# ۵: قفل شدن پورت سرویس توسط یک پروسه بی‌نام در پس‌زمینه (Zombie / Detached Process Socket Lock)

## 📋 صورت مسئله و مشخصات سناریو
- **محیط اجرا:** لینوکس خام (Native Linux)
- **شرح خرابی / تیکت پروداکشن:**
> **تیکت:** #INC-44140 | **اولویت:** High
> گزارش: سرویس API روی پورت `8088` استارت نمی‌شود (`bind: Address already in use`). بررسی اولیه با `ps aux | grep app` پروسسی را نشان نمی‌دهد، اما پورت همچنان در حالت لیسن باقی مانده است.

### 💥 اسکریپت شبیه‌سازی خرابی (Failure Injection)
```bash
python3 -c "
import socket, time
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('0.0.0.0', 8088))
s.listen(5)
while True:
    time.sleep(10)
" &
BLOCK_PID=$!
echo $BLOCK_PID > /tmp/stuck_socket.pid
```

---

## 🔍 گزارش عیب‌یابی و ریشه‌یابی (Troubleshooting)

* **مشکل (Problem):** مسدود ماندن پورت شبکه توسط یک پروسه پایتون فاقد نام استاندارد، در حالی که در ابزارهای رصد پورت به‌سختی قابل شناسایی است.
* **علائم بالینی (Symptoms):** خطای `Address already in use` هنگام بایند شدن به پورت، بدون اینکه `ps aux | grep app` پروسه‌ای را نشان دهد.
* **علت ریشه‌ای (Root Cause):** اجرای پروسه با کاراکتر `&` که ممکن است سیگنال `SIGHUP` دریافت نکند و به‌صورت Detached در پس‌زمینه باقی بماند و سوکت پورت ۸۰۸۸ را نگه دارد.

### 🛠️ مراحل رفع ایراد (Fix)

1. شناسایی سوکت و PID دقیق با `sudo ss -tulpn | grep :8088` یا `sudo lsof -i :8088`.
2. بررسی جزئیات پروسس مسدودکننده با `ps -fp <PID>` یا `pstree -p <PID>`.
3. اجرای ایزوله و پایدار اسکریپت‌های پس‌زمینه با `nohup <command> >/dev/null 2>&1 &` یا `disown` برای جلوگیری از رفتار مشابه در آینده، و خاتمه پردازه مسدودکننده با `sudo kill -15 <PID>` (یا `sudo kill -9 <PID>`).

### 💡 درس‌آموخته مهندسی (Lesson Learned)

برای عیب‌یابی پورت‌های شبکه، `ps aux | grep app` کافی نیست چون اسکریپت‌ها ممکن است تحت پروسس‌های عمومی‌تر مثل `python3` اجرا شوند؛ منبع قطعی کشف اشغال‌کننده پورت، جدول اتصالات سوکت هسته از طریق `ss` یا `lsof -i` است.

---

### 🔄 اسکریپت بازگشت به حالت اول (Revert)

```bash
if [ -f /tmp/stuck_socket.pid ]; then
    sudo kill -9 $(cat /tmp/stuck_socket.pid) 2>/dev/null || true
    rm -f /tmp/stuck_socket.pid
fi
sudo fuser -k 8088/tcp 2>/dev/null || true
```
