from pathlib import Path
import zipfile, shutil, textwrap, os

src = Path("/mnt/data/MarrirPASS_V5_1_BuildReady")
if not src.exists():
    raise FileNotFoundError("مجلد المشروع V5.1 غير موجود.")

# إنشاء نسخة مخصصة للرفع إلى GitHub
out = Path("/mnt/data/MarrirPASS_V5_1_GitHub_Ready")
if out.exists():
    shutil.rmtree(out)
shutil.copytree(src, out)

# GitHub Actions: بناء APK Debug تلقائياً
workflow = out / ".github/workflows/android-apk.yml"
workflow.parent.mkdir(parents=True, exist_ok=True)
workflow.write_text("""name: Build MarrirPASS APK

on:
  workflow_dispatch:
  push:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: gradle

      - name: Set up Android SDK
        uses: android-actions/setup-android@v3

      - name: Install Android SDK Platform 35
        run: |
          yes | sdkmanager --licenses > /dev/null || true
          sdkmanager "platform-tools" "platforms;android-35" "build-tools;35.0.0"

      - name: Build Debug APK
        run: |
          chmod +x gradlew
          ./gradlew :app:assembleDebug

      - name: Rename APK
        run: |
          cp app/build/outputs/apk/debug/app-debug.apk MarrirPASS-v5.1-debug.apk

      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: MarrirPASS-v5.1-debug
          path: MarrirPASS-v5.1-debug.apk
""", encoding="utf-8")

# تعليمات GitHub مبسطة
(out / "GITHUB_BUILD.md").write_text("""# MarrirPASS V5.1 — GitHub APK Build

## رفع المشروع
1. أنشئ مستودعاً جديداً على GitHub.
2. ارفع جميع ملفات هذا المجلد إلى المستودع.
3. تأكد من وجود `.github/workflows/android-apk.yml`.
4. افتح تبويب **Actions**.
5. اختر **Build MarrirPASS APK**.
6. اضغط **Run workflow** إذا لم يبدأ البناء تلقائياً.
7. بعد نجاح البناء افتح نتيجة الـWorkflow.
8. من قسم **Artifacts** نزّل `MarrirPASS-v5.1-debug`.

## ملاحظة
الـAPK Debug تجريبي. التحقق والتصاريح في النسخة الحالية محلية وليست خدمة حكومية فعلية.
""", encoding="utf-8")

# README يوضح أن المشروع GitHub-ready
with (out / "README.md").open("a", encoding="utf-8") as f:
    f.write("""

## GitHub Actions
هذا الإصدار يحتوي على Workflow تلقائي لبناء APK Debug على GitHub.
الناتج: `MarrirPASS-v5.1-debug.apk`
""")

# إنشاء ZIP النهائي
zip_path = Path("/mnt/data/MarrirPASS_V5_1_GitHub_Ready.zip")
if zip_path.exists():
    zip_path.unlink()

with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for f in out.rglob("*"):
        if f.is_file():
            z.write(f, f.relative_to(out))

print(f"تم إنشاء ZIP الجاهز لـ GitHub: {zip_path}")
print("تمت إضافة GitHub Actions لبناء APK Debug تلقائياً.")
