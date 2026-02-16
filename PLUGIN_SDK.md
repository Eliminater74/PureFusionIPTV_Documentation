# PureFusionIPTV Plugin SDK

## Overview

The PureFusionIPTV Plugin SDK allows developers to extend the app's functionality with custom implementations of various features. Plugins run in a sandboxed environment with version compatibility checks.

## Plugin Capabilities

Plugins can provide one or more of the following capabilities:

### 1. EPG Provider
Custom Electronic Program Guide data sources beyond XMLTV.

**Interface:** `EpgProvider`
**Use Cases:**
- Custom API-based EPG services
- Proprietary EPG formats
- Cloud-based EPG aggregators

### 2. Catch-up Provider
VOD/time-shift playback for past programs.

**Interface:** `CatchupProvider`
**Use Cases:**
- Cloud DVR integration
- Time-shift streaming
- VOD playback systems

### 3. Metadata Provider
Movie/show information and artwork.

**Interface:** `MetadataProvider` (coming soon)
**Use Cases:**
- TMDb integration
- IMDb integration
- Custom metadata APIs

### 4. Subtitle Engine
Custom subtitle format parsing and rendering.

**Interface:** `SubtitleEngine` (coming soon)
**Use Cases:**
- Custom subtitle formats
- Real-time subtitle translation
- Subtitle synchronization

### 5. Analytics
Usage tracking and reporting.

**Interface:** `AnalyticsProvider` (coming soon)
**Use Cases:**
- Custom analytics backends
- Usage statistics
- Performance monitoring

### 6. Cloud Storage Provider
Cloud storage integration for backups and sync.

**Interface:** `CloudStorageProvider`
**Use Cases:**
- Google Drive integration
- OneDrive integration
- Dropbox integration
- pCloud, Box, or any cloud service
- Custom WebDAV servers

### 7. Sync Provider
Data synchronization across devices.

**Interface:** `SyncProvider`
**Use Cases:**
- Cloud-based sync (Drive, OneDrive, etc.)
- Local network sync
- P2P device synchronization
- Custom sync backends

### 8. UI Extension
Extend the app's user interface.

**Interface:** `UiExtension`
**Use Cases:**
- Custom settings screens
- Additional menu items
- Action buttons in specific screens
- Custom views in existing layouts

---

## Creating a Plugin

### Step 1: Implement the Plugin Interface

```kotlin
package com.example.myplugin

import android.content.Context
import dev.eliminater.purefusioniptv.plugin.Plugin
import dev.eliminater.purefusioniptv.plugin.PluginCapability

class MyCustomPlugin : Plugin {
    override val id = "com.example.myplugin"
    override val name = "My Custom Plugin"
    override val description = "Provides custom EPG data from XYZ service"
    override val version = "1.0.0"
    override val author = "Your Name"
    override val minAppVersion = "1.0.0"
    override val capabilities = listOf(PluginCapability.EPG_PROVIDER)

    override fun onLoad(context: Context) {
        // Initialize your plugin
        println("MyCustomPlugin loaded!")
    }

    override fun onUnload() {
        // Cleanup resources
        println("MyCustomPlugin unloaded!")
    }
}
```

### Step 2: Implement Capability Interfaces

For an EPG provider, also implement `EpgProvider`:

```kotlin
import dev.eliminater.purefusioniptv.plugin.EpgProvider
import dev.eliminater.purefusioniptv.data.entity.EpgProgram
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

class MyEpgPlugin : Plugin, EpgProvider {
    // ... Plugin interface implementation ...

    override suspend fun fetchPrograms(
        channelId: Long,
        channelEpgId: String,
        startTimeMillis: Long,
        endTimeMillis: Long
    ): Flow<EpgProgram> = flow {
        // Fetch EPG data from your custom source
        val programs = fetchFromYourApi(channelEpgId, startTimeMillis, endTimeMillis)
        programs.forEach { emit(it) }
    }

    override fun canHandle(epgId: String): Boolean {
        // Return true if this plugin can handle this EPG ID format
        return epgId.startsWith("xyz://")
    }

    override fun getUpdateInterval(): Long {
        // Update every 2 hours
        return 2 * 60 * 60 * 1000L
    }

    private suspend fun fetchFromYourApi(
        epgId: String,
        start: Long,
        end: Long
    ): List<EpgProgram> {
        // Your custom API logic here
        return emptyList()
    }
}
```

