from pathlib import Path
import zipfile, textwrap

src = Path("/mnt/data/MarrirPASS_V5_1_GitHub_Ready.zip")
work = Path("/mnt/data/MarrirPASS_V5_1_FINAL")
work.mkdir(parents=True, exist_ok=True)

with zipfile.ZipFile(src, "r") as z:
    z.extractall(work)

workflow = work / ".github" / "workflows" / "android-apk.yml"
workflow.parent.mkdir(parents=True, exist_ok=True)
workflow.write_text(textwrap.dedent("""\
name: Build MarrirPASS APK

on:
  workflow_dispatch:
  push:
    branches: ["main", "master"]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - name: Set up Gradle 8.10.2
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.10.2"

      - name: Set up Android SDK
        uses: android-actions/setup-android@v3

      - name: Install Android SDK 35
        run: |
          yes | sdkmanager --licenses >/dev/null || true
          sdkmanager "platform-tools" "platforms;android-35" "build-tools;35.0.0"

      - name: Build Debug APK
        run: gradle :app:assembleDebug --stacktrace

      - name: Prepare APK
        run: |
          mkdir -p release
          cp app/build/outputs/apk/debug/app-debug.apk release/MarrirPASS-v5.1-debug.apk

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: MarrirPASS-v5.1-debug
          path: release/MarrirPASS-v5.1-debug.apk
          if-no-files-found: error
"""), encoding="utf-8")

(work / "GITHUB_BUILD.md").write_text(textwrap.dedent("""\
# بناء تطبيق مرّر PASS عبر GitHub Actions

1. ارفع محتويات هذا المشروع إلى مستودع GitHub.
2. افتح تبويب Actions.
3. اختر Build MarrirPASS APK.
4. اضغط Run workflow.
5. انتظر اكتمال البناء.
6. افتح نتيجة التشغيل ثم Artifacts.
7. نزّل MarrirPASS-v5.1-debug.
8. فك الضغط وثبّت APK على هاتف Android.

ملاحظة: هذا الإصدار MVP/تجريبي. بيانات التصاريح محلية، وQR الحالي للعرض والتجربة فقط. قبل الاستخدام الحكومي الفعلي يجب إضافة خادم موثوق، قاعدة بيانات، صلاحيات وأدوار، توقيع رقمي، إبطال التصاريح، وسجل تدقيق.
"""), encoding="utf-8")

out = Path("/mnt/data/MarrirPASS_V5_1_FINAL_GitHub.zip")
if out.exists():
    out.unlink()

with zipfile.ZipFile(out, "w", zipfile.ZIP_DEFLATED) as z:
    for p in work.rglob("*"):
        if p.is_file():
            z.write(p, p.relative_to(work))

print(f"تم تجهيز النسخة النهائية: {out}")
print("تم إصلاح Workflow ليستخدم Gradle 8.10.2 مباشرة، دون الحاجة إلى gradlew داخل المشروع.")
