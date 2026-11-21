# سناریو ۰۱: خطای اتصال SSH در مدیریت لوکال Ansible و کشف مفسر پایتون (Inventory & Ad-Hoc Connection)

---

## ۱. صورت مسئله و تیکت پروداکشن (Problem Statement & Production Ticket)

* **شناسه رخداد (Incident ID):** `INC-ANS-001`  
* **سطح اولویت (Incident Severity):** `Medium (P3)`  
* **محیط عملیاتی:** Ubuntu 22.04 LTS (WSL2 / Control Node)  
* **شرح تیکت:** تیم عملیات قصد دارد بدون نصب Agent و از طریق دستورات Ad-Hoc انسیبل، وضعیت اتصال و مشخصات سخت‌افزاری/توزیع سیستم‌عامل نود کنترلر (`localhost`) را قبل از اجرای پلی‌بوک‌های اصلی بررسی کند. اما هنگام اجرای ماژول `ping`، انسیبل به جای اجرای محلی تلاش می‌کند از طریق SSH به `localhost` متصل شود و با خطای `Connection refused` متوقف می‌شود؛ همچنین هشدارهای مربوط به `discovered_interpreter_python` در خروجی لاگ‌ها ظاهر می‌گردد.

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

برای بازتولید این خطا در محیط آزمایشگاهی ایزوله، اسکریپت زیر را اجرا کنید:

```sh
#!/bin/bash
# failure_inject_01.sh
mkdir -p ~/ansible-lab/01-inventory
cd ~/ansible-lab/01-inventory || exit 1

# تعریف اشتباه localhost بدون تعیین نوع اتصال محلی
cat << 'EOF' > inventory.ini
[local]
localhost
EOF

echo "[INFO] Environment injected. Run: ansible -i inventory.ini local -m ping"
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms & Logs)

پس از اجرای دستور تست اتصال Ad-Hoc، خروجی زیر در ترمینال ثبت می‌شود:

```
$ ansible -i inventory.ini local -m ping
[WARNING]: Platform linux on host localhost is using the discovered Python interpreter at /usr/bin/python3.10, but future installation of another Python interpreter could change the meaning of that path.
localhost | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: ssh: connect to host localhost port 22: Connection refused",
    "unreachable": true
}
```

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

1. **فرض پیش‌فرض اتصال SSH:** انسیبل به صورت پیش‌فرض برای تمام هاست‌های تعریف‌شده در فایل `inventory.ini` از پلاگین اتصال `ssh` روی پورت ۲۲ استفاده می‌کند. در محیط‌هایی مانند WSL2 یا کانتینرهای تست که سرویس `sshd` فعال نیست، تلاش برای برقراری نشست SSH به `localhost` با خطای `Connection refused` مواجه می‌شود.  
2. **کشف خودکار مفسر پایتون (Interpreter Discovery):** انسیبل برای اجرای ماژول‌ها روی نود هدف نیازمند باینری پایتون است. در صورت عدم تعیین صریح مسیر پایتون در کانفیگ یا اینونتوری، انسیبل مسیر `/usr/bin/python3.10` را به صورت پویا کشف کرده و هشدار تغییر احتمالی مسیر در آینده را صادر می‌کند.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

### گام ۱: اصلاح فایل `inventory.ini` و تنظیم اتصال محلی

پارامتر `ansible_connection=local` و مسیر دقیق مفسر پایتون را برای هاست `localhost` تعریف می‌کنیم:

```
[local]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
```

### گام ۲: ایجاد فایل `ansible.cfg` برای حذف هشدارهای کشف مفسر (اختیاری اما استاندارد)

```
[defaults]
inventory = ./inventory.ini
interpreter_python = auto_silent
```

### گام ۳: اعتبارسنجی اتصال و استخراج مشخصات سیستم با دستورات Ad-Hoc

```sh
# ۱. تست سلامت ارتباط با ماژول ping
ansible -i inventory.ini local -m ping

# ۲. استخراج اطلاعات توزیع لینوکس و حافظه با فیلتر کردن ماژول setup
ansible -i inventory.ini local -m setup -a "filter=ansible_distribution*"
ansible -i inventory.ini local -m setup -a "filter=ansible_memtotal_mb"
```

**خروجی مورد انتظار (Verification):**

```json
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## ۶. درس‌آموخته مهندسی و اسکریپت بازگشت (Lesson Learned & Revert Script)

### درس‌آموخته مهندسی

* در معماری Agentless انسیبل، تفکیک نوع اتصال (`ssh` برای سرورهای ریموت و `local` برای اجرای دستورات روی نود کنترلر) الزامی است.  
* استفاده از دستورات Ad-Hoc (`ping` و `setup`) سریع‌ترین راه برای راستی‌آزمایی دسترسی، جمع‌آوری Facts سیستم‌عامل و پیش‌نیازهای پایتون پیش از اجرای پلی‌بوک‌های سنگین است.

### اسکریپت بازگشت (Revert Script)

```sh
#!/bin/bash
# revert_01.sh
rm -rf ~/ansible-lab/01-inventory
echo "[OK] Scenario 01 lab files cleaned up."
```

