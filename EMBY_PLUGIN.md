# Emby Media Server Plugin

The Emby plugin enables PureFusionIPTV to connect to your Emby Media Server and access Live TV channels, EPG data, recordings, movies, and TV shows.

## Features

### ✅ Currently Implemented:

1. **Authentication**
   - Server URL configuration
   - Username/password authentication
   - Secure token storage
   - Session persistence

2. **Live TV Channels**
   - Channel list retrieval
   - Channel logos/images
   - Current program information
   - Direct stream playback

3. **EPG (Electronic Program Guide)**
   - Program listings per channel
   - Time-based program queries
   - Program descriptions and metadata
   - Program images/artwork

4. **Server Information**
   - Server name and version
   - Connection status
   - Feature support detection

### 🚧 Coming Soon:

- DVR Recordings playback
- Movies library integration
- TV Shows library integration
- Continue Watching
- Favorites management
- Search integration
- Playback progress tracking

## Setup Instructions

### Prerequisites

1. **Emby Server Installation**
   - Download from: https://emby.media
   - Install on your server/NAS
   - Complete initial setup

2. **Live TV Configuration** (Optional)
   - Configure TV tuner or IPTV source in Emby
   - Set up EPG data source
   - Map channels

### Connecting to Your Emby Server

1. **Find Your Server URL**
   - Local network: `http://192.168.1.100:8096`
   - Remote access: `https://your-domain.com`
   - Don't include `/web` or trailing slash

2. **Get Your Credentials**
   - Username: Your Emby username
   - Password: Your Emby password
   - Note: Create a dedicated user for IPTV if desired

3. **In PureFusionIPTV**
   - Navigate to Settings → Playlists
   - Click "Add Playlist"
   - Select "Media Server (Plugin)"
   - Choose "Emby Server"
   - Enter server URL, username, password
   - Click "Connect"

### Connection Requirements

- **Network Access**: App must reach Emby server
- **Server Running**: Emby server must be online
- **Valid Credentials**: Username/password must be correct
- **Permissions**: User must have appropriate Emby permissions

## API Implementation

The Emby plugin uses the official Emby Server REST API.

### Authentication

```
POST /Users/AuthenticateByName
Headers:
  X-Emby-Authorization: MediaBrowser Client="PureFusionIPTV", ...

Body:
{
  "Username": "your_username",
  "Pw": "your_password"
}

Response:
{
  "AccessToken": "abc123...",
  "User": {
    "Id": "user_id",
    "Name": "Username"
  }
}
```

### Live TV Channels

```
GET /LiveTv/Channels?UserId={userId}
Headers:
  X-Emby-Token: {authToken}

Response:
{
  "Items": [
    {
      "Id": "channel_id",
      "Name": "ESPN",
      "Number": "2.1",
      "ImageTags": { "Primary": "..." },
      "CurrentProgram": { ... }
    }
  ]
}
```

### EPG Programs

```
GET /LiveTv/Programs?ChannelIds={channelId}&StartDate={start}&EndDate={end}
Headers:
  X-Emby-Token: {authToken}

Response:
{
  "Items": [
    {
      "Id": "program_id",
      "Name": "Program Title",
      "Overview": "Description...",
      "StartDate": "2024-01-15T10:00:00.0000000Z",
      "EndDate": "2024-01-15T11:00:00.0000000Z"
    }
  ]
}
```

### Stream URLs

**Live TV Channel:**
```
{serverUrl}/LiveTv/Channels/{channelId}/stream.m3u8?api_key={authToken}
```

**Recording:**
```
{serverUrl}/Videos/{recordingId}/stream.m3u8?api_key={authToken}&Static=true
```

**Movie/Episode:**
```
{serverUrl}/Videos/{itemId}/stream.m3u8?api_key={authToken}
```

## Architecture

### Plugin Structure

```
EmbyPlugin.kt
├── Authentication (✅ Complete)
│   ├── authenticate()
│   ├── signOut()
│   └── isAuthenticated()
│
├── Server Info (✅ Complete)
│   └── getServerInfo()
│
├── Live TV (✅ Complete)
│   ├── getLiveTVChannels()
│   ├── getChannelStreamUrl()
│   └── getChannelPrograms()
│
├── DVR Recordings (🚧 TODO)
│   ├── getRecordings()
│   └── getRecordingStreamUrl()
│
├── Movies (🚧 TODO)
│   ├── getMovies()
│   └── getMediaStreamUrl()
│
├── TV Shows (🚧 TODO)
│   ├── getTVShows()
│   ├── getEpisodes()
│   └── getMediaStreamUrl()
│
└── Features (🚧 TODO)
    ├── search()
    ├── getContinueWatching()
    ├── getFavorites()
    ├── updatePlaybackProgress()
    └── markAsWatched()
```

### Data Flow