### Step 3: Register Your Plugin (Option A: ServiceLoader)

Create `META-INF/services/dev.eliminater.purefusioniptv.plugin.Plugin`:

```
com.example.myplugin.MyEpgPlugin
```

### Step 3: Register Your Plugin (Option B: Explicit Registration)

Register programmatically:

```kotlin
val pluginManager = ... // Get from Hilt
pluginManager.registerPlugin(MyEpgPlugin())
```

---

## Plugin Lifecycle

1. **Discovery**: Plugins are discovered via ServiceLoader or explicit registration
2. **Version Check**: Plugin compatibility is verified against app version
3. **Registration**: Plugin is added to the registry
4. **Load**: `onLoad(context)` is called
5. **Active**: Plugin provides functionality
6. **Unload**: `onUnload()` is called on shutdown

---

## Version Compatibility

Plugins must declare a minimum app version:

```kotlin
override val minAppVersion = "1.0.0"  // Semantic versioning
```

The plugin system automatically checks compatibility:
- Plugin requires 1.0.0, app is 1.2.0 → ✅ Compatible
- Plugin requires 2.0.0, app is 1.5.0 → ❌ Incompatible

---

## Best Practices

### Thread Safety
All plugin methods may be called from background threads. Use coroutines and Flow for async operations:

```kotlin
override suspend fun fetchPrograms(...): Flow<EpgProgram> = flow {
    withContext(Dispatchers.IO) {
        // Network/database operations
    }
}
```

### Error Handling
Handle errors gracefully and log appropriately:

```kotlin
override fun onLoad(context: Context) {
    try {
        initializeApi()
    } catch (e: Exception) {
        Log.e(TAG, "Failed to initialize", e)
    }
}
```

### Resource Cleanup
Always cleanup in `onUnload()`:

```kotlin
override fun onUnload() {
    apiClient?.close()
    cache?.clear()
}
```

### Performance
- Use batch operations when available
- Implement caching to reduce API calls
- Use efficient data structures

---

## Example: Simple EPG Plugin

```kotlin
package com.example.simpleepg

import android.content.Context
import dev.eliminater.purefusioniptv.plugin.*
import dev.eliminater.purefusioniptv.data.entity.EpgProgram
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

class SimpleEpgPlugin : Plugin, EpgProvider {
    override val id = "com.example.simpleepg"
    override val name = "Simple EPG"
    override val description = "Basic EPG provider example"
    override val version = "1.0.0"
    override val author = "Example Author"
    override val minAppVersion = "1.0.0"
    override val capabilities = listOf(PluginCapability.EPG_PROVIDER)

    override fun onLoad(context: Context) {
        println("SimpleEpgPlugin loaded")
    }

    override fun onUnload() {
        println("SimpleEpgPlugin unloaded")
    }

    override suspend fun fetchPrograms(
        channelId: Long,
        channelEpgId: String,
        startTimeMillis: Long,
        endTimeMillis: Long
    ): Flow<EpgProgram> = flow {
        // Example: Return a single program
        emit(EpgProgram(
            id = 0,
            channelId = channelId,
            channelEpgId = channelEpgId,
            title = "Example Program",
            description = "This is an example program from SimpleEpgPlugin",
            startTime = startTimeMillis,
            endTime = endTimeMillis,
            category = "Entertainment",
            icon = null,
            rating = null
        ))
    }

    override fun canHandle(epgId: String): Boolean {
        return epgId.startsWith("simple://")
    }
}
```

---

## Advanced Plugin Examples

### Example: Google Drive Sync Plugin

```kotlin
class GoogleDrivePlugin : Plugin, CloudStorageProvider, SyncProvider {
    override val id = "dev.eliminater.googledrive"
    override val capabilities = listOf(
        PluginCapability.CLOUD_STORAGE,
        PluginCapability.SYNC_PROVIDER
    )

    // CloudStorageProvider implementation
    override suspend fun uploadFile(
        fileName: String,
        folderPath: String,
        inputStream: InputStream,
        mimeType: String,
        progressCallback: ((Float) -> Unit)?
    ): String {
        // 1. Authenticate with Google Drive API
        // 2. Find or create folder
        // 3. Upload file with progress tracking
        // 4. Return file ID
    }

    // SyncProvider implementation
    override suspend fun sync(
        dataType: SyncDataType,
        direction: SyncDirection
    ): SyncResult {
        when (dataType) {
            SyncDataType.SETTINGS -> syncSettings()
            SyncDataType.PLAYLISTS -> syncPlaylists()
            SyncDataType.FAVORITES -> syncFavorites()
            // ...
        }
    }
}
```

