# ۷: خرابی چرخه روت لاگ‌ها به‌دلیل خطای پرمیشن دایرکتوری (Logrotate Insecure Permissions)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس خام (Native Linux)
- **شرح خرابی / تیکت پروداکشن:**
> **تیکت:** #INC-44204 | **اولویت:** Medium
> گزارش: کران‌جاب شبانه چرخش لاگ‌ها شکست خورده و فایل‌های لاگ در `/var/log/custom_app/` بدون فشرده‌سازی و آرشیو شدن، حجیم شده‌اند.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
sudo mkdir -p /var/log/custom_app
echo "Dummy operational log stream" | sudo tee /var/log/custom_app/app.log > /dev/null

cat << 'CONF' | sudo tee /etc/logrotate.d/custom_app > /dev/null
/var/log/custom_app/*.log {
    daily
    rotate 3
    compress
    missingok
    notifempty
    create 0640 root root
}
CONF

sudo chmod 700 /var/log/custom_app
sudo chown -R 1005:1005 /var/log/custom_app
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** شکست تسک دوره‌ای لاگ‌روتیت در فشرده‌سازی و آرشیو لاگ‌های حجیم.
* **علائم بالینی (Symptoms):** فایل‌های لاگ بدون روتیت شدن متورم می‌شوند و اجرای `logrotate -d` خطای `parent directory has insecure permissions` صادر می‌کند.
* **علت ریشه‌ای (Root Cause):** دایرکتوری لاگ متعلق به کاربری غیر از `root` بود و به دلیل نداشتن مجوز اجرایی (`x`) برای سایرین، پردازه logrotate (که با `root` اجرا می‌شود) از ورود به دایرکتوری خودداری می‌کرد.

### 🛠️ مراحل رفع ایراد (Fix)

1. تست دستی در حالت دیباگ: `sudo logrotate -d /etc/logrotate.d/custom_app`.
2. بازگرداندن مالکیت و دسترسی استاندارد:
   ```bash
   sudo chown -R root:root /var/log/custom_app
   sudo chmod 755 /var/log/custom_app
   ```
   یا در صورتی که دایرکتوری باید در تملک یوزر سرویس بماند، افزودن دایرکتیو `su 1005 1005` به کانفیگ `/etc/logrotate.d/custom_app`.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

قبل از تغییر کورکورانه پرمیشن‌ها (مانند `chmod 760` که حق `execute` را از بقیه می‌گیرد)، باید رفتار ابزارها را با فلگ دیباگ (`logrotate -d`) شبیه‌سازی کرد تا خطای دقیق تحلیل شود.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
sudo rm -f /etc/logrotate.d/custom_app
sudo rm -rf /var/log/custom_app
```