```
┌──────────────────────────────────────────────┐
│           PureFusionIPTV App                 │
│  ┌────────────────────────────────────────┐  │
│  │      Plugin Manager                     │  │
│  └────────────┬───────────────────────────┘  │
│               │                               │
│               ▼                               │
│  ┌────────────────────────────────────────┐  │
│  │      EmbyPlugin                         │  │
│  │  - Authentication                       │  │
│  │  - API Calls (OkHttp)                   │  │
│  │  - JSON Parsing (Gson)                  │  │
│  │  - Data Mapping                         │  │
│  └────────────┬───────────────────────────┘  │
└───────────────┼───────────────────────────────┘
                │
                │ HTTPS REST API
                │
                ▼
┌──────────────────────────────────────────────┐
│           Emby Server                         │
│  - Live TV Tuners                             │
│  - EPG Data                                   │
│  - Media Libraries                            │
│  - Transcoding                                │
└──────────────────────────────────────────────┘
```

## Configuration Storage

The plugin stores configuration in SharedPreferences:

**Storage Key:** `emby_plugin`

**Stored Data:**
- `auth_token`: Authentication token
- `server_url`: Server URL
- `user_id`: User ID
- `username`: Username

**Security Notes:**
- Credentials stored in encrypted SharedPreferences
- Token used for all API calls
- Password never stored (only used during auth)

## Error Handling

### Common Issues & Solutions:

1. **"Authentication failed"**
   - Verify server URL is correct
   - Check username/password
   - Ensure server is reachable
   - Check firewall settings

2. **"Not authenticated with Emby server"**
   - Re-authenticate in settings
   - Check if server is still online
   - Verify token hasn't expired

3. **"Empty response"**
   - Server may be overloaded
   - Network connectivity issues
   - Check Emby server logs

4. **"Failed to get Live TV channels"**
   - Verify Live TV is configured in Emby
   - Check user has Live TV permissions
   - Ensure tuner is working

5. **Stream playback fails**
   - Check stream URL in Emby
   - Verify transcoding settings
   - Test stream directly in Emby web UI

## Performance Considerations

1. **API Rate Limiting**
   - Plugin respects Emby's API limits
   - Channels loaded once and cached
   - EPG data requested in time windows

2. **Network Efficiency**
   - Compressed responses
   - Incremental data loading
   - Image URLs (not base64)

3. **Background Processing**
   - All API calls on IO dispatcher
   - Non-blocking Flow streams
   - Coroutine-based async

## Development Notes

### Testing the Plugin

1. **With Real Emby Server:**
   ```kotlin
   val plugin = EmbyPlugin()
   plugin.onLoad(context)

   // Authenticate
   val token = plugin.authenticate(
       serverUrl = "http://192.168.1.100:8096",
       username = "your_username",
       password = "your_password"
   )

   // Get channels
   plugin.getLiveTVChannels().collect { channel ->
       Log.d("Emby", "Channel: ${channel.name}")
   }
   ```

2. **Check Logs:**
   ```
   adb logcat | grep EmbyPlugin
   ```

### Adding New Features

To add movies/TV shows support:

1. Update `getMovies()` in EmbyPlugin.kt
2. Add response data classes (EmbyMoviesResponse, EmbyMovie)
3. Map to MediaServerMovie
4. Test with real Emby server
5. Update documentation

### Contributing

When modifying the plugin:
- Follow existing code patterns
- Add Timber logging for debugging
- Handle errors gracefully
- Update this documentation
- Test with real Emby server

## Troubleshooting Commands

### Test Server Connection:
```bash
curl -X POST "http://your-server:8096/Users/AuthenticateByName" \
  -H "X-Emby-Authorization: MediaBrowser Client=\"Test\", Device=\"Test\", DeviceId=\"test\", Version=\"1.0.0\"" \
  -H "Content-Type: application/json" \
  -d '{"Username":"your_user","Pw":"your_pass"}'
```

### Get Live TV Channels:
```bash
curl "http://your-server:8096/LiveTv/Channels?UserId=USER_ID" \
  -H "X-Emby-Token: YOUR_TOKEN"
```

### Test Stream URL:
```bash
ffprobe "http://your-server:8096/LiveTv/Channels/CHANNEL_ID/stream.m3u8?api_key=YOUR_TOKEN"
```

## Future Enhancements

Planned features:
1. ✅ Live TV and EPG
2. 🚧 DVR Recordings
3. 🚧 Movies library
4. 🚧 TV Shows library
5. 🚧 Continue Watching
6. 🚧 Favorites sync
7. 🚧 Playback progress tracking
8. 🚧 Search integration
9. 🚧 Multiple profiles support
10. 🚧 Transcoding settings

## Support

For Emby-specific issues:
- Emby Forum: https://emby.media/community
- Emby Docs: https://github.com/MediaBrowser/Emby

For plugin issues:
- GitHub Issues: https://github.com/eliminater74/purefusioniptv/issues
- Include Emby server version
- Include plugin error logs
