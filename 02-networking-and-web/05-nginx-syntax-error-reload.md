# ۵: خطای سینتکس در کانفیگ Nginx و شکست Reload سرویس

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس (مستقیم / بدون داکر)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** شکست در استقرار کانفیگ جدید وب‌سرور
> **گزارش:** پس از اعمال تغییرات اخیر روی Virtual Host، دستور reload کردن Nginx با خطا خارج شده و تغییرات جدید لود نمی‌شوند.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak 2>/dev/null || true
sudo sed -i 's/http {/http {\n    server_names_hash_bucket_size 64/' /etc/nginx/nginx.conf
sudo systemctl reload nginx || true
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** شکست در بازخوانی تنظیمات وب‌سرور (`systemctl reload nginx`) و اعمال نشدن تغییرات جدید.
* **علائم بالینی (Symptoms):** دریافت پیام خطای `Job for nginx.service failed` و خطای `invalid number of arguments` در گزارش لاگ.
* **علت ریشه‌ای (Root Cause):** عدم قرار دادن سمی‌کالن (`;`) در انتهای دایرکتیو `server_names_hash_bucket_size 64` در فایل `/etc/nginx/nginx.conf`.

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی و اعتبارسنجی سینتکس با دستور `sudo nginx -t`.
2. مشاهده خط دقیق بروز ارور از خروجی `nginx -t` و اصلاح آن (افزودن `;` جا‌افتاده).
3. اجرای مجدد `sudo nginx -t` و `sudo systemctl reload nginx`.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

هرگز قبل از تست سینتکس فایل کانفیگ با دستور `sudo nginx -t` نباید اقدام به reload یا restart وب‌سرور در محیط پروداکشن کرد؛ این اعتبارسنجی باید بخشی از پایپ‌لاین یا فرآیند استاندارد اعمال تغییرات باشد.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
if [ -f /etc/nginx/nginx.conf.bak ]; then
  sudo cp /etc/nginx/nginx.conf.bak /etc/nginx/nginx.conf
  sudo rm /etc/nginx/nginx.conf.bak
fi
sudo systemctl reload nginx
```
