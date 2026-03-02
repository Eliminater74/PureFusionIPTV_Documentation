# Amazon Fire TV / Fire OS Strategy

## The Core Problem

Fire TV runs Fire OS — an Android fork with no Google Play Services.

| Feature | Google Play Android | Amazon Fire TV |
|---------|--------------------|-|
| Google Play Billing | ✅ Works | ❌ Not available |
| Google Play Integrity API | ✅ Works | ❌ Not available |
| Play Developer API (receipt check) | ✅ Works | ❌ Not available |
| Firebase Auth | ✅ Works | ✅ Works (Fire OS 5+) |
| Firebase Crashlytics | ✅ Works | ✅ Works (Fire OS 5+) |
| Firebase Remote Config | ✅ Works | ✅ Works (Fire OS 5+) |
| Firebase Cloud Messaging (FCM) | ✅ Works | ✅ Works (Fire OS 5+) |
| ExoPlayer / Media3 | ✅ Works | ✅ Works |
| Room, Hilt, Coroutines | ✅ Works | ✅ Works |
| Jetpack Security (EncryptedSharedPrefs) | ✅ Works | ✅ Works |
| Amazon IAP SDK | ❌ Not available | ✅ Works |
| D-pad / remote navigation | Not required | Required |

**Bottom line:** If you use the Firebase JWT entitlement model (see FIREBASE_LICENSING_BACKEND.md),
licensing works identically on both platforms. The store becomes irrelevant to your app's
internal license logic.

---

## Distribution Strategy Decision

You need to decide before writing billing code:

### Option A — Google Play only
- Simplest. Use existing `BillingManager` as-is.
- Fire TV devices cannot install from Play Store.
- You lose the entire Fire TV market.
- No changes needed.

### Option B — Google Play + Amazon Appstore
- Requires: billing abstraction layer + Amazon IAP SDK
- Users on Fire TV buy through Amazon Appstore
- Users on phones/tablets buy through Google Play
- Both purchases → your Firebase backend → same JWT entitlement
- More work but correct approach for Fire TV market

### Option C — Sideload / Direct APK (no store)
- No store billing at all
- Payments via your website (Stripe)
- User logs in → Firebase Auth → gets JWT
- Works on every Android device including Fire TV
- Most piracy risk, but also most flexibility
- Correct if you want to avoid 30% store cut

### Option D — All three (Play + Amazon + Sideload)
- Best market coverage
- Firebase JWT entitlement unifies them all
- Each "store" is just a different way to fund the Firestore `tier` field
- Sideload users pay via web → Stripe webhook updates Firestore
- Play users pay in-app → your server validates with Google, updates Firestore
- Amazon users pay in-app → your server validates with Amazon RVS, updates Firestore

**Recommendation: Start with Option C (sideload/web payments via Stripe), then add
Play and Amazon as optional purchase paths later.** This ships fastest, works on all
devices, and the architecture is identical regardless of which stores you add later.

---

## What Breaks Right Now on Fire TV

### `BillingManager.kt` — Will silently fail
`BillingClient.newBuilder(context).build()` will fail to connect on Fire TV because
there is no Play Store service. Current behavior:
- `BillingManager` connection fails
- `SubscriptionRepository` loads cached encrypted state
- After cache TTL expires → user appears as FREE

**This is acceptable short-term** if you add Firebase JWT entitlement. The JWT path
bypasses `BillingManager` entirely. Fire TV users who paid via Stripe/web will get
their correct tier from Firebase regardless of Play Billing state.

### `ReceiptValidator.kt` — Will skip (acceptable)
When `receiptValidationUrl` is empty, `ReceiptValidator` returns `ValidationResult.Skipped`.
This is already the correct behavior.

When you add a backend, your validation endpoint needs:
```
POST /validateReceipt
Body: { store: "google" | "amazon" | "web", purchaseToken: string, productId: string }
```

---

## Amazon IAP SDK Integration (When You're Ready)

This is future work — do not implement until Firebase JWT entitlement is working.

### Step 1 — Add Amazon IAP SDK

Amazon's IAP SDK is not on Maven Central. You must download it manually:
1. Go to https://developer.amazon.com/docs/in-app-purchasing/iap-install-and-configure-sdk.html
2. Download the SDK JAR
3. Place in `app/libs/in-app-purchasing-2.0.76.jar`
4. Add to `app/build.gradle.kts`:
   ```kotlin
   implementation(files("libs/in-app-purchasing-2.0.76.jar"))
   ```

### Step 2 — Create AmazonBillingProvider

