# Firebase Licensing Backend — Complete Setup Guide

**Firebase project:** `purefusioniptv` (already exists, google-services.json present)
**Firebase BOM:** 33.11.0 (already in build.gradle.kts)

This document is the complete step-by-step guide to turn your existing Firebase project
into a real licensing backend that works for:
- Google Play Store
- Amazon Appstore / Fire TV
- Sideloaded APKs
- Any Android device, Google-less or not

---

## Architecture Overview

```
User Device (Android/Fire TV)
    │
    ├─ Firebase Auth (login — works everywhere including Fire TV)
    │
    └─ Cloud Function: refreshEntitlement()
            │
            ├─ Reads Firestore: users/{uid}/tier, expiresAt
            │
            └─ Returns signed JWT (RS256, 24h TTL)
                    │
                    └─ App verifies JWT locally with embedded public key
                            │
                            └─ SubscriptionRepository trusts verified tier
```

The app **never trusts the store directly**. All platforms (Play, Amazon, sideload)
flow through the same JWT entitlement system. The store is just a payment processor.

---

## What Is Already Done

- [x] Firebase project `purefusioniptv` exists
- [x] `google-services.json` present at `app/google-services.json`
- [x] Firebase BOM 33.11.0 in `app/build.gradle.kts`
- [x] Crashlytics + Analytics + Remote Config active
- [x] `SubscriptionRepository` uses encrypted SharedPreferences (Sprint 5)
- [x] `LicenseAnomalyDetector` wired (Sprint 4)
- [x] `ReceiptValidator` skeleton exists (Sprint 4) — needs JWT path added

## What Still Needs To Be Done

- [ ] Enable Firebase Authentication in Console
- [ ] Enable Firestore in Console
- [ ] Add Auth + Firestore + Functions + Messaging dependencies
- [ ] Initialize Firebase CLI and create `functions/` folder
- [ ] Generate RSA keypair for JWT signing
- [ ] Write and deploy Cloud Functions
- [ ] Embed public key in Android app
- [ ] Wire JWT verification into `SubscriptionRepository`
- [ ] Add login UI (email/password minimum)
- [ ] (Optional) Stripe webhook function for web payments

---

## PART 1 — Firebase Console Setup

### Step 1 — Enable Authentication

1. Go to https://console.firebase.google.com/project/purefusioniptv
2. Left sidebar → **Authentication** → **Get started**
3. **Sign-in method** tab → Enable **Email/Password**
4. Optionally enable **Google** sign-in (good for Play Store users)
5. Save

### Step 2 — Enable Firestore

1. Left sidebar → **Firestore Database** → **Create database**
2. Choose **Production mode** (you will set rules in Step 3)
3. Choose region closest to your users (e.g., `us-central` or `europe-west`)
4. Click **Done**

### Step 3 — Set Firestore Security Rules

In Firestore → **Rules** tab, replace the default with:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users can read their own doc. Only Cloud Functions can write.
    match /users/{userId} {
      allow read: if request.auth != null && request.auth.uid == userId;
      allow write: if false;
    }

    // Devices are server-managed only
    match /devices/{deviceId} {
      allow read, write: if false;
    }

    // Revocation list — readable by any authenticated user
    match /revoked/{tokenId} {
      allow read: if request.auth != null;
      allow write: if false;
    }
  }
}
```

**Publish** the rules.

### Step 4 — Enable Cloud Messaging (FCM)

1. Left sidebar → **Cloud Messaging**
2. It should already be enabled — verify it shows your **Sender ID**
3. Note your **Server key** (needed for sending push from backend)

FCM works on Fire TV (Fire OS 5+). This is how you push license revocations
and force-revalidation commands to devices without waiting for Remote Config TTL.

---

## PART 2 — Android App Dependencies

Open `app/build.gradle.kts` and add inside the `dependencies` block:

```kotlin
// Firebase Auth (login — works on all Android including Fire TV)
implementation("com.google.firebase:firebase-auth-ktx")

// Firestore (optional — only needed if app reads Firestore directly)
// implementation("com.google.firebase:firebase-firestore-ktx")

// Cloud Functions (call refreshEntitlement)
implementation("com.google.firebase:firebase-functions-ktx")

// FCM (push notifications / license revocation)
implementation("com.google.firebase:firebase-messaging-ktx")

// JWT verification (verify signed entitlement tokens)
implementation("io.jsonwebtoken:jjwt-api:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")
```

All of these use the existing Firebase BOM — no version numbers needed for Firebase libs.

Sync Gradle after adding.

---

## PART 3 — Generate RSA Keypair (One Time)

You need an RSA keypair for signing JWTs. The private key lives in Cloud Functions
(server-side only). The public key is embedded in the Android app.

### On Windows (Git Bash):

```bash
# Generate 2048-bit RSA private key
openssl genrsa -out purefusion_license_private.pem 2048