### Example: UI Extension Plugin

```kotlin
class CustomSettingsPlugin : Plugin, UiExtension {
    override val capabilities = listOf(PluginCapability.UI_EXTENSION)

    override fun getSettingsScreens(): List<SettingsScreen> {
        return listOf(
            SettingsScreen(
                id = "custom_sync",
                title = "Cloud Sync",
                iconResourceId = R.drawable.ic_cloud,
                intent = Intent(context, SyncSettingsActivity::class.java),
                category = SettingsCategory.SYNC
            )
        )
    }

    override fun getMenuItems(): List<MenuItem> {
        return listOf(
            MenuItem(
                id = "sync_now",
                title = "Sync Now",
                iconResourceId = R.drawable.ic_sync,
                section = MenuSection.SETTINGS,
                onClick = { context ->
                    // Trigger sync
                }
            )
        )
    }
}
```

### Example: OneDrive Storage Plugin

```kotlin
class OneDrivePlugin : Plugin, CloudStorageProvider {
    override val id = "dev.eliminater.onedrive"
    override val providerName = "OneDrive"
    override val capabilities = listOf(PluginCapability.CLOUD_STORAGE)

    override suspend fun authenticate(context: Context): Intent? {
        // Return Microsoft Sign-In intent
        return MicrosoftAuthClient.getSignInIntent(context)
    }

    override suspend fun uploadFile(...): String {
        // Use Microsoft Graph API to upload
        // POST /me/drive/root:/path/to/file:/content
    }

    override suspend fun downloadFile(...) {
        // Use Microsoft Graph API to download
        // GET /me/drive/items/{id}/content
    }
}
```

---

## Plugin Discovery

PureFusionIPTV discovers plugins using Java ServiceLoader. To make your plugin discoverable:

1. Create file: `src/main/resources/META-INF/services/dev.eliminater.purefusioniptv.plugin.Plugin`
2. Add your plugin class fully qualified name:
   ```
   com.example.myplugin.MyCustomPlugin
   ```

---

## Testing Your Plugin

Use PluginManager to test:

```kotlin
@Test
fun testPlugin() {
    val plugin = MyCustomPlugin()
    val context = ApplicationProvider.getApplicationContext<Context>()

    plugin.onLoad(context)
    assertTrue(plugin.capabilities.contains(PluginCapability.EPG_PROVIDER))
    plugin.onUnload()
}
```

---

## Media Server Provider

### Overview

The **MediaServerProvider** interface enables integration of media servers like Emby, Plex, and Jellyfin into PureFusionIPTV.

### Interface: `MediaServerProvider`

Media server plugins provide:
- Live TV channel access
- DVR recordings playback
- Movies and TV shows library
- Continue watching / in-progress items
- Favorites management
- Search across all content
- Watch progress synchronization

### Key Methods

```kotlin
interface MediaServerProvider : Plugin {
    // Server identification
    val serverType: String  // "Emby", "Plex", "Jellyfin", etc.
    val serverIconResourceId: Int

    // Authentication
    suspend fun authenticate(serverUrl: String, username: String, password: String): String
    suspend fun isAuthenticated(): Boolean
    suspend fun signOut()

    // Live TV
    suspend fun getLiveTVChannels(): Flow<MediaServerChannel>
    suspend fun getChannelStreamUrl(channelId: String): String
    suspend fun getChannelPrograms(channelId: String, startTimeMillis: Long, endTimeMillis: Long): Flow<MediaServerProgram>

    // Recordings
    suspend fun getRecordings(): Flow<MediaServerRecording>
    suspend fun getRecordingStreamUrl(recordingId: String): String

    // Movies & Shows
    suspend fun getMovies(): Flow<MediaServerMovie>
    suspend fun getTVShows(): Flow<MediaServerTVShow>
    suspend fun getEpisodes(showId: String): Flow<MediaServerEpisode>

    // Playback
    suspend fun getMediaStreamUrl(itemId: String, mediaType: MediaType): String
    suspend fun updatePlaybackProgress(itemId: String, positionMillis: Long)
    suspend fun markAsWatched(itemId: String)

    // User features
    suspend fun getContinueWatching(): Flow<MediaServerItem>
    suspend fun getFavorites(): Flow<MediaServerItem>
    suspend fun addToFavorites(itemId: String)
    suspend fun search(query: String): MediaServerSearchResults

    // Capabilities
    fun supportsFeature(feature: MediaServerFeature): Boolean
}
```

