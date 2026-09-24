# ۲: اشغال بودن پورت لوکال‌هاست روی WSL و توقف اجرای کانتینر وب

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** ویندوز (محیط WSL / Docker)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** خطای Bind for 0.0.0.0:8080 failed: port is already allocated
> **گزارش:** تلاش برای بالا آوردن کانتینر جدید با پورت فورواردینگ روی پورت ۸۰۸۰ ناموفق بوده و استقرار کانتینر متوقف شده است.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
docker run -d --name dummy_holder -p 8080:80 nginx:alpine
docker run -d --name main_app -p 8080:80 nginx:alpine || true
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** شکست در راه‌اندازی و اجرای کانتینر جدید به دلیل عدم امکان نگاشت پورت (`Bind for 0.0.0.0:8080 failed: port is already allocated`).
* **علائم بالینی (Symptoms):** کانتینر در وضعیت `Created` باقی می‌ماند و دستور `docker run` با خطای تنظیمات شبکه متوقف می‌شود.
* **علت ریشه‌ای (Root Cause):** پورت `8080` هاست پیش‌تر توسط کانتینر دیگری (`dummy_holder`) اشغال شده بود.

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی خطای دقیق و وضعیت کانتینرها با `docker ps -a` و لاگ‌ها.
2. شناسایی کانتینر اشغال‌کننده با `docker ps --filter "publish=8080"` یا در WSL با `ss -tulpn | grep :8080`.
3. متوقف کردن و حذف کانتینر مزاحم با `docker stop dummy_holder && docker rm dummy_holder` و سپس استارت کانتینر اصلی با `docker start main_app` (به جای اجرای مجدد `docker run`).

### 💡 درس‌آموخته مهندسی (Lesson Learned)

قبل از استقرار کانتینرها باید از خالی بودن پورت‌های مپ‌شده اطمینان حاصل کرد؛ در صورت تداخل با کانتینری که قبلاً ساخته شده اما متوقف مانده، باید با `docker start` آن را اجرا کرد، نه `docker run` مجدد.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
docker rm -f dummy_holder main_app || true
```
