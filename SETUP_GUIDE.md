# React Native Android App — Complete Setup Guide

End-to-end reference for creating, building, and packaging a React Native Android app on macOS — from a clean machine to a signed AAB ready for Google Play Store.

**Project:** HelloWorldPlaystoreTest
**Platform:** macOS (Apple Silicon, Darwin 25.4.0)
**Final output:** Signed `.aab` file uploadable to Google Play Store

---

## Table of Contents

1. [Tools & Dependencies](#1-tools--dependencies)
2. [Environment Setup](#2-environment-setup)
3. [Create the React Native Project](#3-create-the-react-native-project)
4. [Convert to JavaScript (Remove TypeScript)](#4-convert-to-javascript-remove-typescript)
5. [Configure Android Build](#5-configure-android-build)
6. [Replace Default UI](#6-replace-default-ui)
7. [Run on Android Emulator](#7-run-on-android-emulator)
8. [Generate Release Keystore](#8-generate-release-keystore)
9. [Configure Release Signing](#9-configure-release-signing)
10. [Build the AAB](#10-build-the-aab)
11. [Test the AAB Locally](#11-test-the-aab-locally)
12. [Reference: Common Commands](#12-reference-common-commands)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Tools & Dependencies

### What Was Installed

| Tool | Version | Purpose | Install Method |
|---|---|---|---|
| **Node.js** | v24.13.1 | JavaScript runtime for React Native | Pre-installed |
| **npm** | 11.8.0 | Package manager | Pre-installed with Node |
| **Homebrew** | latest | macOS package manager | Pre-installed |
| **OpenJDK 17** | 17.0.19 | Java runtime for Gradle/Android build | `brew install openjdk@17` |
| **Android Command-Line Tools** | 14742923 | Android SDK manager | `brew install --cask android-commandlinetools` |
| **Android SDK Platform-Tools** | 37.0.0 | adb, fastboot | `sdkmanager` |
| **Android SDK Platform 34** | rev 3 | Android API 34 | `sdkmanager` |
| **Android SDK Platform 36** | rev 2 | Android API 36 (auto-installed) | Auto by Gradle |
| **Android Build-Tools** | 34.0.0, 35.0.0, 36.0.0 | Build APK/AAB | Auto by Gradle |
| **Android NDK** | 27.1.12297006 | Native code compilation | Auto by Gradle |
| **Android Emulator System Image** | android-34, arm64-v8a | Pixel 6 emulator | `sdkmanager` |
| **bundletool** | 1.18.3 | Convert AAB to APKs for testing | `brew install bundletool` |
| **React Native CLI** | 20.1.3 | Project scaffolding | `npx @react-native-community/cli` |
| **React Native** | 0.85.2 | Mobile framework | npm dependency |
| **Gradle** | 9.3.1 | Build system (auto-downloaded) | Auto by RN template |

### Project Dependencies (npm)

```json
{
  "dependencies": {
    "react": "19.2.3",
    "react-native": "0.85.2",
    "@react-native/new-app-screen": "0.85.2",
    "react-native-safe-area-context": "^5.5.2"
  },
  "devDependencies": {
    "@babel/core": "^7.25.2",
    "@babel/preset-env": "^7.25.3",
    "@babel/runtime": "^7.25.0",
    "@react-native-community/cli": "20.1.0",
    "@react-native-community/cli-platform-android": "20.1.0",
    "@react-native/babel-preset": "0.85.2",
    "@react-native/eslint-config": "0.85.2",
    "@react-native/jest-preset": "0.85.2",
    "@react-native/metro-config": "0.85.2",
    "eslint": "^8.19.0",
    "jest": "^29.6.3",
    "prettier": "2.8.8",
    "react-test-renderer": "19.2.3"
  }
}
```

---

## 2. Environment Setup

### Step 2.1 — Install JDK 17

```bash
brew install openjdk@17
```

> Avoid `brew install --cask temurin@17` — the cask requires `sudo` which doesn't work in non-interactive sessions. The formula version is keg-only but works without sudo.

### Step 2.2 — Install Android Command-Line Tools

```bash
brew install --cask android-commandlinetools
```

This installs to: `/opt/homebrew/share/android-commandlinetools/`

### Step 2.3 — Set Environment Variables (Permanent)

Add to `~/.zshrc`:

```bash
echo 'export JAVA_HOME="/opt/homebrew/opt/openjdk@17"' >> ~/.zshrc
echo 'export ANDROID_HOME="/opt/homebrew/share/android-commandlinetools"' >> ~/.zshrc
echo 'export ANDROID_SDK_ROOT="$ANDROID_HOME"' >> ~/.zshrc
echo 'export PATH="$JAVA_HOME/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Step 2.4 — Verify Installation

```bash
java -version          # Should show: openjdk 17.0.19
node --version         # Should show: v24.13.1 or similar
adb --version          # Should show: Android Debug Bridge version
```

### Step 2.5 — Install Android SDK Components

```bash
# Accept licenses
yes | sdkmanager --licenses

# Install required packages
sdkmanager "platform-tools" \
           "platforms;android-34" \
           "build-tools;34.0.0" \
           "system-images;android-34;google_apis;arm64-v8a"
```

### Step 2.6 — Create Android Virtual Device (Emulator)

```bash
avdmanager create avd \
  -n "Pixel_6_API_34" \
  -k "system-images;android-34;google_apis;arm64-v8a" \
  -d "pixel_6"
```

---

## 3. Create the React Native Project

### Step 3.1 — Initialize Project

```bash
cd /Users/mohammedfajlulkarim/projects/HelloWorld_AndroidApp

npx @react-native-community/cli@latest init PlaystoreTestApp
```

> **Note:** The CLI rejects "HelloWorld" in project names because it's the default placeholder. We use `PlaystoreTestApp` as the technical name and set the display name to `HelloWorldPlaystoreTest` in `app.json`.

When prompted "Do you want to install CocoaPods now?", answer **N** (Android-only project).

---

## 4. Convert to JavaScript (Remove TypeScript)

The default RN template ships with TypeScript. Since we want plain JavaScript:

### Step 4.1 — Delete TypeScript Files

```bash
cd PlaystoreTestApp
rm -f App.tsx tsconfig.json Gemfile
```

### Step 4.2 — Rename Test File

```bash
mv __tests__/App.test.tsx __tests__/App.test.js
```

### Step 4.3 — Update `package.json`

Remove these `devDependencies`:
- `@react-native/typescript-config`
- `@types/jest`
- `@types/react`
- `@types/react-test-renderer`
- `typescript`

Then run:

```bash
npm install
```

---

## 5. Configure Android Build

### Step 5.1 — Update `android/build.gradle`

Change `minSdkVersion` to **24** (React Native 0.85.x requires API 24+):

```groovy
buildscript {
    ext {
        buildToolsVersion = "36.0.0"
        minSdkVersion = 24       // ← was 24, kept at 24 (RN 0.85 requirement)
        compileSdkVersion = 36
        targetSdkVersion = 36
        ndkVersion = "27.1.12297006"
        kotlinVersion = "2.1.20"
    }
    ...
}
```

### Step 5.2 — Create `android/local.properties`

```properties
sdk.dir=/opt/homebrew/share/android-commandlinetools
```

### Step 5.3 — Update `app.json`

```json
{
  "name": "PlaystoreTestApp",
  "displayName": "HelloWorldPlaystoreTest"
}
```

---

## 6. Replace Default UI

### Step 6.1 — Create `App.js`

```javascript
import React from 'react';
import {StyleSheet, Text, View} from 'react-native';

export default function App() {
  return (
    <View style={styles.container}>
      <Text style={styles.text}>Hello World - Play Store Test</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#ffffff',
  },
  text: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#000000',
  },
});
```

`index.js` automatically resolves `import App from './App'` to `App.js`.

---

## 7. Run on Android Emulator

### Step 7.1 — Start the Emulator

```bash
emulator -avd Pixel_6_API_34 -no-snapshot-load -no-audio &

# Wait for full boot
adb wait-for-device shell 'until [ "$(getprop sys.boot_completed)" = "1" ]; do sleep 2; done; echo Booted'
```

### Step 7.2 — Build & Install Debug Build

```bash
cd PlaystoreTestApp
npx react-native run-android
```

> First build takes **~10–15 minutes** because Gradle downloads:
> - NDK (~2.4 GB)
> - Build-Tools 35 & 36
> - SDK Platform 36
> Subsequent builds take **~1–2 minutes**.

### Step 7.3 — Start Metro (in separate terminal)

```bash
cd PlaystoreTestApp
npx react-native start --port 8081
```

### Step 7.4 — Connect App to Metro

```bash
adb -s emulator-5554 reverse tcp:8081 tcp:8081
adb -s emulator-5554 shell am start -n com.playstoretestapp/.MainActivity
```

The app will show "Bundling X%..." then display the Hello World screen.

---

## 8. Generate Release Keystore

### Step 8.1 — Create Keystore

```bash
cd /Users/mohammedfajlulkarim/projects/HelloWorld_AndroidApp/PlaystoreTestApp

keytool -genkey -v \
  -keystore playstore-release-key.jks \
  -alias playstore-key \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000
```

When prompted:
- **Keystore password**: `Android` (chosen for this project)
- **Key password**: `Android` (same as keystore)
- **Name/Org/City/Country**: Fill in or skip

### Step 8.2 — CRITICAL: Backup the Keystore

⚠️ **Never lose this file or its passwords!**
- Without it, you cannot publish updates to the same app on Play Store
- Back up to: external drive, password manager, encrypted cloud storage

### Step 8.3 — Add Keystore to `.gitignore`

```bash
echo "*.jks" >> .gitignore
echo "*.keystore" >> .gitignore
```

> The `.jks` file should **NEVER** be committed to git.

---

## 9. Configure Release Signing

### Step 9.1 — Update `android/app/build.gradle`

Find the `signingConfigs` block and add a `release` config. Update the `release` build type to use it:

```groovy
android {
    ...
    signingConfigs {
        debug {
            storeFile file('debug.keystore')
            storePassword 'android'
            keyAlias 'androiddebugkey'
            keyPassword 'android'
        }
        release {                                                 // ← ADD THIS
            storeFile file('../../playstore-release-key.jks')
            storePassword 'Android'
            keyAlias 'playstore-key'
            keyPassword 'Android'
        }
    }
    buildTypes {
        debug {
            signingConfig signingConfigs.debug
        }
        release {
            signingConfig signingConfigs.release                  // ← CHANGE from .debug
            minifyEnabled enableProguardInReleaseBuilds
            proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
        }
    }
}
```

> Path `../../playstore-release-key.jks` resolves from `android/app/` → up to project root.

---

## 10. Build the AAB

### Step 10.1 — Build Release Bundle

```bash
cd PlaystoreTestApp/android
./gradlew bundleRelease
```

### Step 10.2 — Locate the AAB

The AAB is generated at:

```
PlaystoreTestApp/android/app/build/outputs/bundle/release/app-release.aab
```

### Step 10.3 — Verify the AAB (optional)

```bash
$ANDROID_HOME/cmdline-tools/latest/bin/apkanalyzer \
  manifest print \
  android/app/build/outputs/bundle/release/app-release.aab
```

You should see:
- `package="com.playstoretestapp"`
- `versionCode="1"`
- `versionName="1.0"`

---

## 11. Test the AAB Locally

AABs cannot be installed directly. Use `bundletool` to convert it to APKs:

### Step 11.1 — Install bundletool

```bash
brew install bundletool
```

### Step 11.2 — Convert AAB to APK Set

```bash
cd PlaystoreTestApp

bundletool build-apks \
  --bundle=android/app/build/outputs/bundle/release/app-release.aab \
  --output=/tmp/app.apks \
  --ks=playstore-release-key.jks \
  --ks-pass=pass:Android \
  --ks-key-alias=playstore-key \
  --key-pass=pass:Android
```

### Step 11.3 — Uninstall Debug Version First

```bash
adb uninstall com.playstoretestapp
```

> Required because Android blocks installing an app with a different signature over an existing one.

### Step 11.4 — Install Release APKs

```bash
bundletool install-apks --apks=/tmp/app.apks
```

### Step 11.5 — Launch the App

```bash
adb -s emulator-5554 shell am start -n com.playstoretestapp/.MainActivity
```

For the release build to display content, you need either:
- Metro running + reverse port (development), OR
- The JS bundle baked into the APK (production/release default)

The release build automatically bundles JS during build, so it works **without** Metro.

---

## 12. Reference: Common Commands

### Environment

```bash
# Check Java
java -version

# Check Android SDK
sdkmanager --list_installed

# Check connected devices
adb devices
```

### Emulator

```bash
# List AVDs
avdmanager list avd

# Start emulator
emulator -avd Pixel_6_API_34 -no-snapshot-load -no-audio &

# Stop emulator
adb -s emulator-5554 emu kill
```

### Build

```bash
cd PlaystoreTestApp/android

# Debug APK
./gradlew assembleDebug

# Release APK (signed)
./gradlew assembleRelease

# Release AAB (signed)
./gradlew bundleRelease

# Clean build artifacts
./gradlew clean
```

### Output Locations

| Build | Output Path |
|---|---|
| Debug APK | `android/app/build/outputs/apk/debug/app-debug.apk` |
| Release APK | `android/app/build/outputs/apk/release/app-release.apk` |
| Release AAB | `android/app/build/outputs/bundle/release/app-release.aab` |

### Metro

```bash
# Start dev server
npx react-native start --port 8081

# Reset cache
npx react-native start --reset-cache
```

### App Management

```bash
# Install APK
adb install app-debug.apk

# Uninstall app
adb uninstall com.playstoretestapp

# Launch app
adb shell am start -n com.playstoretestapp/.MainActivity

# Force-stop app
adb shell am force-stop com.playstoretestapp

# View app logs
adb logcat | grep -i "playstoretestapp\|reactnative"
```

---

## 13. Troubleshooting

| Problem | Solution |
|---|---|
| `java: command not found` | `export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"` |
| `adb: command not found` | `export PATH="$ANDROID_HOME/platform-tools:$PATH"` |
| `sdk.dir not found` | Create `android/local.properties` with `sdk.dir=...` |
| `minSdkVersion 21 < 24` error | Set `minSdkVersion = 24` in `android/build.gradle` |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | `adb uninstall com.playstoretestapp` then reinstall |
| `bundletool: Unable to determine ADB location` | Set `ANDROID_HOME` env var or use `--adb=...` flag |
| `bundletool: No connected devices` | Start emulator first with `emulator -avd Pixel_6_API_34 &` |
| Metro can't open new terminal | Run `npx react-native start` manually in separate terminal |
| App stuck on "Bundling 40%" | Ensure `adb reverse tcp:8081 tcp:8081` was set up |
| System UI ANR on emulator | `adb shell am crash com.android.systemui` to restart UI |
| First build very slow (15 min) | Normal — Gradle downloads NDK (2.4 GB) and SDK packages |
| Subsequent builds slow | Run `cd android && ./gradlew clean` then rebuild |

---

## Project Structure

```
HelloWorld_AndroidApp/
├── SETUP_GUIDE.md                        ← this file
├── README.md
└── PlaystoreTestApp/
    ├── App.js                            ← main UI
    ├── index.js                          ← entry point
    ├── app.json                          ← display name
    ├── package.json
    ├── babel.config.js
    ├── metro.config.js
    ├── playstore-release-key.jks         ← release keystore (DO NOT COMMIT)
    ├── .gitignore
    ├── __tests__/
    │   └── App.test.js
    └── android/
        ├── build.gradle                  ← project-level config (minSdk, compileSdk)
        ├── local.properties              ← SDK path (DO NOT COMMIT)
        ├── settings.gradle
        ├── gradle.properties
        ├── gradlew
        ├── gradlew.bat
        ├── gradle/wrapper/
        └── app/
            ├── build.gradle              ← app-level config (signing)
            ├── debug.keystore
            ├── proguard-rules.pro
            ├── build/                    ← build output (DO NOT COMMIT)
            │   └── outputs/
            │       ├── apk/
            │       └── bundle/release/app-release.aab    ← AAB FOR PLAY STORE
            └── src/main/
                ├── AndroidManifest.xml
                ├── java/com/playstoretestapp/
                │   ├── MainActivity.kt
                │   └── MainApplication.kt
                └── res/
                    ├── drawable/
                    ├── mipmap-*/         ← app icons
                    └── values/
                        ├── strings.xml
                        └── styles.xml
```

---

## Key Configuration Summary

| File | Purpose | Critical Settings |
|---|---|---|
| `android/build.gradle` | Project-level Gradle config | `minSdkVersion = 24`, `compileSdkVersion = 36`, `ndkVersion = "27.1.12297006"` |
| `android/app/build.gradle` | App-level Gradle config | `applicationId = "com.playstoretestapp"`, signing configs, `versionCode`, `versionName` |
| `android/local.properties` | Local SDK paths | `sdk.dir=/opt/homebrew/share/android-commandlinetools` |
| `app.json` | RN app metadata | `displayName = "HelloWorldPlaystoreTest"` |
| `package.json` | npm dependencies | `react-native: 0.85.2` |
| `playstore-release-key.jks` | Release signing key | Alias: `playstore-key`, Password: `Android` |

---

## Security Reminders

⚠️ **NEVER commit these files to git:**
- `playstore-release-key.jks` (release keystore)
- `android/local.properties` (machine-specific paths)
- `node_modules/` (npm dependencies)
- `android/build/` and `android/app/build/` (build artifacts)
- `.env` files (if used)

✅ **Already in `.gitignore`:**
- `*.jks`
- `*.keystore`
- Standard React Native ignores

---

## Next Steps After AAB Generation

1. **Test locally** — use `bundletool` to install on emulator (Section 11)
2. **Create Google Play Console account** — pay one-time $25 fee at [play.google.com/console](https://play.google.com/console)
3. **Create app listing** — name, description, screenshots, icon, feature graphic
4. **Complete content rating** — questionnaire about app content
5. **Upload AAB** — start with Internal Testing → Production
6. **Submit for review** — Google reviews in 1–3 days for new apps

---

**Document Version:** 1.0
**Last Updated:** 2026-05-03
**Project:** HelloWorldPlaystoreTest
**React Native Version:** 0.85.2
