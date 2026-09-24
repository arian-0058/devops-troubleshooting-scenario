# ۴: بروز خطای مجوزی داخل کانتینر به دلیل اشتباه در Mount Volume

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** ویندوز (محیط WSL / Docker)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** خطای Permission Denied یا بالا نیامدن وب‌سرور داکر پس از مونت والیوم
> **گزارش:** کانتینر Nginx لاگ خطای دسترسی به مسیر دایرکتوری داده‌های استاتیک ثبت می‌کند و پاسخ ۴۰۳ یا ۵۰۰ می‌دهد.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
mkdir -p /tmp/docker_static_test
echo "Docker Web App" > /tmp/docker_static_test/index.html
chmod 000 /tmp/docker_static_test/index.html
docker run -d --name perm_web -p 8085:80 -v /tmp/docker_static_test:/usr/share/nginx/html:ro nginx:alpine
curl -I http://localhost:8085 || true
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** دریافت خطای `HTTP 403 Forbidden` و عدم امکان سرویس‌دهی صفحات وب توسط Nginx.
* **علائم بالینی (Symptoms):** ثبت خطای `open() "/usr/share/nginx/html/index.html" failed (13: Permission denied)` در `docker logs`.
* **علت ریشه‌ای (Root Cause):** تنظیم دسترسی `000` روی فایل مونت‌شده در هاست (`/tmp/docker_static_test/index.html`) که دسترسی خواندن را از پروسه وب‌سرور داخل کانتینر سلب کرده بود.

### 🛠️ مراحل رفع ایراد (Fix)

1. لاگ‌گیری از کانتینر با `docker logs perm_web`.
2. بررسی فایل‌های داخل کانتینر با `docker exec perm_web ls -la /usr/share/nginx/html`.
3. تصحیح دسترسی فایل در هاست با `sudo chmod 644 /tmp/docker_static_test/index.html` و `sudo chmod 755 /tmp/docker_static_test`، سپس `docker restart perm_web`.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

در Bind Mountها، دسترسی‌های سیستم‌فایل هاست مستقیماً به داخل کانتینر منتقل می‌شوند؛ پروسه‌های بدون دسترسی روت داخل کانتینر برای خواندن فایل‌ها حداقل به مجوز Read (`4`) روی فایل و Execute (`5` یا `7`) روی تمام دایرکتوری‌های والد نیاز دارند.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
docker rm -f perm_web || true
rm -rf /tmp/docker_static_test
```
