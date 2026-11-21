# سناریو ۰۴: ماژولارسازی با Ansible Roles، بهینه‌سازی با Loop و رفع خطای Dry-Run (`--check`)

---

## ۱. صورت مسئله و تیکت پروداکشن (Problem Statement & Production Ticket)

* **شناسه رخداد (Incident ID):** `INC-ANS-004`  
* **سطح اولویت (Incident Severity):** `Medium (P3)`  
* **محیط عملیاتی:** Ubuntu 22.04 LTS / Production Provisioning  
* **شرح تیکت:** با افزایش تعداد پکیج‌های مورد نیاز سرور (`nginx`، `ufw`، `fail2ban`) و پیچیده‌تر شدن تسک‌ها، نگهداری کدها در یک فایل پلی‌بوک واحد خوانایی و قابلیت استفاده مجدد را از بین برده است. همچنین هنگام اجرای تست شبیه‌سازی (`--check`) قبل از اعمال تغییرات روی سرور، پلی‌بوک به دلیل عدم دریافت رمز ارتقای دسترسی (`sudo`) در مرحله `Gathering Facts` با خطای بحرانی متوقف می‌شود.

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

اسکریپت زیر ساختار اولیه نقش `web_setup` را ایجاد کرده و اجرای Dry-Run بدون فلگ احراز هویت `sudo` را شبیه‌سازی می‌کند:

```sh
#!/bin/bash
# failure_inject_04.sh
mkdir -p ~/ansible-lab/04-roles/roles
cd ~/ansible-lab/04-roles || exit 1

cat << 'EOF' > inventory.ini
[local]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
EOF

ansible-galaxy init roles/web_setup >/dev/null 2>&1

cat << 'EOF' > site.yml
---
- name: Setup Web Server
  hosts: local
  become: yes
  roles:
    - web_setup
EOF

echo "[INFO] Run 'ansible-playbook -i inventory.ini site.yml --check' to reproduce sudo failure."
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms & Logs)

اجرای دستور `--check` بدون ارائه رمز `sudo` منجر به شکست فوری پلی‌بوک در اولین گام می‌شود:

```
$ ansible-playbook -i inventory.ini site.yml --check
PLAY [Setup Web Server] ********************************************************

TASK [Gathering Facts] *********************************************************
fatal: [localhost]: FAILED! => {
    "ansible_facts": {},
    "changed": false,
    "module_stderr": "sudo: a password is required\n",
    "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error",
    "rc": 1
}
```

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

1. **نیاز به احراز هویت Sudo حتی در حالت `--check`:** زمانی که `become: yes` در سطح Play فعال است، انسیبل حتی برای اجرای ماژول `setup` (جمع‌آوری Facts) و شبیه‌سازی تغییرات در حالت Dry-Run، دستورات را از طریق `sudo` اجرا می‌کند. در صورتی که کاربر دسترسی `NOPASSWD` در `/etc/sudoers` نداشته باشد، نبود فلگ `-K` (`--ask-become-pass`) باعث خطای `sudo: a password is required` می‌شود.  
2. **تکرار کد در نصب پکیج‌ها:** نوشتن تسک‌های جداگانه برای هر پکیج (`nginx`، `ufw`، `fail2ban`) زمان اجرا را افزایش داده و نگهداری متغیرها را دشوار می‌کند؛ انتقال لیست پکیج‌ها به `defaults/main.yml` و استفاده از `loop` این مشکل را حل می‌کند.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

### گام ۱: تعریف متغیرهای پیش‌فرض در `roles/web_setup/defaults/main.yml`

```
---
app_env: "Production"
required_packages:
  - nginx
  - ufw
  - fail2ban
```

### گام ۲: پیاده‌سازی تسک‌ها با حلقه `loop` در `roles/web_setup/tasks/main.yml`

```
---
- name: Install required packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
    cache_valid_time: 3600
  loop: "{{ required_packages }}"

- name: Deploy index.html template
  ansible.builtin.template:
    src: index.html.j2
    dest: /var/www/html/index.html
    owner: www-data
    group: www-data
    mode: '0644'
  notify: Restart Nginx

- name: Ensure Nginx is running and enabled
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: yes
```

### گام ۳: قرار دادن Handler در `roles/web_setup/handlers/main.yml` و تمپلیت در `templates/index.html.j2`

```
# roles/web_setup/handlers/main.yml
---
- name: Restart Nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

### گام ۴: اعتبارسنجی سینتکس، Dry-Run امن و اجرای نهایی

```sh
# ۱. بررسی سینتکس YAML
ansible-playbook -i inventory.ini site.yml --syntax-check

# ۲. اجرای شبیه‌سازی (Dry-Run) همراه با احراز هویت sudo
ansible-playbook -i inventory.ini site.yml --check -K

# ۳. اجرای نهایی Role
ansible-playbook -i inventory.ini site.yml -K
```

---

## ۶. درس‌آموخته مهندسی و اسکریپت بازگشت (Lesson Learned & Revert Script)

### درس‌آموخته مهندسی

* **معماری ماژولار با Roles:** تفکیک منطق پروژه به پوشه‌های استاندارد (`tasks`، `handlers`، `templates`، `defaults`) زیربنای اصلی پروژه‌های بزرگ اتوماسیون است و امکان استفاده مجدد از یک Role در چندین محیط و پروژه مختلف را فراهم می‌کند.  
* استفاده از `--syntax-check` و `--check -K` به عنوان دو گیت پیش‌اعتبارسنجی (Pre-flight Checks) از بروز خطاهای تایپی و منطقی در سرورهای پروداکشن جلوگیری می‌کند.

### اسکریپت بازگشت (Revert Script)

```sh
#!/bin/bash
# revert_04.sh
rm -rf ~/ansible-lab/04-roles
echo "[OK] Scenario 04 Role lab cleaned up."
```