### Example: Emby Plugin

```kotlin
class EmbyPlugin : Plugin, MediaServerProvider {
    override val id = "dev.eliminater.purefusioniptv.emby"
    override val name = "Emby Server"
    override val serverType = "Emby"
    override val capabilities = listOf(PluginCapability.MEDIA_SERVER)

    private var authToken: String? = null
    private var serverUrl: String? = null

    override suspend fun authenticate(serverUrl: String, username: String, password: String): String {
        // Emby authentication API call
        val client = OkHttpClient()
        val json = """{"Username": "$username", "Pw": "$password"}"""
        val body = json.toRequestBody("application/json".toMediaType())
        val request = Request.Builder()
            .url("$serverUrl/Users/AuthenticateByName")
            .post(body)
            .addHeader("X-Emby-Authorization", getAuthHeader())
            .build()

        val response = client.newCall(request).execute()
        val authResponse = Gson().fromJson(response.body?.string(), EmbyAuthResponse::class.java)

        this.authToken = authResponse.AccessToken
        this.serverUrl = serverUrl

        // Save credentials
        saveCredentials(authToken, serverUrl)

        return authToken
    }

    override suspend fun getLiveTVChannels(): Flow<MediaServerChannel> = flow {
        requireAuthentication()

        // Call Emby Live TV API
        val response = client.get("$serverUrl/LiveTv/Channels?UserId=$userId") {
            header("X-Emby-Token", authToken)
        }

        val channels = Gson().fromJson(response.bodyAsText(), EmbyChannelsResponse::class.java)
        channels.Items.forEach { channel ->
            emit(MediaServerChannel(
                id = channel.Id,
                name = channel.Name,
                number = channel.Number,
                imageUrl = "$serverUrl/Items/${channel.Id}/Images/Primary",
                hasEPG = true,
                currentProgram = channel.CurrentProgram?.toMediaServerProgram()
            ))
        }
    }

    override suspend fun getChannelStreamUrl(channelId: String): String {
        requireAuthentication()
        // Return Emby live stream URL
        return "$serverUrl/LiveTv/Channels/$channelId/stream.m3u8?api_key=$authToken"
    }

    override suspend fun getMovies(): Flow<MediaServerMovie> = flow {
        requireAuthentication()

        // Call Emby Movies API
        val response = client.get("$serverUrl/Items?IncludeItemTypes=Movie&UserId=$userId&Recursive=true") {
            header("X-Emby-Token", authToken)
        }

        val movies = Gson().fromJson(response.bodyAsText(), EmbyItemsResponse::class.java)
        movies.Items.forEach { movie ->
            emit(MediaServerMovie(
                id = movie.Id,
                title = movie.Name,
                description = movie.Overview,
                year = movie.ProductionYear,
                durationMillis = movie.RunTimeTicks / 10000,
                imageUrl = "$serverUrl/Items/${movie.Id}/Images/Primary",
                genres = movie.Genres,
                rating = movie.OfficialRating,
                communityRating = movie.CommunityRating
            ))
        }
    }

    override fun supportsFeature(feature: MediaServerFeature): Boolean {
        return when (feature) {
            MediaServerFeature.LIVE_TV,
            MediaServerFeature.DVR_RECORDINGS,
            MediaServerFeature.MOVIES,
            MediaServerFeature.TV_SHOWS,
            MediaServerFeature.TRANSCODING -> true
            else -> false
        }
    }
}
```

### Using Media Server Plugins

