

```markdown
# 🛠️ آزمایشگاه جامع عیب‌یابی زیرساخت و دواپس (Incident & Troubleshooting Lab)

مجموعه سناریوهای واقعی و شبیه‌سازی‌شده از رخدادهای پروداکشن، گزارش‌های خرابی زیرساخت، ریشه‌یابی خطاها (Root Cause Analysis) و تدوین الگوهای مهندسی پایدار در لینوکس، شبکه، داکر و پایپ‌لاین‌های CI/CD.

---

## 🎯 چرا این مخزن ساخته شد؟
در مهندسی سیستم‌ها و DevOps، راه‌اندازی اولیه‌ی سرویس‌ها تنها بخش کوچکی از کار است؛ مهارت اصلی در زمان قطعی سرویس، بن‌بست‌های دیتابیس، نشت منابع دیسک و شکست فرآیندهای استقرار نمایان می‌شود. 

تمام سناریوهای ثبت‌شده در این ریپازیتوری از ساختار مهندسی ۶ مرحله‌ای پیروی می‌کنند:
1. **صورت مسئله و سناریو (Ticket):** شرح دقیق خرابی و اولویت Incident.
2. **اسکریپت شبیه‌سازی (Failure Injection):** اسکریپت آماده جهت بازتولید ایزوله‌ی خرابی روی سیستم آزمایشگاهی.
3. **علائم بالینی و لاگ‌ها (Symptoms):** رفتار خطا در مانیتورینگ و کدهای خطای ثبت‌شده.
4. **تحلیل علت ریشه‌ای (Root Cause):** کالبدشکافی فنی چرایی رخداد خطا در لایه سیستم‌عامل، کرنل، شبکه یا ابزار.
5. **مراحل برطرف‌سازی (Fix):** دستورات دقیق و اصلاحات زیرساختی گام‌به‌گام.
6. **درس‌آموخته مهندسی (Lesson Learned):** بهترین الگوهای معماری جهت پیشگیری از تکرار مجدد حادثه در پروداکشن.

---

## 📂 ساختار کلی و ماژول‌های فنی

```text
├── 01-linux-system-troubleshooting/   # حوادث سیستم‌عامل، کرنل، اینود و لاگ‌ها
├── 02-networking-and-web/              # ارتباطات شبکه، فایروال، وب‌سرور و DNS
├── 03-docker-containers/               # چرخه حیات کانتینرها، شبکه و کش بیلد
└── 04-cicd-and-automation/             # گلوگاه‌های استقرار، پایپ‌لاین و امنیت DevSecOps

