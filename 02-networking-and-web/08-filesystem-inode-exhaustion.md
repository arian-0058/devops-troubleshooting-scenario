# **🚨 سناریو ۰۸: ناتوانی سرویس در نوشتن فایل و خطای عجیب دیسک (Inode Exhaustion)**

* **دسته‌بندی (Module):** `01-linux-system-troubleshooting`  
* **سطح حساسیت (Incident Severity):** `P2 (High)`  
* **هدف آزمایشگاه:** تشخیص پر شدن ظرفیت جدول Inode فایل‌سیستم در شرایطی که فضای ذخیره‌سازی (Data Blocks) همچنان خالی است.

# **۱. صورت مسئله و تیکت حادثه پروداکشن (Production Incident Ticket)**

* **سطح حساسیت:** P2 (High)  
* **گزارش تیم مانیتورینگ / عملیات:** سیستم آپلود فایل و ثبت لاگ‌های تراکنش در اپلیکیشن از کار افتاده است. تیم عملیات فضای دیسک را با دستور `df -h` چک کرده و گزارش داده که بیش از ۵۰ درصد فضای پارتیشن روت (`/`) خالی است، اما سیستم اجازه ساخت هیچ فایل جدیدی را نمی‌دهد.

# **۲. اسکریپت شبیه‌سازی خرابی (Failure Injection — `break_system.sh`)**

این اسکریپت یک فضای ایزوله کوچک در `/mnt/prod_storage` می‌سازد تا به پارتیشن اصلی سیستم شما آسیبی نزند، و سپس مشکل را در آن ایجاد می‌کند:

```
#!/bin/bash
if [ "$EUID" -ne 0 ]; then
  echo "Please run as root"
  exit 1
fi

echo "Preparing storage environment..."
sleep 3

mkdir -p /mnt/prod_storage
dd if=/dev/zero of=/tmp/prod_disk.img bs=1M count=20 status=none
mkfs.ext4 -N 100 -q /tmp/prod_disk.img
mount -o loop /tmp/prod_disk.img /mnt/prod_storage

for i in $(seq 1 95); do
  touch /mnt/prod_storage/session_$i.tmp
done

echo "Incident active on /mnt/prod_storage!"
```

# **۳. علائم بالینی و لاگ‌ها (Symptoms)**

* توقف سرویس‌ها هنگام تلاش برای ساخت فایل سشن (`session`)، کش (`cache`) یا لاگ جدید با خطای سیستمی `No space left on device` (کد خطای کرنل `ENOSPC`).  
* گزارش وجود فضای خالی فراوان (بیش از ۵۰٪ ظرفیت آزاد) در خروجی دستور استاندارد `df -h`.

# **۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)**

* **علت ریشه‌ای:** اتمام ظرفیت جدول Inode در فایل‌سیستم (`Inode Exhaustion`) در عین وجود فضای فیزیکی خالی (`Data Blocks`).  
* **مکانیزم فنی:** در فایل‌سیستم‌های لینوکسی مانند `ext4`، هر فایل یا دایرکتوری علاوه بر بلوک‌های داده (که محتوای فایل را نگه می‌دارند)، به یک ساختار داده به نام `Inode` نیاز دارد که متادیتا (مالکیت، مجوزها، زمان تغییر و اشاره‌گر به بلوک‌های داده) را ذخیره می‌کند. تعداد Inodeها هنگام فرمت شدن پارتیشن (`mkfs`) به صورت ثابت تعیین می‌شود. اگر برنامه‌ای تعداد بسیار زیادی فایل صفر بایتی یا بسیار کوچک (مانند فایل‌های Session، Cache یا Queue) تولید کند، تمام Inodeهای موجود مصرف می‌شوند. در این حالت، کرنل لینوکس برای ساخت فایل جدید ساختار Inode آزادی پیدا نمی‌کند و دقیقاً همان خطای آشنای `No space left on device` را برمی‌گرداند.

# **۵. راهنمای عیب‌یابی و مراحل اصلاح (Troubleshooting & Remediation)**

1. **بررسی فضای خالی دیسک بر حسب بایت (مشاهده اینکه فضای خالی وجود دارد):**

```
df -h /mnt/prod_storage
```

2. **تست نوشتن یک فایل کوچک در مسیر مورد نظر برای بازتولید خطا:**

```
touch /mnt/prod_storage/test.txt
```

*(مشاهده خطای `cannot touch '/mnt/prod_storage/test.txt': No space left on device`.)*

3. **بررسی وضعیت جدول Inode در فایل‌سیستم‌های مانت‌شده:**

```
df -i /mnt/prod_storage
```

4. **شناسایی گلوگاه در خروجی دستور بالا:**  
   مشاهده مقدار `IUse% = 100%` و `IFree = 0` در خروجی `df -i`.  
5. **بررسی فایل‌های ریز و انباشته‌شده در دایرکتوری برای پیدا کردن منشأ پر شدن جدول:**

```
ls -la /mnt/prod_storage | head -n 20
ls /mnt/prod_storage | wc -l
```

6. **پاک‌سازی فایل‌های موقت اضافی برای آزادسازی Inodeها:**

```
rm -f /mnt/prod_storage/session_*.tmp
```

*(تایید رفع مشکل با اجرای مجدد `df -i /mnt/prod_storage` و ساخت فایل تست `touch /mnt/prod_storage/test.txt`.)*

# **۶. درس‌آموخته مهندسی و اسکریپت پاک‌سازی (Lesson Learned & Revert)**

## **درس‌آموخته مهندسی (Lesson Learned)**

* در سیستم‌های مانیتورینگ (مانند Prometheus Node Exporter)، علاوه بر تعریف Alert روی مصرف فضای دیسک (`node_filesystem_avail_bytes`)، باید حتماً هشدار جداگانه برای مصرف Inodeها (`node_filesystem_files_free`) تعریف شود.  
* برای فایل‌های موقت پرتعداد (Sessionها یا فایل‌های صف)، استفاده از مکانیزم‌های پاک‌سازی دوره‌ای (`systemd-tmpfiles` یا Cron Job) یا انتقال ذخیره‌سازی به ابزارهای In-Memory مانند Redis ضروری است.

## **اسکریپت پاک‌سازی اضطراری (`safe_fix.sh`)**

با اجرای این اسکریپت، محیط آزمایشی ساخته‌شده به طور کامل حذف و پاک‌سازی می‌شود:

```
#!/bin/bash
if [ "$EUID" -ne 0 ]; then
  echo "Please run as root"
  exit 1
fi

umount /mnt/prod_storage 2>/dev/null
rm -rf /mnt/prod_storage /tmp/prod_disk.img
echo "Storage environment cleaned up."
```

