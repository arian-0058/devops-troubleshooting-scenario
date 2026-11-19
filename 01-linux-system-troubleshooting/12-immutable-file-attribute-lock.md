# **🚨 سناریو ۰۹: قفل شدن تغییرات پیکربندی حتی برای کاربر Root (Immutable File Attribute)**

* **دسته‌بندی (Module):** `01-linux-system-troubleshooting`  
* **سطح حساسیت (Incident Severity):** `P2 (High)`  
* **هدف آزمایشگاه:** شناسایی و مدیریت ویژگی‌های توسعه‌یافته فایل‌سیستم (`Extended Attributes` / `chattr`) در شرایطی که مجوزهای استاندارد POSIX عادی به نظر می‌رسند.

---

# **۱. صورت مسئله و تیکت حادثه پروداکشن (Production Incident Ticket)**

* **سطح حساسیت:** P2 (High)  
* **گزارش تیم استقرار (Deployment):** تیم استقرار قصد دارد پارامتر `API_TIMEOUT` را در فایل تنظیمات سرویس پرداخت در مسیر `/opt/payment-gateway/settings.conf` افزایش دهد تا خطاهای تایم‌اوت در ساعات اوج ترافیک برطرف شود؛ اما اسکریپت استقرار و حتی ادمین سیستم با دسترسی `root` هنگام ذخیره تغییرات یا جابه‌جایی فایل خطا دریافت می‌کنند.

---

# **۲. اسکریپت شبیه‌سازی خرابی (Failure Injection — `break_system.sh`)**

این اسکریپت را با دسترسی `sudo` اجرا کنید و بدون بررسی محتوای آن، عیب‌یابی را شروع کنید:

```
#!/bin/bash
if [ "$EUID" -ne 0 ]; then
  echo "Please run as root"
  exit 1
fi

echo "Initializing production service environment..."
sleep 3

mkdir -p /opt/payment-gateway
cat << 'EOF' > /opt/payment-gateway/settings.conf
APP_ENV=production
DB_POOL_SIZE=50
API_TIMEOUT=30
EOF

chattr +i /opt/payment-gateway/settings.conf

echo "Environment ready. Incident triggered!"
```

---

# **۳. علائم بالینی و لاگ‌ها (Symptoms)**

* دریافت خطای `Operation not permitted` (کد خطای `EPERM`) یا پیام `Read-only` در ویرایشگرهایی مانند `vim` و `nano` هنگام تلاش برای ویرایش، تغییر نام یا حذف فایل `/opt/payment-gateway/settings.conf` توسط کاربر `root`.  
* عادی بودن مجوزهای استاندارد فایل (`-rw-r--r--`) و مالکیت آن (`root:root`) در خروجی `ls -la`.  
* وضعیت `rw` (قابل نوشتن) بودن پارتیشن در جدول مانت سیستم‌عامل.

---

# **۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)**

* **علت ریشه‌ای:** فعال بودن صفت تغییرناپذیری یا Immutable Attribute (`+i`) در سطح متادیتای فایل‌سیستم (`ext4` / `xfs`).  
* **مکانیزم فنی:** مجوزهای سنتی POSIX (`rwx`) تنها سطح دسترسی کاربران عادی را کنترل می‌کنند و کاربر `root` (دارای قابلیت `CAP_DAC_OVERRIDE`) به طور پیش‌فرض می‌تواند از آن‌ها عبور کند. با این حال، فایل‌سیستم‌های مدرن لینوکس از ویژگی‌های توسعه‌یافته (`Extended Attributes`) پشتیبانی می‌کنند که توسط ابزارهای `lsattr` و `chattr` مدیریت می‌شوند. وقتی بیت `i` روی یک Inode تنظیم شود، هسته لینوکس (`VFS Layer`) هرگونه تغییر در محتوا، تغییر نام، ایجاد لینک سخت یا حذف فایل را مسدود کرده و کد خطای `EPERM` (`Operation not permitted`) را حتی برای کاربر `root` برمی‌گرداند. ادمین‌ها معمولاً از این ویژگی برای محافظت از فایل‌های حیاتی در برابر تغییرات ناخواسته یا بدافزارها استفاده می‌کنند.

---

# **۵. راهنمای عیب‌یابی و مراحل اصلاح (Troubleshooting & Remediation)**

1. **بررسی مجوزهای استاندارد، مالکیت و نوع فایل مورد نظر:**

```
ls -la /opt/payment-gateway/settings.conf
```

2. **اطمینان از هویت کاربر فعلی و اینکه فایل‌سیستم به صورت Read-Only مانت نشده است:**

```
id
findmnt -T /opt/payment-gateway/settings.conf
```

*(مشاهده گزینه `rw` در وضعیت مانت پارتیشن.)*

3. **تست نوشتن یک فایل جدید در همان دایرکتوری برای اثبات اینکه مشکل از کل دایرکتوری یا پارتیشن نیست:**

```
touch /opt/payment-gateway/test_write
```

*(این دستور بدون خطا انجام می‌شود و نشان می‌دهد مشکل روی خود فایل `settings.conf` متمرکز است؛ سپس فایل تستی را با `rm -f /opt/payment-gateway/test_write` پاک کنید.)*

4. **بررسی ویژگی‌های توسعه‌یافته فایل‌سیستم (Extended Attributes) روی فایل:**

```
lsattr /opt/payment-gateway/settings.conf
```

5. **شناسایی پرچم قفل‌کننده در خروجی دستور بالا:**  
   مشاهده پرچم `i` (Immutable) در خروجی به صورت `----i---------e-------`.  
6. **حذف ویژگی تغییرناپذیری برای بازگرداندن دسترسی نوشتن:**

```
sudo chattr -i /opt/payment-gateway/settings.conf
```

*(تایید رفع مشکل با بررسی `lsattr /opt/payment-gateway/settings.conf` و امکان ویرایش فایل.)*

---

# **۶. درس‌آموخته مهندسی و اسکریپت پاک‌سازی (Lesson Learned & Revert)**

## **درس‌آموخته مهندسی (Lesson Learned)**

* هر زمان کاربر `root` روی یک فایل‌سیستم `rw` با خطای `Operation not permitted` مواجه شد، بلافاصله علاوه بر `ls -la`، ابزارهای `lsattr` (Extended Attributes)، `getfacl` (ACLها) و وضعیت ماژول‌های امنیتی کرنل (`SELinux` / `AppArmor`) را بررسی کنید.  
* در صورت استفاده از `chattr +i` برای سخت‌سازی فایل‌های حساس، حتماً این موضوع را در مستندات Runbook سرویس و پایپ‌لاین‌های استقرار (CI/CD / Ansible) ثبت کنید تا در زمان به‌روزرسانی‌های برنامه‌ریزی‌شده، پرچم به صورت کنترل‌شده برداشته و مجدداً اعمال شود.

## **اسکریپت پاک‌سازی اضطراری (`safe_fix.sh`)**

در صورت نیاز به بازگردانی سریع وضعیت سیستم و حذف محیط آزمایشی، این اسکریپت را اجرا کنید:

```
#!/bin/bash
if [ "$EUID" -ne 0 ]; then
  echo "Please run as root"
  exit 1
fi

chattr -i /opt/payment-gateway/settings.conf 2>/dev/null
rm -rf /opt/payment-gateway
echo "System recovered safely."
```

