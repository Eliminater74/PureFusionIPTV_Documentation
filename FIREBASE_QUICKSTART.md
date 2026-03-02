# 🔥 Firebase Setup - 10 Minute Quickstart

## Step 1: Create Firebase Project (3 minutes)

1. Open https://console.firebase.google.com/
2. Click **"Add project"** or **"Create a project"**
3. Project name: `PureFusionIPTV` (or whatever you prefer)
4. Click **Continue**
5. **Disable Google Analytics** (recommended for privacy, can enable later)
   - Toggle OFF "Enable Google Analytics for this project"
6. Click **Create project**
7. Wait ~30 seconds for setup
8. Click **Continue**

---

## Step 2: Add Android App (2 minutes)

1. On the Firebase Console home, click the **Android icon** (or "+ Add app")
2. Fill in:
   - **Android package name**: `dev.eliminater.purefusioniptv`
   - **App nickname**: `PureFusionIPTV` (optional but recommended)
   - **Debug signing certificate SHA-1**: Leave blank (not needed for Crashlytics)
3. Click **Register app**

---

## Step 3: Download Config File (1 minute)

1. Click **Download google-services.json**
2. **IMPORTANT**: This file contains API keys - keep it private!
3. Save it to your Downloads folder

---

## Step 4: Install Config File (30 seconds)

**Option A - Manual** (recommended):
```bash
# Copy from Downloads to app directory
cp ~/Downloads/google-services.json app/google-services.json

# Or on Windows:
copy %USERPROFILE%\Downloads\google-services.json app\google-services.json
```

**Option B - Let me do it**:
Just tell me the path where you downloaded it, and I'll copy it for you.

---

## Step 5: Enable Crashlytics (2 minutes)

1. In Firebase Console, click **Crashlytics** in the left sidebar (under "Product categories" → "Operations")
2. Click **"Enable Crashlytics"**
3. Follow the setup wizard:
   - ✅ Add Firebase SDK (already done in code)
   - ✅ Force a test crash (we'll do this next)
4. Click **Finish setup**

---

## Step 6: Verify Installation (2 minutes)

Build and install the app:

```bash
# Build release APK (triggers Crashlytics symbol upload)
./gradlew assembleRelease

# Install on connected device
adb install app/build/outputs/apk/release/app-release-unsigned.apk

# Or build debug for faster testing:
./gradlew assembleDebug
adb install app/build/outputs/apk/debug/app-debug.apk
```

Check logs for Crashlytics initialization:
```bash
adb logcat | grep -i "crashlytics\|firebase"
```

**Expected output**:
```
I/FirebaseApp: Device unlocked: initializing all Firebase APIs for app [DEFAULT]
D/FirebaseCrashlytics: Crashlytics initialized
I/PureFusionApp: PureFusionIPTV initialized
```

---

## Step 7: Test Crash Reporting (Optional - 2 minutes)

Add this to `MainActivity.onCreate()` temporarily:

```kotlin
// TEMPORARY - Remove after testing
if (BuildConfig.DEBUG) {
    throw RuntimeException("Test Crashlytics crash")
}
```

Rebuild, install, launch app → should crash immediately.

Wait 5-10 minutes, then check Firebase Console → Crashlytics → should see the crash report.

**IMPORTANT**: Remove the test crash code after verification!

---

## ✅ Success Checklist

- [ ] Firebase project created
- [ ] Android app registered in Firebase
- [ ] `google-services.json` downloaded
- [ ] File copied to `app/google-services.json`
- [ ] Build succeeds with `./gradlew assembleRelease`
- [ ] Crashlytics enabled in Firebase Console
- [ ] Crashlytics initialization seen in logcat
- [ ] (Optional) Test crash appears in Firebase Console

---

## 🔒 Security - CRITICAL

Add this to `.gitignore` **RIGHT NOW**:

```bash
echo "app/google-services.json" >> .gitignore
```

**Why?**: The `google-services.json` file contains API keys. If you commit it to a public repo, anyone can use your Firebase quota.

The template file (`app/google-services.json.template`) is safe to commit.

---

## 🐛 Troubleshooting

### Build fails with "google-services.json not found"
**Solution**: Make sure file is at `app/google-services.json` (not in root directory)

### Crashlytics not showing crashes
**Possible causes**:
1. Wait 5-10 minutes (indexing delay)
2. Check you're building **release** variant (debug builds don't send crashes by default)
3. Verify `setCrashlyticsCollectionEnabled(!BuildConfig.DEBUG)` in PureFusionApp.kt

### "Too many INVALID_ARGUMENT errors"
**Solution**: Your `google-services.json` is corrupted. Re-download from Firebase Console.

---

## 🎉 What You Get

Once Firebase is set up:

✅ **Automatic crash reporting** - All production crashes sent to Firebase
✅ **Non-fatal exception tracking** - `Timber.e()` logs to Crashlytics
✅ **Custom context keys** - App version, version code, log tags
✅ **Stack traces** - Full deobfuscated stack traces with ProGuard mapping
✅ **StrictMode violations** - Debug builds flash screen on violations

**No code changes needed** - everything is already wired up!

---

## ⏭️ After Firebase Setup

Once this is done, you're ready for:
1. **Wire PredictivePreBufferManager** (2 hours) - See DEPLOYMENT_GUIDE.md Step 3
2. **Test on device** (4 hours) - Verify pre-buffer works
3. **Ship Phase 1** 🚀

---

**Total time**: ~10 minutes
**Difficulty**: Easy (mostly clicking buttons)
**Blockers**: None (I'll help if you get stuck)

Ready? Let's go! 🔥