```kotlin
interface BillingProvider {
    fun startConnection(onReady: () -> Unit, onFailed: () -> Unit)
    fun querySubscriptions(onResult: (List<PurchaseInfo>) -> Unit)
    fun launchPurchaseFlow(activity: Activity, productId: String)
}

class GoogleBillingProvider @Inject constructor(...) : BillingProvider { ... }
class AmazonBillingProvider @Inject constructor(...) : BillingProvider { ... }
```

### Step 3 — Detect store at runtime

```kotlin
fun isAmazonDevice(): Boolean {
    return try {
        Class.forName("com.amazon.device.iap.PurchasingService")
        true
    } catch (e: ClassNotFoundException) {
        false
    }
}
```

### Step 4 — Amazon Receipt Verification Service (RVS)

Amazon RVS endpoint:
```
GET https://appstore-sdk.amazon.com/version/1.0/verifyReceiptId/developer/{developerId}/user/{userId}/receiptId/{receiptId}
```

Headers:
- `Authorization: Basic <base64(developer_secret)>`

Response: JSON with `betaProduct`, `cancelDate`, `gracePeriodEndDate`, `parentProductId`,
`productId`, `productType`, `purchaseDate`, `quantity`, `receiptId`, `renewalDate`, `term`,
`termSku`, `testTransaction`

Your backend calls this to verify Amazon purchases, then updates Firestore the same way
as Google purchases.

---

## Amazon Appstore Submission Checklist

When ready to submit to Amazon Appstore:

1. **APK must not reference Google Play Billing SDK classes directly in the manifest**
   - If you have `BillingManager` compiled in, this is fine — Amazon won't reject it
   - But you must handle `BillingClient` connection failures gracefully (already done)

2. **Remove/gate Play-specific features for Amazon builds**
   - Consider a `amazon` build flavor in `build.gradle.kts`
   - Fire TV build can exclude `BillingManager` entirely

3. **D-pad navigation must work for ALL interactive elements**
   - Test every screen with only a D-pad
   - Every button, card, input must be focusable
   - Focus highlight must be clearly visible (check `CertificatePinConfig.kt` focus colors)

4. **No Google branding in UI**
   - Remove "Continue with Google" buttons from any login UI
   - Amazon review rejects apps with Google sign-in prompts shown on Fire TV

5. **APK size**
   - Amazon Appstore accepts APKs up to 500MB
   - Use AAB (Android App Bundle) for Play, APK for Amazon (Amazon doesn't support AAB yet)

6. **Build configuration for separate APK:**
   ```kotlin
   // app/build.gradle.kts
   flavorDimensions += "store"
   productFlavors {
       create("google") {
           dimension = "store"
           // default — Play Store
       }
       create("amazon") {
           dimension = "store"
           applicationIdSuffix = ".amazon"
           buildConfigField("String", "STORE", "\"amazon\"")
       }
   }
   ```

---

## NDK Emulator Detection on Fire TV

`license_validator.cpp` currently detects emulators by checking for:
- `/dev/socket/qemud`
- `/dev/qemu_pipe`

**Risk:** Amazon's internal Fire TV test environment and some Fire TV emulators may
trigger these checks. Before enforcing (not just detecting), test on:
- Real Fire TV Stick 4K
- Fire TV Cube
- Amazon's own test environment if you have access

The current implementation **reports but does not block** — this is correct behavior
until you have verified it does not false-positive on real Fire TV hardware.

To test, check `NativeLicenseValidator.checkEnvironment()` output on a physical Fire TV.
If it returns `EnvironmentStatus.Compromised` on real hardware, adjust the detection
thresholds in `license_validator.cpp`.

---

## Fire TV Specific Android Features

### Input handling
Fire TV uses a physical remote. Ensure:
- All `View`s that should be interactive have `android:focusable="true"`
- `android:nextFocusUp/Down/Left/Right` set correctly for complex layouts
- No dialogs that require touch to dismiss

### Background restrictions
Fire TV aggressively kills background processes to free RAM for the active app.
Your `LicenseAnomalyDetector` and background sync must handle `onTrimMemory` and
process death gracefully. Encrypted SharedPreferences writes with `.apply()` (async)
are safe — they flush before process death.

### 4K / HDR
Fire TV Stick 4K and Fire TV Cube support 4K HDR. `DeviceCapabilityProfiler` will
correctly detect this. Ensure your player respects `supportsHardwareDecode4K` before
attempting 4K streams.

### Audio
Fire TV passes audio through HDMI. Dolby Atmos and DTS passthrough work on supported
hardware. ExoPlayer handles this automatically when the stream contains the correct
audio tracks.
