# Stalker Portal / MAG Portal Support

PureFusionIPTV now supports Stalker Portal middleware, enabling users to connect to IPTV providers that use the Stalker Portal protocol (also known as MAG Portal).

## What is Stalker Portal?

Stalker Portal is the middleware protocol used by MAG devices (made by Infomir) and various STB (Set-Top Box) emulators. It's a popular alternative to M3U playlists and Xtream Codes API.

**Compatible with:**
- MAG devices (MAG 250, 322, 424, etc.)
- STB Emulator apps
- Smart STB apps
- TVIP devices
- Other Stalker Portal-compatible players

## Features

### ✅ Currently Implemented:

1. **Authentication**
   - Handshake and session token management
   - MAC-based authentication (auto-generated if not provided)
   - URL-based simplified authentication
   - Supports both traditional and modern Stalker portals

2. **Channel Management**
   - Channel list retrieval
   - Channel logos/images
   - Genre/category support
   - Channel numbers and ordering

3. **EPG Support**
   - EPG data fetching via Stalker API
   - Current and upcoming programs
   - Program descriptions and metadata
   - XMLTV ID mapping

4. **Device Emulation**
   - Emulates MAG 322 device by default
   - Automatic MAC address generation
   - Proper User-Agent and headers
   - Cookie management

### 🚧 Coming Soon:

- Time-shifted playback (catch-up TV)
- VOD (Video On Demand) from Stalker portals
- Favorites sync with portal
- Parental controls via portal
- Recording management

## Setup Instructions

### Adding a Stalker Portal Playlist

1. **Open PureFusionIPTV**
   - Navigate to Settings → Playlists
   - Click "Add Playlist"

2. **Select Stalker Portal**
   - Choose "Stalker Portal" from the playlist type options

3. **Enter Portal URL**
   - Format: `http://portal.provider.com/username/password`
   - Or: `http://portal.provider.com` (if no auth required)
   - Example: `http://stb.iptveditor.com/Michael/elim0154`

4. **MAC Address (Optional)**
   - Leave empty to auto-generate
   - Or enter your device MAC: `00:1A:79:XX:XX:XX`
   - Use the MAC provided by your IPTV provider if required

5. **Click Connect**
   - The app will connect to the portal
   - Channels will be synced automatically
   - EPG data will be downloaded if available

## URL Formats Supported

### Simplified URL (Recommended)
```
http://portal.provider.com/username/password
```
This is the format shown in your screenshot - copy/paste the entire URL.

### Server Only (No Authentication)
```
http://portal.provider.com
```
Some portals don't require username/password in the URL.

### With Port Number
```
http://portal.provider.com:8080/username/password
```

### HTTPS
```
https://portal.provider.com/username/password
```

## How It Works

### API Flow

1. **Handshake**
   - App connects to portal
   - Receives session token
   - Token used for all subsequent requests

2. **Authentication** (if required)
   - Validates credentials
   - Gets user profile
   - Checks account status

3. **Channel Retrieval**
   - Fetches complete channel list
   - Downloads genres/categories
   - Gets channel logos and metadata

4. **EPG Sync**
   - Downloads program guide data
   - Maps programs to channels
   - Caches for offline use

5. **Stream Playback**
   - Generates time-limited stream URLs
   - Handles load balancing
   - Supports multiple stream formats

### Under the Hood

PureFusionIPTV stores Stalker channels in the database with:
- `streamUrl`: Portal URL + channel ID
- `containerExtension`: Stores the Stalker `cmd` for stream generation
- `username`: Stores MAC address (if provided)
- `epgId`: XMLTV ID for EPG matching
- `xtreamStreamId`: Stalker channel ID (numeric)

When playing a Stalker channel:
1. App reads the channel's Stalker ID and cmd
2. Calls Stalker API `create_link` to get actual stream URL
3. Passes stream URL to ExoPlayer
4. Stream plays with low latency

## Troubleshooting

### Connection Failed

**Problem:** "Failed to connect to Stalker portal"

**Solutions:**
- Verify the portal URL is correct
- Check if the portal is online
- Ensure you have internet connectivity
- Try removing `/c/` or trailing slashes from URL

### No Channels Loaded

**Problem:** "No channels found in Stalker portal"

**Solutions:**
- Check if your account is active
- Verify your subscription hasn't expired
- Try reconnecting (Settings → Playlists → Sync)
- Contact your IPTV provider

### Authentication Failed

**Problem:** "Authentication failed"

**Solutions:**
- Some portals don't require authentication (this is normal)
- Verify username/password in URL
- Check if MAC address is required by provider
- Try without MAC address first (auto-generate)

### Channels Won't Play

**Problem:** Channels load but don't play

**Solutions:**
- Check your internet speed (recommend 10+ Mbps)
- Try a different channel
- Restart the app
- Clear app cache and re-sync

### Wrong MAC Address

**Problem:** Provider requires specific MAC address

**Solutions:**
- Go to Settings → Playlists
- Edit your Stalker playlist
- Enter the MAC address provided by your IPTV provider
- Format: `00:1A:79:XX:XX:XX`
- Save and re-sync

## Provider Compatibility

### Tested and Working:
- IPTVEditor Stalker Portal
- Generic Stalker Portal v5+
- MAG-compatible portals

