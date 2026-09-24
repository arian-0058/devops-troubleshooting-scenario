# ۵: گیر افتادن پروسه بیلد به دلیل نبود فایل در Build Context

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** ویندوز (محیط WSL / VSCode)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** خطای failed to compute cache key: "/app/src not found" در زمان Docker Build
> **گزارش:** فرآیند ساخت ایمیج روی سیستم توسعه‌دهنده با خطای عدم یافتن مسیر مواجه شده و متوقف می‌شود.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
mkdir -p /tmp/build_fail_test
cd /tmp/build_fail_test
cat << 'EOF' > Dockerfile
FROM alpine:latest
WORKDIR /workspace
COPY ./non_existent_folder /workspace/
CMD ["ls", "-la"]
EOF
docker build -t test-build-image . || true
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** توقف فرآیند ساخت ایمیج با خطای `failed to compute cache key` حین اجرای `docker build`.
* **علائم بالینی (Symptoms):** خروج اضطراری `docker build` در مرحله دستور `COPY` با پیام `failed to calculate checksum ... "/non_existent_folder": not found`.
* **علت ریشه‌ای (Root Cause):** ارجاع دستور `COPY` در داکرفایل به دایرکتوری‌ای (`./non_existent_folder`) که در Build Context وجود خارجی نداشت.

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی لایه‌های Dockerfile و مسیر Build Context.
2. بررسی محتویات دایرکتوری جاری در مقایسه با دستورات `COPY` یا `ADD`.
3. اصلاح داکرفایل با تغییر مسیر منبع به فایل/مسیر معتبر موجود در کانتکست (مانند `COPY . /workspace/`).

### 💡 درس‌آموخته مهندسی (Lesson Learned)

دستور `COPY` و `ADD` فقط به فایل‌هایی دسترسی دارند که درون مسیر Build Context مشخص‌شده هنگام بیلد (آرگومان انتهایی `docker build .`) قرار دارند؛ خارج از آن یا فایل‌های ناموجود بلافاصله پروسه بیلد را متوقف می‌کنند.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
rm -rf /tmp/build_fail_test
docker rmi test-build-image 2>/dev/null || true
```
