# ۳: اتمام ظرفیت Inode علی‌رغم وجود فضای خالی (Inode Exhaustion)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس خام (Native Linux)
- **شرح خرابی / تیکت پروداکشن:**
> **تیکت:** #INC-44115 | **اولویت:** Critical
> گزارش: هنگام ایجاد فایل جدید با خطای `No space left on device` مواجه می‌شویم، در حالی که `df -h` نشان می‌دهد دیسک چندین گیگابایت فضای خالی دارد.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
TARGET_DIR="/tmp/session_cache_leak"
mkdir -p "$TARGET_DIR"
python3 -c "
import os
for i in range(120000):
    try:
        with open(f'$TARGET_DIR/sess_{i}.tmp', 'w') as f:
            f.write('')
    except OSError:
        break
"
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** خطای `No space left on device` هنگام ساخت فایل، علی‌رغم وجود چند گیگابایت فضای خالی بر حسب بایت.
* **علائم بالینی (Symptoms):** خروجی `df -h` ظرفیت خالی قابل‌توجهی نشان می‌دهد، اما ابزارهای ساخت فایل با شکست مواجه می‌شوند.
* **علت ریشه‌ای (Root Cause):** تولید بیش از حد فایل‌های با حجم صفر (Session Cache Leaks) که کل ساختار جدول Inode پارتیشن را اشغال کرده‌اند.

### 🛠️ مراحل رفع ایراد (Fix)

1. پایش درصد مصرف اینود در پارتیشن‌ها با `df -i`.
2. یافتن دایرکتوری دارای بیشترین تعداد فایل با `sudo find / -xdev -printf '%h\n' | sort | uniq -c | sort -k 1 -nr | head -20`.
3. پاک‌سازی کارآمد فایل‌ها بدون سرریز بافر آرگومان با `find /tmp/session_cache_leak -type f -delete`.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

مانیتورینگ سیستم باید همیشه پارامتر `df -i` (تعداد اینودها) را هم‌تراز با `df -h` (فضای ذخیره‌سازی) پایش کند؛ اجرای `rm *` روی دایرکتوری با ده‌ها هزار فایل خطای `Argument list too long` تولید می‌کند و باید از `find ... -delete` استفاده شود.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
find /tmp/session_cache_leak -type f -delete 2>/dev/null || true
rm -rf /tmp/session_cache_leak 2>/dev/null || true
```
