# PureFusionIPTV — Where We Are & What To Do Tomorrow

A plain-English summary of everything built so far and exactly what to do next.

**Last updated:** 2026-02-28

---

## What The App Is

An Android TV / Fire TV IPTV player. Think TiViMate, but yours.

Targets:
- Android TV (Google Play Store)
- Amazon Fire TV / Firestick (sideload or Amazon Appstore)
- Any Android phone/tablet (sideload or Play Store)

Users pay once on their Android phone via Google Play. They then log in on their
Fire TV or any other device and their subscription follows them automatically.

---

## How The Subscription System Works

This is the most important thing to understand.

```
User buys Monthly / Yearly / Lifetime on Android phone (Google Play)
        ↓
BillingManager confirms purchase — tier active immediately on that device
        ↓
[Future] syncPlayPurchase Cloud Function verifies purchase and writes to Firestore
        ↓
User logs into any other device (Fire TV, tablet, sideloaded phone)
        ↓
refreshEntitlement() calls Cloud Function → reads Firestore → issues signed JWT
        ↓
JWT proves their tier on that device for 24 hours without hitting the server again
```

**Right now, the "Future" step is manual** — after a user buys, you go into
Firebase Console and set their Firestore tier by hand. Fine for 10 beta testers.
Not fine for 1000 users. That's the next big thing to build after Play account is live.

### The Three Tiers

| Tier | How to get it | What it unlocks |
|------|--------------|-----------------|
| FREE | Default, no login needed | 1 playlist, 3-day EPG, basic playback |
| PREMIUM | Monthly or Yearly Google Play subscription | Everything — unlimited playlists, 14-day EPG, multi-view, plugins, recordings, cloud sync |
| LIFETIME | One-time Google Play purchase | Same as PREMIUM, forever |

### Product IDs (hardcoded in app — do not change)

```
purefusion_monthly    → Monthly subscription
purefusion_yearly     → Yearly subscription
purefusion_lifetime   → One-time lifetime purchase
```

---

## What Is Built (The Big Picture)

### Core Player
- ExoPlayer / Media3 based
- FastZap system for sub-250ms channel switching
- Dual player warm pool (one playing, one preloading)
- Surface reuse to avoid flicker on zap
- Adaptive bitrate and LoadControl tuning
- Frame rate auto-switching for 4K

### EPG / TV Guide
- Full 7-14 day grid EPG
- Timeline scrolling, category sidebar, program details
- Optimised for 50,000+ entries without memory issues

### Multi-View
- Infrastructure built to show 2-4 channels simultaneously
- Feature-gated (PREMIUM only)
- Full UI integration is the next UI work item

### Settings
- General (account, backup, startup options)
- Playback (player settings, buffer tuning)
- Appearance (themes, colors, transparency)
- EPG (source, refresh, display options)
- Parental controls (settings exist, PIN UI not built yet)
- Recording (settings exist, native DVR not built)
- Plugins (SDK exists, marketplace UI not built)
- Remote control (D-pad customisation)

### Security
- R8 + ProGuard obfuscation on release builds
- NDK native license validator (detects Frida, debuggers, emulators)
- License anomaly detection (flags token reuse, rapid tier switching)
- Encrypted SharedPreferences for all subscription state (AES-256-GCM)
- Firebase JWT for cross-device licensing (RS256, 24h TTL)
- Firestore rules: clients can NEVER write their own tier

### Billing
- Google Play Billing v7 fully integrated
- Subscription screen with real prices from Play
- Restore purchases button (required by Play policy)
- Tier flows: FREE → PREMIUM → LIFETIME

### Firebase
- Authentication: email/password login and registration
- Firestore: user tier storage (`users/{uid}` documents)
- Cloud Functions: `refreshEntitlement`, `revokeDevice`, `setUserTier`
- FCM: push to force revalidation or revoke license instantly
- Crashlytics: crash reporting with subscription context stamped on every crash
- Remote Config: feature flags, A/B infrastructure, kill switches

### First Run Flow
```
App installs
    ↓
SetupWizardActivity checks: is user logged in?
    ↓ No
LoginActivity — sign in or create account, or skip
    ↓
SetupWizardActivity — add playlist, restore backup, cloud sync, etc.
    ↓
MainActivity — main player
```

### Settings → General → Account
Shows the logged-in user's email, their current tier, and links to:
- Subscription & Upgrade screen (the purchase screen)
- Sign in / Create account
- Sign out

---

## What To Do Tomorrow (In Order)

### 1. Create Google Play Developer Account ($25)

Go to: https://play.google.com/console

- Pay $25 one-time fee
- Fill in developer profile (name that users see on store)
- Account verified usually within a few hours, sometimes up to 48h

Full instructions: `DOCS/GOOGLE_PLAY_SETUP.md` Step 1

---

### 2. Create Your App In Play Console

- App name: `PureFusion IPTV`
- Package: `dev.eliminater.purefusioniptv` (must match exactly)
- Free app with in-app purchases
- Language: English

Full instructions: `DOCS/GOOGLE_PLAY_SETUP.md` Step 2

---

### 3. Create The Three Products

This is critical. The product IDs in Play Console must match the app exactly.

**Subscriptions** (Monetize → Subscriptions):
- `purefusion_monthly` — Monthly, auto-renewing
- `purefusion_yearly` — Yearly, auto-renewing

**In-app product** (Monetize → In-app products):
- `purefusion_lifetime` — One-time purchase

Suggested prices (you decide):
- Monthly: $4.99/mo
- Yearly: $34.99/yr (saves ~42% vs monthly)
- Lifetime: $59.99 (one-time)

Full instructions: `DOCS/GOOGLE_PLAY_SETUP.md` Step 3

---

### 4. Set Up Merchant Account

Play Console → Setup → Payments profile

