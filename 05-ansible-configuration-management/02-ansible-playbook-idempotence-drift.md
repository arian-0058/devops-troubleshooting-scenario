# سناریو ۰۲: انحراف پیکربندی (Configuration Drift) و نقض اصل تکرارپذیری در Playbookها (Idempotence)

---

## ۱. صورت مسئله و تیکت پروداکشن (Problem Statement & Production Ticket)

* **شناسه رخداد (Incident ID):** `INC-ANS-002`  
* **سطح اولویت (Incident Severity):** `Medium (P3)`  
* **محیط عملیاتی:** Ubuntu 22.04 LTS  
* **شرح تیکت:** اسکریپت‌های شل قدیمی تیم زیرساخت برای نصب ابزارهای پایش پایه (`curl`، `htop`، `tree`) و ایجاد دایرکتوری‌های عملیاتی (`/opt/devops_lab`) در هر بار اجرا، فایل‌ها را بدون بررسی وضعیت فعلی بازنویسی کرده و باعث تغییر Timestamp فایل‌ها و ایجاد تغییرات ناخواسته در محیط پروداکشن می‌شوند. نیاز است این فرآیند با نوشتن یک Playbook استاندارد و رعایت کامل اصل **Idempotence** بازطراحی شود.

---

## ۲. اسکریپت شبیه‌سازی خرابی (Failure Injection)

اسکریپت زیر وضعیت دستکاری‌شده و غیراستاندارد سرور را شبیه‌سازی می‌کند:

```sh
#!/bin/bash
# failure_inject_02.sh
mkdir -p ~/ansible-lab/02-playbook
cd ~/ansible-lab/02-playbook || exit 1

cat << 'EOF' > inventory.ini
[local]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
EOF

# پاکسازی محیط برای شبیه‌سازی سرور خام بدون ابزارها و دایرکتوری lab
sudo rm -rf /opt/devops_lab
sudo apt-get remove -y tree >/dev/null 2>&1

echo "[INFO] Server drifted to unprovisioned state."
```

---

## ۳. علائم بالینی و لاگ‌ها (Symptoms & Logs)

در صورت اجرای اسکریپت‌های غیرتکرارپذیر یا اجرای پلی‌بوک بدون سطح دسترسی `become: yes`، خطاها و رفتارهای زیر مشاهده می‌شود:

```
TASK [Update apt cache and install curl, htop, and tree] ***********************
fatal: [localhost]: FAILED! => {
    "changed": false,
    "msg": "Failed to lock apt for exclusive operation: Failed to lock directory /var/lib/apt/lists/: E:Could not open lock file /var/lib/apt/lists/lock - open (13: Permission denied)"
}
```

همچنین در اجرای مجدد اسکریپت‌های سنتی، خروجی همچنان وضعیت تغییر (`changed > 0`) را گزارش می‌دهد.

---

## ۴. تحلیل علت ریشه‌ای (Root Cause Analysis \- RCA)

1. **عدم ارتقای سطح دسترسی (Privilege Escalation):** تغییرات سطح سیستم‌عامل (نصب پکیج با `apt` و نوشتن در مسیر `/opt`) نیازمند دسترسی `root` است که در انسیبل باید صراحتاً با `become: yes` و دریافت رمز `sudo` (`-K`) فعال شود.  
2. **تفاوت رویکرد Imperative با Declarative:** اسکریپت‌های Bash دستورات را به صورت دستوری اجرا می‌کنند، در حالی که ماژول‌های بومی انسیبل (`apt`، `file`، `copy`) وضعیت مطلوب (Desired State) را بررسی کرده و تنها در صورت مغایرت، تغییر ایجاد می‌کنند.

---

## ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

### گام ۱: نگارش پلی‌بوک استاندارد `setup_tools.yml`

```
---
- name: Setup basic DevOps tools and lab directory
  hosts: local
  become: yes

  tasks:
    - name: Update apt cache and install curl, htop, and tree
      ansible.builtin.apt:
        name:
          - curl
          - htop
          - tree
        state: present
        update_cache: yes
        cache_valid_time: 3600

    - name: Create /opt/devops_lab directory
      ansible.builtin.file:
        path: /opt/devops_lab
        state: directory
        owner: root
        group: root
        mode: '0755'

    - name: Create info file inside devops_lab
      ansible.builtin.copy:
        content: "Provisioned by Ansible\n"
        dest: /opt/devops_lab/info.txt
        owner: root
        group: root
        mode: '0644'
```

### گام ۲: اجرا و اثبات عملی Idempotence

```sh
# اجرای اول: اعمال تغییرات روی سرور خام (خروجی باید changed=3 باشد)
ansible-playbook -i inventory.ini setup_tools.yml -K

# اجرای دوم بلافاصله پس از اجرای اول: اثبات Idempotence (خروجی باید changed=0 باشد)
ansible-playbook -i inventory.ini setup_tools.yml -K
```

**خروجی تایید در اجرای دوم:**

```
PLAY RECAP *********************************************************************
localhost                  : ok=4    changed=0    unreachable=0    failed=0
```

---

## ۶. درس‌آموخته مهندسی و اسکریپت بازگشت (Lesson Learned & Revert Script)

### درس‌آموخته مهندسی

* **اصل طلایی Idempotence:** هر پلی‌بوک پروداکشنی باید به گونه‌ای نوشته شود که اجرای ۱۰ باره‌ی آن دقیقاً همان نتیجه اجرای اول را داشته باشد و در اجراهای بعدی `changed=0` گزارش دهد.  
* استفاده از پارامتر `cache_valid_time: 3600` در ماژول `apt` از آپدیت غیرضروری کش مخازن در اجراهای متوالی جلوگیری کرده و Idempotence کامل را حفظ می‌کند.

### اسکریپت بازگشت (Revert Script)

```sh
#!/bin/bash
# revert_02.sh
sudo rm -rf /opt/devops_lab
rm -rf ~/ansible-lab/02-playbook
echo "[OK] Scenario 02 artifacts removed."
```

