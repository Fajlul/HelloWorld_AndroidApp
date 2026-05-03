# AAB Build & Test — Command Reference

Complete cumulative list of every command used to:
1. Set up the environment
2. Start the Android emulator
3. Generate the signed `.aab` file
4. Convert `.aab` → `.apk` set
5. Install and run the APK on the emulator

**Project:** HelloWorldPlaystoreTest
**Project Path:** `/Users/mohammedfajlulkarim/projects/HelloWorld_AndroidApp/PlaystoreTestApp`

---

## Visual Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          OVERALL WORKFLOW                                     │
└──────────────────────────────────────────────────────────────────────────────┘

   ┌─────────────────────┐
   │  STEP 1: ENVIRONMENT │
   │   export JAVA_HOME   │
   │   export ANDROID_HOME│
   │   export PATH        │
   └──────────┬──────────┘
              │
              ▼
   ┌─────────────────────┐         ┌─────────────────────────┐
   │  STEP 2: EMULATOR   │────────►│   Pixel_6_API_34 AVD    │
   │  emulator -avd ...  │         │   (Android 14, arm64)    │
   │  adb wait-for-device│         └─────────────────────────┘
   └──────────┬──────────┘
              │
              ▼
   ┌─────────────────────┐         ┌─────────────────────────┐
   │  STEP 3: BUILD AAB   │────────►│   app-release.aab       │
   │  ./gradlew           │         │   (signed with .jks)    │
   │      bundleRelease   │         │   ~30-50 MB             │
   └──────────┬──────────┘         └─────────────────────────┘
              │
              ▼
   ┌─────────────────────┐         ┌─────────────────────────┐
   │  STEP 4: AAB → APKS  │────────►│   /tmp/app.apks         │
   │  bundletool          │         │   (split APK set)       │
   │      build-apks      │         └─────────────────────────┘
   └──────────┬──────────┘
              │
              ▼
   ┌─────────────────────┐         ┌─────────────────────────┐
   │  STEP 5: UNINSTALL   │────────►│   Removes debug build   │
   │  adb uninstall       │         │   (signature mismatch)  │
   └──────────┬──────────┘         └─────────────────────────┘
              │
              ▼
   ┌─────────────────────┐         ┌─────────────────────────┐
   │  STEP 6: INSTALL APK │────────►│   App on emulator       │
   │  bundletool          │         │   (release-signed)      │
   │      install-apks    │         └─────────────────────────┘
   └──────────┬──────────┘
              │
              ▼
   ┌─────────────────────┐         ┌─────────────────────────┐
   │  STEP 7: LAUNCH APP  │────────►│   "Hello World - Play   │
   │  adb shell am start  │         │    Store Test"           │
   └─────────────────────┘         └─────────────────────────┘
```

---

## STEP 1 — Environment Setup (Required Before Every Session)

### Set Environment Variables for Current Session

```bash
export JAVA_HOME="/opt/homebrew/opt/openjdk@17"
export ANDROID_HOME="/opt/homebrew/share/android-commandlinetools"
export ANDROID_SDK_ROOT="$ANDROID_HOME"
export PATH="$JAVA_HOME/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$PATH"
```

### Verify Environment

```bash
java -version       # Should show: openjdk version "17.0.19"
adb --version       # Should show: Android Debug Bridge version
emulator -version   # Should show: Android emulator version
```

### Make Permanent (One-Time Setup)

```bash
echo 'export JAVA_HOME="/opt/homebrew/opt/openjdk@17"' >> ~/.zshrc
echo 'export ANDROID_HOME="/opt/homebrew/share/android-commandlinetools"' >> ~/.zshrc
echo 'export ANDROID_SDK_ROOT="$ANDROID_HOME"' >> ~/.zshrc
echo 'export PATH="$JAVA_HOME/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

---

## STEP 2 — Start the Android Emulator

### Start Emulator in Background

```bash
emulator -avd Pixel_6_API_34 -no-snapshot-load -no-audio &
```

