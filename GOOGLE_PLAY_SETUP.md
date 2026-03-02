# Google Play Store Setup Guide

Complete step-by-step instructions for publishing PureFusionIPTV and enabling in-app purchases.

---

## Step 1 — Create Your Google Play Developer Account

1. Go to https://play.google.com/console
2. Sign in with your Google account (use a permanent account, not a throwaway)
3. Click **Get started**
4. Accept the Developer Distribution Agreement
5. Pay the **one-time $25 USD registration fee** (credit/debit card)
6. Fill in your developer profile:
   - Developer name: what users will see (e.g. "PureFusion Media")
   - Email address: support contact shown on store listing
   - Website: optional for now
7. Account takes up to **48 hours** to be verified — usually faster

---

## Step 2 — Create Your App in Play Console

1. Go to Play Console → **All apps** → **Create app**
2. Fill in:
   - App name: `PureFusion IPTV`
   - Default language: English (United States)
   - App or game: **App**
   - Free or paid: **Free** (you will use in-app purchases for monetization)
3. Accept the declarations → **Create app**

---

## Step 3 — Create In-App Products (Subscriptions + Lifetime)

This is the most important step. Your app uses these exact product IDs — do NOT change them:

| Product | ID | Type |
|---|---|---|
| Monthly | `purefusion_monthly` | Subscription |
| Yearly | `purefusion_yearly` | Subscription |
| Lifetime | `purefusion_lifetime` | One-time (in-app product) |

### 3a — Create Subscriptions (Monthly + Yearly)

1. In your app → left sidebar → **Monetize** → **Subscriptions**
2. Click **Create subscription**
3. For **Monthly**:
   - Product ID: `purefusion_monthly` ← exact, case-sensitive
   - Name: `PureFusion Premium Monthly`
   - Description: `Full access to all PureFusion IPTV features. Cancel anytime.`
   - Click **Add a base plan**
   - Billing period: **Monthly**
   - Price: set your price (e.g. $4.99/month)
   - Renewal type: Auto-renewing
   - Click **Save** → **Activate**
4. Repeat for **Yearly**:
   - Product ID: `purefusion_yearly` ← exact, case-sensitive
   - Name: `PureFusion Premium Yearly`
   - Description: `Full access to all PureFusion IPTV features. Billed annually.`
   - Billing period: **Yearly**
   - Price: set your price (e.g. $34.99/year)
   - Click **Save** → **Activate**

### 3b — Create Lifetime (One-Time Purchase)

1. Left sidebar → **Monetize** → **In-app products**
2. Click **Create product**
3. Fill in:
   - Product ID: `purefusion_lifetime` ← exact, case-sensitive
   - Name: `PureFusion Lifetime Access`
   - Description: `One-time purchase. Lifetime access to all premium features.`
   - Status: **Active**
   - Price: set your price (e.g. $59.99)
4. Click **Save**

---

## Step 4 — Set Up a Merchant Account (To Receive Money)

1. In Play Console → **Setup** → **Payments profile**
2. Click **Create payments profile**
3. You will be redirected to Google Pay & Wallet Console
4. Fill in:
   - Business type: Individual or Business
   - Legal name (your real name or business name)
   - Address
   - Bank account details (for payouts)
5. Submit — Google will verify within 1-3 business days
6. You will NOT receive any payments until this is complete

---

## Step 5 — Upload Your First APK (Internal Testing)

You need to upload an APK before products become testable.

### Build the Release APK

In Android Studio terminal or Git Bash:
```bash
cd I:/GITHUB/Projects/PureFusionIPTV
./gradlew assembleRelease
```

APK location after build:
```
app/build/outputs/apk/release/app-release.apk
```

> Note: If the release APK is unsigned, you need to set up signing first.
> See the **Signing** section below before proceeding.

### Upload to Internal Testing

1. Play Console → your app → **Testing** → **Internal testing**
2. Click **Create new release**
3. Under **App bundles and APKs** → upload your APK or AAB
4. Release name: e.g. `0.1.0-alpha`
5. Release notes: e.g. "Internal testing release"
6. Click **Save** → **Review release** → **Start rollout to Internal testing**

### Add Testers

1. In **Internal testing** → **Testers** tab
2. Click **Create email list**
3. Name it: `Alpha Testers`
4. Add your testers' Gmail addresses (up to 100 for internal testing)
5. Click **Save**
6. Share the opt-in URL with your testers — they click it to join

---

## Step 6 — Set Up App Signing

Play Store requires APKs to be signed. Two options:

### Option A — Google Play App Signing (Recommended)

Google manages your keystore. Safer — you can't lose your key.

1. When uploading your first release, Play Console offers "Let Google manage and protect your app signing key"
2. Select this option
3. Upload an **upload key** (Android Studio can generate one for you)

