# PureFusionIPTV — Production Readiness TODO

This document tracks everything intentionally incomplete, placeholder, or deferred.
Nothing here is a bug — the app builds and runs correctly. These are items requiring
external setup, credentials, or deployment decisions before a full production release.

**Last updated:** 2026-02-28

---

## DONE — Completed Items

| Item | Status |
|------|--------|
| Firebase Auth (email/password) | DONE — enabled in console |
| Firebase Firestore | DONE — created, security rules published (write=false for clients) |
| Firebase Cloud Messaging (FCM) | DONE — wired in app, PureFusionMessagingService registered |
| Firebase Cloud Functions | DONE — 3 functions deployed (refreshEntitlement, revokeDevice, setUserTier) |
| RSA keypair for JWT | DONE — private key in Secret Manager, public key embedded in APK |
| EntitlementManager.kt | DONE — verifies JWT, caches tier, wired into SubscriptionRepository |
| Login screen (email/password) | DONE — LoginActivity, launches from wizard and settings |
| Subscription screen | DONE — SubscriptionActivity with real Play prices, all 3 tiers |
| Google Play Billing integration | DONE — BillingManager v7, subscriptions + lifetime |
| Settings account section | DONE — email, tier, login, logout, subscription link in General |
| Wizard login step | DONE — SetupWizardActivity launches LoginActivity on first run |

---

## 1. Google Play Developer Account — PENDING TOMORROW

**Status:** Not yet created. $25 one-time fee. See `DOCS/GOOGLE_PLAY_SETUP.md`.

**What blocks you without it:**
- Cannot create in-app products (`purefusion_monthly`, `purefusion_yearly`, `purefusion_lifetime`)
- Subscription screen shows "—" prices until products exist in Play Console
- Cannot test real purchases (license testing requires Play Console account)

**What works without it:**
- App installs and runs fine via sideload
- Login / Firebase Auth works
- Firebase tier system works (you can manually set tier in Firestore)
- All app features work based on locally cached tier

**Action tomorrow:**
1. Create account at play.google.com/console — $25
2. Create app: `PureFusion IPTV`, package `dev.eliminater.purefusioniptv`
3. Create 3 products — exact IDs matter, see `DOCS/GOOGLE_PLAY_SETUP.md` Step 3
4. Upload APK to Internal Testing track
5. Add testers, set their tier to LIFETIME via Firestore manually

---

## 2. Cross-Device Play Purchase Sync — TODO (syncPlayPurchase)

**Status:** Manual workaround in place. Auto-sync not yet built.

**Current behavior:**
- User buys on Android phone → tier active on that device immediately (Google Play Billing)
- Fire TV / sideloaded device → tier NOT automatically updated in Firestore
- Manual fix: go to Firebase Console → Firestore → set uid document tier manually

**Why client cannot write to Firestore:**
Firestore security rule is `allow write: if false` for clients — this is CORRECT and
intentional. If clients could write their own tier, a modded APK could set itself to
LIFETIME in 30 seconds. Do NOT change this rule.

**The correct solution (future):**
Build `syncPlayPurchase` Cloud Function:
```
Android app sends purchaseToken → Cloud Function
    → Cloud Function calls Google Play Developer API to verify token
    → If valid → writes tier to Firestore
    → refreshEntitlement() on all devices gets updated JWT
```

**Requirements to build this:**
- Google Play Developer API credentials (service account JSON)
- Link Play Console to Google Cloud project (Play Console → Setup → API access)
- Add `google-auth-library` to Cloud Functions
- Requires Play Console account (tomorrow)

**TODO location in code:**
`billing/SubscriptionRepository.kt` line ~196 — TODO comment marks exactly where
the Cloud Function call should go after purchase confirmed.

---

## 3. Certificate Pinning — PLACEHOLDER (no backend domain yet)

**File:** `app/src/main/java/dev/eliminater/purefusioniptv/security/CertificatePinConfig.kt`

**Current state:** Placeholder pins. `NetworkModule.kt` detects placeholders and skips
pinning automatically — app works correctly without real pins.

**When you have a backend domain:**
```bash
# Run in Git Bash to get your pin:
openssl s_client -connect YOUR_DOMAIN:443 </dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | base64
```

Then update `CertificatePinConfig.kt`:
```kotlin
val PINS = mapOf(
    "api.yourdomain.com" to listOf(
        "sha256/REAL_PIN_FROM_ABOVE=",
        "sha256/BACKUP_PIN="   // Need 2 minimum
    )
)
```

**Emergency escape hatch:** Firebase Remote Config key `cert_pin_override = true`
bypasses pinning for all users instantly if you have a cert rotation emergency.

---

## 4. Receipt Validation Backend — NOT DEPLOYED

**File:** `app/src/main/java/dev/eliminater/purefusioniptv/billing/ReceiptValidator.kt`

**Current state:** `receiptValidationUrl` is empty in Remote Config → `ReceiptValidator`
returns `ValidationResult.Skipped` → billing falls back to local Google Play state.
App works correctly, just without server-side receipt verification.

