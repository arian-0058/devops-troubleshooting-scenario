# ۱: عدم دسترسی به وب‌سایت به دلیل خطای مجوز فایل‌ها (403 Forbidden)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس (مستقیم / بدون داکر)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** خطای 403 Forbidden روی صفحه اصلی وب‌سایت
> **گزارش:** کاربران هنگام باز کردن وب‌سایت با خطای `403 Forbidden` مواجه می‌شوند. سرویس Nginx در حال اجراست ولی محتوا نمایش داده نمی‌شود.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
sudo mkdir -p /var/www/html
echo "<h1>Production Web App</h1>" | sudo tee /var/www/html/index.html
sudo chmod 600 /var/www/html/index.html
sudo chmod 700 /var/www/html
sudo systemctl restart nginx
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** کاربران هنگام درخواست صفحه اصلی وب‌سایت با خطای HTTP `403 Forbidden` مواجه می‌شدند.
* **علائم بالینی (Symptoms):** سرویس Nginx فعال (`active/running`) بود و لاگ وب‌سرور خطای `open() "/var/www/html/index.html" failed (13: Permission denied)` را ثبت کرده بود.
* **علت ریشه‌ای (Root Cause):** اعمال دسترسی نامناسب (`chmod 600` برای فایل و `700` برای دایرکتوری)، مانع از خوانده شدن فایل توسط کاربر اجراکننده Worker Processهای وب‌سرور (`www-data`) شده بود.

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی لاگ‌های خطا با `sudo tail -n 20 /var/log/nginx/error.log`.
2. بررسی کاربر اجراکننده Nginx (معمولاً `www-data`).
3. اصلاح دسترسی به حالت استاندارد وب: `sudo chmod 755 /var/www/html` برای دایرکتوری و `sudo chmod 644 /var/www/html/index.html` برای فایل.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

در ساختار وب‌سرورهای چندکاربره لینوکس، وب‌سرور با سطح دسترسی کاربر غیرریشه اجرا می‌شود؛ برای دسترسی به فایل‌ها، علاوه بر پرمیشن خواندن فایل (`r`)، تمامی دایرکتوری‌های والد نیز باید دارای دسترسی اجرایی (`x`) برای Others باشند.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
sudo chmod 755 /var/www/html
sudo chmod 644 /var/www/html/index.html
```
