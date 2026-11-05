# سناریوی 08: بن‌بست ارتباطی و خطای Connection Reset ناشی از بایند شدن اپلیکیشن به لوپ‌بک کانتینر

- **محیط هدف:** سیستم‌عامل لینوکس / ویندوز (WSL2) همراه با داکر انجین  
- **سطح اولویت و رخداد:** Incident Severity: P2 (Service Outage / Localhost Port Misconfiguration)

---

## ۱. صورت مسئله و تیکت پروداکشن (Incident Report)

> **متن تیکت:**  
> «کانتینر پایتون مربوط به سرویس وضعیت سیستم با موفقیت ساخته شده و در وضعیت `Up` با پورت فورواردینگ `5000:5000` در حال اجراست. لاگ‌های کانتینر نشان می‌دهند که وب‌سرور آماده دریافت درخواست است، اما اجرای دستور `curl localhost:5000` از داخل هاست (یا WSL) یا باز کردن آن در مرورگر ویندوز، خطای زیر را برمی‌گرداند:  
> `curl: (56) Recv failure: Connection reset by peer`»

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

برای بازتولید دقیق سناریو در محیط آزمایشگاهی، دستورات زیر را اجرا کنید:

```sh
#!/usr/bin/env bash
set -e

WORKDIR="$HOME/docker-loopback-incident"
mkdir -p "$WORKDIR"
cd "$WORKDIR"

# ۱. ساخت وب‌اپلیکیشن تستی پایتون که به اشتباه روی 127.0.0.1 بایند شده است
cat << 'EOF' > server.py
import http.server
import socketserver

PORT = 5000
# اشکال ریشه‌ای: بایند شدن به لوپ‌بک داخلی کانتینر
HOST = "127.0.0.1"

Handler = http.server.SimpleHTTPRequestHandler
with socketserver.TCPServer((HOST, PORT), Handler) as httpd:
    print(f"Server serving on {HOST}:{PORT}")
    httpd.serve_forever()
EOF

# ۲. ایجاد Dockerfile سبک
cat << 'EOF' > Dockerfile
FROM python:3.11-alpine
WORKDIR /app
COPY server.py .
CMD ["python", "server.py"]
EOF

# ۳. بیلد و اجرا
docker build -t loopback-app .
docker run -d --name loopback-container -p 5000:5000 loopback-app

echo "[!] Failure injected. Try querying service: curl -v http://localhost:5000"
```

---

## ۳. علائم بالینی، شواهد و لاگ‌ها (Symptoms & Triage)

### الف) وضعیت پردازه کانتینر

کانتینر به ظاهر کاملاً سالم و فعال است:

```sh
docker ps
# CONTAINER ID   IMAGE          COMMAND              STATUS         PORTS
# 43cf17b05a55   loopback-app   "python server.py"   Up 2 minutes   0.0.0.0:5000->5000/tcp, [::]:5000->5000/tcp   loopback-container
```

### ب) خروجی تست اتصال از سمت کلاینت/هاست

```sh
curl -v http://localhost:5000
```

**شواهد دریافتی:**

* هندشیک سه‌مرحله‌ای TCP با موفقیت شروع شده (`Connected to localhost (127.0.0.1) port 5000`).  
* به محض ارسال درخواست HTTP GET، اتصال از سمت مقصد قطع می‌شود:

```
* Recv failure: Connection reset by peer
* Closing connection 0
curl: (56) Recv failure: Connection reset by peer
```

### ج) پایش سوکت‌های داخل کانتینر

با بررسی سوکت‌های شبکه از درون کانتینر، وضعیت بایندینگ مشخص می‌شود:

```sh
docker exec loopback-container netstat -tlpn
```

**خروجی تشخیصی:**

```
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:5000          0.0.0.0:*               LISTEN      1/python
```

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

1. **ایزولاسیون فضای نام شبکه (Network Namespace):** هر کانتینر لینوکسی دارای Network Namespace مستقل، جدول مسیریابی (Routing Table) و اینترفیس‌های لوپ‌بک (`lo`) و مجازی خود (`eth0`) است.  
2. **تفاوت اینترفیس `127.0.0.1` با `0.0.0.0`:**  
   * زمانی که وب‌سرور داخل کانتینر روی آدرس `127.0.0.1` گوش می‌دهد، پردازه **صرفاً** پکت‌هایی را می‌پذیرد که از اینترفیس لوپ‌بک محلی همان کانتینر منشأ گرفته باشند.  
   * مکانیزم Port Forwarding داکر (`-p 5000:5000` یا Docker Proxy) بسته‌های ارسالی از هاست را از طریق Bridge به اینترفیس خارجی کانتینر (`eth0` کانتینر با IP مثلاً `172.17.0.2`) فوروارد می‌کند.  
