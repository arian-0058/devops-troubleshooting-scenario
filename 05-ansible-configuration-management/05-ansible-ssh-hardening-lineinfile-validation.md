# سناریو ۰۵: سخت‌سازی امنیتی SSH با `lineinfile` و شکست اعتبارسنجی `sshd -t` به دلیل نبود `/run/sshd`

---

## ۱. صورت مسئله و تیکت پروداکشن (Problem Statement & Production Ticket)

* **شناسه رخداد (Incident ID):** `INC-ANS-005`  
* **سطح اولویت (Incident Severity):** `Critical (P1)`  
* **محیط عملیاتی:** Ubuntu 22.04 LTS (WSL2 / Cloud VM Hardening)  
* **شرح تیکت:** در فرآیند سخت‌سازی خودکار سرور (`base_security`)، باید دسترسی ورود مستقیم `root` و احراز هویت با رمز عبور در سرویس SSH غیرفعال شوند. هرگونه خطای سینتکسی در فایل `sshd_config` می‌تواند منجر به قفل شدن دائمی دسترسی ادمین به سرور (Admin Lockout) شود. بنابراین تغییر خطوط باید با استفاده از `lineinfile` و مکانیزم اعتبارسنجی `validate: 'sshd -t -f %s'` انجام شود؛ اما هنگام اجرای اعتبارسنجی در محیط ایزوله، ابزار `sshd -t` به دلیل نبود دایرکتوری سیستمی `/run/sshd` شکست می‌خورد.

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

اسکریپت زیر وضعیت محیطی را که دایرکتوری Privilege Separation دیمن SSH در آن وجود ندارد شبیه‌سازی می‌کند:

```sh
#!/bin/bash
# failure_inject_05.sh
mkdir -p ~/ansible-lab/05-ssh-hardening
cd ~/ansible-lab/05-ssh-hardening || exit 1

cat << 'EOF' > inventory.ini
[local]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
EOF

# حذف دایرکتوری /run/sshd برای شبیه‌سازی محیط کانتینری/WSL که سرویس sshd هنوز استارت نشده است
sudo rm -rf /run/sshd
sudo rm -f /tmp/sshd_config.test /tmp/sshd_config.bak

echo "[INFO] /run/sshd removed. Running sshd -t will now fail."
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms & Logs)

هنگام اجرای تسک `lineinfile` همراه با پارامتر `validate: 'sshd -t -f %s'`، انسیبل از ذخیره فایل جلوگیری کرده و خطای زیر را برمی‌گرداند:

```
TASK [Harden SSH settings in test file with syntax validation] *****************
failed: [localhost] (item={'regexp': '^#?PermitRootLogin\\s+', 'line': 'PermitRootLogin no'}) => {
    "changed": false,
    "msg": "failed to validate: rc:255 error:Missing privilege separation directory: /run/sshd\r\n"
}
```

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

1. **وابستگی باینری `sshd` به دایرکتوری Runtime:** باینری `sshd` حتی در حالت تست پیکربندی (`-t`) وجود دایرکتوری `/run/sshd` را به عنوان مسیر جداسازی سطح دسترسی (Privilege Separation) بررسی می‌کند. در سیستم‌هایی که سرویس `ssh` هنوز یک بار هم توسط `systemd` استارت نشده باشد (مانند WSL2 یا ایمیج‌های پایه داکر)، این پوشه در مسیر موقت `/run` ایجاد نشده است.  
2. **اهمیت دروازه اعتبارسنجی (`validate`):** پارامتر `validate` فایل موقت ویرایش‌شده (`%s`) را قبل از جایگزینی روی فایل اصلی تست می‌کند و تا زمانی که خروجی دستور اعتبارسنجی `0` نشود، تغییری روی فایل هدف اعمال نمی‌کند.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

### گام ۱: نگارش پلی‌بوک امن `server_audit_and_harden.yml`

در این پلی‌بوک ابتدا وجود فایل با `stat` و `register` بررسی شده، با شرط `when` نسخه پشتیبان گرفته می‌شود، دایرکتوری `/run/sshd` تضمین می‌گردد و سپس تغییرات سخت‌سازی با اعتبارسنجی اعمال می‌شوند:

```
---
- name: Audit and Harden SSH Configuration Safely
  hosts: local
  become: yes

  tasks:
    - name: Check if sshd_config exists
      ansible.builtin.stat:
        path: /etc/ssh/sshd_config
      register: sshd_file

    - name: Backup original sshd_config if it exists
      ansible.builtin.copy:
        src: /etc/ssh/sshd_config
        dest: /tmp/sshd_config.bak
        remote_src: yes
        mode: '0600'
      when: sshd_file.stat.exists

    - name: Create a test copy of sshd_config
      ansible.builtin.copy:
        src: /etc/ssh/sshd_config
        dest: /tmp/sshd_config.test
        remote_src: yes
        mode: '0600'
      when: sshd_file.stat.exists

    - name: Ensure /run/sshd directory exists for sshd validation
      ansible.builtin.file:
        path: /run/sshd
        state: directory
        owner: root
        group: root
        mode: '0755'

    - name: Harden SSH settings in test file with syntax validation
      ansible.builtin.lineinfile:
        path: /tmp/sshd_config.test
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        validate: 'sshd -t -f %s'
      loop:
        - { regexp: '^#?PermitRootLogin\s+', line: 'PermitRootLogin no' }
        - { regexp: '^#?PasswordAuthentication\s+', line: 'PasswordAuthentication no' }
```

### گام ۲: اجرا و راستی‌آزمایی تنظیمات اعمال‌شده

```sh
ansible-playbook -i inventory.ini server_audit_and_harden.yml -K
sudo grep -E "^(PermitRootLogin|PasswordAuthentication)" /tmp/sshd_config.test
```

**خروجی مورد انتظار:**

```
PermitRootLogin no
PasswordAuthentication no
```

---

## ۶. درس‌آموخته مهندسی و اسکریپت بازگشت (Lesson Learned & Revert Script)

### درس‌آموخته مهندسی

* **پیشگیری از Lockout در اتوماسیون:** هرگز فایل‌های حساس سیستمی مانند `/etc/ssh/sshd_config` یا `/etc/sudoers` را بدون استفاده از پارامتر `validate` (`sshd -t -f %s` یا `visudo -cf %s`) ویرایش نکنید.  
* **الگوهای Regex تکرارپذیر در `lineinfile`:** استفاده از پیشوند `^#?` باعث می‌شود چه پارامتر در فایل اولیه کامنت شده باشد و چه فعال، با مقدار امن جایگزین شود و از ایجاد خطوط تکراری در اجراهای بعدی جلوگیری گردد.

### اسکریپت بازگشت (Revert Script)

```sh
#!/bin/bash
# revert_05.sh
sudo rm -f /tmp/sshd_config.test /tmp/sshd_config.bak
rm -rf ~/ansible-lab/05-ssh-hardening
echo "[OK] Scenario 05 temporary hardening files cleaned up."
```

