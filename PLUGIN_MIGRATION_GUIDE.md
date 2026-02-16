# Plugin Migration Guide

## Migrating Google Sync to Plugin

This guide shows how to migrate the existing Google Sync functionality to a plugin using the new Plugin SDK.

---

## Step 1: Create Plugin Module (Separate Gradle Module)

**Why separate module?** Plugins should be independent, optional, and potentially distributed separately.

### Create new module:

```
app/plugins/googledrive/
├── build.gradle.kts
├── src/main/
│   ├── AndroidManifest.xml
│   ├── java/dev/eliminater/purefusioniptv/plugins/googledrive/
│   │   ├── GoogleDrivePlugin.kt
│   │   └── GoogleDriveSyncProvider.kt
│   └── resources/META-INF/services/
│       └── dev.eliminater.purefusioniptv.plugin.Plugin
```

### build.gradle.kts for plugin module:

```kotlin
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
    id("com.google.dagger.hilt.android")
}

android {
    namespace = "dev.eliminater.purefusioniptv.plugins.googledrive"
    compileSdk = 36

    defaultConfig {
        minSdk = 25
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

dependencies {
    // Main app (for plugin interfaces)
    implementation(project(":app"))

    // Google Play Services
    implementation("com.google.android.gms:play-services-auth:21.0.0")
    implementation("com.google.apis:google-api-services-drive:v3-rev20231212-2.0.0")

    // Kotlin
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.0")

    // Hilt
    implementation("com.google.dagger:hilt-android:2.51")
    kapt("com.google.dagger:hilt-compiler:2.51")

    // Timber for logging
    implementation("com.jakewharton.timber:timber:5.0.1")
}
```

---

## Step 2: Identify Code to Migrate

### Current Google Sync code locations:

1. **Authentication** - Where is Google Sign-In handled?
   - Look for: `GoogleSignInClient`, `GoogleSignInAccount`

2. **Drive API** - Where are Drive operations?
   - Look for: `Drive`, `DriveClient`, file upload/download

3. **Sync Logic** - Where is sync orchestrated?
   - Look for: Settings sync, playlist sync, favorites sync

4. **UI** - Where are sync settings/controls?
   - Look for: Sync buttons, sync status displays

---

## Step 3: Implement CloudStorageProvider

### GoogleDrivePlugin.kt:

