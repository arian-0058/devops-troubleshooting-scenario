# سناریو: خطاهای مقطعی ۵۰۲ در درگاه API هنگام بار ترافیکی (Ephemeral Port Exhaustion & TIME_WAIT)

- **دسته‌بندی (Module):** `03-docker-containers` / `01-linux-system-troubleshooting`
- **نام پیشنهادی فایل در مخزن:** `03-docker-containers/07-ephemeral-port-exhaustion-502.md`
- **شناسه تیکت:** `INC-2026-0849`
- **سطح حساسیت:** `P2 - High`
- **تکنولوژی‌های درگیر:** `Linux Kernel TCP/IP Stack`, `sysctl`, `Nginx Reverse Proxy`, `Docker Network Namespace`

---

## ۱. صورت مسئله و تیکت پروداکشن (Production Incident Ticket)

* **شناسه تیکت:** `INC-2026-0849`
* **سطح حساسیت:** `P2 - High`
* **گزارش کاربر و مانیتورینگ:** تیم فرانت‌اند گزارش داده است که درگاه سرویس (`http://localhost:8082`) رفتار بسیار ناپایداری دارد. وقتی یک درخواست تکی با `curl` ارسال می‌شود، سرویس معمولاً پاسخ `200 OK` برمی‌گرداند؛ اما به محض اینکه صفحه در مرورگر باز می‌شود یا چند درخواست پشت‌سرهم ارسال می‌گردد، بخش زیادی از درخواست‌ها با خطای **502 Bad Gateway** شکست می‌خورند. مصرف CPU، رم، فضای دیسک و تعداد File Descriptorها کاملاً عادی گزارش شده است.
* **خطای ظاهری:** دریافت متناوب `502 Bad Gateway` تحت درخواست‌های متوالی به `http://localhost:8082`.

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

برای بازتولید دقیق این حادثه در محیط آزمایشگاهی ایزوله، اسکریپت زیر را اجرا کنید:

```bash
#!/usr/bin/env bash
# File: break_system.sh
set -e

WORKDIR="/tmp/incident_lab_02"
rm -rf "$WORKDIR"
mkdir -p "$WORKDIR"

docker rm -f api-gateway app-engine >/dev/null 2>&1 || true
docker network rm lab02_net >/dev/null 2>&1 || true

docker network create lab02_net >/dev/null

cat << 'EOF' > "$WORKDIR/nginx.conf"
events { worker_connections 1024; }
http {
    server {
        listen 80;
        location / {
            proxy_pass http://app-engine:8000;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
EOF

docker run -d --name app-engine --network lab02_net python:3.11-alpine \
  python3 -m http.server 8000 >/dev/null

docker run -d --name api-gateway --network lab02_net --privileged -p 8082:80 \
  -v "$WORKDIR/nginx.conf:/etc/nginx/nginx.conf:ro" nginx:alpine >/dev/null

sleep 5

# محدود کردن شدید بازه پورت‌های موقت (Ephemeral Ports) در فضای نام شبکه کانتینر
docker exec api-gateway sysctl -w net.ipv4.ip_local_port_range="32768 32771" >/dev/null

echo "Environment is ready at http://localhost:8082"
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms & Diagnostics)

برای بررسی رفتار متناوب سرویس تحت درخواست‌های متوالی و وضعیت سوکت‌های شبکه در فضای نام (Network Namespace) کانتینر پراکسی، دستورات زیر اجرا می‌شود:

```bash
# ۱. شبیه‌سازی ارسال ۸ درخواست متوالی (۴ درخواست اول موفق و مابقی 502 می‌شوند)
for i in {1..8}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8082; done

# ۲. بررسی لاگ خطاهای Nginx (مشاهده خطای Cannot assign requested address)
docker logs --tail 20 api-gateway

# ۳. بررسی سقف File Descriptor (اطمینان از عدم پر شدن ulimit)
docker exec api-gateway sh -c "ulimit -n"

# ۴. مشاهده وضعیت سوکت‌های TCP و انباشت اتصالات در وضعیت TIME_WAIT
docker exec api-gateway ss -tan

