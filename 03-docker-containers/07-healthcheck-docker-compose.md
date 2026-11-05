# سناریوی 07: بن‌بست استارت سرویس‌ها در Docker Compose بر اثر Healthcheck ناموفق در ایمیج Minimal

- **حوزه تخصصی:** Containerization & Service Orchestration
- **محیط آزمایشگاهی:** Linux / WSL2 / Docker Compose v2
- **سطح اولویت رخداد (Incident Severity):** High (P2 - Service Deployment Deadlock)

---

### ۱. صورت مسئله و تیکت پروداکشن (Incident Report)
در معماری سرویس جدید، کانتینر `frontend-proxy` برای استارت به سلامت کانتینر کش دیتابیس (`cache-service`) وابسته است (`condition: service_healthy`). با اجرای دستور `docker compose up -d` سرویس پروکسی هیچ‌گاه بالا نمی‌آید و روی وضعیت زیر متوقف شده و تایم‌اوت می‌خورد:

```text
dependency failed to start: container cache-service is unhealthy
```

---

### ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)
اسکریپت شل زیر محیط ایزوله را در دایرکتوری آزمایشگاهی ساخته و نقص هلث‌چک را فعال می‌کند:

```bash
#!/usr/bin/env bash
set -e

WORKDIR="$HOME/compose-deadlock-lab"
mkdir -p "$WORKDIR"
cd "$WORKDIR"

cat << 'EOF' > docker-compose.yml
services:
  cache-service:
    image: redis:alpine
    container_name: cache-service
    healthcheck:
      # خطای ریشه‌ای: دستور curl در ایمیج سبک redis:alpine وجود ندارد
      test: ["CMD-SHELL", "curl -f http://localhost:6379 || exit 1"]
      interval: 3s
      timeout: 2s
      retries: 2
      start_period: 2s

  frontend-proxy:
    image: nginx:alpine
    container_name: frontend-proxy
    depends_on:
      cache-service:
        condition: service_healthy
    ports:
      - "8089:80"
EOF

docker compose up -d || true
echo "[!] Failure injected. Inspect with: docker compose ps"
```

---

### ۳. علائم بالینی و لاگ‌های تشخیصی (Symptoms & Triage)
بررسی وضعیت استک با دستور `docker ps -a` یا `docker compose ps` نشان می‌دهد که کانتینر وابسته اصلاً اجرا نشده است:

```text
CONTAINER ID   IMAGE          STATUS                          NAMES
24006007390b   nginx:alpine   Created                         frontend-proxy
bb420c0eada7   redis:alpine   Up About a minute (unhealthy)   cache-service
```

بررسی ریشه‌ای وضعیت سلامت کانتینر با دستور بازرسی عمیق داکر:
```bash
docker inspect cache-service
```

خروجی ثبت‌شده در بلاک `State.Health.Log`:
```json
"Health": {
  "Status": "unhealthy",
  "FailingStreak": 76,
  "Log": [
    {
      "ExitCode": 1,
      "Output": "/bin/sh: curl: not found\n"
    }
  ]
}
```

---

### ۴. تحلیل علت ریشه‌ای (Root Cause Analysis - RCA)
1. ایمیج‌های مینیمال مبتنی بر آلپاین (`alpine`) فاقد ابزارهای متداول شبکه مانند `curl` یا `wget` هستند.
2. دستور هلث‌چک برای ارزیابی کانتینر فراخوانی شده، اما با خطای عدم وجود باینری (`/bin/sh: curl: not found`) و کد خطای ۱۲۷/۱ مواجه شده است.
3. در داکر کامپوز، شرط `service_healthy` به این معناست که پروسه استارت کانتینر بعدی تا زمان سبز شدن وضعیت کانتینر پیش‌نیاز متوقف می‌ماند؛ در نتیجه سیستم وارد بن‌بست (Deadlock) شده است.

---

### ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)
فایل `docker-compose.yml` ویرایش شده و پروب هلث‌چک با دستور استاندارد و بومی ردیس (`redis-cli ping`) جایگزین گردید:

```yaml
services:
  cache-service:
    image: redis:alpine
    container_name: cache-service
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 2s

  frontend-proxy:
    image: nginx:alpine
    container_name: frontend-proxy
    depends_on:
      cache-service:
        condition: service_healthy
    ports:
      - "8089:80"
```

اعمال مجدد و استارت سرویس‌ها:
```bash
docker compose up -d
```

**تایید سلامت:**
خروجی تست سلامت با `docker inspect cache-service` مقدار خروج صفر و پاسخ معتبر ردیس را برمی‌گرداند:
```json
"ExitCode": 0,
"Output": "PONG\n"
```
سرویس `frontend-proxy` نیز با موفقیت روی پورت `8089` آنلاین شد.

---

### ۶. درس‌آموخته مهندسی و پاکسازی محیط (Lesson Learned & Revert)
* **قاعده بومی‌سازی هلث‌چک:** در ایمیج‌های کانتینری حداقلی (Minimal/Distroless)، هرگز نباید فرض را بر وجود باینری‌های پرکاربرد سیستمی گذاشت؛ پروب‌های سلامت باید مستقیماً با کلاینت داخلی سرویس (مانند `redis-cli` یا اسکریپت‌های پایتون داخلی) نوشته شوند.
* **اسکریپت پاکسازی محیط (revert_scenario_4.sh):**

```bash
#!/usr/bin/env bash
set -e

cd "$HOME/compose-deadlock-lab" 2>/dev/null || true
docker compose down -v 2>/dev/null || true
rm -rf "$HOME/compose-deadlock-lab"
echo "[+] Cleanup complete: Docker compose stack stopped and cleaned."
```