```kotlin
package dev.eliminater.purefusioniptv.plugins.googledrive

import android.content.Context
import android.content.Intent
import com.google.android.gms.auth.api.signin.GoogleSignIn
import com.google.android.gms.auth.api.signin.GoogleSignInClient
import com.google.android.gms.auth.api.signin.GoogleSignInOptions
import com.google.android.gms.common.api.Scope
import com.google.api.client.googleapis.extensions.android.gms.auth.GoogleAccountCredential
import com.google.api.client.http.javanet.NetHttpTransport
import com.google.api.client.json.gson.GsonFactory
import com.google.api.services.drive.Drive
import com.google.api.services.drive.DriveScopes
import dev.eliminater.purefusioniptv.plugin.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.withContext
import timber.log.Timber
import java.io.InputStream
import java.io.OutputStream

class GoogleDrivePlugin : Plugin, CloudStorageProvider {

    override val id = "dev.eliminater.purefusioniptv.googledrive"
    override val name = "Google Drive Sync"
    override val description = "Sync app data using Google Drive"
    override val version = "1.0.0"
    override val author = "Eliminater74"
    override val minAppVersion = "1.0.0"
    override val capabilities = listOf(PluginCapability.CLOUD_STORAGE)
    override val providerName = "Google Drive"

    private var appContext: Context? = null
    private var signInClient: GoogleSignInClient? = null
    private var driveService: Drive? = null

    override fun onLoad(context: Context) {
        Timber.d("GoogleDrivePlugin: Loading")
        appContext = context.applicationContext

        val signInOptions = GoogleSignInOptions.Builder(GoogleSignInOptions.DEFAULT_SIGN_IN)
            .requestEmail()
            .requestScopes(Scope(DriveScopes.DRIVE_FILE))
            .build()

        signInClient = GoogleSignIn.getClient(context, signInOptions)

        // Check if already signed in
        val account = GoogleSignIn.getLastSignedInAccount(context)
        if (account != null) {
            initializeDriveService(context, account.email ?: "")
        }

        Timber.d("GoogleDrivePlugin: Loaded")
    }

    override fun onUnload() {
        Timber.d("GoogleDrivePlugin: Unloading")
        driveService = null
        signInClient = null
        appContext = null
    }

    override suspend fun isAuthenticated(): Boolean {
        return withContext(Dispatchers.IO) {
            appContext?.let {
                GoogleSignIn.getLastSignedInAccount(it) != null
            } ?: false
        }
    }

    override suspend fun authenticate(context: Context): Intent? {
        return signInClient?.signInIntent
    }

    override suspend fun handleAuthResult(resultCode: Int, data: Intent?): Boolean {
        return withContext(Dispatchers.IO) {
            try {
                val task = GoogleSignIn.getSignedInAccountFromIntent(data)
                val account = task.result
                appContext?.let { initializeDriveService(it, account.email ?: "") }
                true
            } catch (e: Exception) {
                Timber.e(e, "GoogleDrivePlugin: Authentication failed")
                false
            }
        }
    }

    override suspend fun signOut() {
        withContext(Dispatchers.IO) {
            signInClient?.signOut()?.await()
            driveService = null
        }
    }

    override suspend fun getUserDisplayName(): String? {
        return appContext?.let {
            GoogleSignIn.getLastSignedInAccount(it)?.displayName
        }
    }

    override suspend fun getUserEmail(): String? {
        return appContext?.let {
            GoogleSignIn.getLastSignedInAccount(it)?.email
        }
    }

    private fun initializeDriveService(context: Context, accountName: String) {
        val credential = GoogleAccountCredential.usingOAuth2(
            context,
            listOf(DriveScopes.DRIVE_FILE)
        ).apply {
            selectedAccountName = accountName
        }

        driveService = Drive.Builder(
            NetHttpTransport(),
            GsonFactory.getDefaultInstance(),
            credential
        )
            .setApplicationName("PureFusionIPTV")
            .build()
    }

    override suspend fun uploadFile(
        fileName: String,
        folderPath: String,
        inputStream: InputStream,
        mimeType: String,
        progressCallback: ((Float) -> Unit)?
    ): String {
        return withContext(Dispatchers.IO) {
            // TODO: Implement Drive API file upload
            // 1. Find or create folder
            // 2. Upload file with progress tracking
            // 3. Return file ID
            throw NotImplementedError("Upload not yet implemented")
        }
    }

    override suspend fun downloadFile(
        fileId: String,
        outputStream: OutputStream,
        progressCallback: ((Float) -> Unit)?
    ) {
        withContext(Dispatchers.IO) {
            // TODO: Implement Drive API file download
            throw NotImplementedError("Download not yet implemented")
        }
    }

    override suspend fun listFiles(folderPath: String): Flow<CloudFile> = flow {
        // TODO: Implement Drive API file listing
    }

    override suspend fun deleteFile(fileId: String) {
        withContext(Dispatchers.IO) {
            // TODO: Implement Drive API file deletion
        }
    }

    override suspend fun fileExists(fileName: String, folderPath: String): Boolean {
        return withContext(Dispatchers.IO) {
            // TODO: Implement file existence check
            false
        }
    }

    override suspend fun getFileMetadata(fileId: String): CloudFile? {
        return withContext(Dispatchers.IO) {
            // TODO: Implement metadata retrieval
            null
        }
    }

    override suspend fun getStorageQuota(): StorageQuota {
        return withContext(Dispatchers.IO) {
            try {
                val about = driveService?.about()?.get()?.setFields("storageQuota")?.execute()
                val quota = about?.storageQuota
                StorageQuota(
                    totalBytes = quota?.limit ?: 0L,
                    usedBytes = quota?.usage ?: 0L,
                    availableBytes = (quota?.limit ?: 0L) - (quota?.usage ?: 0L)
                )
            } catch (e: Exception) {
                Timber.e(e, "Failed to get storage quota")
                StorageQuota(0L, 0L, 0L)
            }
        }
    }

    override suspend fun createFolder(folderName: String, parentPath: String): String {
        return withContext(Dispatchers.IO) {
            // TODO: Implement folder creation
            ""
        }
    }
}
```

---

## Step 4: Implement SyncProvider

### GoogleDriveSyncProvider.kt:

