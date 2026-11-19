# **🚨 سناریو ۰۸: نادیده گرفته شدن رکوردهای محلی در تفکیک نام دامنه (NSSwitch Hosts Resolution)**

* **دسته‌بندی (Module):** `02-networking-and-web`  
* **سطح حساسیت (Incident Severity):** `P1 (Critical)`  
* **هدف آزمایشگاه:** تفکیک مسیرهای Name Resolution در کتابخانه استاندارد لینوکس (`glibc` / `nsswitch.conf`) و عیب‌یابی نادیده گرفته شدن فایل `/etc/hosts`.

# **۱. صورت مسئله و تیکت حادثه پروداکشن (Production Incident Ticket)**

* **سطح حساسیت:** P1 (Critical)  
* **گزارش تیم مانیتورینگ / شبکه:** سرویس‌های داخلی سرور پس از اعمال یک تغییر امنیتی روی تنظیمات شبکه، دیگر قادر به برقراری ارتباط با سرویس احراز هویت داخلی با نام میزبان `auth.internal.corp` نیستند. نکته عجیب اینجاست که دسترسی به دامنه‌های اینترنتی (مانند `google.com`) بدون هیچ مشکلی برقرار است و رکورد مربوط به `auth.internal.corp` نیز دقیقاً در فایل میزبان‌های محلی سرور (`/etc/hosts`) ثبت شده است.

# **۲. اسکریپت شبیه‌سازی خرابی (Failure Injection — `break_system.sh`)**

این اسکریپت را با دسترسی `sudo` اجرا کنید و پس از پایان، بدون خواندن کدها شروع به عیب‌یابی کنید:

```
#!/bin/bash
if [ "$EUID" -ne 0 ]; then
  echo "Please run as root"
  exit 1
fi

echo "Applying network configuration..."
sleep 4

cp -p /etc/nsswitch.conf /var/cache/.sys_state_bak

grep -q "auth.internal.corp" /etc/hosts || echo "127.0.0.1 auth.internal.corp" >> /etc/hosts
|---|dns/' /etc/nsswitch.conf

echo "Done. Incident triggered!"
```

# **۳. علائم بالینی و لاگ‌ها (Symptoms)**

* توقف برنامه‌ها و ابزارهای خط فرمان هنگام فراخوانی `auth.internal.corp` با خطای `Could not resolve host: auth.internal.corp` یا `Name or service not known`.  
* موفقیت کامل در تفکیک نام و پینگ دامنه‌های خارجی مانند `google.com`.  
* وجود رکورد صحیح `127.0.0.1 auth.internal.corp` در فایل `/etc/hosts` بدون هیچ‌گونه خطای تایپی.

# **۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)**

* **علت ریشه‌ای:** حذف منبع `files` از بخش `hosts:` در فایل پیکربندی Name Service Switch (`/etc/nsswitch.conf`).  
* **مکانیزم فنی:** در سیستم‌عامل لینوکس، کتابخانه استاندارد `glibc` هنگام فراخوانی توابع تفکیک نام (مانند `getaddrinfo`) مستقیماً به سراغ فایل `/etc/hosts` یا سرورهای DNS نمی‌رود؛ بلکه ابتدا فایل `/etc/nsswitch.conf` را می‌خواند تا ترتیب و منابع مجاز برای جستجو را تعیین کند. در حالت استاندارد (`hosts: files dns`)، سیستم ابتدا فایل محلی `/etc/hosts` (`files`) را بررسی کرده و در صورت نیافتن نام، پرس‌وجو را به سرورهای DNS موجود در `/etc/resolv.conf` (`dns`) ارسال می‌کند. با حذف کلیدواژه `files`، کتابخانه `glibc` فایل `/etc/hosts` را کاملاً نادیده می‌گیرد و تمام دامنه‌ها (حتی دامنه‌های داخلی تعریف‌شده در سیستم) را مستقیماً از DNS استعلام می‌گیرد که منجر به دریافت پاسخ `NXDOMAIN` می‌شود.

# **۵. راهنمای عیب‌یابی و مراحل اصلاح (Troubleshooting & Remediation)**

1. **بازتولید خطا و بررسی تفاوت تفکیک نام خارجی و داخلی:**

```
ping -c 2 google.com
ping -c 2 auth.internal.corp
```

*(پینگ `google.com` موفق است، اما `auth.internal.corp` خطای `Name or service not known` می‌دهد.)*

2. **بررسی فایل میزبان‌های محلی برای اطمینان از وجود رکورد و نبود خطای تایپی یا کاراکتر مخفی:**

```
cat -A /etc/hosts
```

*(مشاهده وجود خط `127.0.0.1 auth.internal.corp$`.)*

3. **استعلام از کتابخانه استاندارد تفکیک نام سیستم‌عامل (NSS) برای دیدن اینکه سیستم‌عامل واقعاً چه چیزی می‌بیند:**

```
getent hosts auth.internal.corp
```

*(این دستور هیچ خروجی برنمی‌گرداند و ثابت می‌کند فایل `/etc/hosts` خوانده نمی‌شود.)*

4. **بررسی ترتیب و منابع جستجوی سرویس‌های نام در فایل پیکربندی Name Service Switch:**

```
grep "^hosts:" /etc/nsswitch.conf
```

5. **شناسایی نقص پیکربندی در خروجی دستور بالا:**  
   مشاهده اینکه منبع `files` از خط `hosts:` حذف شده و فقط `dns` باقی مانده است (`hosts:          dns`).  
6. **بازگرداندن منبع `files` به ابتدای خط `hosts:` در فایل `/etc/nsswitch.conf`:**

```
sudo sed -i 's/^hosts:.*/hosts:          files dns/' /etc/nsswitch.conf
```

*(تایید رفع مشکل با اجرای `getent hosts auth.internal.corp` و مشاهده خروجی `127.0.0.1       auth.internal.corp`.)*

# **۶. درس‌آموخته مهندسی و اسکریپت پاک‌سازی (Lesson Learned & Revert)**

## **درس‌آموخته مهندسی (Lesson Learned)**

* ابزارهای سطح DNS مانند `dig` یا `nslookup` مستقیماً از سرورهای DNS استعلام می‌گیرند و فایل `/etc/hosts` یا `nsswitch.conf` را بررسی نمی‌کنند؛ بنابراین برای عیب‌یابی دقیق Name Resolution در لینوکس همیشه از دستور `getent hosts <hostname>` استفاده کنید تا رفتار واقعی کتابخانه `glibc` را ببینید.  
* پیش از ویرایش خودکار فایل‌های سیستمی با `sed` در اسکریپت‌های سخت‌سازی، حتماً از فایل اصلی نسخه پشتیبان تهیه کرده و پس از اعمال تغییر، یک تست دود (Smoke Test) روی دامنه‌های داخلی و خارجی اجرا کنید.

## **اسکریپت پاک‌سازی اضطراری (`safe_fix.sh`)**

در صورتی که نتوانستید مشکل را پیدا کنید، این اسکریپت تنظیمات سیستم را به حالت اولیه برمی‌گرداند:

```
#!/bin/bash
if [ "$EUID" -ne 0 ]; then
  echo "Please run as root"
  exit 1
fi

if [ -f /var/cache/.sys_state_bak ]; then
  mv /var/cache/.sys_state_bak /etc/nsswitch.conf
else
  sed -i 's/^hosts:.*/hosts:          files dns/' /etc/nsswitch.conf
fi

sed -i '/auth.internal.corp/d' /etc/hosts
echo "System recovered safely."
```

