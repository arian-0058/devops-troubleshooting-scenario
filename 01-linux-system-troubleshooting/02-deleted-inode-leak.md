# ۲: باقی ماندن فضای دیسک پس از حذف فایل به‌دلیل هندل باز پروسه (Deleted Inode / File Descriptor Leak)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس خام (Native Linux)
- **شرح خرابی / تیکت پروداکشن:**
> **تیکت:** #INC-44102 | **اولویت:** High
> گزارش مانیتورینگ: آلارم دیسک سرور روی پارتیشن روت فعال شده و `df -h` مصرف را بالای ۹۰٪ نشان می‌دهد. تیم توسعه اعلام کرده فایل لاگ حجیم را با `rm` حذف کرده‌اند، اما فضای دیسک همچنان آزاد نشده است.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
mkdir -p /tmp/app_leak
dd if=/dev/zero of=/tmp/app_leak/active_stream.log bs=1M count=600 status=none
tail -f /tmp/app_leak/active_stream.log > /dev/null &
SLEEP_PID=$!
echo $SLEEP_PID > /tmp/app_leak/proc.pid
rm -f /tmp/app_leak/active_stream.log
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** پر شدن دیسک روت و عدم آزادسازی فضا پس از حذف فایل لاگ با دستور `rm`.
* **علائم بالینی (Symptoms):** دستور `df -h` مصرف دیسک را بالای ۹۰٪ نشان می‌دهد، اما `du -sh` دایرکتوری هیچ فضای اشغال‌شده‌ای پیدا نمی‌کند.
* **علت ریشه‌ای (Root Cause):** پردازه‌ای در پس‌زمینه (`tail -f`) همچنان File Descriptor فایل حذف‌شده را باز نگه داشته است؛ لینوکس فضا را تا زمان بسته شدن کامل هندل پردازه آزاد نمی‌کند (Unlinked Open File).

### 🛠️ مراحل رفع ایراد (Fix)

1. شناسایی پردازه و فایل قفل‌شده با دستور `sudo lsof +L1` یا `sudo lsof | grep '(deleted)'`.
2. متوقف کردن پروسه مسدودکننده با `killall tail` یا بستن امن PID مربوطه (`kill -9 <PID>`).

### 💡 درس‌آموخته مهندسی (Lesson Learned)

دستور `rm` فقط اشاره‌گر دایرکتوری (dentry link) را حذف می‌کند نه لزوماً اینود و بلاک‌های داده را؛ برای خالی کردن سریع لاگ‌های پروسه‌های فعال در پروداکشن، به‌جای `rm` باید فایل را با دستور `cat /dev/null > /path/to/file.log` خالی (Truncate) کرد.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
if [ -f /tmp/app_leak/proc.pid ]; then
    kill -9 $(cat /tmp/app_leak/proc.pid) 2>/dev/null || true
fi
pkill -f "tail -f /tmp/app_leak/active_stream.log" 2>/dev/null || true
rm -rf /tmp/app_leak
```
