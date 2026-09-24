# ۶: پر شدن دیسک مجازی WSL/Docker و خطای No Space Left On Device

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** ویندوز (محیط WSL / Docker)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** خطای resource exhaustion و عدم امکان اجرای کانتینر یا Pull ایمیج جدید
> **گزارش:** ایمیج‌های بی‌استفاده و Volumeهای معلق باعث پر شدن فضای مربوط به Docker Storage شده و دستورات داکر ارور کمبود حافظه می‌دهند.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
docker volume create dangling_vol_1
docker volume create dangling_vol_2
docker pull alpine:latest
docker pull busybox:latest
docker tag busybox:latest dummy:latest
docker rmi busybox:latest
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** اشغال حجم بالای دیسک ناشی از تجمع لایه‌ها، ایمیج‌های استفاده‌نشده و والیوم‌های معلق (Dangling).
* **علائم بالینی (Symptoms):** پر شدن ظرفیت حافظه داکر، افزایش بی‌رویه حجم در `docker system df` و کندی یا شکست در استقرار کانتینرها و Pull ایمیج‌های جدید.
* **علت ریشه‌ای (Root Cause):** عدم پاک‌سازی والیوم‌های بی‌نام حاصل از تست‌ها و باقی ماندن ایمیج‌های فاقد تگ یا بدون کانتینر فعال در استوریج داکر.

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی میزان حجم مصرفی منابع مختلف داکر با `docker system df`.
2. مشاهده ایمیج‌ها و والیوم‌های معلق با `docker images -f "dangling=true"` و `docker volume ls -f "dangling=true"`.
3. پاک‌سازی امن منابع بی‌استفاده: `docker volume prune -a` و `docker image prune -a` (یا `docker system prune -f`).

### 💡 درس‌آموخته مهندسی (Lesson Learned)

اجرای دوره‌ای دستورات `prune` فیلتردار برای سیستم‌های توسعه ضروری است؛ این دستورات ذاتاً ایمن هستند و هرگز ایمیج‌ها یا والیوم‌هایی که توسط کانتینرهای فعال در حال استفاده هستند را حذف نمی‌کنند.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
docker system prune -a --volumes -f
```
