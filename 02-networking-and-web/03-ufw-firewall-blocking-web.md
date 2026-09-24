# ۳: مسدود شدن ترافیک وب توسط فایروال محلی (UFW / iptables)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس (مستقیم / بدون داکر)
- **شرح خرابی / تیکت پروداکشن:**
> **عنوان:** عدم دریافت پاسخ (Connection Timed Out) از سرور
> **گزارش:** سرویس Nginx فعال است و `curl localhost` داخل سرور پاسخ می‌دهد، اما درخواست‌ها از بیرون بی‌پاسخ مانده و تایم‌اوت می‌شوند.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
sudo systemctl start nginx
sudo ufw --force enable
sudo ufw default deny incoming
sudo ufw delete allow 80/tcp || true
sudo ufw delete allow 'Nginx HTTP' || true
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** سرویس وب از طریق `curl localhost` پاسخ `200 OK` می‌داد اما کلاینت‌های خارجی دچار تایم‌اوت می‌شدند.
* **علائم بالینی (Symptoms):** در خروجی `sudo iptables -L -n -v`، پکت‌های ورودی به زنجیره `ufw-user-input` هدایت و توسط پالیسی پیش‌فرض `DROP` می‌شدند؛ `sudo ufw status verbose` وضعیت `Default: deny (incoming)` را بدون رول صریح برای پورت ۸۰/۴۴۳ نشان می‌داد.
* **علت ریشه‌ای (Root Cause):** فعال بودن فایروال UFW با سیاست پیش‌فرض مسدودسازی ترافیک ورودی و عدم تعریف رول صریح برای پورت‌های ترافیک وب.

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی وضعیت فایروال با `sudo ufw status verbose` یا `sudo iptables -L -n -v`.
2. تعریف قوانین مجاز برای سرویس‌ها و ریلود فایروال:
   ```bash
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw allow 22/tcp
   sudo ufw reload
   ```

### 💡 درس‌آموخته مهندسی (Lesson Learned)

هنگام اعمال تنظیمات فایروال و تغییر پالیسی ورودی به `deny`، قبل از هر کاری باید پورت مدیریت از راه دور (`22/tcp`) و پورت‌های ترافیک برنامه‌ها باز شوند تا از قطع دسترسی به سرور (Lockout) جلوگیری شود.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
sudo ufw allow 80/tcp
sudo ufw allow 22/tcp
sudo ufw disable
```