# Extract public key
openssl rsa -in purefusion_license_private.pem -pubout -out purefusion_license_public.pem

# View the public key (you'll embed this in the app)
cat purefusion_license_public.pem
```

**CRITICAL:**
- `purefusion_license_private.pem` → goes into Cloud Functions environment. NEVER commit to git.
- `purefusion_license_public.pem` → embed in Android app (it's public, safe to include)

Store the private key in a password manager (Bitwarden, 1Password, etc.) as a backup.

---

## PART 4 — Firebase CLI and Cloud Functions

### Step 1 — Install Firebase CLI (one time)

```bash
npm install -g firebase-tools
```

Verify:
```bash
firebase --version
```

### Step 2 — Login

```bash
firebase login
```

This opens a browser. Log in with the Google account that owns `purefusioniptv`.

### Step 3 — Initialize Functions in your project

```bash
cd I:\GITHUB\Projects\PureFusionIPTV
firebase init functions
```

Prompts:
- **Use existing project** → select `purefusioniptv`
- **Language** → TypeScript
- **ESLint** → Yes (recommended)
- **Install dependencies now** → Yes

This creates:
```
functions/
    src/
        index.ts       ← your function code goes here
    package.json
    tsconfig.json
firebase.json
.firebaserc
```

### Step 4 — Install JWT library

```bash
cd functions
npm install jsonwebtoken @types/jsonwebtoken
```

### Step 5 — Set your private key as a Firebase secret

```bash
firebase functions:secrets:set LICENSE_PRIVATE_KEY
```

Paste the entire contents of `purefusion_license_private.pem` when prompted
(including the `-----BEGIN PRIVATE KEY-----` header and footer lines).

This stores the key encrypted in Google Cloud Secret Manager.
It is never in your code or git history.

### Step 6 — Write the Cloud Functions

Replace `functions/src/index.ts` with:

```typescript
import { onCall, HttpsError } from "firebase-functions/v2/https";
import { defineSecret } from "firebase-functions/params";
import * as admin from "firebase-admin";
import * as jwt from "jsonwebtoken";

admin.initializeApp();

const PRIVATE_KEY = defineSecret("LICENSE_PRIVATE_KEY");

// ─── refreshEntitlement ───────────────────────────────────────────────────────
// Called by the Android app after login.
// Returns a signed JWT with the user's current tier and 24h expiry.
export const refreshEntitlement = onCall(
  { secrets: [PRIVATE_KEY] },
  async (request) => {
    if (!request.auth) {
      throw new HttpsError("unauthenticated", "Login required");
    }

    const uid = request.auth.uid;
    const db = admin.firestore();

    // Fetch user document
    const userDoc = await db.collection("users").doc(uid).get();

    if (!userDoc.exists) {
      // Auto-create free tier on first call
      await db.collection("users").doc(uid).set({
        tier: "FREE",
        expiresAt: 0,
        deviceLimit: 2,
        anomalyScore: 0,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        source: "none",
      });
    }

    const userData = (await db.collection("users").doc(uid).get()).data()!;

    // Check if subscription is actually active
    const now = Math.floor(Date.now() / 1000);
    const isActive = userData.expiresAt === 0 || userData.expiresAt > now;
    const tier = isActive ? (userData.tier ?? "FREE") : "FREE";

    // Issue signed JWT (24 hour TTL)
    const payload = {
      uid: uid,
      tier: tier,
      iat: now,
      exp: now + (60 * 60 * 24), // 24 hours
    };

    const token = jwt.sign(payload, PRIVATE_KEY.value(), { algorithm: "RS256" });

    return { token, tier, expiresAt: payload.exp };
  }
);

// ─── revokeDevice ─────────────────────────────────────────────────────────────
// Admin use only — revoke a specific device fingerprint.
export const revokeDevice = onCall(
  { secrets: [PRIVATE_KEY] },
  async (request) => {
    // Only callable from your admin dashboard (check admin claim)
    if (!request.auth?.token?.admin) {
      throw new HttpsError("permission-denied", "Admin only");
    }
    const { deviceId } = request.data;
    await admin.firestore().collection("revoked").doc(deviceId).set({
      revokedAt: admin.firestore.FieldValue.serverTimestamp(),
    });
    return { success: true };
  }
);

