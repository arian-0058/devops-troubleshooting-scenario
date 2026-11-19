# سناریو: اختلال دسترسی به سرویس Core پس از دیپلوی شبانه (Nginx Upstream DNS Caching)

- **دسته‌بندی (Module):** `03-docker-containers` / `02-networking-and-web`
- **نام پیشنهادی فایل در مخزن:** `03-docker-containers/06-nginx-docker-dns-cache-502.md`
- **شناسه تیکت:** `INC-2026-0841`
- **سطح حساسیت:** `P1 - Critical`
- **تکنولوژی‌های درگیر:** `Docker Networking (IPAM & Embedded DNS)`, `Nginx Reverse Proxy`, `Linux`

---

## ۱. صورت مسئله و تیکت پروداکشن (Production Incident Ticket)

* **شناسه تیکت:** `INC-2026-0841`
* **سطح حساسیت:** `P1 - Critical`
* **گزارش کاربر و مانیتورینگ:** پس از ری‌استارت و به‌روزرسانی سرویس‌های داخلی در ساعت ۰۲:۰۰ بامداد، درگاه اصلی سرویس (`http://localhost:8081`) برای تمام کاربران با خطای **HTTP 502 Bad Gateway** مواجه شده است. بررسی‌های اولیه تیم شیفت نشان می‌دهد که تمام کانتینرهای مربوط به این استک در وضعیت `Up` قرار دارند و هیچ کرش یا ری‌استارت مداومی در لیست کانتینرها دیده نمی‌شود.
* **خطای ظاهری:** دریافت خطای `502 Bad Gateway` هنگام ارسال درخواست به `http://localhost:8081`.

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

برای بازتولید دقیق این حادثه در محیط آزمایشگاهی ایزوله، اسکریپت زیر را اجرا کنید:

```bash
#!/usr/bin/env bash
# File: break_system.sh
set -e

WORKDIR="/tmp/incident_lab_01"
rm -rf "$WORKDIR"
mkdir -p "$WORKDIR"

docker rm -f proxy-edge backend-core worker-sync >/dev/null 2>&1 || true
docker network rm lab01_net >/dev/null 2>&1 || true

docker network create --subnet=172.29.10.0/24 lab01_net >/dev/null

cat << 'EOF' > "$WORKDIR/nginx.conf"
events { worker_connections 1024; }
http {
    server {
        listen 80;
        location / {
            proxy_pass http://backend-core:8000;
            proxy_connect_timeout 3s;
            proxy_read_timeout 5s;
        }
    }
}
EOF

docker run -d --name backend-core --network lab01_net python:3.11-alpine \
  python3 -m http.server 8000 >/dev/null

docker run -d --name worker-sync --network lab01_net python:3.11-alpine \
  sleep infinity >/dev/null

docker run -d --name proxy-edge --network lab01_net -p 8081:80 \
  -v "$WORKDIR/nginx.conf:/etc/nginx/nginx.conf:ro" nginx:alpine >/dev/null

sleep 5

# شبیه‌سازی دیپلوی مجدد سرویس‌ها با ترتیب استارت متفاوت
docker rm -f backend-core worker-sync >/dev/null
docker run -d --name worker-sync --network lab01_net python:3.11-alpine \
  sleep infinity >/dev/null
docker run -d --name backend-core --network lab01_net python:3.11-alpine \
  python3 -m http.server 8000 >/dev/null

echo "Environment is ready at http://localhost:8081"
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms & Diagnostics)

در گام نخست عیب‌یابی غیرتهاجمی (Non-Destructive Troubleshooting)، وضعیت درگاه، کانتینرها، لاگ‌های Nginx و نگاشت شبکه بررسی می‌شود:

```bash
# ۱. بررسی وضعیت پاسخ درگاه
curl -i http://localhost:8081

# ۲. بررسی وضعیت کانتینرها (همه در وضعیت Up هستند)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# ۳. بررسی لاگ خطاهای Nginx (مشاهده Connection refused به IP قدیمی)
docker logs --tail 20 proxy-edge

# ۴. بررسی IP فعلی کانتینرها در شبکه داخلی داکر
docker network inspect lab01_net

# ۵. بررسی پاسخ DNS داخلی داکر از درون کانتینر پراکسی
docker exec proxy-edge getent hosts backend-core

