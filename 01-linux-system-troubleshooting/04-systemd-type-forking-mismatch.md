# ۴: خرابی بالا آمدن سرویس سیستمی به‌دلیل نوع نادرست Type در Systemd

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس خام (Native Linux)
- **شرح خرابی / تیکت پروداکشن:**
> **تیکت:** #INC-44129 | **اولویت:** Medium
> گزارش: سرویس `custom-collector.service` پس از ری‌لود سیستم در وضعیت `activating (start)` تایم‌اوت شده و وارد حالت `failed` می‌شود؛ در حالی که با دستور دستی کار می‌کند.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
mkdir -p /opt/collector
cat << 'APP' > /opt/collector/run.sh
#!/bin/bash
while true; do
    sleep 2
done
APP
chmod +x /opt/collector/run.sh

cat << 'UNIT' | sudo tee /etc/systemd/system/custom-collector.service > /dev/null
[Unit]
Description=Custom Metric Collector Service
After=network.target

[Service]
Type=forking
ExecStart=/opt/collector/run.sh
TimeoutStartSec=6
Restart=on-failure

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl start custom-collector.service || true
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** تایم‌اوت شدن و رفتن سرویس به وضعیت Failed در زمان بالا آمدن (`activating (start)`) در حالی که اسکریپت به‌صورت دستی کار می‌کند.
* **علائم بالینی (Symptoms):** خطای `Job failed because a timeout was exceeded` در `systemctl status` و لوپ ری‌استارت‌ها در `journalctl -xeu`.
* **علت ریشه‌ای (Root Cause):** تنظیم `Type=forking` برای اسکریپتی که در فورگراند اجرا می‌شود؛ Systemd منتظر خروج پروسه والد می‌ماند و پس از پایان `TimeoutStartSec` سرویس را متوقف می‌کند.

### 🛠️ مراحل رفع ایراد (Fix)

1. بررسی لاگ اجرایی با `sudo systemctl status custom-collector.service` یا `sudo journalctl -u custom-collector.service -e`.
2. ویرایش یونیت فایل در `/etc/systemd/system/custom-collector.service` و تغییر `Type=forking` به `Type=simple` یا `Type=exec`.
3. اجرای `sudo systemctl daemon-reload` و استارت مجدد سرویس.

### 💡 درس‌آموخته مهندسی (Lesson Learned)

پس از هرگونه تغییر در فایل‌های یونیت Systemd، اجرای `systemctl daemon-reload` الزامی است؛ سرویس‌هایی که خروجی فورگراند دارند باید با تایپ `simple` یا `exec` تعریف شوند، نه `forking`.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
sudo systemctl stop custom-collector.service 2>/dev/null || true
sudo systemctl disable custom-collector.service 2>/dev/null || true
sudo rm -f /etc/systemd/system/custom-collector.service
sudo rm -rf /opt/collector
sudo systemctl daemon-reload
```