| Flag | Purpose |
|---|---|
| `-avd Pixel_6_API_34` | The AVD name to launch |
| `-no-snapshot-load` | Cold boot (avoids snapshot corruption) |
| `-no-audio` | Disables audio (faster startup) |
| `&` | Runs in background |

### Wait for Emulator to Fully Boot

```bash
adb wait-for-device shell 'until [ "$(getprop sys.boot_completed)" = "1" ]; do sleep 2; done; echo Booted'
```

### Verify Device is Connected

```bash
adb devices
```

Expected output:
```
List of devices attached
emulator-5554   device
```

---

## STEP 3 — Build the AAB File

### Navigate to Android Directory

```bash
cd /Users/mohammedfajlulkarim/projects/HelloWorld_AndroidApp/PlaystoreTestApp/android
```

### Run Gradle Bundle Release

```bash
./gradlew bundleRelease
```

### AAB Output Location

```
PlaystoreTestApp/android/app/build/outputs/bundle/release/app-release.aab
```

### Optional: Verify AAB Contents

```bash
$ANDROID_HOME/cmdline-tools/latest/bin/apkanalyzer \
  manifest print \
  app/build/outputs/bundle/release/app-release.aab
```

---

## STEP 4 — Convert AAB to APK Set (using bundletool)

### Install bundletool (One-Time)

```bash
brew install bundletool
```

### Navigate Back to Project Root

```bash
cd /Users/mohammedfajlulkarim/projects/HelloWorld_AndroidApp/PlaystoreTestApp
```

### Convert AAB → APK Set

```bash
bundletool build-apks \
  --bundle=android/app/build/outputs/bundle/release/app-release.aab \
  --output=/tmp/app.apks \
  --ks=playstore-release-key.jks \
  --ks-pass=pass:Android \
  --ks-key-alias=playstore-key \
  --key-pass=pass:Android
```

| Flag | Value | Purpose |
|---|---|---|
| `--bundle` | Path to `.aab` | Source AAB file |
| `--output` | `/tmp/app.apks` | Generated APK set archive |
| `--ks` | `playstore-release-key.jks` | Keystore file |
| `--ks-pass` | `pass:Android` | Keystore password (literal "Android") |
| `--ks-key-alias` | `playstore-key` | Key alias inside keystore |
| `--key-pass` | `pass:Android` | Key password |

> **Note:** The `pass:` prefix tells bundletool the password is provided inline (vs reading from a file).

---

## STEP 5 — Uninstall Existing Debug Build

⚠️ **Required step** — Android blocks installing differently-signed apps over an existing one.

```bash
adb uninstall com.playstoretestapp
```

Expected output:
```
Success
```

---

## STEP 6 — Install the Release APK

```bash
bundletool install-apks --apks=/tmp/app.apks
```

> **If you get "No connected devices found":** make sure Step 2 (emulator) is running.
>
> **If you get "Unable to determine ADB location":** make sure Step 1 (env vars) is set, especially `ANDROID_HOME`.

---

## STEP 7 — Launch the App

```bash
adb -s emulator-5554 shell am start -n com.playstoretestapp/.MainActivity
```

### Verify App is Running

```bash
adb -s emulator-5554 shell dumpsys activity activities | grep "topResumedActivity"
```

Expected output (excerpt):
```
topResumedActivity=ActivityRecord{... com.playstoretestapp/.MainActivity ...}
```

---

## Complete Sequence — Copy & Paste Block

For convenience, here's the **entire sequence as one block** you can run end-to-end:

