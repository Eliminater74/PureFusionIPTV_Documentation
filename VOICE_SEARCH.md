# Voice Search & Voice Commands

PureFusionIPTV includes comprehensive voice search and voice command support for Android TV.

## Features

### 1. Voice Search via Android TV Remote
Press the microphone button on your Android TV remote to trigger voice search:
- Say a channel name: "ESPN", "CNN", "BBC News"
- Say a movie or show name: "Breaking Bad", "The Avengers"
- Say a search query: "Sports channels", "News"

### 2. In-App Voice Search Button
Click the microphone icon in the search screen to trigger voice input.

### 3. Search Suggestions
Recent search queries are saved and displayed as suggestions for quick access.

### 4. Direct Channel Playback
When you search for a channel name, you can immediately play it by selecting it from the results.

## Implementation Details

### Components

#### 1. Searchable Configuration (`res/xml/searchable.xml`)
Defines the search behavior:
- Voice search mode enabled
- Search suggestions from recent queries
- Global search integration
- Free-form voice language model

#### 2. SearchSuggestionsProvider
Content provider that stores and retrieves recent search queries:
- Authority: `dev.eliminater.purefusioniptv.search.suggestions`
- Provides suggestions as user types
- Integrates with Android's search framework

#### 3. SearchActivity
Handles all search operations:
- Text search with real-time results
- Voice search intent handling (`ACTION_SEARCH`)
- Search suggestion selection (`ACTION_VIEW`)
- Search filters (All, Channels, Movies, Shows, Programs)
- Search history management

#### 4. MainActivity Voice Intent Handling
Processes direct playback commands:
- `ACTION_PLAY_CHANNEL`: Play channel by ID
- `ACTION_PLAY_PROGRAM`: Play EPG program by ID
- Legacy search result handling

### Intent Actions

```kotlin
// Play a specific channel
Intent(ACTION_PLAY_CHANNEL).apply {
    putExtra(EXTRA_CHANNEL_ID, channelId)
}

// Play a specific program
Intent(ACTION_PLAY_PROGRAM).apply {
    putExtra(EXTRA_CHANNEL_ID, channelId)
    putExtra(EXTRA_PROGRAM_ID, programId)
}
```

### Search Flow

1. **Voice Input Triggered**
   - User presses microphone button on Android TV remote
   - OR user clicks voice search button in SearchActivity

2. **Voice Recognition**
   - Android's voice recognition service processes speech
   - Returns recognized text as search query

3. **Search Execution**
   - Query is sent to SearchViewModel
   - Database is searched for matching channels, programs, movies, shows
   - Results are displayed in categorized sections

4. **Result Selection**
   - User selects a result
   - SearchActivity sends playback intent to MainActivity
   - MainActivity plays the selected content
   - Query is saved to search history

## Permissions Required

The following permissions are already declared in `AndroidManifest.xml`:

```xml
<!-- Voice search on Android TV -->
<uses-permission android:name="android.permission.RECORD_AUDIO"/>

<!-- Microphone hardware feature (not required, but recommended) -->
<uses-feature android:name="android.hardware.microphone" android:required="false"/>
```

## Testing Voice Search

### On Android TV Device
1. Navigate to PureFusionIPTV
2. Press the microphone button on your remote
3. Say a channel name or search term
4. View search results
5. Select a result to play

### In SearchActivity
1. Open search from the main menu
2. Click the microphone icon in the search bar
3. Say your search query
4. Results appear automatically

### From Android TV Home Screen
1. On the Android TV home screen, press the microphone button
2. Say "Open PureFusion IPTV"
3. Then say a channel name or search term
4. App will open and display search results

## Voice Search Architecture

```
┌─────────────────────────────────────────┐
│     Android TV Voice Button Press        │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│   Android Voice Recognition Service      │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│   Intent: ACTION_SEARCH                  │
│   Extra: QUERY = "ESPN"                  │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│         SearchActivity                   │
│   - Receives query                       │
│   - Saves to search history              │
│   - Performs search                      │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│       SearchViewModel                    │
│   - Queries database                     │
│   - Returns matching results             │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│      Display Results                     │
│   - Channels                             │
│   - Programs                             │
│   - Movies                               │
│   - Shows                                │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│     User Selects Result                  │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│         MainActivity                     │
│   - Receives playback intent            │
│   - Plays selected channel/content       │
└─────────────────────────────────────────┘
```

## Search Database Schema

Voice search queries the following database tables:

### Channels
- Searches: `name`, `tvgId`
- Stream types: LIVE, VOD, SERIES

### EPG Programs
- Searches: `title`, `description`
- Filters by current and upcoming programs

### Search Results Grouping
Results are automatically grouped by type:
1. Live TV Channels
2. Movies (VOD)
3. TV Shows (Series)
4. EPG Programs

## Performance Considerations

1. **Search Debouncing**: 300ms debounce prevents excessive database queries
2. **Indexed Searches**: Database columns are indexed for fast lookups
3. **Result Limiting**: Results are limited to prevent UI lag
4. **Background Processing**: All searches run on IO dispatcher

## Future Enhancements

Potential improvements for voice control:

1. **Direct Voice Commands**
   - "Play ESPN" → Directly plays ESPN without showing search results
   - "Next channel" → Channel navigation
   - "Previous channel" → Channel navigation
   - "Volume up/down" → Volume control

2. **Contextual Voice Search**
   - "Show me sports channels"
   - "What's playing now on HBO"
   - "Record this show"

3. **Voice Settings**
   - "Open settings"
   - "Show TV guide"
   - "Switch to movie mode"

4. **Multi-Language Support**
   - Voice recognition in multiple languages
   - Channel name matching in different languages

## Troubleshooting

### Voice Search Not Working
1. Check microphone permission is granted
2. Verify Android TV remote has working microphone
3. Check internet connection (voice recognition may require network)
4. Try clearing app data and reconfiguring

### No Search Results
1. Verify playlist is loaded
2. Check EPG data is synced
3. Try searching for known channel names
4. Check search history to see if query was received

### Voice Button Opens Wrong App
1. Set PureFusionIPTV as default search app in Android TV settings
2. Clear defaults for other IPTV apps
3. Restart Android TV device

## Technical Notes

- Voice recognition requires `RECORD_AUDIO` permission
- Search provider uses `SearchRecentSuggestionsProvider`
- All voice intents are handled by `SearchActivity`
- Results are processed by `SearchViewModel` using Kotlin Flow
- Database queries use Room with proper indexing
- Voice search works offline once content is cached