// ─── stripeWebhook ────────────────────────────────────────────────────────────
// Stripe calls this when a subscription is created, renewed, or cancelled.
// See STRIPE_SETUP.md for configuration.
// Uncomment when Stripe is configured.
//
// export const stripeWebhook = onRequest(async (req, res) => {
//   const sig = req.headers["stripe-signature"] as string;
//   const event = stripe.webhooks.constructEvent(req.rawBody, sig, STRIPE_WEBHOOK_SECRET);
//   if (event.type === "customer.subscription.created" ||
//       event.type === "customer.subscription.updated") {
//     const sub = event.data.object as Stripe.Subscription;
//     const uid = sub.metadata.firebase_uid;
//     const tier = sub.metadata.tier ?? "PREMIUM";
//     const expiresAt = sub.current_period_end;
//     await admin.firestore().collection("users").doc(uid).update({
//       tier, expiresAt, source: "stripe"
//     });
//   }
//   if (event.type === "customer.subscription.deleted") {
//     const sub = event.data.object as Stripe.Subscription;
//     const uid = sub.metadata.firebase_uid;
//     await admin.firestore().collection("users").doc(uid).update({
//       tier: "FREE", expiresAt: 0
//     });
//   }
//   res.json({ received: true });
// });
```

### Step 7 — Deploy

```bash
firebase deploy --only functions
```

Your function URL will be printed:
```
https://us-central1-purefusioniptv.cloudfunctions.net/refreshEntitlement
```

---

## PART 5 — Android App Integration

### Files to create/modify:

**1. Add public key as a raw resource**

Create file: `app/src/main/res/raw/license_public_key.pem`

Paste the contents of `purefusion_license_public.pem` into this file.

**2. Create `EntitlementManager.kt`**

Location: `app/src/main/java/dev/eliminater/purefusioniptv/billing/EntitlementManager.kt`

```kotlin
@Singleton
class EntitlementManager @Inject constructor(
    @ApplicationContext private val context: Context,
    @SubscriptionPrefs private val prefs: SharedPreferences
) {
    companion object {
        private const val KEY_JWT = "entitlement_jwt"
        private const val KEY_TIER = "entitlement_tier"
        private const val KEY_EXPIRES = "entitlement_expires"
    }

    private val functions = Firebase.functions

    // Call this after login, and periodically (e.g., app foreground)
    suspend fun refreshEntitlement(): EntitlementResult = withContext(Dispatchers.IO) {
        val currentUser = Firebase.auth.currentUser
            ?: return@withContext EntitlementResult.NotLoggedIn

        try {
            val result = functions
                .getHttpsCallable("refreshEntitlement")
                .call()
                .await()

            val data = result.data as Map<*, *>
            val token = data["token"] as String
            val tier = data["tier"] as String
            val expiresAt = (data["expiresAt"] as Number).toLong()

            // Verify the JWT signature before trusting it
            if (!verifyJwt(token)) {
                Timber.e("EntitlementManager: JWT signature verification FAILED")
                return@withContext EntitlementResult.InvalidToken
            }

            // Store in encrypted prefs
            prefs.edit()
                .putString(KEY_JWT, token)
                .putString(KEY_TIER, tier)
                .putLong(KEY_EXPIRES, expiresAt)
                .apply()

            EntitlementResult.Success(tier = SubscriptionTier.fromString(tier))
        } catch (e: Exception) {
            Timber.w(e, "EntitlementManager: refresh failed, using cached")
            loadCached()
        }
    }

    fun loadCached(): EntitlementResult {
        val tier = prefs.getString(KEY_TIER, "FREE") ?: "FREE"
        val expires = prefs.getLong(KEY_EXPIRES, 0L)
        val nowSec = System.currentTimeMillis() / 1000

        return if (expires == 0L || expires > nowSec) {
            EntitlementResult.Success(SubscriptionTier.fromString(tier))
        } else {
            EntitlementResult.Expired
        }
    }

    private fun verifyJwt(token: String): Boolean = try {
        val keyStream = context.resources.openRawResource(R.raw.license_public_key)
        val pemContent = keyStream.bufferedReader().readText()
        val keyBytes = Base64.decode(
            pemContent.replace("-----BEGIN PUBLIC KEY-----", "")
                      .replace("-----END PUBLIC KEY-----", "")
                      .replace("\n", "").trim(),
            Base64.DEFAULT
        )
        val keySpec = X509EncodedKeySpec(keyBytes)
        val publicKey = KeyFactory.getInstance("RSA").generatePublic(keySpec)
        val parser = Jwts.parser().verifyWith(publicKey as java.security.PublicKey).build()
        parser.parseSignedClaims(token)
        true
    } catch (e: Exception) {
        Timber.e(e, "JWT verification failed")
        false
    }
}