```bash
# ─────────────────────────────────────────────────────
# STEP 1: Environment
# ─────────────────────────────────────────────────────
export JAVA_HOME="/opt/homebrew/opt/openjdk@17"
export ANDROID_HOME="/opt/homebrew/share/android-commandlinetools"
export ANDROID_SDK_ROOT="$ANDROID_HOME"
export PATH="$JAVA_HOME/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$PATH"

# ─────────────────────────────────────────────────────
# STEP 2: Start emulator
# ─────────────────────────────────────────────────────
emulator -avd Pixel_6_API_34 -no-snapshot-load -no-audio &
adb wait-for-device shell 'until [ "$(getprop sys.boot_completed)" = "1" ]; do sleep 2; done; echo Booted'

# ─────────────────────────────────────────────────────
# STEP 3: Build AAB
# ─────────────────────────────────────────────────────
cd /Users/mohammedfajlulkarim/projects/HelloWorld_AndroidApp/PlaystoreTestApp/android
./gradlew bundleRelease

# ─────────────────────────────────────────────────────
# STEP 4: Convert AAB → APK set
# ─────────────────────────────────────────────────────
cd /Users/mohammedfajlulkarim/projects/HelloWorld_AndroidApp/PlaystoreTestApp
bundletool build-apks \
  --bundle=android/app/build/outputs/bundle/release/app-release.aab \
  --output=/tmp/app.apks \
  --ks=playstore-release-key.jks \
  --ks-pass=pass:Android \
  --ks-key-alias=playstore-key \
  --key-pass=pass:Android

# ─────────────────────────────────────────────────────
# STEP 5: Uninstall old debug build
# ─────────────────────────────────────────────────────
adb uninstall com.playstoretestapp

# ─────────────────────────────────────────────────────
# STEP 6: Install release APK
# ─────────────────────────────────────────────────────
bundletool install-apks --apks=/tmp/app.apks

# ─────────────────────────────────────────────────────
# STEP 7: Launch app
# ─────────────────────────────────────────────────────
adb -s emulator-5554 shell am start -n com.playstoretestapp/.MainActivity
```

---

## Visual: File & Path Mapping

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      WHERE EVERYTHING LIVES                              │
└─────────────────────────────────────────────────────────────────────────┘

  TOOL LOCATIONS                          PROJECT FILES
  ──────────────                          ─────────────

  /opt/homebrew/                          PlaystoreTestApp/
  ├── opt/openjdk@17/                     ├── playstore-release-key.jks ◄─ keystore
  │   └── bin/java                        ├── App.js
  │                                       ├── package.json
  └── share/android-commandlinetools/     │
      ├── cmdline-tools/latest/bin/       └── android/
      │   ├── adb (no, in platform-tools) │   ├── build.gradle ◄─── minSdk: 24
      │   ├── sdkmanager                  │   ├── local.properties
      │   ├── avdmanager                  │   ├── gradlew ◄────────── build wrapper
      │   └── apkanalyzer                 │   │
      │                                   │   └── app/
      ├── platform-tools/                 │       ├── build.gradle ◄ signing config
      │   └── adb ◄────────── debug bridge│       │
      │                                   │       └── build/outputs/
      ├── emulator/                       │           ├── apk/
      │   └── emulator                    │           │   └── debug/
      │                                   │           │       └── app-debug.apk
      └── platforms/android-34/           │           │
                                          │           └── bundle/release/
                                          │               └── app-release.aab ◄── AAB
                                          │
                                          └── /tmp/app.apks ◄── APK set output
```

---

## Visual: Signing Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  HOW SIGNING WORKS DURING AAB BUILD                      │
└─────────────────────────────────────────────────────────────────────────┘

   ┌────────────────────────┐
   │ playstore-release-key  │
   │      .jks (keystore)   │
   └──────────┬─────────────┘
              │
              │ Contains:
              │   - Private Key (secret)
              │   - Public Certificate
              │   - Alias: playstore-key
              │   - Password: Android
              ▼
   ┌────────────────────────┐
   │     Gradle Signing     │
   │  (./gradlew bundle...) │
   └──────────┬─────────────┘
              │
              │ Uses private key to:
              │   1. Hash app contents
              │   2. Encrypt hash with private key
              │   3. Embed signature in AAB
              │   4. Embed PUBLIC certificate in AAB
              │      (NEVER the private key!)
              ▼
   ┌────────────────────────┐
   │   app-release.aab      │
   │                        │
   │   Contents:            │
   │   ✅ Compiled code     │
   │   ✅ JS bundle         │
   │   ✅ Resources         │
   │   ✅ Public certificate│
   │   ✅ Cryptographic sig │
   │   ❌ Private key       │
   │   ❌ Keystore file     │
   └────────────────────────┘
```

