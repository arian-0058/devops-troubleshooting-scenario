# ۶: شکست احراز هویت کلید SSH به‌دلیل ناامنی مجوزهای دسترسی (StrictModes)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس خام (Native Linux)
- **شرح خرابی / تیکت پروداکشن:**
> **تیکت:** #INC-44171 | **اولویت:** Critical
> گزارش: پس از به‌روزرسانی پیکربندی سیستم، کاربران امکان لاگین با کلید SSH به اکانت سرویسی `svc_deploy` را از دست داده‌اند و سرویس SSH مدام درخواست پسورد می‌کند (کلید Public نادیده گرفته می‌شود).

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
sudo id -u svc_deploy &>/dev/null || sudo useradd -m -s /bin/bash svc_deploy
sudo mkdir -p /home/svc_deploy/.ssh
sudo touch /home/svc_deploy/.ssh/authorized_keys

sudo chmod 777 /home/svc_deploy
sudo chmod 777 /home/svc_deploy/.ssh
sudo chmod 666 /home/svc_deploy/.ssh/authorized_keys
sudo chown -R svc_deploy:svc_deploy /home/svc_deploy
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** عدم امکان لاگین کاربران با کلید SSH به حساب سرویسی `svc_deploy` و درخواست مکرر کلمه عبور توسط OpenSSH.
* **علائم بالینی (Symptoms):** کلید عمومی نادیده گرفته می‌شود و در `/var/log/auth.log` یا `journalctl -u ssh`، پیام `Authentication refused: bad ownership or modes for directory/file` ثبت می‌گردد.
* **علت ریشه‌ای (Root Cause):** قابلیت پیش‌فرض `StrictModes yes` در `sshd_config`، دایرکتوری خانگی، پوشه `.ssh` یا فایل `authorized_keys` را در صورت داشتن دسترسی نوشتن برای گروه/سایرین، ناامن تشخیص داده و احراز هویت کلید را رد می‌کند.

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی لاگ امنیتی با `sudo tail -n 30 /var/log/auth.log` یا `sudo journalctl -u ssh -e`.
2. بررسی مفهوم `StrictModes` در `/etc/ssh/sshd_config`.
3. بازگرداندن پرمیشن‌ها به استاندارد حداقل دسترسی:
   ```bash
   sudo chmod 755 /home/svc_deploy
   sudo chmod 700 /home/svc_deploy/.ssh
   sudo chmod 600 /home/svc_deploy/.ssh/authorized_keys
   sudo chown -R svc_deploy:svc_deploy /home/svc_deploy
   ```

### 💡 درس‌آموخته مهندسی (Lesson Learned)

استفاده از پرمیشن باز (مانند `777`) نه‌تنها مشکل دسترسی را حل نمی‌کند بلکه محافظ‌های امنیتی هسته SSH را فعال می‌سازد. کاراکتر `~` همیشه به خانه کاربر جاری شل اشاره دارد و در دستورات مدیریتی چندکاربره باید آدرس‌دهی کامل (`/home/<user>/...`) استفاده شود.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
sudo userdel -r svc_deploy 2>/dev/null || true
```