sealed class EntitlementResult {
    data class Success(val tier: SubscriptionTier) : EntitlementResult()
    data object NotLoggedIn : EntitlementResult()
    data object InvalidToken : EntitlementResult()
    data object Expired : EntitlementResult()
}
```

**3. Wire into `SubscriptionRepository`**

In `SubscriptionRepository`, after `processPurchases()` succeeds, call
`entitlementManager.refreshEntitlement()` to sync the server-issued tier.
The server tier overrides the local Play Billing tier — server is the authority.

---

## PART 6 — Firestore User Document Management

You manage user tiers directly in Firestore Console or via admin scripts.

### Manually upgrade a user to PREMIUM (testing):
1. Firebase Console → Firestore → `users` collection
2. Find the document with the user's UID
3. Update field `tier` to `"PREMIUM"`
4. Update field `expiresAt` to a Unix timestamp (e.g., `9999999999` for permanent)

### Find a user's UID:
Firebase Console → Authentication → Users tab → find by email → copy UID

### Revoke a user:
Set `tier` to `"FREE"` and `expiresAt` to `0`. On next app launch, the JWT
refresh will return FREE tier.

---

## PART 7 — FCM Push for Instant Revocation

When you need to revoke a license immediately (don't want to wait for 24h JWT expiry):

### From your backend / Firebase Console:

```bash
# Send a "force_revalidate" push to a specific device token
curl -X POST https://fcm.googleapis.com/fcm/send \
  -H "Authorization: key=YOUR_SERVER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "DEVICE_FCM_TOKEN",
    "data": {
      "type": "force_revalidate",
      "reason": "subscription_cancelled"
    }
  }'
```

### In Android, handle in `FirebaseMessagingService`:

```kotlin
class PureFusionMessagingService : FirebaseMessagingService() {
    override fun onMessageReceived(message: RemoteMessage) {
        when (message.data["type"]) {
            "force_revalidate" -> {
                // Trigger EntitlementManager.refreshEntitlement()
                // This will hit Cloud Functions and get updated tier
            }
            "license_revoked" -> {
                // Clear cached tier, force FREE immediately
            }
        }
    }
}
```

Register in AndroidManifest.xml:
```xml
<service
    android:name=".PureFusionMessagingService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

---

## PART 8 — Stripe Integration (When Ready)

When you want to accept web payments (works for all platforms including Fire TV):

1. Create account at https://stripe.com
2. Get your API keys from Stripe Dashboard
3. Create products and prices in Stripe
4. Build a simple checkout page (or use Stripe Payment Links — no code needed)
5. In the checkout metadata, include `firebase_uid` (get this after Firebase login)
6. Uncomment the `stripeWebhook` function in `functions/src/index.ts`
7. Add Stripe secret and webhook secret to Firebase secrets:
   ```bash
   firebase functions:secrets:set STRIPE_SECRET_KEY
   firebase functions:secrets:set STRIPE_WEBHOOK_SECRET
   ```
8. Register the webhook URL in Stripe Dashboard:
   ```
   https://us-central1-purefusioniptv.cloudfunctions.net/stripeWebhook
   ```
9. Events to listen for:
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`

Stripe Payment Links require zero backend code on your website — Stripe hosts
the checkout page. You just send the user to the link.

---

## PART 9 — Key Rotation

JWT keys should be rotated periodically (or after any security incident).

### Rotation procedure:
1. Generate new RSA keypair (same openssl commands as Part 3)
2. Add new private key as a NEW Firebase secret: `LICENSE_PRIVATE_KEY_V2`
3. Update Cloud Function to sign with new key
4. Update `app/src/main/res/raw/license_public_key.pem` with new public key
5. Deploy Cloud Functions
6. Release app update
7. Old tokens expire naturally within 24h
8. After 24h, remove old secret

---

## Summary: Priority Order

1. **Enable Auth + Firestore in Console** — 10 minutes
2. **Set Firestore security rules** — 5 minutes
3. **Generate RSA keypair** — 2 minutes
4. **Install Firebase CLI, init functions** — 15 minutes
5. **Deploy Cloud Functions** — 10 minutes
6. **Add dependencies to Android app** — 5 minutes
7. **Create EntitlementManager.kt** — implement JWT verification
8. **Wire into SubscriptionRepository** — server tier overrides store tier
9. **Add login screen** — minimum viable: email/password
10. **Add FCM handler** — for instant revocation
11. **Stripe** — when ready to take payments

Steps 1–6 can be done in under an hour and give you a working backend
immediately, before a single line of Android app code changes.