Enter your bank account so Google can pay you. Takes 1-3 days to verify.

Full instructions: `DOCS/GOOGLE_PLAY_SETUP.md` Step 4

---

### 5. Sign Your APK

You need a signed release APK before you can upload to Play Console.

Option A (recommended): Use Google Play App Signing — Google holds the master key,
you create an upload key. Safest option.

Option B: Self-managed keystore — you hold the `.jks` file, do NOT lose it.

Full instructions: `DOCS/GOOGLE_PLAY_SETUP.md` Step 6

---

### 6. Upload To Internal Testing

Build and upload your APK to the Internal Testing track. This activates your
product IDs so billing actually works during testing.

```bash
# Build release APK:
cd I:/GITHUB/Projects/PureFusionIPTV
./gradlew assembleRelease
# APK: app/build/outputs/apk/release/app-release.apk
```

Upload via Play Console → Testing → Internal testing → Create new release

Full instructions: `DOCS/GOOGLE_PLAY_SETUP.md` Step 5

---

### 7. Give Your 10 Testers Lifetime Access

For each tester:
1. Have them install the app (via Internal Testing link or sideload APK)
2. Have them create an account in the app (email/password)
3. They tell you their email address
4. You go to Firebase Console → Authentication → find their UID
5. Firebase Console → Firestore → `users` collection → their UID
6. Set or create the document:
   ```
   tier: "LIFETIME"
   expiresAt: 9999999999
   deviceLimit: 5
   anomalyScore: 0
   ```
7. They reopen the app → Settings → General → Account → tier shows LIFETIME

Full instructions: `DOCS/GOOGLE_PLAY_SETUP.md` Step 7 + 8

---

### 8. Test Purchases Yourself

Once products are created and APK is on Internal Testing track:

1. Add yourself to Play Console → Setup → License testing (your Gmail)
2. Install app via the Internal Testing opt-in link
3. Open Settings → General → Account → Subscription & Upgrade
4. Prices should load from Google Play
5. Tap a plan — Play purchase sheet appears
6. Complete purchase — you are NOT charged (license tester)
7. Verify tier updates in app

Full instructions: `DOCS/GOOGLE_PLAY_SETUP.md` Step 7

---

## The One Big Gap To Fix After Play Account Is Live

**Cross-device sync after Play purchase.**

Right now:
- Buy on phone → works on phone ✅
- Buy on phone → auto-updates Fire TV ❌ (manual Firestore edit needed)

To fix this properly, we need to build `syncPlayPurchase` Cloud Function.

This requires:
- Google Play Developer API credentials (Play Console → Setup → API access)
- Service account JSON linked to your Cloud project
- New Cloud Function added to `functions/src/index.ts`

Once your Play account exists, we can build this in one session.

See: `DOCS/PRODUCTION_READINESS_TODO.md` Section 2 for technical details.

---

## What To Work On Today (No Play Account Needed)

Since Play account is tomorrow, here are things that can be done right now:

### Option A — MultiView UI Integration
MultiView infrastructure is built but not fully wired into the main UI.
Users can't actually activate it yet from the player screen.
High impact — this is a key differentiator vs TiViMate.

### Option B — Sprint 6: Remote Config Enforcement
Wire the enforcement flags (`force_online_validation`, `grace_period_hours`, etc.)
into `SubscriptionRepository`. Makes the security model complete.

### Option C — Parental Controls UI
Build the PIN entry/setup screen and channel filtering.
Settings storage exists, just needs the UI layer.

### Option D — App Icon + Splash Screen
You'll need these for the Play Store listing anyway.
Get the visual identity sorted before the store listing rush.

### Option E — Testing: Sideload The APK
Build the debug APK, install on a real device or Fire TV stick,
walk through the complete user flow end to end:
- First launch → wizard → login → add playlist → watch → settings → subscription screen

This will surface real UX issues before you go to the store.

---

## Document Index

| Document | Read it when... |
|----------|----------------|
| `DOCS/SESSION_SUMMARY.md` | This file — start here |
| `DOCS/PRODUCTION_READINESS_TODO.md` | Checking what's done vs pending |
| `DOCS/GOOGLE_PLAY_SETUP.md` | Setting up Play Console tomorrow |
| `DOCS/FIREBASE_LICENSING_BACKEND.md` | Understanding the Firebase/JWT system |
| `DOCS/AMAZON_FIRETV_STRATEGY.md` | Fire TV distribution decisions |
| `DOCS/DEPLOYMENT_GUIDE.md` | How to build and deploy APKs |
| `DOCS/FASTZAP_ARCHITECTURE.md` | How the fast channel switching works |
| `DOCS/PLUGIN_SDK.md` | Plugin system for third-party developers |
| `DOCS/EMBY_PLUGIN.md` | Emby media server integration |

---

## Key Numbers / IDs To Know

| Thing | Value |
|-------|-------|
| App package name | `dev.eliminater.purefusioniptv` |
| Firebase project | `purefusioniptv` |
| FCM Sender ID | `706920887579` |
| Firebase account | eliminater74@gmail.com |
| Monthly product ID | `purefusion_monthly` |
| Yearly product ID | `purefusion_yearly` |
| Lifetime product ID | `purefusion_lifetime` |
| Firestore collection | `users/{uid}` |
| Cloud Functions region | `us-central1` |

---

## Security Rules (Do Not Change These)

Firestore rules — clients can ONLY read their own document, NEVER write:
```js
match /users/{userId} {
  allow read: if request.auth.uid == userId;
  allow write: if false;  // Only Cloud Functions can write
}
```

If you change `allow write: if false` to anything else, a modded APK can
set itself to LIFETIME in 30 seconds. This rule stays as-is forever.
Only the `setUserTier` Cloud Function (admin-gated) can modify tiers.
