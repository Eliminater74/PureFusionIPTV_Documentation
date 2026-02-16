# PureFusion IPTV

<p align="center">
  <img src="app/src/main/res/mipmap-xxxhdpi/ic_launcher_round.png" width="120" alt="PureFusion IPTV Logo">
</p>

<p align="center">
  <strong>A modern, feature-rich IPTV player for Android TV and mobile devices</strong>
</p>

<p align="center">
  <a href="#features">Features</a> •
  <a href="#screenshots">Screenshots</a> •
  <a href="#installation">Installation</a> •
  <a href="#usage">Usage</a> •
  <a href="#keyboard-shortcuts">Controls</a> •
  <a href="#building">Building</a> •
  <a href="#changelog">Changelog</a>
</p>

---

## About

PureFusion IPTV is a sleek, TiViMate-inspired IPTV player built with modern Android architecture. It provides a seamless viewing experience for Live TV, Movies, and TV Shows with an intuitive interface designed for both remote control navigation on Android TV and touch input on mobile devices.

## Features

### 📺 Content Browsing
- **Live TV** - Browse and watch live television channels organized by category
- **Movies (VOD)** - Access your video-on-demand movie library
- **TV Shows (Series)** - Browse episodic content with series organization
- **My List** - Save channels to your personal watchlist
- **Search** - Find content across all categories

### 📡 TV Guide (EPG)
- Full-featured Electronic Program Guide with TiViMate-style grid layout
- 6-hour time span view with scrollable timeline
- Current program highlighting with progress indicators
- Program details with descriptions and duration
- Current time indicator
- Auto-scrolls to currently playing channel
- Highlights the channel you're watching
- Keyboard navigation support (Page Up/Down for time jumping)

### 🎬 Player Features
- Hardware-accelerated video playback via ExoPlayer/Media3
- Closed Caption (CC) toggle support
- Channel strip overlay with quick navigation
- Quick Actions bar (Favorites, Record, CC, Audio, More)
- Resume last channel on startup (configurable)
- Double-click to go fullscreen from TV Guide

### 📋 Playlist Management
- M3U/M3U8 playlist support
- Xtream Codes API support
- Multiple playlist sources
- Auto-sync with configurable intervals (6hr, 12hr, 24hr, Weekly)
- Handles `#EXTINF` metadata including logos and group titles
- Stream type detection (Live, VOD, Series)

### 🚀 Setup Wizard
- **Add Playlist** - M3U URL or Xtream Codes login
- **Restore Backup** - Restore from local or cloud backup files
- **Cloud Sync** - Google Drive, Dropbox, OneDrive, WebDAV
- **Network Share** - Connect to SMB/CIFS/NAS on local network
- **Local File** - Browse and import M3U files from device
- **Device Sync** - Copy settings from another PureFusion device

### ⚙️ Settings & Customization
- **Playlist Management** - Add, edit, and remove playlist sources
- **EPG Sync** - Configure EPG sources and sync schedule
- **Startup Behavior** - Resume last channel or start at main page
- **Playback Settings** - Configure player preferences
- **Appearance** - Theme customization options

### 🎮 Navigation
- Full D-pad/remote control support for Android TV
- Touch-friendly interface for mobile
- Intuitive back navigation
- Channel strip for quick channel switching
- Sidebar navigation between sections

## Screenshots

*Coming soon*

## Installation

### Requirements
- Android 7.0 (API 24) or higher
- Android TV or Android mobile device

### Download
Download the latest APK from the [Releases](https://github.com/Eliminater74/PureFusionIPTV/releases) page.

### Manual Installation
1. Download the APK file
2. Enable "Install from unknown sources" in your device settings
3. Open the APK file to install
4. Launch PureFusion IPTV

## Usage

### Adding a Playlist
1. Open the app and go to **Settings** (gear icon in sidebar)
2. Select **Playlists**
3. Tap **Add Playlist**
4. Enter your M3U playlist URL and a name
5. Save and the playlist will sync automatically

### Adding EPG Data
1. Go to **Settings** → **EPG Sync**
2. Enter your EPG/XMLTV URL
3. Configure sync interval
4. Tap **Sync Now** to fetch program data

### Watching Content
- Navigate to **TV**, **Movies**, or **Shows** sections
- Browse categories on the left, channels on the right
- **Single click** - Select/preview channel
- **Double click** or **Enter** - Play channel fullscreen
- Press **Back** to return to browser

### Using the TV Guide
- Press **G** or select **TV Guide** from channel strip
- Navigate with arrow keys or D-pad
- **Single click** on channel - Play in background (stay in guide)
- **Double click** on channel - Play fullscreen (exit guide)
- **Page Up/Down** - Jump 3 hours back/forward
- **Y button** - Jump to current time

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `G` | Open TV Guide |
| `Enter` / `OK` | Select / Play |
| `Back` | Go back / Exit overlay |
| `Space` | Play/Pause |
| `C` | Toggle Closed Captions |
| `Up/Down` | Navigate channels |
| `Left/Right` | Navigate / Seek |
| `Page Up` | Previous channel group |
| `Page Down` | Next channel group |

### In TV Guide
| Key | Action |
|-----|--------|
| `Page Up` | Jump back 3 hours |
| `Page Down` | Jump forward 3 hours |
| `Y` | Jump to now |
| `Rewind` | Previous day |
| `Fast Forward` | Next day |

## Building

### Prerequisites
- Android Studio Hedgehog or newer
- JDK 17 or higher
- Android SDK with API 34

### Build Steps
```bash
# Clone the repository
git clone https://github.com/Eliminater74/PureFusionIPTV.git
cd PureFusionIPTV

# Build debug APK
./gradlew assembleDebug

# Build release APK (requires signing configuration)
./gradlew assembleRelease
```

The APK will be generated in `app/build/outputs/apk/`

## Tech Stack

- **Language**: Kotlin
- **Architecture**: MVVM with Repository pattern
- **UI**: Android Views with ViewBinding and Data Binding
- **Dependency Injection**: Hilt
- **Database**: Room
- **Networking**: Retrofit + OkHttp
- **Media Playback**: AndroidX Media3 (ExoPlayer)
- **Image Loading**: Glide
- **Async**: Kotlin Coroutines + Flow
- **Logging**: Timber

## Project Structure

```
app/
├── src/main/java/dev/eliminater/purefusioniptv/
│   ├── data/
│   │   ├── entity/          # Room entities (Channel, EpgProgram, Playlist)
│   │   ├── dao/             # Data Access Objects
│   │   ├── repository/      # Data repositories
│   │   └── remote/          # API services
│   ├── di/                  # Hilt dependency injection modules
│   ├── player/              # PlayerManager and playback logic
│   ├── sync/                # Playlist and EPG sync workers
│   └── ui/
│       ├── MainActivity.kt  # Main browser activity
│       ├── channels/        # Channel adapters and components
│       ├── epg/             # TV Guide components
│       ├── groups/          # Category/group adapters
│       └── settings/        # Settings screens
└── src/main/res/
    ├── layout/              # XML layouts
    ├── drawable/            # Icons and drawables
    └── values/              # Colors, strings, themes
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Inspired by [TiViMate](https://tivimate.com/) - The gold standard for IPTV players
- Built with [AndroidX Media3](https://developer.android.com/media/media3) for robust video playback
- Icons from [Material Design Icons](https://materialdesignicons.com/)

## Disclaimer

PureFusion IPTV is a media player application. It does not provide any content. Users are responsible for ensuring they have the legal right to access any content they add to the application.

---

<p align="center">
  Made with ❤️ by <a href="https://github.com/Eliminater74">Eliminater74</a>
</p>