### Should Work (Not Tested):
- Ministra TV Platform
- Stalker Middleware 5.x
- Custom Stalker installations

### Known Issues:
- Some older Stalker v4 portals may have compatibility issues
- Portals with custom authentication may not work
- Heavily modified Stalker installations may need adjustments

## Comparison: Stalker vs M3U vs Xtream

| Feature | Stalker Portal | M3U | Xtream Codes |
|---------|---------------|-----|--------------|
| Channel List | ✅ | ✅ | ✅ |
| EPG Data | ✅ | ⚠️ (Separate) | ✅ |
| VOD/Movies | 🚧 | ✅ | ✅ |
| TV Shows | 🚧 | ✅ | ✅ |
| Time-shift | 🚧 | ❌ | ✅ |
| Categories | ✅ | ✅ | ✅ |
| Device Lock | ✅ (MAC) | ❌ | ❌ |
| Security | ✅ Session | ⚠️ URL | ⚠️ Credentials |

## Advanced Configuration

### Custom Device Emulation

Currently emulates **MAG 322** by default. Future versions will allow:
- Custom device model selection
- Custom User-Agent strings
- Custom timezone settings
- Hardware version override

### Load Balancing

Stalker Portal supports load balancing across multiple stream servers.
PureFusionIPTV automatically:
- Selects the best server
- Falls back to alternates if primary fails
- Handles server rotation

### Debugging

Enable debug logging to troubleshoot issues:

```bash
adb logcat | grep "StalkerApiService"
```

Look for:
- Handshake response
- Session token
- Channel count
- API errors

## Security Notes

- Session tokens are temporary and expire
- Tokens are stored in memory only
- MAC addresses are stored securely
- No passwords are stored (only used during initial auth)
- Portal URL may contain credentials (stored encrypted)

## API Endpoints Used

PureFusionIPTV uses these Stalker Portal API endpoints:

- `GET /portal.php?type=stb&action=handshake` - Get session token
- `GET /portal.php?type=stb&action=do_auth` - Authenticate
- `GET /portal.php?type=stb&action=get_profile` - User info
- `GET /portal.php?type=itv&action=get_all_channels` - Channel list
- `GET /portal.php?type=itv&action=get_ordered_list` - Paginated channels
- `GET /portal.php?type=itv&action=get_genres` - Categories
- `GET /portal.php?type=itv&action=create_link` - Stream URL
- `GET /portal.php?type=itv&action=get_epg_info` - EPG data
- `GET /portal.php?type=itv&action=get_short_epg` - Quick EPG

## Developer Notes

### Code Structure

```
StalkerApiService.kt
├── Handshake & Authentication
├── Channel Management
│   ├── getAllChannels()
│   ├── getOrderedList()
│   └── createLink()
├── Category Management
│   └── getGenres()
└── EPG Management
    ├── getEpgInfo()
    └── getShortEpg()
```

### Data Models

All Stalker API responses are modeled in `StalkerModels.kt`:
- `StalkerHandshakeResponse`
- `StalkerChannelsResponse`
- `StalkerChannel`
- `StalkerGenresResponse`
- `StalkerEpgResponse`
- `StalkerStreamLinkResponse`

### Repository Integration

The `PlaylistRepository` handles Stalker sync:
- `syncStalkerPlaylist()` - Main sync method
- Converts Stalker channels to app's Channel model
- Stores metadata for later stream URL generation
- Handles errors gracefully

## FAQ

**Q: Is my MAC address sent to the portal?**
A: Yes, the MAC address (auto-generated or provided) is sent in the Cookie header and User-Agent. This is required by the Stalker protocol.

**Q: Can I use the same MAC on multiple devices?**
A: Some providers allow it, others don't. Check with your IPTV provider.

**Q: Do I need a MAG device?**
A: No! PureFusionIPTV emulates a MAG device. No physical hardware needed.

**Q: Will this work with Tivimate/MyTVOnline?**
A: Those apps use Xtream API or M3U. Stalker Portal requires a Stalker-compatible player like PureFusionIPTV.

**Q: How do I get a Stalker Portal URL?**
A: Your IPTV provider will give you the portal URL. It usually starts with `http://` and may include your username/password in the path.

**Q: Can I import my Stalker Portal channels to M3U?**
A: No, Stalker uses time-limited stream URLs that expire. M3U playlists need static URLs.

## Support

For Stalker Portal issues:
- GitHub Issues: https://github.com/eliminater74/purefusioniptv/issues
- Include portal error logs (mask sensitive info)
- Specify portal provider if possible

For provider-specific issues:
- Contact your IPTV provider first
- They may have specific setup instructions
- Some providers block non-MAG devices (rare)

## Credits

Stalker Portal implementation based on:
- Stalker Middleware documentation
- MAG device protocol analysis
- Community reverse engineering efforts
- IPTVEditor Stalker portal reference

## Future Enhancements

Planned features:
1. VOD/Movie support from Stalker portals
2. TV Shows/Series browsing
3. Catch-up TV (time-shift)
4. Recording management
5. Multi-profile support
6. Favorites sync with portal
7. Parental controls
8. Custom device model selection
9. Stream quality selection
10. Advanced debugging tools
