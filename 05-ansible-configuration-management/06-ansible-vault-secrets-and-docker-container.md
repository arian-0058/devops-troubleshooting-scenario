# سناریو ۰۶: مدیریت امن اسرار با Ansible Vault، جلوگیری از نشت لاگ (`no_log`) و استقرار کانتینر Docker

---

## ۱. صورت مسئله و تیکت پروداکشن (Problem Statement & Production Ticket)

* **شناسه رخداد (Incident ID):** `INC-ANS-006`  
* **سطح اولویت (Incident Severity):** `Critical (P1)`  
* **محیط عملیاتی:** Ubuntu 22.04 LTS / Docker Engine  
* **شرح تیکت:** برای استقرار سرویس‌های کانتینری (مانند پایگاه داده In-Memory ردیس و سرویس‌های اپلیکیشن)، نگهداری رمزهای عبور (`db_root_password` و `app_secret_key`) به صورت متن ساده (Plaintext) در مخزن گیت نقض قوانین امنیت اطلاعات است. همچنین هنگام تزریق این متغیرها به فایل `.env` توسط انسیبل، مقادیر رمز در خروجی استاندارد (Stdout) و لاگ‌های CI/CD چاپ شده و فایل `.env` با پرمیشن باز (`0644`) برای تمام کاربران سیستم قابل خواندن می‌شود.

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

اسکریپت زیر وضعیت ناامن نگهداری رمزهای خام و دسترسی باز فایل `.env` را شبیه‌سازی می‌کند:

```sh
#!/bin/bash
# failure_inject_06.sh
mkdir -p ~/ansible-lab/06-vault-docker
cd ~/ansible-lab/06-vault-docker || exit 1
sudo mkdir -p /opt/devops_lab

# شبیه‌سازی فایل .env ناامن با پرمیشن قابل خواندن برای همه (World-Readable)
echo "DB_ROOT_PASSWORD=PlainTextInsecurePassword123" | sudo tee /opt/devops_lab/.env >/dev/null
sudo chmod 0644 /opt/devops_lab/.env

echo "[INFO] Insecure .env created with 0644 permissions:"
ls -l /opt/devops_lab/.env
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms & Logs)

در صورت عدم استفاده از `Ansible Vault` و `no_log: true`، هنگام اجرای پلی‌بوک با حالت Verbose (`-v`) یا بروز خطا، مقادیر محرمانه مستقیماً در خروجی ترمینال افشا می‌شوند:

```
TASK [Create secure .env file from vaulted secrets] ****************************
changed: [localhost] => {
    "changed": true,
    "invocation": {
        "module_args": {
            "content": "DB_ROOT_PASSWORD=SuperSecretPass2026!\nAPP_SECRET_KEY=9f8e7d6c5b4a\n",
            "dest": "/opt/devops_lab/.env",
            "mode": "0644"
        }
    }
}
```

علاوه بر این، در صورت نصب نبودن کالکشن داکر، خطای زیر رخ می‌دهد:

```
ERROR! couldn't resolve module/action 'community.docker.docker_container'
```

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

1. **افشای داده‌های حساس در لاگ اجرا:** انسیبل به صورت پیش‌فرض تمام آرگومان‌های ارسالی به ماژول‌ها (از جمله محتوای `content` در ماژول `copy`) را در خروجی ثبت می‌کند. در محیط‌های CI/CD این لاگ‌ها در تاریخچه پایپ‌لاین ذخیره شده و باعث نشت اعتبارنامه‌ها می‌شوند.  
2. **پرمیشن‌های بیش از حد باز (Over-permissive Files):** دسترسی پیش‌فرض `0644` اجازه می‌دهد هر کاربر غیرمجاز روی سیستم محتوای فایل `.env` را بخواند؛ این فایل باید به `0600` و مالکیت `root` محدود شود.  
3. **وابستگی به کالکشن‌های خارجی انسیبل:** از نسخه‌های جدید `ansible-core`، ماژول‌های مدیریت داکر در کالکشن مستقل `community.docker` قرار دارند و پیش از اجرا باید نصب شوند.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

### گام ۱: نصب پیش‌نیاز کالکشن داکر و ساخت فایل رمزنگاری‌شده با Ansible Vault

```sh
# نصب کالکشن رسمی داکر در انسیبل
ansible-galaxy collection install community.docker

# ساخت فایل رمزنگاری‌شده با الگوریتم AES-256
ansible-vault create secrets.yml
```

محتوای داخل `secrets.yml`:

```
---
db_root_password: "SuperSecretDBPassword_2026!"
app_secret_key: "a94f82c1e7b04d3981f2c6a5e4d8b701"
```

### گام ۲: نگارش پلی‌بوک `deploy_secure_stack.yml` با `no_log: true` و ماژول کانتینری

```
---
- name: Deploy Secrets Securely and Launch Redis Container
  hosts: local
  become: yes
  vars_files:
    - secrets.yml

  tasks:
    - name: Ensure /opt/devops_lab directory exists
      ansible.builtin.file:
        path: /opt/devops_lab
        state: directory
        owner: root
        group: root
        mode: '0750'

    - name: Generate restricted .env file from encrypted vault variables
      ansible.builtin.copy:
        dest: /opt/devops_lab/.env
        owner: root
        group: root
        mode: '0600'
        content: |
          DB_ROOT_PASSWORD={{ db_root_password }}
          APP_SECRET_KEY={{ app_secret_key }}
      no_log: true

    - name: Deploy Redis container with restart policy
      community.docker.docker_container:
        name: devops-redis-cache
        image: redis:alpine
        state: started
        restart_policy: always
        ports:
          - "127.0.0.1:6379:6379"
```

### گام ۳: اجرا همراه با رمز Vault و اعتبارسنجی امنیتی

```sh
# اجرای پلی‌بوک و دریافت رمز Vault به صورت تعاملی
ansible-playbook -i inventory.ini deploy_secure_stack.yml -K --ask-vault-pass

# تایید پرمیشن محدود 0600 روی فایل .env
sudo ls -l /opt/devops_lab/.env

# تایید اجرای کانتینر Redis با سیاست restart_policy
docker ps --filter "name=devops-redis-cache"
```

---

## ۶. درس‌آموخته مهندسی و اسکریپت بازگشت (Lesson Learned & Revert Script)

### درس‌آموخته مهندسی

* **دفاع چندلایه برای Secrets (Defense in Depth):** رمزنگاری فایل در مخزن گیت با `ansible-vault` به تنهایی کافی نیست؛ باید هم‌زمان در سطح اجرا از `no_log: true` برای پاکسازی لاگ‌ها و در سطح سیستم‌عامل از پرمیشن `0600` استفاده کرد.  
* **ایزوله‌سازی پورت کانتینرها:** بایند کردن پورت ردیس به `127.0.0.1:6379:6379` به جای `0.0.0.0` از دسترسی مستقیم شبکه خارجی به سرویس دیتابیس جلوگیری می‌کند.

### اسکریپت بازگشت (Revert Script)

```sh
#!/bin/bash
# revert_06.sh
docker rm -f devops-redis-cache 2>/dev/null || true
sudo rm -f /opt/devops_lab/.env
rm -rf ~/ansible-lab/06-vault-docker
echo "[OK] Scenario 06 container and secret files cleaned up."
```

