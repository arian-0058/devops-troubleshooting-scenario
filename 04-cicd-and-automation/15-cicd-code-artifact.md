# سناریوی 15: تصادم نام آرتیفکت‌ها و Race Condition در استراتژی ماتریسی (Matrix Jobs)

* **حوزه تخصصی:** CI/CD Automation & GitHub Actions Pipelines

* **محیط آزمایشگاهی:** GitHub Actions Workflow Engine

* **سطح اولویت رخداد (Incident Severity):** High (P2 - Silent Artifact Corruption / Incomplete Release)

### ۱. صورت مسئله و تیکت پروداکشن (Incident Report)

در یک پایپ‌لاین چندنسخه‌ای، مرحله بیلد پکیج روی ۳ نسخه پایتون (`3.10`، `3.11` و `3.12`) به صورت ماتریسی اجرا می‌شود. هر جاب آرتیفکت پکیج ساخته‌شده خود را آپلود می‌کند. در جاب نهاییِ انتشار (`deploy-package`)، خروجی پکیج‌ها ناقص است و هر بار تنها پکیج آخرین ورژنی که تصادفاً کارش تمام شده در مسیر دایرکتوری قرار دارد و سایر نسخه‌ها حذف شده‌اند.

### ۲. ورک‌فلو ناقص با باگ معماری (Flawed Workflow)

```
name: Multi-Version Build & Release

on:
  push:
    branches: [ main ]

jobs:
  build-matrix:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Build Wheel Package
        run: |
          mkdir -p dist
          python -m pip install --upgrade build
          python -m build --outdir dist/

      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          # باگ: تمام جاب‌های همزمان با یک کلید نام اقدام به آپلود می‌کنند
          name: app-dist-package
          path: dist/

  deploy-package:
    needs: build-matrix
    runs-on: ubuntu-latest
    steps:
      - name: Download Artifacts
        uses: actions/download-artifact@v4
        with:
          name: app-dist-package
          path: final-release/

      - name: Validate Release Files
        run: |
          ls -la final-release/
          # تست همواره به دلیل نبود هر سه فایل با شکست مواجه می‌شود

```

### ۳. علائم بالینی و لاگ‌های خطا (Symptoms & Failure Logs)

در لاگ‌های اجرای مرحله `Upload Artifact` روی رانرهای موازی، هشدارهای همزمانی و بازنویسی مشاهده می‌شود:

```
Warning: Artifact "app-dist-package" already exists. An artifact with this name has already been uploaded by another job.
Error: Artifact upload failed or overwritten due to static artifact key conflicts across parallel runners.

```

در مرحله `Validate Release Files` نیز صرفاً یک نسخه پکیج وجود دارد و اعتبارسنجی پکیج‌های چندنسخه‌ای مردود اعلام می‌شود.

### ۴. تحلیل علت ریشه‌ای (Root Cause Analysis - RCA)

1. جاب‌های ماتریسی (`strategy: matrix`) روی رانرهای مجزا و کاملاً همزمان (Parallel Execution) اجرا می‌شوند.

2. استفاده از مقدار ثابت برای کلید ذخیره‌سازی (`name: app-dist-package`) در اکشن `actions/upload-artifact@v4` موجب وقوع **Race Condition** در لایه ذخیره‌سازی ابری گیت‌هاب می‌شود.

3. در اکشن‌های مدرن CI/CD، آپلود با نام یکسان یا منجر به شکست بیلد شده یا در صورت مجاز بودن بازنویسی، آرتیفکت رانری که دیرتر تمام شود جایگزین فایل‌های قبلی می‌شود (Silent Overwrite).

### ۵. مراحل اصلاح و رفع ایراد (Remediation & Fix)

پایپ‌لاین به نحوی بازطراحی شد که کلید ذخیره‌سازی با متغیر محیطی ماتریس ایزوله شود و در مرحله دانلود، با الگوی تطبیقی (`pattern`) تجمیع گردد:

```
name: Multi-Version Build & Release

on:
  push:
    branches: [ main ]

jobs:
  build-matrix:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Build Wheel Package
        run: |
          mkdir -p dist
          python -m pip install --upgrade build
          python -m build --outdir dist/

      - name: Upload Dynamic Matrix Artifact
        uses: actions/upload-artifact@v4
        with:
          # تفکیک کلید بر اساس نگارش ماتریس
          name: app-dist-py${{ matrix.python-version }}
          path: dist/
          retention-days: 1

  deploy-package:
    needs: build-matrix
    runs-on: ubuntu-latest
    steps:
      - name: Download All Release Artifacts
        uses: actions/download-artifact@v4
        with:
          # تجمیع تمام بسته‌ها با الگوی تطبیقی بدون تداخل نام
          pattern: app-dist-py*
          path: final-release/

      - name: Validate Release Files
        run: |
          echo "Listing all acquired packages:"
          ls -R final-release/

```

### ۶. درس‌آموخته مهندسی (Lesson Learned)

* **ایزوله‌سازی کانتکست در جاب‌های موازی:** در هرگونه استراتژی ماتریسی، تمام منابع اشتراکی بیرونی (شامل کلیدهای آرتیفکت، نام کانتینرهای تست، و تگ‌های ایمیج) باید از متغیرهای ترکیب ماتریس (`matrix context`) بهره ببرند.

* **مکانیزم چندبخشی در اکشن‌های مدرن:** نسخه چهارم اکشن‌های گیت‌هاب (`upload-artifact@v4` و `download-artifact@v4`) از رفتار ادغام ضمنی پشتیبانی نمی‌کنند؛ بهترین راهکار تفکیک صریح با نام‌های پویا و سپس دانلود با `pattern` جهت تجمیع ساختار درختی پکیج‌هاست.