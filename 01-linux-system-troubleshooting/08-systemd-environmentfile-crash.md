# سناریوی 08: کرش مکرر سرویس Systemd ناشی از خطای منابع و دسترسی در EnvironmentFile

## ۱. صورت مسئله و تیکت پروداکشن (Incident Ticket)

* **سرویس متاثر:** `app-auth.service` (سرویس احراز هویت داخلی)  
* **سطح رخداد:** P1 \- Critical (قطع دسترسی و DownTime سرویس احراز هویت)  
* **شرح تیکت:** «سرویس احراز هویت پس از اعمال تنظیمات متغیرهای محیطی بالا نمی‌آید. دستور استارت با خطای سیستمی متوقف شده و پردازه وارد چرخه بازیابی مکرر (Restart Loop) شده است.»

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

برای بازتولید این رخداد در محیط آزمایشگاهی، اسکریپت زیر را با دسترسی روت اجرا کنید:

```sh
#!/usr/bin/env bash
set -e

# ۱. متوقف کردن نسخه فعال سرویس
systemctl stop app-auth.service 2>/dev/null || true

# ۲. ایجاد ساختار متناقض با EnvironmentFile (دایرکتوری به‌جای فایل متنی)
rm -rf /etc/app-auth.env
mkdir -p /etc/app-auth.env

# ۳. تنظیم واحد سرویس در Systemd با مکانیزم بازیابی خودکار
cat << 'EOF' > /etc/systemd/system/app-auth.service
[Unit]
Description=Demo Authentication Service
After=network.target

[Service]
Type=simple
EnvironmentFile=/etc/app-auth.env
ExecStart=/bin/sh -c 'echo "App running on port $APP_PORT"; while true; do sleep 10; done'
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF

# ۴. ریلود دیمن و اجرای سرویس
systemctl daemon-reload
echo "[*] Starting app-auth.service..."
systemctl start app-auth.service || true
echo "[!] Failure injected: Service should be failing in restart loop."
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms)

هنگام تلاش برای استارت یا بررسی وضعیت سرویس، نشانه‌های زیر ثبت می‌شوند:

**۱. خروجی دستور `systemctl status app-auth.service`:**

```
● app-auth.service - Demo Authentication Service
     Loaded: loaded (/etc/systemd/system/app-auth.service; disabled; preset: enabled)
     Active: activating (auto-restart) (Result: resources)
```

**۲. خروجی لاگ‌های ژورنال (`journalctl -u app-auth.service -e --no-pager`):**

```
systemd[1]: app-auth.service: Failed to load environment files: Is a directory
systemd[1]: app-auth.service: Failed to spawn 'start' task: Is a directory
systemd[1]: app-auth.service: Failed with result 'resources'.
systemd[1]: Failed to start app-auth.service - Demo Authentication Service.
systemd[1]: app-auth.service: Scheduled restart job, restart counter is at 10.
```

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

1. **رفتار پارسر فایل محیطی Systemd:** دیرکتیو `EnvironmentFile=` در سیستم‌دی انتظار یک فایل متنی با فرمت کلید/مقدار (`KEY=VALUE`) را دارد. زمانی که مسیر داده‌شده از نوع دایرکتوری، سوکت یا فایل غیرقابل‌دسترسی باشد، سیستم‌عامل خطای سیستمی `EISDIR` بازمی‌گرداند.  
2. **بروز وضعیت `Result: resources`:** سیستم‌دی خطای عدم توانایی در خواندن ساختار فایل محیطی را در دسته کمبود/نامعتبر بودن منابع دسترسی سیستم قرار داده و پیش از اجرای دستور اصلی `ExecStart` تسک را متوقف می‌کند.  
3. **چرخه ری‌استارت بی‌پایان:** به دلیل تنظیم `Restart=on-failure` و `RestartSec=2`، سرویس مکرراً تلاش می‌کند مقدار محیطی را بارگذاری کند و شمارنده ری‌استارت پیوسته افزایش می‌یابد.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

**گام ۱: متوقف کردن چرخه ری‌استارت سرویس**

```sh
sudo systemctl stop app-auth.service
```

**گام ۲: پاکسازی ساختار نامعتبر و ایجاد فایل پیکربندی صحیح**

```sh
sudo rm -rf /etc/app-auth.env

# نوشتن مقادیر معتبر با دسترسی روت
cat << 'EOF' | sudo tee /etc/app-auth.env
APP_ENV=production
APP_PORT=8080
DB_SECRET="P@ssw0rd_Secure_2026"
DATABASE_URL="postgres://user:password@localhost/authdb"
EOF
```

**گام ۳: ریلود دیمن و راه‌اندازی مجدد سرویس**

```sh
sudo systemctl daemon-reload
sudo systemctl restart app-auth.service
```

**گام ۴: تایید سلامت اجرای سرویس**

```sh
sudo systemctl status app-auth.service
```

*خروجی موفقیت‌آمیز:*

```
● app-auth.service - Demo Authentication Service
     Loaded: loaded (/etc/systemd/system/app-auth.service; disabled; preset: enabled)
     Active: active (running)
   Main PID: 5112 (sh)
```

---

## ۶. درس‌آموخته‌های مهندسی و اسکریپت بازگشت (Lesson Learned & Revert)

* **پرهیز از خطاهای ریدایرکت با Sudo:** دستوراتی مانند `sudo echo "..." >> /path` عمل ریدایرکت را با شل کاربر عادی انجام می‌دهند و به خطای `Permission denied` منجر می‌شوند. همیشه از الگوی `echo "..." | sudo tee /path` استفاده کنید.  
* **نحوه تفسیر کاراکتر `#` در Systemd:** در فایل‌های `EnvironmentFile`، علامت `#` تنها در ابتدای سطر به عنوان کامنت شناخته می‌شود؛ در میانه سطر جزئی از مقدار متغیر در نظر گرفته می‌شود. مقادیر دارای کاراکترهای خاص باید داخل کوتیشن جفت‌شده قرار گیرند.  
* **پرمیشن‌های فایل محیطی:** فایل‌های حاوی مقادیر محرمانه باید دارای پرمیشن `600` باشند (`chmod 600 /etc/app-auth.env`).

### اسکریپت پاکسازی محیط آزمایشی (Revert Script)

برای برگرداندن سرور آزمایشگاهی به حالت خام و تمیز:

```sh
#!/usr/bin/env bash
sudo systemctl stop app-auth.service 2>/dev/null || true
sudo rm -f /etc/systemd/system/app-auth.service
sudo rm -rf /etc/app-auth.env
sudo systemctl daemon-reload
echo "[+] Cleanup complete. System reverted."
```

