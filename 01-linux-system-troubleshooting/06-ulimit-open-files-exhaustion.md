# ۶: محدودیت تعداد فایل‌های باز کاربر و کرش تحت بار (ulimit / Too Many Open Files)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس خام (Native Linux)
- **شرح خرابی / تیکت پروداکشن:**
> **تیکت:** #INC-44155 | **اولویت:** High
> گزارش: اپلیکیشن پردازش داده هنگامی که اتصالات همزمان افزایش می‌یابد با ارور `Too many open files` کرش می‌کند، در حالی که منابع سخت‌افزاری سرور کاملاً آزاد هستند.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
cat << 'APP' > /tmp/leak_fds.py
import resource, os, sys
resource.setrlimit(resource.RLIMIT_NOFILE, (30, 1024))
opened_files = []
try:
    for i in range(100):
        opened_files.append(open(f"/tmp/fd_test_{i}.tmp", "w"))
except OSError as e:
    print(f"[CRASH ERROR]: System failed to allocate file descriptor -> {e}", file=sys.stderr)
    sys.exit(1)
APP
python3 /tmp/leak_fds.py || true
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** کرش ناگهانی پردازه‌های پربار هنگام پذیرش اتصالات همزمان در سطح سیستم‌عامل.
* **علائم بالینی (Symptoms):** خطای صریح سیستم‌عامل `[Errno 24] Too many open files` در لاگ‌های خطا، در حالی که CPU و RAM آزاد هستند.
* **علت ریشه‌ای (Root Cause):** اتمام سقف مجاز تخصیص File Descriptor که به‌صورت پیش‌فرض روی مقادیر پایین (معمولاً ۱۰۲۴ یا کمتر) محدود شده است.

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی محدودیت‌های عمومی هسته با `cat /proc/sys/fs/file-max` و `ulimit -n`.
2. بررسی محدودیت‌های سطح کاربر در `/etc/security/limits.conf` و افزایش سقف با:
   ```
   *    soft    nofile    65535
   *    hard    nofile    65535
   ```
3. در سرویس‌های Systemd، افزودن `LimitNOFILE=65535` در بخش `[Service]` فایل یونیت.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

پروسه‌هایی که سریعاً کرش می‌کنند در خروجی `ps aux` باقی نمی‌مانند. حل مشکل ظرفیت اتصالات باید از طریق تنظیمات کرنل و کنترل دسترسی‌های سیستم‌عامل انجام شود، نه با دور زدن تست و هاردکد در سورس برنامه.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
rm -f /tmp/leak_fds.py /tmp/fd_test_*.tmp
```
