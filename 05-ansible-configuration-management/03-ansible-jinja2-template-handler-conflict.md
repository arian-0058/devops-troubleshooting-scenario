# سناریو ۰۳: پیکربندی پویا با Jinja2، تریگر شرطی Handlerها و رفع تداخل پورت Nginx

---

## ۱. صورت مسئله و تیکت پروداکشن (Problem Statement & Production Ticket)

* **شناسه رخداد (Incident ID):** `INC-ANS-003`  
* **سطح اولویت (Incident Severity):** `High (P2)`  
* **محیط عملیاتی:** Ubuntu 22.04 LTS / Nginx Web Server  
* **شرح تیکت:** وب‌سرور Nginx باید با یک صفحه وضعیت پویا که نام هاست (`ansible_hostname`)، نسخه سیستم‌عامل و محیط استقرار (`Staging`) را نمایش می‌دهد راه‌اندازی شود. در حین استقرار، به دلیل وجود Virtual Hostهای قدیمی در `/etc/nginx/sites-enabled/`، تداخل پورت رخ داده و همچنین ری‌استارت‌های غیرضروری سرویس در زمان عدم تغییر فایل کانفیگ باعث اختلال لحظه‌ای می‌شود.

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

اسکریپت زیر یک کانفیگ معیوب و متداخل در مسیر `sites-enabled` ایجاد می‌کند تا خطای راه‌اندازی مجدد Nginx شبیه‌سازی شود:

```sh
#!/bin/bash
# failure_inject_03.sh
mkdir -p ~/ansible-lab/03-templates-handlers
cd ~/ansible-lab/03-templates-handlers || exit 1

cat << 'EOF' > inventory.ini
[local]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
EOF

# تزریق کانفیگ خراب در Nginx برای شبیه‌سازی خطای سینتکس و تداخل
echo "invalid_directive on;" | sudo tee /etc/nginx/sites-enabled/broken.local >/dev/null

echo "[INFO] Broken virtual host injected into /etc/nginx/sites-enabled/broken.local"
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms & Logs)

هنگام تلاش برای ری‌استارت سرویس Nginx توسط انسیبل یا بررسی وضعیت سرویس، خطای زیر ثبت می‌شود:

```
RUNNING HANDLER [Restart Nginx] ************************************************
fatal: [localhost]: FAILED! => {
    "changed": false,
    "msg": "Unable to restart service nginx: Job for nginx.service failed because the control process exited with error code."
}
```

و با بررسی لاگ اعتبارسنجی کانفیگ:

```
$ sudo nginx -t
nginx: [emerg] unknown directive "invalid_directive" in /etc/nginx/sites-enabled/broken.local:1
nginx: configuration file /etc/nginx/nginx.conf test failed
```

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

1. **عدم پاکسازی کانفیگ‌های قدیمی یا نامعتبر:** Nginx در زمان راه‌اندازی تمام فایل‌های موجود در `/etc/nginx/sites-enabled/` را بارگذاری می‌کند؛ وجود هر فایل اضافی یا دارای خطای سینتکسی مانع از بالا آمدن کل سرویس می‌شود.  
2. **نبود مکانیزم رویدادمحور (Event-Driven Reload):** ری‌استارت کردن سرویس وب‌سرور در هر بار اجرای پلی‌بوک (حتی زمانی که فایل HTML یا کانفیگ تغییر نکرده است) باعث قطع ارتباطات فعال کاربران می‌شود؛ این عملیات باید صرفاً از طریق `notify` و `handlers` اجرا شود.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

### گام ۱: پاکسازی کانفیگ‌های معیوب و اعتبارسنجی پیش از استقرار

```sh
sudo rm -f /etc/nginx/sites-enabled/broken.local
sudo nginx -t
```

### گام ۲: ایجاد قالب پویا با Jinja2 (`index.html.j2`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Ansible Web Server</title>
</head>
<body>
    <h1>Deployed with Ansible!</h1>
    <ul>
        <li><strong>Hostname:</strong> {{ ansible_hostname }}</li>
        <li><strong>OS:</strong> {{ ansible_distribution }} {{ ansible_distribution_version }}</li>
        <li><strong>Environment:</strong> {{ app_env }}</li>
    </ul>
</body>
</html>
```

### گام ۳: نگارش پلی‌بوک `deploy_nginx.yml` با متغیرها و Handler

```
---
- name: Install Nginx and deploy custom Jinja2 template
  hosts: local
  become: yes
  vars:
    app_env: "Staging"

  tasks:
    - name: Install Nginx web server
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Ensure Nginx service is started and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy custom index.html using template module
      ansible.builtin.template:
        src: index.html.j2
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

### گام ۴: اجرا و تایید خروجی رندر شده

```sh
ansible-playbook -i inventory.ini deploy_nginx.yml -K
curl -s http://localhost | grep -E "(Hostname|Environment)"
```

---

## ۶. درس‌آموخته مهندسی و اسکریپت بازگشت (Lesson Learned & Revert Script)

### درس‌آموخته مهندسی

* **استفاده از Jinja2 برای حذف Hardcoding:** ترکیب متغیرهای کشف‌شده سیستم (`ansible_hostname` و `ansible_distribution`) با متغیرهای سفارشی پلی‌بوک (`app_env`) امکان استفاده از یک فایل قالب واحد برای محیط‌های مختلف (`Staging` و `Production`) را فراهم می‌کند.  
* **کاربرد صحیح Handlerها:** Handlerها تنها زمانی اجرا می‌شوند که تسک فراخوانی‌کننده (`notify`) وضعیت `changed` داشته باشد و در انتهای Play اجرا می‌شوند.

### اسکریپت بازگشت (Revert Script)

```sh
#!/bin/bash
# revert_03.sh
sudo rm -f /etc/nginx/sites-enabled/broken.local
rm -rf ~/ansible-lab/03-templates-handlers
echo "[OK] Scenario 03 cleaned up."
```