3. **مکانیزم وقوع خطای TCP RST:**  
   * پکت ارسالی هاست به پورت ۵۰۰۰ روی اینترفیس `eth0` کانتینر تحویل داده می‌شود.  
   * هسته لینوکس داخل کانتینر بررسی می‌کند که هیچ پردازه‌ای روی `eth0:5000` یا اینترفیس عمومی (`INADDR_ANY` یا `0.0.0.0`) در وضعیت `LISTEN` قرار ندارد.  
   * بنابراین، استک TCP هسته فوراً یک بسته `TCP RST` (Reset) صادر کرده و اتصال را قطع می‌کند که در لایه کلاینت با خطای `Connection reset by peer (Code 56)` ظاهر می‌شود.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

### گام اول: اصلاح فایل سرور (`server.py`)

آدرس بایندینگ را از `127.0.0.1` به `0.0.0.0` تغییر دهید تا وب‌سرور روی تمامی کارت‌های شبکه کانتینر فعال شود:

```py
import http.server
import socketserver

PORT = 5000
# اصلاح: اتصال به تمامی اینترفیس‌های شبکه کانتینر
HOST = "0.0.0.0"

Handler = http.server.SimpleHTTPRequestHandler
with socketserver.TCPServer((HOST, PORT), Handler) as httpd:
    print(f"Server serving on {HOST}:{PORT}")
    httpd.serve_forever()
```

### گام دوم: متوقف‌سازی و پاکسازی کانتینر قدیمی

```sh
docker rm -f loopback-container
```

### گام سوم: بیلد مجدد ایمیج و راه‌اندازی کانتینر جدید

```sh
cd "$HOME/docker-loopback-incident"
docker build -t loopback-app .
docker run -d --name loopback-container -p 5000:5000 loopback-app
```

### گام چهارم: اعتبارسنجی (Validation)

اجرای مجدد درخواست HTTP از هاست:

```sh
curl -v http://localhost:5000
```

**نتیجه موفقیت‌آمیز:**

```
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/3.11.18
...
<!DOCTYPE HTML>
<html>
...
Directory listing for /
...
```

---

## ۶. درس‌آموخته مهندسی و الگوهای پیشگیری (Lessons Learned)

* **قانون عمومی بایندینگ کانتینرها:** تمامی دیمن‌ها و وب‌سرورهایی که درون کانتینر اجرا می‌شوند (مانند Flask, Django, Node.js, FastAPI, Nginx, Uvicorn) باید صراحتاً روی آدرس `0.0.0.0` پیکربندی شوند؛ مگر اینکه سرویس صرفاً برای ارتباط محلی پروسه‌های درون همان کانتینر طراحی شده باشد.  
* **شناسایی تله‌های فریم‌ورک‌ها:** اکثر فریم‌ورک‌های توسعه وب (مانند `flask run` یا `uvicorn`) به صورت پیش‌فرض امنیتی روی `127.0.0.1` اجرا می‌شوند. در داکرفایل یا Entrypoint باید همواره آرگومان `--host 0.0.0.0` صراحتاً پاس داده شود.  
* **تفاوت Connection Refused با Connection Reset:**  
  * خطای `Connection Refused` معمولاً نشان‌دهنده بسته بودن پورت روی هاست یا خطا در مپینگ داکر است.  
  * خطای `Connection Reset by Peer` در معماری داکر نشان‌دهنده آن است که پورت روی هاست باز است و به کانتینر می‌رسد، اما داخل کانتینر ترافیک توسط هسته ریست می‌شود.

---

## اسکریپت بازگشت و پاکسازی آزمایشگاه (Cleanup Script)

```sh
#!/usr/bin/env bash
set -e

docker rm -f loopback-container 2>/dev/null || true
docker rmi -f loopback-app 2>/dev/null || true
rm -rf "$HOME/docker-loopback-incident"
echo "[+] Cleanup complete: Container and lab directory removed."
```

