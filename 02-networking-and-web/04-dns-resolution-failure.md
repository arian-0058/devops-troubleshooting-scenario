# ۴: اختلال در تفکیک نام دامنه (DNS Resolution Failure)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس (مستقیم / بدون داکر)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** سرور قادر به اتصال به اینترنت و دانلود پکیج‌ها/آپدیت‌ها نیست
> **گزارش:** ارتباط شبکه با IPهای مستقیم برقرار است اما دستوراتی مثل `apt update` یا `curl https://google.com` با خطای `Could not resolve host` مواجه می‌شوند.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
sudo cp /etc/resolv.conf /etc/resolv.conf.bak
echo "nameserver 192.0.2.1" | sudo tee /etc/resolv.conf
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** عدم امکان اتصال به اینترنت از طریق نام دامنه و شکست در اجرای دستوراتی مانند `curl` یا `apt update`.
* **علائم بالینی (Symptoms):** تایم‌اوت یا خطای `Could not resolve host` در اجرای درخواست‌های مبتنی بر نام دامنه‌های خارجی.
* **علت ریشه‌ای (Root Cause):** تنظیم یک آدرس نیم‌سرور نامعتبر/غیرقابل‌دسترس (`192.0.2.1`) در فایل پیکربندی `/etc/resolv.conf`.

### 🛠️ مراحل رفع ایراد (Fix)

1. تست ارتباط لایه شبکه با IP مستقیم: `ping 8.8.8.8`.
2. تست لایه DNS با `dig` یا `nslookup` یا `host google.com`.
3. بازگرداندن تنظیمات صحیح به `/etc/resolv.conf` (یا استفاده از DNS معتبر مانند `8.8.8.8` / `1.1.1.1`) و ری‌استارت سرویس تفکیک نام با `sudo systemctl restart systemd-resolved`.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

در عیب‌یابی مشکلات شبکه، همیشه باید ابتدا لایه IP مستقیم و سپس لایه DNS تست شود تا زمان رفع ایراد بین قطعی فیزیکی شبکه و خطای تفکیک نام هدر نرود.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
if [ -f /etc/resolv.conf.bak ]; then
  sudo cp /etc/resolv.conf.bak /etc/resolv.conf
  sudo rm /etc/resolv.conf.bak
fi
```