**Risk without it:** A user could potentially share a purchase token. Low risk for
now — anomaly detection already flags token reuse.

**To deploy (after Play account + Developer API access):**
1. Add `validateReceipt` Cloud Function to `functions/src/index.ts`
2. Function calls Google Play Developer API with purchase token
3. Returns valid/invalid
4. Set `receipt_validation_url` in Firebase Remote Config to the function URL

**See:** `DOCS/FIREBASE_LICENSING_BACKEND.md` for full implementation steps.

---

## 5. Remote Config Enforcement — PARTIALLY WIRED

**Current state:** Flags exist in Remote Config, but enforcement logic is deferred.

| Flag | Current behavior | Required behavior |
|------|-----------------|-------------------|
| `force_online_validation` | Not enforced | If true → skip ValidationResult.Skipped, require real validation |
| `grace_period_enabled` | Always 72h | If false → deny access immediately on billing lapse |
| `grace_period_hours` | Hardcoded 72 | Should read from Remote Config |
| `anomaly_score_hard_cutoff` | Not read | If anomaly score ≥ cutoff → deny access |
| `receipt_validation_url` | Read, empty = skip | When set → must validate, no skip |

**Sprint 6 scope** — wire these flags into `SubscriptionRepository.processPurchases()`
and `isInGracePeriod()`.

---

## 6. NDK Enforcement — DETECTION ONLY (not blocking)

**Files:** `app/src/main/cpp/license_validator.cpp`, `NativeLicenseValidator.kt`

**Current state:** Detects debuggers, Frida, emulators, and reports via `EnvironmentStatus`.
Logs the finding but does NOT block access — stealth mode.

**To enforce when ready:**
```kotlin
// In SubscriptionRepository.processPurchases():
val envStatus = nativeValidator.checkEnvironment()
if (envStatus is EnvironmentStatus.Compromised) {
    _subscriptionTier.value = SubscriptionTier.FREE
}
```

**Recommendation:** Keep stealth mode for now. Blocking causes false positives on
legitimate rooted devices. Let anomaly detection accumulate score first.

---

## 7. Parental Controls — UI NOT BUILT

**Current state:** Feature-gated (`PremiumFeature.PARENTAL_CONTROLS`), settings keys exist
in `SettingsManager`, but there is no PIN entry screen or channel filtering enforcement.

**To build:**
- PIN entry/setup dialog
- Channel age rating filter
- PIN gate on settings screen itself
- Parental unlock flow when accessing blocked content

**Priority:** Medium — not needed for initial launch.

---

## 8. App Signing for Release — REQUIRED BEFORE PLAY UPLOAD

**Current state:** Debug APK works fine. Release APK needs a signing keystore.

**Steps:**
1. Android Studio → Build → Generate Signed Bundle / APK
2. Create new keystore → save `.jks` file somewhere safe (NOT in git)
3. Store passwords safely (password manager, not in code)
4. Add signing config to `app/build.gradle.kts` (use environment variables or local.properties, not hardcoded)

**Or:** Use Google Play App Signing (recommended) — Google holds the key, you upload
with an upload key. Safer because you can't lose the signing key.

---

## 9. Firebase App Distribution — OPTIONAL (useful for testers)

Send APKs to testers without Play Store:
```
Firebase Console → App Distribution → Enable
./gradlew assembleRelease appDistributionUploadRelease
```
Testers get an email link to download. Works on all Android devices including Fire TV.

---

## 10. Summary — What Blocks Each Release Type

### Sideload / Direct APK (can do TODAY)
- Nothing blocking. Build APK, share file, done.
- Testers create account in app, you set their Firestore tier manually.

### Internal Testing via Play Store (need tomorrow — Play account)
- Create Play developer account ($25)
- Create the 3 in-app products in Play Console
- Upload APK to Internal Testing track
- Add testers to license testing list

### Public Production Release (need all of the following)
- [ ] Play developer account + verified merchant account (bank details)
- [ ] App signing keystore configured
- [ ] Store listing complete (icon, screenshots, description, content rating, data safety)
- [ ] Real in-app products activated in Play Console
- [ ] `syncPlayPurchase` Cloud Function built (for cross-device sync)
- [ ] Receipt validation backend deployed
- [ ] Remote Config enforcement flags wired (Sprint 6)

---

## Related Docs

| Doc | Purpose |
|-----|---------|
| `DOCS/GOOGLE_PLAY_SETUP.md` | Step-by-step Play Console setup, products, testing |
| `DOCS/FIREBASE_LICENSING_BACKEND.md` | Firebase Auth, Firestore, Cloud Functions, JWT |
| `DOCS/AMAZON_FIRETV_STRATEGY.md` | Fire TV distribution, billing gap, sideload strategy |
| `DOCS/DEPLOYMENT_GUIDE.md` | Build and deployment process |
| `DOCS/FASTZAP_ARCHITECTURE.md` | Player zap architecture |
| `DOCS/PLUGIN_SDK.md` | Plugin system for developers |