```kotlin
package dev.eliminater.purefusioniptv.plugins.googledrive

import android.content.Context
import dev.eliminater.purefusioniptv.plugin.*
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

class GoogleDriveSyncProvider(
    private val storageProvider: GoogleDrivePlugin
) : Plugin, SyncProvider {

    override val id = "dev.eliminater.purefusioniptv.googledrive.sync"
    override val name = "Google Drive Sync"
    override val description = "Sync app data using Google Drive"
    override val version = "1.0.0"
    override val author = "Eliminater74"
    override val minAppVersion = "1.0.0"
    override val capabilities = listOf(PluginCapability.SYNC_PROVIDER)

    private val _syncState = MutableStateFlow<SyncState>(SyncState.Idle)
    override val syncState: StateFlow<SyncState> = _syncState.asStateFlow()

    override fun onLoad(context: Context) {
        // Initialize sync provider
    }

    override fun onUnload() {
        // Cleanup
    }

    override suspend fun initialize(): Boolean {
        return storageProvider.isAuthenticated()
    }

    override suspend fun syncAll(direction: SyncDirection): SyncResult {
        // TODO: Implement full sync
        return SyncResult(
            success = false,
            dataType = SyncDataType.SETTINGS,
            itemsSynced = 0,
            conflicts = emptyList(),
            error = "Not implemented"
        )
    }

    override suspend fun sync(dataType: SyncDataType, direction: SyncDirection): SyncResult {
        // TODO: Implement type-specific sync
        return SyncResult(
            success = false,
            dataType = dataType,
            itemsSynced = 0,
            conflicts = emptyList(),
            error = "Not implemented"
        )
    }

    override suspend fun getLastSyncTime(dataType: SyncDataType): Long? {
        // TODO: Retrieve from preferences
        return null
    }

    override suspend fun setAutoSync(enabled: Boolean, intervalMinutes: Int) {
        // TODO: Configure WorkManager periodic sync
    }

    override suspend fun isAutoSyncEnabled(): Boolean {
        // TODO: Check preferences
        return false
    }

    override suspend fun resolveConflict(conflict: SyncConflict, resolution: ConflictResolution) {
        // TODO: Implement conflict resolution
    }

    override suspend fun getPendingConflicts(): List<SyncConflict> {
        // TODO: Retrieve from database
        return emptyList()
    }

    override suspend fun clearSyncData() {
        // TODO: Clear sync metadata
    }
}
```

---

## Step 5: Register Plugin

### Create ServiceLoader registration:

**File:** `src/main/resources/META-INF/services/dev.eliminater.purefusioniptv.plugin.Plugin`

```
dev.eliminater.purefusioniptv.plugins.googledrive.GoogleDrivePlugin
```

---

## Step 6: Update Main App

### settings.gradle.kts:

```kotlin
include(":app")
include(":app:plugins:googledrive")
```

### app/build.gradle.kts:

```kotlin
dependencies {
    // ... existing dependencies ...

    // Plugins
    implementation(project(":app:plugins:googledrive"))
}
```

---

## Migration Checklist

- [ ] Create plugin module structure
- [ ] Move Google Sign-In code to GoogleDrivePlugin
- [ ] Implement CloudStorageProvider methods
- [ ] Move sync logic to GoogleDriveSyncProvider
- [ ] Implement SyncProvider methods
- [ ] Create ServiceLoader registration file
- [ ] Update settings.gradle.kts
- [ ] Update app/build.gradle.kts
- [ ] Test authentication flow
- [ ] Test file upload/download
- [ ] Test sync operations
- [ ] Remove old Google Sync code from main app
- [ ] Update UI to use PluginManager.getCloudStorageProviders()

---

## Testing the Plugin

### In your Activity:

```kotlin
@AndroidEntryPoint
class SettingsActivity : AppCompatActivity() {

    @Inject
    lateinit var pluginManager: PluginManager

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Get Google Drive plugin
        val drivePlugin = pluginManager.getCloudStorageProviders()
            .firstOrNull { it.providerName == "Google Drive" }

        if (drivePlugin != null) {
            lifecycleScope.launch {
                if (!drivePlugin.isAuthenticated()) {
                    // Launch auth
                    val authIntent = drivePlugin.authenticate(this@SettingsActivity)
                    authIntent?.let { startActivityForResult(it, REQUEST_AUTH) }
                } else {
                    // Already authenticated
                    val email = drivePlugin.getUserEmail()
                    val quota = drivePlugin.getStorageQuota()
                    Timber.d("Signed in as: $email")
                    Timber.d("Storage: ${quota.usedBytes}/${quota.totalBytes}")
                }
            }
        }
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        if (requestCode == REQUEST_AUTH) {
            val drivePlugin = pluginManager.getCloudStorageProviders()
                .firstOrNull { it.providerName == "Google Drive" }

            lifecycleScope.launch {
                val success = drivePlugin?.handleAuthResult(resultCode, data) ?: false
                if (success) {
                    Toast.makeText(this, "Authenticated!", Toast.LENGTH_SHORT).show()
                }
            }
        }
    }

    companion object {
        private const val REQUEST_AUTH = 1001
    }
}
```

---

## Benefits of Plugin Architecture

1. **Modularity** - Google Sync is separate from core app
2. **Optional** - Users can choose not to include it
3. **Maintainability** - Easier to update/fix independently
4. **Extensibility** - Easy to add OneDrive, Dropbox plugins
5. **Testing** - Can test plugin independently
6. **Distribution** - Could distribute plugins separately (future)

---

## Next Steps

1. **Start with authentication** - Get Google Sign-In working in plugin
2. **Implement file operations** - Upload/download to Drive
3. **Migrate sync logic** - Move existing sync code
4. **Update UI** - Use PluginManager instead of direct Google Sync calls
5. **Test thoroughly** - Ensure feature parity with old implementation
6. **Clean up** - Remove old Google Sync code from main app

---

## Questions?

See PLUGIN_SDK.md for full API documentation and examples.
