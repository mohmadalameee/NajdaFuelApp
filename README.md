name: Build APK

on:
  push:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Locate or extract project
        id: locate
        run: |
          if [ -f "FuelDistributionApp.zip" ]; then
            echo "وجدت ملف مضغوط - يتم فك الضغط..."
            unzip -o FuelDistributionApp.zip -d extracted
            PROJECT_DIR=$(find extracted -name "build.gradle" -not -path "*/build/*" | head -1 | xargs dirname)
            if [ -z "$PROJECT_DIR" ]; then
              echo "تعذر العثور على build.gradle داخل الملف المضغوط!"
              exit 1
            fi
            echo "project_dir=$PROJECT_DIR" >> "$GITHUB_OUTPUT"
            echo "تم العثور على المشروع في: $PROJECT_DIR"
          elif [ -f "build.gradle" ] && [ -d "app" ]; then
            echo "المشروع موجود بدون ضغط في جذر المستودع"
            echo "project_dir=." >> "$GITHUB_OUTPUT"
          elif [ -f "FuelDistributionApp/build.gradle" ]; then
            echo "المشروع موجود داخل مجلد FuelDistributionApp"
            echo "project_dir=FuelDistributionApp" >> "$GITHUB_OUTPUT"
          else
            echo "تعذر العثور على المشروع! تأكد من رفع FuelDistributionApp.zip أو محتويات المشروع في جذر المستودع."
            exit 1
          fi

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Set up Android SDK
        uses: android-actions/setup-android@v3

      - name: Install Android SDK components
        run: |
          sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0"

      - name: Grant execute permission for gradlew
        working-directory: ${{ steps.locate.outputs.project_dir }}
        run: |
          if [ -f "gradlew" ]; then
            chmod +x gradlew
            echo "gradlew موجود وتم منحه صلاحية التنفيذ"
          else
            echo "تحذير: gradlew غير موجود! سيتم استخدام gradle النظامي."
          fi

      - name: Cache Gradle dependencies
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/build.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-

      - name: Build debug APK
        working-directory: ${{ steps.locate.outputs.project_dir }}
        run: |
          if [ -f "gradlew" ]; then
            ./gradlew assembleDebug --stacktrace
          else
            gradle assembleDebug --stacktrace
          fi

      - name: Upload APK as artifact
        uses: actions/upload-artifact@v4
        with:
          name: FuelDistributionApp-debug-apk
          path: ${{ steps.locate.outputs.project_dir }}/app/build/outputs/apk/debug/app-debug.apk
          if-no-files-found: warn
          