# ۶. بررسی کانفیگ فعال Nginx
docker exec proxy-edge cat /etc/nginx/nginx.conf
```

* **خروجی کلیدی لاگ‌ها:** در لاگ `proxy-edge` خطای `connect() failed (111: Connection refused) while connecting to upstream` ثبت شده که نشان می‌دهد Nginx همچنان در حال ارسال درخواست به آدرس `172.29.10.2:8000` است، در حالی که `getent hosts backend-core` آدرس `172.29.10.3` را برمی‌گرداند.

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis - RCA)

* **علت ریشه‌ای:** کش شدن دائمی آدرس IP سرویس Upstream در حافظه Nginx همراه با جابه‌جایی پویای آدرس IP کانتینرها توسط سیستم مدیریت IP داکر (IPAM) پس از بازسازی کانتینرها.
* **مکانیزم فنی در سطح شبکه و سرویس:**
  1. هنگام استارت اولیه کانتینر `proxy-edge`، وب‌سرور Nginx نام میزبان `backend-core` را از طریق DNS داخلی داکر (`127.0.0.11`) ترجمه کرده و آدرس اولیه (`172.29.10.2`) را در حافظه خود کش می‌کند. در نسخه متن‌باز Nginx، هنگامی که آدرس مقصد به صورت ایستا در دستور `proxy_pass` نوشته شود، تنها یک‌بار در زمان لود شدن کانفیگ Resolve می‌شود و TTL رکوردهای DNS نادیده گرفته می‌شود.
  2. در جریان دیپلوی شبانه و بازسازی کانتینرها با ترتیب جدید، سیستم IPAM داکر اولین آدرس IP آزاد ساب‌نت (`172.29.10.2`) را به `worker-sync` اختصاص داد و کانتینر `backend-core` آدرس `172.29.10.3` را دریافت کرد.
  3. با وجود به‌روز شدن رکورد در DNS داخلی داکر (`127.0.0.11`)، Nginx بدون انجام Reload همچنان ترافیک را به `.2:8000` (کانتینر `worker-sync` که هیچ پورتی باز نکرده است) هدایت می‌کرد که منجر به دریافت `Connection refused` از کرنل و بازگشت خطای `502 Bad Gateway` به کلاینت شد.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

### الف) رفع فوری حادثه (Immediate Mitigation)
بارگذاری مجدد پیکربندی Nginx جهت تخلیه کش DNS و وادار کردن آن به استعلام مجدد آدرس جدید `backend-core`:

```bash
docker exec proxy-edge nginx -s reload
curl -i http://localhost:8081
```

### ب) راهکار پایدار در محیط پروداکشن (Permanent Production Fix)
تنظیم صریح `resolver` روی DNS داخلی داکر (`127.0.0.11`) با مقدار `valid` کوتاه و تعریف آدرس Upstream در قالب متغیر تا Nginx در هر درخواست بر اساس TTL اقدام به ترجمه مجدد نام دامنه کند:

```nginx
http {
    resolver 127.0.0.11 valid=10s ipv6=off;

    server {
        listen 80;
        location / {
            set $upstream_core "http://backend-core:8000";
            proxy_pass $upstream_core;
            proxy_connect_timeout 3s;
            proxy_read_timeout 5s;
        }
    }
}
```

---

## ۶. درس‌آموخته مهندسی و اسکریپت بازگشت (Lesson Learned & Revert)

### 💡 درس‌آموخته‌های مهندسی
* **پرهیز از Resolve ایستا در محیط‌های پویا:** در معماری‌های مبتنی بر کانتینر (Docker / Kubernetes)، آدرس IP کانتینرها ذاتاً موقتی (Ephemeral) است. تعریف ایستا نام سرویس در `proxy_pass` بدون `resolver` و متغیر، یک نقطه شکست پنهان (Silent Failure) در زمان Rolling Update ایجاد می‌کند.
* **بازبینی چرخه دیپلوی:** هر پایپ‌لاین CD که سرویس‌های Backend را بازسازی می‌کند، باید یا از Dynamic DNS Resolution در لایه پراکسی بهره ببرد یا پس از تغییر کانتینرهای Upstream، سلامت مسیریابی لایه Edge را با Smoke Test بررسی کند.

### 🧹 اسکریپت پاک‌سازی و بازگشت محیط (`safe_fix.sh`)

```bash
#!/usr/bin/env bash
# File: safe_fix.sh
docker rm -f proxy-edge backend-core worker-sync >/dev/null 2>&1 || true
docker network rm lab01_net >/dev/null 2>&1 || true
rm -rf /tmp/incident_lab_01
echo "Lab 01 cleaned up successfully."
```