```

---

## 📋 فهرست سناریوهای بررسی‌شده

### 🐧 ماژول ۱: سیستم‌عامل و هسته لینوکس (Linux Troubleshooting)

* **[01-disk-full-dummy-fill.md](https://www.google.com/search?q=01-linux-system-troubleshooting/01-disk-full-dummy-fill.md&utm_source=gemini):** پر شدن ظرفیت دیسک ناشی از لاگ‌های حجیم و تحلیل با ابزارهای `df` و `du`.
* **[02-deleted-inode-leak.md](https://www.google.com/search?q=01-linux-system-troubleshooting/02-deleted-inode-leak.md&utm_source=gemini):** عدم آزادسازی فضای دیسک پس از دستور `rm` به‌علت باز ماندن File Descriptor در پروسه‌ها و استفاده از Truncate.
* **[03-inode-exhaustion.md](https://www.google.com/search?q=01-linux-system-troubleshooting/03-inode-exhaustion.md&utm_source=gemini):** خطای `No space left on device` بر اثر اتمام جدول Inodeها علی‌رغم وجود فضای دیسک و پاکسازی امن با `find -delete`.
* **[04-systemd-type-forking-mismatch.md](https://www.google.com/search?q=01-linux-system-troubleshooting/04-systemd-type-forking-mismatch.md&utm_source=gemini):** تایم‌اوت شدن و شکست سرویس در Systemd به‌علت عدم همخوانی نوع `Type=forking` با رفتار فورگراند پردازه.
* **[05-zombie-process-port-lock.md](https://www.google.com/search?q=01-linux-system-troubleshooting/05-zombie-process-port-lock.md&utm_source=gemini):** قفل شدن سوکت شبکه توسط پروسه‌های پس‌زمینه بدون نام و کشف عمیق با زنجیره `ss` و `lsof`.
* **[06-ulimit-open-files-exhaustion.md](https://www.google.com/search?q=01-linux-system-troubleshooting/06-ulimit-open-files-exhaustion.md&utm_source=gemini):** کرش اپلیکیشن‌های پرترافیک به‌علت محدودیت سقف File Descriptors (`Too many open files`) و افزایش آن در سطح کرنل و یونیت‌فایل.
* **[07-logrotate-permission-failure.md](https://www.google.com/search?q=01-linux-system-troubleshooting/07-logrotate-permission-failure.md&utm_source=gemini):** شکست چرخه فشرده‌سازی لاگ‌ها به‌خاطر دسترسی‌های ناامن و اشکال‌زدایی با حالت شبیه‌سازی `logrotate -d`.

---

### 🌐 ماژول ۲: شبکه، وب‌سرور و دسترسی (Networking & Web)

* **[01-nginx-403-permission-chmod.md](https://www.google.com/search?q=02-networking-and-web/01-nginx-403-permission-chmod.md&utm_source=gemini):** ریشه‌یابی خطای `403 Forbidden` در انجین‌ایکس ناشی از مجوزهای ناکافی کاربر `www-data` روی دایرکتوری‌های ریشه.
* **[02-port-80-python-http-server.md](https://www.google.com/search?q=02-networking-and-web/02-port-80-python-http-server.md&utm_source=gemini):** تداخل در اتصال به سوکت و اشغال بودن پورت‌های استاندارد وب توسط سرویس‌های سرگردان.
* **[03-ufw-firewall-blocking-web.md](https://www.google.com/search?q=02-networking-and-web/03-ufw-firewall-blocking-web.md&utm_source=gemini):** تایم‌اوت شدن درخواست‌های بیرونی سرور ناشی از پالیسی `DROP` فایروال و پیشگیری از قفل شدن دسترسی ادمین (Lockout).
* **[04-dns-resolution-failure.md](https://www.google.com/search?q=02-networking-and-web/04-dns-resolution-failure.md&utm_source=gemini):** تفکیک خطای قطعی فیزیکی از اختلالات لایه DNS و حل مشکل نیم‌سرورها در `/etc/resolv.conf`.
* **[05-nginx-syntax-error-reload.md](https://www.google.com/search?q=02-networking-and-web/05-nginx-syntax-error-reload.md&utm_source=gemini):** شکست در ری‌لود پیکربندی وب‌سرور و ضرورت گیت اعتبارسنجی سینتکس با `nginx -t` پیش از انتشار تغییرات.
* **[06-ssh-strictmodes-auth-failure.md](https://www.google.com/search?q=02-networking-and-web/06-ssh-strictmodes-auth-failure.md&utm_source=gemini):** رد احراز هویت کلید SSH به‌دلیل باز بودن بیش از حد مجوزهای دسترسی پوشه `.ssh` ناشی از قابلیت `StrictModes`.

---

### 🐳 ماژول ۳: ارکستراسیون و کانتینرهای داکر (Docker Containers)

* **[01-docker-network-isolation.md](https://www.google.com/search?q=03-docker-containers/01-docker-network-isolation.md&utm_source=gemini):** خطای عدم دسترسی و تفکیک نام کانتینرها در شبکه پیش‌فرض `bridge` و پیاده‌سازی Custom Network برای فعال‌سازی DNS داخلی.
* **[02-docker-port-8080-conflict.md](https://www.google.com/search?q=03-docker-containers/02-docker-port-8080-conflict.md&utm_source=gemini):** مهار خطای پورت فورواردینگ مکرر و استقرار صحیح با تمایز بین `docker start` و `docker run`.
* **[03-docker-crashloop-exit-0.md](https://www.google.com/search?q=03-docker-containers/03-docker-crashloop-exit-0.md&utm_source=gemini):** خاتمه ناگهانی کانتینرها به‌علت پایان یافتن پروسه `PID 1` و نحوه اجرای پایدار سرویس‌ها در پس‌زمینه.
* **[04-docker-bind-mount-permission.md](https://www.google.com/search?q=03-docker-containers/04-docker-bind-mount-permission.md&utm_source=gemini):** ناهماهنگی مجوز فایل‌های مونت‌شده هاست داخل کانتینر و خطاهای Permission Denied.
* **[05-docker-build-context-cache-key.md](https://www.google.com/search?q=03-docker-containers/05-docker-build-context-cache-key.md&utm_source=gemini):** شکست در فرآیند ساخت ایمیج به‌دلیل ارجاع دستور `COPY` به خارج از Build Context.
* **[06-docker-disk-space-dangling-prune.md](https://www.google.com/search?q=03-docker-containers/06-docker-disk-space-dangling-prune.md&utm_source=gemini):** انباشت Volumeهای معلق و لایه‌های بی‌نام، و روش‌های پاک‌سازی ایمن بدون آسیب زدن به کانتینرهای فعال.

---

### 🚀 ماژول ۴: اتوماسیون پایپ‌لاین و امنیت (CI/CD & Automation)

* **[01-artifact-not-shared-between-jobs.md](https://www.google.com/search?q=04-cicd-and-automation/01-artifact-not-shared-between-jobs.md&utm_source=gemini):** ایزوله بودن دیسک رانرهای Ephemeral در GitHub Actions و جابه‌جایی صحیح داده‌ها با Artifacts.
* **[02-test-failure-skips-report-upload.md](https://www.google.com/search?q=04-cicd-and-automation/02-test-failure-skips-report-upload.md&utm_source=gemini):** توقف کامل مراحل بعدی در صورت شکست تست‌ها و استخراج شواهد با شرط `if: always()`.
* **[03-cd-disk-leak-repeated-deploy.md](https://www.google.com/search?q=04-cicd-and-automation/03-cd-disk-leak-repeated-deploy.md&utm_source=gemini):** پر شدن فضای سرور استقرار مداوم ناشی از انباشت ایمیج‌های Dangling با هر بار Pull.
* **[04-cicd-build-time-caching-ops402.md](https://www.google.com/search?q=04-cicd-and-automation/04-cicd-build-time-caching-ops402.md&utm_source=gemini):** کاهش زمان بیلد از ۱۸ دقیقه با اصلاح ترتیب لایه‌های داکرفایل و کشینگ اکسترنال Buildx.
* **[05-cicd-rollback-tagging-smoketest-ops403.md](https://www.google.com/search?q=04-cicd-and-automation/05-cicd-rollback-tagging-smoketest-ops403.md&utm_source=gemini):** مقابله با Anti-pattern تگ `latest`، تگ‌گذاری با Git SHA، تست دود خودکار (Smoke Test) و مکانیزم رول‌بک سریع.
* **[06-cicd-runner-disk-cleanup-ops404.md](https://www.google.com/search?q=04-cicd-and-automation/06-cicd-runner-disk-cleanup-ops404.md&utm_source=gemini):** مدیریت چرخه پاک‌سازی هوشمند با فیلتر عمر لایه‌ها (`until=72h`) در رانرهای اختصاصی (Self-Hosted).
* **[07-cicd-image-vulnerability-scan-ops405.md](https://www.google.com/search?q=04-cicd-and-automation/07-cicd-image-vulnerability-scan-ops405.md&utm_source=gemini):** پیاده‌سازی Shift-Left Security در CI/CD با لینتر **Hadolint** و اسکنر آسیب‌پذیری **Trivy**.
* **[08-cicd-environment-promotion-gate-ops406.md](https://www.google.com/search?q=04-cicd-and-automation/08-cicd-environment-promotion-gate-ops406.md&utm_source=gemini):** پیاده‌سازی گیت تایید دستی (Manual Approval) و رعایت اصل Build Once, Deploy Everywhere در انتقال ایمیج بین استیج‌ها.
* **[09-cicd-migration-concurrency-lock-ops407.md](https://www.google.com/search?q=04-cicd-and-automation/09-cicd-migration-concurrency-lock-ops407.md&utm_source=gemini):** جلوگیری از Deadlock دیتابیس در زمان مرج‌های هم‌زمان با اعمال قفل انحصاری `concurrency`.
* **[10-cicd-bluegreen-zero-downtime-ops408.md](https://www.google.com/search?q=04-cicd-and-automation/10-cicd-bluegreen-zero-downtime-ops408.md&utm_source=gemini):** حذف قطعی‌های لحظه‌ای (502 Bad Gateway) از طریق سوئیچ بدون وقفه ترافیک با معماری Blue/Green.
* **[11-cicd-secrets-leak-buildkit-ops409.md](https://www.google.com/search?q=04-cicd-and-automation/11-cicd-secrets-leak-buildkit-ops409.md&utm_source=gemini):** جلوگیری از نشت توکن‌ها در تاریخچه لایه‌های داکر با BuildKit Secret Mounting و اسکنر خودکار Gitleaks.
* **[12-cicd-dockerhub-rate-limit-mirror-ops410.md](https://www.google.com/search?q=04-cicd-and-automation/12-cicd-dockerhub-rate-limit-mirror-ops410.md&utm_source=gemini):** عبور از خطای Rate Limit داکر هاب در زیرساخت‌های سازمانی با راه‌اندازی Pull-Through Cache رجیستری محلی.
* **[13-cicd-flaky-build-lockfile-ops411.md](https://www.google.com/search?q=04-cicd-and-automation/13-cicd-flaky-build-lockfile-ops411.md&utm_source=gemini):** ایجاد بیلد تکرارپذیر (Deterministic Build) با اجباری کردن Lockfileها و دستورات نصب تمیز (`npm ci`).
* **[14-cicd-test-isolation-sharding-ops413.md](https://www.google.com/search?q=04-cicd-and-automation/14-cicd-test-isolation-sharding-ops413.md&utm_source=gemini):** کوتاه کردن زمان تست‌های E2E و رفع آلودگی داده‌ها با تکنیک Test Sharding و ران‌تایم‌های سبک Ephemeral.

---

## 🛠️ نحوه استفاده و بازتولید سناریوها

تمام سناریوها به شکلی طراحی شده‌اند که روی ماشین مجازی محلی، سرور تستی یا محیط داکر قابل پیاده‌سازی و آزمایش باشند:

۱. مخزن را کلون کنید:

```bash
git clone [https://github.com/](https://github.com/)<your-username>/<repo-name>.git
cd <repo-name>

```

۲. وارد دایرکتوری سناریوی مورد نظر شوید و مستند مربوطه را مطالعه کنید:

```bash
cat 01-linux-system-troubleshooting/02-deleted-inode-leak.md

```

۳. اسکریپت شبیه‌سازی خرابی (Failure Injection) مندرج در مستند را اجرا کرده و شواهد را با ابزارهای معرفی‌شده عیب‌یابی کنید.

۴. پس از رفع ایراد، با اجرای اسکریپت بازگشت (Revert Script) پایان هر فایل، محیط سیستم را به حالت اولیه و پاک برگردانید.

```