---

## Visual: Why You Need bundletool

```
┌─────────────────────────────────────────────────────────────────────────┐
│              AAB vs APK — Different Purposes                            │
└─────────────────────────────────────────────────────────────────────────┘

  PRODUCTION FLOW (Google Play):              LOCAL TESTING FLOW:
  ─────────────────────────────                ────────────────────

    app-release.aab                              app-release.aab
         │                                            │
         │ upload                                     │ bundletool build-apks
         ▼                                            ▼
    ┌──────────┐                                /tmp/app.apks
    │   Play   │                                (APK set archive)
    │  Store   │                                     │
    └────┬─────┘                                     │ bundletool install-apks
         │ optimizes per device                      ▼
         ▼                                      ┌─────────────┐
    Device-specific APK                         │  Emulator   │
         │                                      └─────────────┘
         ▼
    User's phone

    ⚠ AABs CANNOT be installed directly on a device.
    ⚠ APKs CAN be installed directly.
    ⚠ bundletool simulates what Play Store does.
```

---

## Quick Command Cheat Sheet

| Goal | Command |
|---|---|
| Set environment | `export JAVA_HOME=...; export ANDROID_HOME=...; export PATH=...` |
| List AVDs | `avdmanager list avd` |
| Start emulator | `emulator -avd Pixel_6_API_34 &` |
| Wait for boot | `adb wait-for-device shell 'until [ "$(getprop sys.boot_completed)" = "1" ]; do sleep 2; done'` |
| Check devices | `adb devices` |
| Build AAB | `cd android && ./gradlew bundleRelease` |
| AAB → APK set | `bundletool build-apks --bundle=...aab --output=/tmp/app.apks --ks=...jks ...` |
| Uninstall app | `adb uninstall com.playstoretestapp` |
| Install APK set | `bundletool install-apks --apks=/tmp/app.apks` |
| Launch app | `adb shell am start -n com.playstoretestapp/.MainActivity` |
| View logs | `adb logcat | grep playstoretestapp` |
| Force-stop app | `adb shell am force-stop com.playstoretestapp` |
| Stop emulator | `adb -s emulator-5554 emu kill` |

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `command not found: adb` | PATH missing | Re-run Step 1 export commands |
| `command not found: java` | JAVA_HOME not set | Re-run Step 1 export commands |
| `Unable to determine ADB location` | bundletool can't find ADB | Set `ANDROID_HOME` env var |
| `No connected devices found` | Emulator not running | Run Step 2 emulator commands |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | Old debug build still installed | Run Step 5 `adb uninstall ...` |
| `Keystore was tampered with` | Wrong password | Verify password is `Android` (capital A) |
| `BUILD FAILED` (Gradle) | Various — check log | Run `./gradlew clean` then retry |
| App shows white screen | JS bundle issue | Check Metro running; or rebuild with `bundleRelease` |

---

## Key Variables Used in This Project

| Variable | Value |
|---|---|
| Project name (technical) | `PlaystoreTestApp` |
| App display name | `HelloWorldPlaystoreTest` |
| Package ID | `com.playstoretestapp` |
| Main activity | `com.playstoretestapp/.MainActivity` |
| Keystore file | `playstore-release-key.jks` |
| Keystore password | `Android` |
| Key alias | `playstore-key` |
| Key password | `Android` |
| AVD name | `Pixel_6_API_34` |
| Emulator device ID | `emulator-5554` |
| Min Android SDK | 24 (Android 7.0) |
| Compile SDK | 36 |
| Target SDK | 36 |

---

**Document Version:** 1.0
**Last Updated:** 2026-05-03
**Use Case:** Quick reference for rebuilding and testing the AAB
