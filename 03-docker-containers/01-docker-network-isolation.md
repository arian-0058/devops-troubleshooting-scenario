# ۱: عدم دسترسی سرویس‌ها به دیتابیس داکر به دلیل نامعتبر بودن Docker Network

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** ویندوز (محیط WSL / Docker)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** خطای Connection Refused در ارتباط بین کانتینر وب و دیتابیس
> **گزارش:** کانتینر Nginx بالا آمده اما هنگام ارسال درخواست به نام کانتینر دیتابیس (`db`)، خطای نامعتبر بودن هاست و عدم اتصال دریافت می‌شود.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
docker network create web_net || true
docker run -d --name test_db --network bridge redis:alpine
docker run -d --name test_web --network web_net nginx:alpine
docker exec test_web ping -c 2 test_db || true
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** خطای `ping: bad address` و عدم تفکیک نام و برقراری ارتباط بین کانتینر وب (`test_web`) و دیتابیس (`test_db`).
* **علائم بالینی (Symptoms):** وب‌سرور بالا می‌آید اما امکان ارسال یا دریافت پاسخ از سمت نام کانتینر دیتابیس را ندارد.
* **علت ریشه‌ای (Root Cause):** کانتینر `test_web` روی شبکه سفارشی `web_net` قرار داشت، در حالی که کانتینر `test_db` روی شبکه پیش‌فرض `bridge` اجرا شده بود (شبکه پیش‌فرض `bridge` قابلیت Service Discovery و DNS داخلی بر پایه نام کانتینر ندارد).

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی شبکه هر کانتینر با `docker inspect test_db -f '{{json .NetworkSettings.Networks}}'` و مشابه برای `test_web`.
2. اطمینان از اینکه کانتینرها روی یک شبکه سفارشی مشترک (Custom Bridge Network) قرار دارند.
3. اتصال کانتینر دیتابیس به شبکه کانتینر وب: `docker network connect web_net test_db`.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

برای استفاده از قابلیت Built-in DNS داکر و تفکیک نام کانتینرها (Service Discovery)، تمام سرویس‌های مرتبط باید حتماً روی یک شبکه سفارشی مشترک (User-defined Bridge Network) قرار بگیرند.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
docker rm -f test_db test_web || true
docker network rm web_net || true
```