```kotlin
@AndroidEntryPoint
class BrowseActivity : AppCompatActivity() {

    @Inject
    lateinit var pluginManager: PluginManager

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Get available media servers
        val mediaServers = pluginManager.getMediaServerProviders()

        mediaServers.forEach { server ->
            Timber.d("Found media server: ${server.serverType}")
        }

        // Get authenticated server
        lifecycleScope.launch {
            val embyServer = pluginManager.getAuthenticatedMediaServer()

            if (embyServer != null) {
                // Load live TV channels
                embyServer.getLiveTVChannels().collect { channel ->
                    Timber.d("Channel: ${channel.name} (${channel.number})")
                    // Add to channel list
                }
            } else {
                // Show authentication screen
                showServerAuthDialog()
            }
        }
    }

    private fun showServerAuthDialog() {
        val mediaServers = pluginManager.getMediaServerProviders()

        // Show server selection dialog
        val serverNames = mediaServers.map { it.serverType }.toTypedArray()

        MaterialAlertDialogBuilder(this)
            .setTitle("Connect to Media Server")
            .setItems(serverNames) { _, which ->
                val selectedServer = mediaServers[which]
                showAuthenticationForm(selectedServer)
            }
            .show()
    }

    private fun showAuthenticationForm(server: MediaServerProvider) {
        // Show input dialog for server URL, username, password
        val view = layoutInflater.inflate(R.layout.dialog_server_auth, null)
        val urlInput = view.findViewById<EditText>(R.id.serverUrlInput)
        val usernameInput = view.findViewById<EditText>(R.id.usernameInput)
        val passwordInput = view.findViewById<EditText>(R.id.passwordInput)

        MaterialAlertDialogBuilder(this)
            .setTitle("Connect to ${server.serverType}")
            .setView(view)
            .setPositiveButton("Connect") { _, _ ->
                val url = urlInput.text.toString()
                val username = usernameInput.text.toString()
                val password = passwordInput.text.toString()

                lifecycleScope.launch {
                    try {
                        val token = server.authenticate(url, username, password)
                        Toast.makeText(this@BrowseActivity, "Connected!", Toast.LENGTH_SHORT).show()
                        loadServerContent(server)
                    } catch (e: Exception) {
                        Toast.makeText(this@BrowseActivity, "Failed: ${e.message}", Toast.LENGTH_LONG).show()
                    }
                }
            }
            .setNegativeButton("Cancel", null)
            .show()
    }
}
```

### Media Server Features

The `MediaServerFeature` enum defines supported capabilities:

- `LIVE_TV` - Live TV channels with EPG
- `DVR_RECORDINGS` - Recorded content playback
- `MOVIES` - Movie library access
- `TV_SHOWS` - TV show library access
- `MUSIC` - Music library access
- `PHOTOS` - Photo library access
- `TRANSCODING` - Server-side transcoding
- `DIRECT_PLAY` - Direct stream playback
- `SUBTITLES` - Subtitle support
- `MULTIPLE_PROFILES` - Multiple user profiles
- `PARENTAL_CONTROLS` - Parental control features
- `CONTINUE_WATCHING` - Resume watching support
- `FAVORITES` - Favorites/liked items
- `SEARCH` - Search functionality

### Supported Media Servers

The plugin system supports any media server that provides an API:

- **Emby** - Full support (see example plugin)
- **Plex** - Compatible (requires Plex API implementation)
- **Jellyfin** - Compatible (API similar to Emby)
- **Channels DVR** - Compatible
- **TVHeadend** - Compatible
- **Custom servers** - Any server with HTTP API

### Plugin Dependencies

Media server plugins typically require:

```gradle
dependencies {
    // HTTP client
    implementation("com.squareup.okhttp3:okhttp:4.12.0")

    // JSON parsing
    implementation("com.google.code.gson:gson:2.10.1")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.0")

    // Main app (for plugin interfaces)
    implementation(project(":app"))
}
```

---

## Future Capabilities

Coming soon:
- **MetadataProvider**: Movie/show information
- **SubtitleEngine**: Custom subtitle formats
- **AnalyticsProvider**: Usage tracking
- **ThemeProvider**: Custom UI themes
- **ChannelSource**: Custom channel providers

---

## Support

For plugin development questions:
- GitHub Issues: https://github.com/Eliminater74/PureFusionIPTV/issues
- Documentation: https://github.com/Eliminater74/PureFusionIPTV/wiki

---

**Note:** The Plugin SDK is currently in beta. APIs may change in future versions. Always check compatibility before releasing plugins.