### Option B — Self-Managed Keystore

You keep your own keystore file. You must NEVER lose it — if you lose it you cannot update your app.

In Android Studio:
1. **Build** → **Generate Signed Bundle / APK**
2. Choose **APK**
3. **Create new keystore**
4. Fill in alias, passwords, certificate info
5. Save the `.jks` file somewhere safe (NOT in git)
6. Build signed release APK

Add to `app/build.gradle.kts`:
```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file("path/to/your.jks")
            storePassword = "yourpassword"
            keyAlias = "youralias"
            keyPassword = "yourpassword"
        }
    }
    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

---

## Step 7 — Test Purchases Before Going Live

Google Play has a **License Testing** feature so you can test purchases without being charged.

### Add License Testers

1. Play Console → **Setup** → **License testing**
2. Add Gmail addresses of testers
3. These accounts can make purchases that are free and cancellable

### Test on a Real Device

1. Install the app via the Internal Testing opt-in link (NOT sideload)
2. The app must be installed from Play Store for billing to work
3. Open Settings → General → Account → Subscription & Upgrade
4. Prices should now load from Play (may take a few minutes after activation)
5. Tap a plan — Google Play purchase sheet appears
6. Complete the purchase — you will NOT be charged (license tester)
7. After purchase, the app should update to PREMIUM/LIFETIME

---

## Step 8 — Cross-Device Sync After Purchase (Fire TV / Sideload)

After a user purchases on Android (Play Store):

**Automatic on that device:** The tier is immediately active via Google Play Billing.

**For Fire TV / sideloaded devices:** Until you build the automatic server sync (future), use this manual process:

1. User purchases on their Android phone
2. User tells you their email (or you see the purchase in Play Console)
3. Go to Firebase Console → Authentication → find their UID
4. Go to Firestore → `users` collection → their UID document
5. Set:
   ```
   tier: "PREMIUM"   (or "LIFETIME")
   expiresAt: 1893456000   (for PREMIUM — year 2030)
   expiresAt: 9999999999   (for LIFETIME)
   ```
6. User opens app on Fire TV, logs in → `refreshEntitlement` fires → gets correct tier

> Future improvement: Build the `syncPlayPurchase` Cloud Function that auto-syncs
> Google Play purchase tokens to Firestore. This removes the manual step.
> Documented in DOCS/FIREBASE_LICENSING_BACKEND.md.

---

## Step 9 — Full Store Listing (Before Public Launch)

Required before you can publish to Production:

1. **Store listing** → fill in:
   - Short description (80 chars max)
   - Full description (4000 chars max)
   - App icon: 512x512 PNG
   - Feature graphic: 1024x500 PNG
   - Screenshots: at least 2, for phone AND tablet AND TV

2. **Content rating** → complete the questionnaire (takes 5 min)

3. **Target audience** → select age group

4. **App access** → select "All functionality is available without special access"
   (or explain how testers can access)

5. **Ads** → declare whether your app contains ads (it doesn't)

6. **Data safety** → fill in what data you collect:
   - Email address (Firebase Auth) — collected, user-provided, for account management
   - Purchase history (Google Play) — collected, for app functionality
   - No data sold to third parties

---

## Step 10 — Go Live

Once store listing is complete and you have tested purchases:

1. Play Console → **Production** → **Create new release**
2. Upload your signed APK/AAB
3. Write release notes
4. **Review release** → **Start rollout to Production**
5. Google reviews new apps — typically **1-3 days** for first submission

---

## Quick Reference — Product IDs

These are hardcoded in the app. Do NOT change them in Play Console.

```
purefusion_monthly    — Subscription, Monthly billing
purefusion_yearly     — Subscription, Yearly billing
purefusion_lifetime   — In-app product, One-time purchase
```

---

## Quick Reference — Pricing Suggestions

| Plan | Suggested Price | Notes |
|------|----------------|-------|
| Monthly | $4.99/mo | Entry point |
| Yearly | $34.99/yr | ~42% saving vs monthly |
| Lifetime | $59.99 | ~1 year payback |

Adjust to whatever you feel is right. You can change prices later without
changing product IDs.

---

## Troubleshooting

**Products show "—" price in app:**
- Products not yet activated in Play Console, OR
- App installed via sideload instead of Play Store, OR
- BillingClient not yet connected (wait a few seconds and reopen screen)

**"This item is not available" error:**
- App package name in Play Console must match `applicationId` in `build.gradle.kts`
- Current app ID: `dev.eliminater.purefusioniptv`

**Purchases don't unlock tier:**
- Check Logcat for BillingManager / SubscriptionRepository logs
- Ensure you are signed in to the same Google account that made the purchase

**Fire TV not getting updated tier after phone purchase:**
- Manual Firestore update required until auto-sync is built (see Step 8)
