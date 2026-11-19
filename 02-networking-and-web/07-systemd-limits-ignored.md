# سناریوی ۰۷: کرش پروسه تحت بار سنگین به دلیل نادیده گرفته شدن تنظیمات Limits توسط Systemd

- **محیط هدف:** سرور لینوکس نیتیو (هسته سیستم‌عامل، پروسه‌ها و مدیریت منابع با Systemd)  
- **سطح اولویت حادثه (Severity):** P1 / Critical (منجر به Service Outage و ناتوانی در ثبت لاگ‌های حساس)  
- **دایرکتوری هدف در مخزن:** `01-linux-system-troubleshooting/`

---

### ۱. صورت مسئله و تیکت پروداکشن (Incident Ticket)

> **تیکت پشتیبانی و گزارش مانیتورینگ:** «سرویس دریافت لاگ سیستم (`mock-logger.service`) تحت ترافیک بالا با خطای `Too many open files` کرش می‌کند. مهندس زیرساخت مقدار محدودیت `nofile` کاربر اجرایی (`logrunner`) را در مسیر `/etc/security/limits.conf` (و دایرکتوری `/etc/security/limits.d/`) روی `65535` تنظیم کرده است، اما پروسه پس از راه‌اندازی همچنان روی سقف پیش‌فرض ۱۰۲۴ فایل قفل مانده و سیستم از اختصاص فایل دیسکریپتور (File Descriptor) جدید به آن خودداری می‌کند.»

---

### ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

برای بازتولید این رخداد در محیط آزمایشگاهی/تست، اسکریپت `inject_scenario_7.sh` را با دسترسی root یا `sudo` اجرا کنید:

```sh
#!/usr/bin/env bash
set -e

# ۱. ساخت کاربر اختصاصی
id -u logrunner &>/dev/null || useradd -m -s /bin/bash logrunner

# ۲. ایجاد کانفیگ PAM limits (که برای سرویس‌های سیستمی بی‌اثر است)
cat << 'EOF' > /etc/security/limits.d/99-logrunner.conf
logrunner soft nofile 65535
logrunner hard nofile 65535
EOF

# ۳. تعریف سرویس Systemd بدون محدودیت صریح منابع
cat << 'EOF' > /etc/systemd/system/mock-logger.service
[Unit]
Description=Mock Logger Engine
After=network.target

[Service]
Type=simple
User=logrunner
# دستور زیر سقف فعلی باز کردن فایل را بررسی و لاگ می‌کند
ExecStart=/bin/bash -c 'echo "Current NOFILE limit: $(ulimit -n)"; while true; do sleep 10; done'
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl restart mock-logger.service
echo "[!] Failure injected. Inspect logs: journalctl -u mock-logger.service -e --no-pager"
```

---

### ۳. علائم بالینی و لاگ‌ها (Symptoms & Triage)

1. **بررسی لاگ سرویس:**

```sh
journalctl -u mock-logger.service -n 10 --no-pager
```

   *خروجی مشاهده‌شده:*

```
Current NOFILE limit: 1024
```

   سرویس به وضوح نشان می‌دهد که سقف فایل دیسکریپتورهای در دسترس آن ۱۰۲۴ است، نه ۶۵۵۳۵.

   

2. **بررسی سقف زنده کرنل برای پروسه جاری:**

```sh
PID=$(pgrep -f "mock-logger")
grep 'Max open files' /proc/${PID}/limits
```

   *خروجی مشاهده‌شده:*

```
Max open files            1024                 524288               files
```

   مقدار Soft Limit کرنل برای این پروسه ۱۰۲۴ باقی مانده است.

---

### ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

* **جداسازی مسیر نشست‌های کاربری (PAM) از سرویس‌های سیستمی (Systemd):** فایل‌های `/etc/security/limits.conf` و `/etc/security/limits.d/*.conf` توسط ماژول `pam_limits.so` در زمان باز شدن نشست‌های تعاملی (Interactive Login Sessions نظیر SSH یا لاگین محلی) بارگذاری و اعمال می‌شوند.  
* **معماری Systemd (PID 1):** سرویس‌هایی که به عنوان Daemons توسط `systemd` اجرا و مدیریت می‌شوند، از استک PAM عبور نمی‌کنند (مگر اینکه ماژول PAM صراحتاً با `PAMName=` فراخوانی شود). به صورت پیش‌فرض، Systemd محدودیت‌های پیش‌فرض خود هسته سیستم‌عامل یا مقادیر کانفیگ‌شده در `/etc/systemd/system.conf` را به Child Processها ارث می‌دهد که برای `LimitNOFILE` اغلب ۱۰۲۴ (Soft Limit) است.  
* **نتیجه:** تغییر در `limits.conf` هیچ تأثیری بر دیمن‌ها و سرویس‌های سیستمیِ تحت کنترل Systemd ندارد.

---

### ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

برای رفع اصولی و اعمال سقف منابع به سرویس:

#### روش اول: ویرایش فایل سرویس یا افزودن Drop-in Override (روش استاندارد)

1. فایل یونیت سرویس `/etc/systemd/system/mock-logger.service` را باز کنید یا با دستور زیر یک override بسازید:

```sh
sudo systemctl edit mock-logger.service
```

2. دایرکتیو `LimitNOFILE=65535` را در بخش `[Service]` اضافه کنید:

```
[Service]
LimitNOFILE=65535
```

3. بازخوانی پیکربندی و راه‌اندازی مجدد سرویس:

```sh
sudo systemctl daemon-reload
sudo systemctl restart mock-logger.service
```

#### اعتبارسنجی (Verification):

بررسی مجدد محدودیت زنده در کرنل:

```sh
grep 'Max open files' /proc/$(pgrep -f "mock-logger")/limits
```

*خروجی مطلوب:*

```
Max open files            65535                65535                files
```

---

### ۶. درس‌آموخته مهندسی و پاکسازی (Lessons Learned & Revert)

* **اصل مهندسی سیستم:** در توزیع‌های مدرن لینوکس مبتنی بر Systemd، مدیریت منابع (CPU, Memory, NOFILE, NPROC) برای سرویس‌های پس‌زمینه همواره باید از طریق Unit Fileها (`LimitNOFILE`, `LimitNPROC`, `MemoryMax`) یا فایل سراسری `/etc/systemd/system.conf` تنظیم شود، نه فایل‌های موروثی PAM (`limits.conf`).  
* **اسکریپت پاکسازی (`revert_scenario_7.sh`):**

```sh
#!/usr/bin/env bash
set -e

systemctl stop mock-logger.service 2>/dev/null || true
rm -f /etc/systemd/system/mock-logger.service
rm -f /etc/security/limits.d/99-logrunner.conf
systemctl daemon-reload
userdel -r logrunner 2>/dev/null || true
echo "[+] Cleanup complete: Service, limits configs, and user purged."
```

