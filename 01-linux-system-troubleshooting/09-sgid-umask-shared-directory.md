# سناریوی 09: اختلال در دسترسی و نوشتن در دایرکتوری اشتراکی تیمی (SGID و umask)

## ۱. صورت مسئله و تیکت پروداکشن (Incident Ticket)

* **محیط هدف:** سرور لینوکس نیتیو (مدیریت دسترسی‌ها، فایل‌سیستم و مجوزهای پیشرفته)  
* **سطح رخداد:** P2 \- Major (اختلال در فرآیند اشتراک فایل و توسعه تیمی)  
* **شرح تیکت:** «دو مهندس به نام‌های dev-alice و dev-bob هر دو عضو گروه لینوکسی devops-team هستند و روی پروژه مشترکی در مسیر `/srv/shared-project` کار می‌کنند. هر فایلی که dev-alice ایجاد می‌کند، توسط dev-bob قابل ویرایش یا بازنویسی نیست و خطای `Permission denied` برمی‌گرداند.»

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

برای بازتولید این رخداد در محیط آزمایشگاهی، اسکریپت زیر را با دسترسی روت اجرا کنید:

```sh
#!/usr/bin/env bash
set -e

# ۱. ساخت گروه و کاربران تستی
groupadd -f devops-team
id -u dev-alice &>/dev/null || useradd -m -s /bin/bash -g devops-team dev-alice
id -u dev-bob &>/dev/null || useradd -m -s /bin/bash -g devops-team dev-bob

# ۲. ایجاد مسیر اشتراکی بدون بیت SGID و بدون Default ACL
mkdir -p /srv/shared-project
chown root:devops-team /srv/shared-project
chmod 775 /srv/shared-project

# ۳. شبیه‌سازی ایجاد فایل توسط کاربر اول با umask محدودکننده
sudo -u dev-alice bash -c "umask 027 && echo 'Initial Config' > /srv/shared-project/deploy.conf"
echo "[!] Failure injected: /srv/shared-project/deploy.conf created without group write permission."
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms)

هنگام بررسی وضعیت مجوزهای فایل یا تلاش کاربر دوم برای ویرایش، نشانه‌های زیر مشاهده می‌شوند:

**۱. خروجی بررسی مشخصات فایل ایجادشده (`ls -l /srv/shared-project/deploy.conf`):**

```
-rw-r----- 1 dev-alice devops-team 15 Nov  2 18:46 /srv/shared-project/deploy.conf
```

*مشاهده می‌شود که مجوز گروه صرفاً فقط‌خواندنی (`r--`) است و اجازه نوشتن (`w`) ندارد.*

**۲. خروجی خطای کاربر دوم (`dev-bob`) در زمان ویرایش یا نوشتن:**

```sh
sudo -u dev-bob bash -c "echo 'New Setting' >> /srv/shared-project/deploy.conf"
```

```
bash: /srv/shared-project/deploy.conf: Permission denied
```

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

1. **تأثیر ماسک پیش‌فرض کاربر (`umask`):** فرآیند ساخت فایل در لینوکس تحت کنترل مقدار `umask` شل هر کاربر قرار دارد. مقدار `umask 027` دسترسی نوشتن برای اعضای گروه را از بین برده و پرمیشن را به `640` (`-rw-r-----`) تقلیل می‌دهد.  
2. **فقدان بیت Set-GID (SGID) روی دایرکتوری والد:** در صورت عدم فعال بودن بیت SGID (`2xxx`) روی پوشه اشتراکی، فایل‌های جدید ممکن است با گروه پیش‌فرض کاربر ساخته شده و ارث‌بری گروهی رعایت نشود.  
3. **عدم تعریف پیش‌فرض‌های لیست کنترل دسترسی (Default ACLs):** تغییر مستقیم مجوز یک فایل موجود (مانند `chmod g+w deploy.conf`) تنها مشکل همان فایل منفرد را حل می‌کند، اما فایل‌های آینده به دلیل نبود `Default ACL` مجدداً دچار کمبود پرمیشن نوشتن برای گروه می‌شوند.

---

## ۵. مراحل اصلاح و رفع ایراد پایدار (Remediation & Fix)

**گام ۱: فعال‌سازی بیت SGID روی دایرکتوری مشترک** فعال‌سازی بیت `g+s` سبب می‌شود تمام فایل‌ها و پوشه‌هایی که در آینده ساخته می‌شوند، به طور خودکار گروه مالکیت پوشه والد (`devops-team`) را به ارث ببرند:

```sh
sudo chmod 2775 /srv/shared-project
```

**گام ۲: تنظیم لیست کنترل دسترسی پیش‌فرض (Default ACL)** برای اینکه بدون وابستگی به مقدار `umask` کاربران، همیشه مجوز خواندن، نوشتن و اجرا به گروه تعلق گیرد:

```sh
# اعمال ACL روی فایل‌ها و پوشه‌های موجود
sudo setfacl -R -m g:devops-team:rwx /srv/shared-project

# اعمال Default ACL برای تمام فایل‌ها و زیرپوشه‌های آینده
sudo setfacl -d -m g:devops-team:rwx /srv/shared-project
```

**گام ۳: تصحیح مجوز فایل‌های موجود از قبل**

```sh
sudo chmod g+rw /srv/shared-project/deploy.conf
```

**گام ۴: تایید نهایی صحت عملکرد سناریو** تست ایجاد فایل توسط `dev-alice` و بازنویسی موفقیت‌آمیز آن توسط `dev-bob`:

```sh
# ایجاد فایل جدید با umask سخت‌گیرانه توسط آلیس
sudo -u dev-alice bash -c "umask 027 && echo 'Config v1' > /srv/shared-project/test-sync.txt"

# اضافه کردن خط جدید توسط باب
sudo -u dev-bob bash -c "echo 'Appended by Bob' >> /srv/shared-project/test-sync.txt"

# مشاهده نتیجه خروجی
cat /srv/shared-project/test-sync.txt
```

*خروجی مورد انتظار:*

```
Config v1
Appended by Bob
```

---

## ۶. درس‌آموخته‌های مهندسی و اسکریپت بازگشت (Lesson Learned & Revert)

* **تمایز حل موردی با اصلاح ساختاری:** اجرای `chmod` روی فایل تکی، مشکل فرآیند (Process Issue) را حل نمی‌کند. برای مسیرهای اشتراکی تیمی باید قوانین سطح فایل‌سیستم (`SGID` و `Default POSIX ACL`) پیاده‌سازی شوند.  
* **کارکرد بیت SGID:** عدد `2` در ابتدای اکتال پرمیشن (`2775`) ارث‌بری خودکار گروه را تضمین می‌کند.  
* **قدرت Default ACL:** پرچم `-d` در دستور `setfacl` یک الگو در اینود دایرکتوری ثبت می‌کند که بر محدودیت‌های ناشی از `umask` شخصی کاربران اولویت دارد.

### اسکریپت بازگشت به حالت اول (Revert Script)

برای پاکسازی کاربران، گروه‌ها و مسیر تستی از سیستم:

```sh
#!/usr/bin/env bash
set -e

rm -rf /srv/shared-project
userdel -r dev-alice 2>/dev/null || true
userdel -r dev-bob 2>/dev/null || true
groupdel devops-team 2>/dev/null || true
echo "[+] Cleanup complete: Users, groups, and directories removed."
```