# ۵. بررسی پارامتر بازه پورت‌های موقت کرنل در کانتینر
docker exec api-gateway sysctl net.ipv4.ip_local_port_range
```

* **خروجی کلیدی لاگ‌ها:** در لاگ `api-gateway` خطای سیستمی `connect() to 172.x.x.x:8000 failed (99: Cannot assign requested address) while connecting to upstream` مشاهده می‌شود و خروجی `ss -tan` نشان می‌دهد تمام پورت‌های آزاد در وضعیت `TIME-WAIT` قفل شده‌اند.

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis - RCA)

* **علت ریشه‌ای:** اتمام بازه پورت‌های موقت کرنل (Ephemeral Port Exhaustion) در فضای نام شبکه (Network Namespace) کانتینر پراکسی به دلیل محدود شدن پارامتر `net.ipv4.ip_local_port_range` و انباشت سوکت‌ها در وضعیت `TIME_WAIT`.
* **مکانیزم فنی در سطح سیستم‌عامل و شبکه:**
  1. هر بار که Nginx به عنوان Reverse Proxy یک اتصال TCP جدید به سمت سرویس بک‌اند (`app-engine:8000`) برقرار می‌کند، کرنل لینوکس یک پورت مبدا آزاد از بازه `net.ipv4.ip_local_port_range` به آن سوکت اختصاص می‌دهد.
  2. از آنجا که سرور مقصد (`python3 -m http.server`) از HTTP/1.0 استفاده می‌کند و اتصال را بلافاصله می‌بندد (و همچنین در کانفیگ Nginx بلوک `upstream` با قابلیت `keepalive` تعریف نشده است)، پس از اتمام هر درخواست HTTP، اتصال TCP بسته شده و سوکت در سمت کلاینت (`api-gateway`) برای جلوگیری از تداخل بسته‌های سرگردان (Delayed Packets) به مدت ۶۰ ثانیه وارد وضعیت `TIME_WAIT` می‌شود.
  3. محدود شدن بازه پورت‌های محلی به `32768 32771` باعث می‌شود کل ظرفیت پورت‌های خروجی این کانتینر به **۴ پورت همزمان** کاهش یابد. چهار درخواست اول با موفقیت انجام می‌شوند اما تمام پورت‌ها در وضعیت `TIME_WAIT` قفل می‌مانند؛ در نتیجه از درخواست پنجم به بعد، فراخوانی سیستمی `connect()` با خطای کرنل `EADDRNOTAVAIL` (`99: Cannot assign requested address`) شکست خورده و Nginx خطای `502 Bad Gateway` برمی‌گرداند. پس از گذشت ۶۰ ثانیه و آزاد شدن سوکت‌های `TIME_WAIT`، مجدداً چند درخواست اول موفق می‌شوند که همین موضوع خطا را به صورت مقطعی (Intermittent) جلوه می‌دهد.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

### الف) رفع فوری حادثه (Immediate Mitigation)
گسترش بازه پورت‌های موقت کرنل به محدوده استاندارد عملیاتی و تایید پایداری سرویس تحت درخواست‌های متوالی:

```bash
docker exec api-gateway sysctl -w net.ipv4.ip_local_port_range="10240 65535"
for i in {1..8}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8082; done
```

### ب) راهکار پایدار در محیط پروداکشن (Permanent Production Fix)
1. **فعال‌سازی Connection Pooling (Upstream Keepalive) در Nginx:** استفاده از بلاک `upstream` به همراه `keepalive` برای بازاستفاده از اتصالات TCP باز و جلوگیری از ایجاد و بستن مداوم سوکت‌ها در هر درخواست:

```nginx
http {
    upstream app_backend {
        server app-engine:8000;
        keepalive 64;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://app_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
```

2. **بهینه‌سازی پارامترهای شبکه کرنل (`sysctl`):** تنظیم بازه پورت‌های محلی (`net.ipv4.ip_local_port_range = 10240 65535`) و فعال‌سازی بازاستفاده امن از سوکت‌های `TIME_WAIT` برای اتصالات خروجی (`net.ipv4.tcp_tw_reuse = 1`) در سطح هاست یا مانیفست کانتینر.

---

## ۶. درس‌آموخته مهندسی و اسکریپت بازگشت (Lesson Learned & Revert)

### 💡 درس‌آموخته‌های مهندسی
* **تفاوت منابع فایل‌سیستم با منابع شبکه:** خطاهای `502 Bad Gateway` در ترافیک بالا همیشه ناشی از CPU، رم یا کمبود File Descriptor (`ulimit -n`) نیستند؛ اتمام سوکت‌های TCP و پورت‌های مبدا (Socket Tuple Exhaustion) یکی از رایج‌ترین گلوگاه‌های پنهان در لایه Reverse Proxy است.
* **اهمیت Keepalive بین Proxy و Upstream:** بدون تنظیم `upstream keepalive` در Nginx، هر درخواست HTTP کلاینت به یک Handshake کامل TCP و ورود یک پورت به چرخه ۶۰ ثانیه‌ای `TIME_WAIT` تبدیل می‌شود که ظرفیت توان عملیاتی (Throughput) پراکسی را به شدت محدود می‌کند.

### 🧹 اسکریپت پاک‌سازی و بازگشت محیط (`safe_fix.sh`)

```bash
#!/usr/bin/env bash
# File: safe_fix.sh
docker rm -f api-gateway app-engine >/dev/null 2>&1 || true
docker network rm lab02_net >/dev/null 2>&1 || true
rm -rf /tmp/incident_lab_02
echo "Lab 02 cleaned up successfully."
```
