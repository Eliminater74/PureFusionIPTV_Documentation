# Changelog

All notable changes to PureFusion IPTV will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Enhanced Setup Wizard** - New welcome screen with multiple options for getting started:
  - Add Playlist (M3U/Xtream)
  - Restore from Backup
  - Cloud Sync (Google Drive, Dropbox, OneDrive, WebDAV)
  - Network Share (SMB/NAS/Samba)
  - Local File browser
  - Sync from Another Device
- **TV Guide Enhancement** - TV Guide now highlights and auto-scrolls to the currently playing channel
- **Startup Behavior Setting** - Choose between "Resume last channel" (fullscreen) or "Start at main page"
- **Channel Strip & Quick Actions** - Mutually exclusive overlays (showing one hides the other) like TiViMate
- **TV Shows Icon** - New icon for Series/TV Shows categories in the group list

### Changed
- **Category Names** - Removed "LIVE:" prefix from TV channel categories for cleaner display
- **Back Navigation** - Simplified to stay in current category/section instead of jumping to "All Channels"
- **TV Guide Channel Click** - Single click plays in background and updates highlight; double click exits to fullscreen

### Fixed
- Fixed startup behavior not properly hiding browser when resuming last channel
- Fixed channel strip and quick actions bar showing at the same time
- Fixed back button incorrectly navigating to TV section when in other sections

---

## [1.0.0] - 2026-01-11

### Added

#### Core Features
- **Live TV Browsing** - Browse live television channels organized by category/group
- **Movies (VOD)** - Video-on-demand movie library support
- **TV Shows (Series)** - Episodic content with series organization
- **My List** - Personal watchlist for saving favorite channels
- **Search** - Content search across all categories

#### TV Guide / EPG
- Full-featured Electronic Program Guide with grid layout
- 6-hour time span view with scrollable timeline
- Current program highlighting with progress indicators
- Program details panel with descriptions and duration
- Real-time current time indicator
- Auto-scroll to current time on open
- Keyboard/remote navigation support
- Time jumping (Page Up/Down for 3 hours, Rewind/FF for days)
- Jump to now functionality (Y button)

#### Player
- Hardware-accelerated video playback via ExoPlayer/Media3
- Closed Caption (CC) toggle support
- Channel strip overlay for quick channel switching
- Quick Actions bar (Favorites, Record, CC, Audio, More)
- Double-click detection for fullscreen from TV Guide
- Background playback support in TV Guide

#### Playlist Management
- M3U/M3U8 playlist import from URL
- Multiple playlist support
- Auto-sync with configurable intervals (6hr, 12hr, 24hr, Weekly)
- `#EXTINF` metadata parsing (logos, group titles, stream types)
- Automatic stream type detection (Live, VOD, Series)

#### EPG Sync
- XMLTV EPG data import
- Configurable sync intervals
- Manual sync option
- Program data caching in local database

#### Settings
- Playlist management (add, edit, remove)
- EPG sync configuration
- Startup behavior preferences
- Playback settings
- Appearance customization (coming soon)
- Backup & Restore (coming soon)

#### Navigation & UI
- TiViMate-inspired dark theme design
- Full D-pad/remote control support for Android TV
- Touch-friendly interface for mobile devices
- Sidebar navigation between sections
- Category/group browsing with icons
- Channel logos with fallback placeholders
- Recently watched tracking
- Favorites system

### Technical
- Kotlin with Coroutines and Flow
- MVVM architecture with Repository pattern
- Hilt dependency injection
- Room database for local storage
- Retrofit for network requests
- AndroidX Media3 for playback
- Glide for image loading
- WorkManager for background sync tasks
- Data Binding and ViewBinding

---

## Version History

| Version | Date | Highlights |
|---------|------|------------|
| 1.0.0 | 2026-01-11 | Initial release with full IPTV functionality |

---

## Upcoming Features

- [ ] Recording functionality
- [ ] Multiple audio track selection
- [ ] Subtitle/CC track selection
- [ ] Picture-in-Picture mode
- [ ] Chromecast support
- [ ] Parental controls
- [ ] Channel sorting options
- [ ] Custom channel ordering
- [ ] Backup/Restore settings
- [ ] Theme customization
- [ ] Multi-language support
- [ ] Catch-up/Timeshift support
- [ ] Series episode tracking

---

## Contributing

Found a bug or have a feature request? Please open an issue on [GitHub](https://github.com/Eliminater74/PureFusionIPTV/issues).

Want to contribute code? See the [Contributing Guide](README.md#contributing) in the README.
