# ۲: اشغال بودن پورت ۸۰ توسط پردازه ناشناس (Port Conflict)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس (مستقیم / بدون داکر)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** عدم استارت سرویس Nginx پس از راه‌اندازی مجدد
> **گزارش:** سرویس Nginx استارت نمی‌شود و دستور `systemctl start nginx` با خطا مواجه می‌شود. ترافیک وب قطع است.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
sudo systemctl stop nginx
sudo python3 -m http.server 80 &
sudo systemctl start nginx
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** سرویس Nginx استارت نمی‌شد و `systemctl start nginx` با خطای خروج غیرعادی فرآیند کنترل مواجه می‌شد.
* **علائم بالینی (Symptoms):** لاگ Nginx خطای `bind() to 0.0.0.0:80 failed (98: Address already in use)` را نشان می‌داد و `sudo ss -tulpn | grep 80` نشان داد پورت توسط پروسس پایتون اشغال شده است.
* **علت ریشه‌ای (Root Cause):** یک پردازه مستقل دیگر روی آدرس `0.0.0.0:80` در حالت `LISTEN` قرار داشت و امکان بایند شدن سوکت Nginx وجود نداشت.

### 🛠️ مراحل رفع ایراد (Fix)

1. مشاهده وضعیت سرویس با `sudo systemctl status nginx` یا `journalctl -xeu nginx`.
2. بررسی پورت‌های در حال گوش دادن با `sudo ss -tulpn | grep :80` یا `sudo lsof -i :80`.
3. متوقف‌سازی مستقیم پردازه اشغال‌کننده با `sudo fuser -k 80/tcp` یا `sudo kill -9 <PID>` و سپس اجرای مجدد `systemctl start nginx`.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

خطای `Address already in use` در پروداکشن معمولاً نشان‌دهنده نمونه‌های قبلی معلق (Zombie/Hanging processes) یا سرویس‌های تداخلی است؛ ابزارهای `ss`، `lsof` و `fuser` سریع‌ترین زنجیره تشخیص و رفع تداخل سوکت شبکه هستند.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
sudo fuser -k 80/tcp || true
sudo systemctl restart nginx
```
