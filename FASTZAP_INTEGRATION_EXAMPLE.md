# FastZap Integration Example for MainActivity

This document shows how to integrate FastZap into the existing MainActivity.

## Current MainActivity Pattern

```kotlin
// Current approach (single PlayerManager)
@Inject
lateinit var playerManager: PlayerManager

private fun playChannel(channel: Channel) {
    playerManager.playChannel(channel)
}
```

## FastZap Integration (Option 1: Direct Replacement)

### Step 1: Replace PlayerManager with UnifiedPlayerManager

```kotlin
// In MainActivity.kt
@Inject
lateinit var unifiedPlayerManager: UnifiedPlayerManager  // Was: playerManager

// In onCreate
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    // Set video surface
    binding.playerView.player = unifiedPlayerManager.getPlayer()

    // Bind to surface lifecycle
    binding.playerView.videoSurfaceView?.holder?.addCallback(object : SurfaceHolder.Callback {
        override fun surfaceCreated(holder: SurfaceHolder) {
            unifiedPlayerManager.setSurface(holder.surface)
        }

        override fun surfaceDestroyed(holder: SurfaceHolder) {
            unifiedPlayerManager.setSurface(null)
        }

        override fun surfaceChanged(holder: SurfaceHolder, format: Int, width: Int, height: Int) {}
    })

    // Collect zap metrics (optional, for debug)
    lifecycleScope.launch {
        repeatOnLifecycle(Lifecycle.State.STARTED) {
            unifiedPlayerManager.zapTime.collect { zapTimeMs ->
                if (zapTimeMs > 0) {
                    Timber.d("Channel zap completed in ${zapTimeMs}ms")
                }
            }
        }
    }
}
```

### Step 2: Update playChannel Method

```kotlin
private fun playChannel(channel: Channel) {
    // Option A: Simple replacement (no pre-buffering optimization)
    unifiedPlayerManager.playChannel(channel)

    // Option B: With smart pre-buffering (RECOMMENDED)
    val nextChannels = predictNextChannels(
        currentChannel = channel,
        allChannels = currentChannelList,
        strategy = NavigationStrategy.LINEAR,
        maxPredictions = 3
    )
    unifiedPlayerManager.playChannel(channel, nextChannels)

    // Option C: Use smart play extension (BEST)
    unifiedPlayerManager.smartPlayChannel(
        channel = channel,
        allChannels = currentChannelList,
        strategy = NavigationStrategy.LINEAR,
        recentChannels = getRecentlyWatchedChannels()
    )
}
```

### Step 3: Update Channel Navigation Handlers

```kotlin
// D-pad navigation (channel up/down)
override fun onKeyDown(keyCode: Int, event: KeyEvent?): Boolean {
    return when (keyCode) {
        KeyEvent.KEYCODE_DPAD_UP -> {
            val prevChannel = getPreviousChannel()
            if (prevChannel != null) {
                // Pre-buffer next channels in the list
                val upcomingChannels = getUpcomingChannels(prevChannel, direction = "up")
                unifiedPlayerManager.playChannel(prevChannel, upcomingChannels)
            }
            true
        }
        KeyEvent.KEYCODE_DPAD_DOWN -> {
            val nextChannel = getNextChannel()
            if (nextChannel != null) {
                // Pre-buffer next channels in the list
                val upcomingChannels = getUpcomingChannels(nextChannel, direction = "down")
                unifiedPlayerManager.playChannel(nextChannel, upcomingChannels)
            }
            true
        }
        else -> super.onKeyDown(keyCode, event)
    }
}

// Helper to get upcoming channels for pre-buffering
private fun getUpcomingChannels(currentChannel: Channel, direction: String): List<Channel> {
    val currentIndex = currentChannelList.indexOfFirst { it.id == currentChannel.id }
    if (currentIndex == -1) return emptyList()

    return when (direction) {
        "down" -> {
            // User is navigating down, pre-buffer next 3-5 channels
            currentChannelList.subList(
                (currentIndex + 1).coerceAtMost(currentChannelList.size),
                (currentIndex + 6).coerceAtMost(currentChannelList.size)
            )
        }
        "up" -> {
            // User is navigating up, pre-buffer previous 3-5 channels
            currentChannelList.subList(
                (currentIndex - 5).coerceAtLeast(0),
                currentIndex.coerceAtLeast(0)
            ).reversed()
        }
        else -> emptyList()
    }
}
```

### Step 4: Update RecyclerView Channel Selection

```kotlin
// When user selects channel from list
channelAdapter.onChannelClick = { channel, position ->
    // Get visible channels for pre-buffering
    val layoutManager = channelRecyclerView.layoutManager as LinearLayoutManager
    val firstVisible = layoutManager.findFirstVisibleItemPosition()
    val lastVisible = layoutManager.findLastVisibleItemPosition()

    val visibleChannels = currentChannelList.subList(
        firstVisible.coerceAtLeast(0),
        (lastVisible + 1).coerceAtMost(currentChannelList.size)
    )

    // Play with pre-buffer context
    unifiedPlayerManager.playChannel(channel, visibleChannels)

    // Optionally update pre-buffer queue on scroll
    channelRecyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
        override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
            if (newState == RecyclerView.SCROLL_STATE_IDLE) {
                val newFirstVisible = layoutManager.findFirstVisibleItemPosition()
                val newLastVisible = layoutManager.findLastVisibleItemPosition()
                val newVisibleChannels = currentChannelList.subList(
                    newFirstVisible.coerceAtLeast(0),
                    (newLastVisible + 1).coerceAtMost(currentChannelList.size)
                )
                unifiedPlayerManager.updatePreBufferQueue(newVisibleChannels)
            }
        }
    })
}
```

### Step 5: Update Favorites Handling

```kotlin
private fun playFavoriteChannel(channel: Channel) {
    // Get all favorites for smart prediction
    val allFavorites = currentChannelList.filter { it.isFavorite }

    // Predict next favorites
    val nextFavorites = predictNextChannels(
        currentChannel = channel,
        allChannels = allFavorites,
        strategy = NavigationStrategy.FAVORITES,
        maxPredictions = 3
    )

    unifiedPlayerManager.playChannel(channel, nextFavorites)
}
```

### Step 6: Add Settings Toggle (Optional)

```kotlin
// In settings or debug menu
private fun toggleFastZapMode() {
    val currentMode = unifiedPlayerManager.zapMode.value
    val newMode = when (currentMode) {
        ZapMode.STANDARD -> ZapMode.FAST_ZAP
        ZapMode.FAST_ZAP -> ZapMode.STANDARD
    }

    unifiedPlayerManager.setZapMode(newMode)

    Toast.makeText(
        this,
        "Zap mode: ${newMode.name}",
        Toast.LENGTH_SHORT
    ).show()
}
```

### Step 7: Update onDestroy

```kotlin
override fun onDestroy() {
    unifiedPlayerManager.release()
    super.onDestroy()
}
```

## FastZap Integration (Option 2: Gradual Migration)

If you want to test FastZap without replacing PlayerManager everywhere:

### Keep Both Managers

```kotlin
// Keep existing PlayerManager
@Inject
lateinit var playerManager: PlayerManager

// Add UnifiedPlayerManager (optional, controlled by setting)
@Inject
lateinit var unifiedPlayerManager: UnifiedPlayerManager

// Setting to control which to use
private var useFastZap: Boolean = false  // Can be stored in SettingsManager

private fun playChannel(channel: Channel) {
    if (useFastZap) {
        // Use FastZap
        val nextChannels = predictNextChannels(
            currentChannel = channel,
            allChannels = currentChannelList,
            strategy = NavigationStrategy.LINEAR
        )
        unifiedPlayerManager.playChannel(channel, nextChannels)
    } else {
        // Use standard PlayerManager
        playerManager.playChannel(channel)
    }
}

private fun getActivePlayer(): ExoPlayer? {
    return if (useFastZap) {
        unifiedPlayerManager.getPlayer()
    } else {
        playerManager.getPlayer()
    }
}
```

## Performance Monitoring Integration

### Add Debug Overlay (Optional)

```kotlin
// In MainActivity, add debug overlay
private fun setupDebugOverlay() {
    if (BuildConfig.DEBUG) {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                launch {
                    unifiedPlayerManager.zapTime.collect { zapTimeMs ->
                        binding.debugZapTime.text = "Zap: ${zapTimeMs}ms"
                    }
                }

                launch {
                    unifiedPlayerManager.isBuffering.collect { isBuffering ->
                        binding.debugBuffering.visibility = if (isBuffering) View.VISIBLE else View.GONE
                    }
                }
            }
        }

        // Log metrics every 10 channel zaps
        var zapCount = 0
        lifecycleScope.launch {
            unifiedPlayerManager.zapTime.collect {
                zapCount++
                if (zapCount % 10 == 0) {
                    unifiedPlayerManager.getZapMetrics()?.logSummary()
                }
            }
        }
    }
}
```

### Add to Debug Menu

```kotlin
// In debug menu or long-press action
private fun showFastZapMetrics() {
    val metrics = unifiedPlayerManager.getZapMetrics() ?: return

    val message = """
        FastZap Performance:
        • Total zaps: ${metrics.getTotalZaps()}
        • Avg zap time: ${metrics.getAverageZapTime()}ms
        • Pre-buffer hit rate: ${(metrics.getPreBufferHitRate() * 100).toInt()}%
        • Errors: ${metrics.getErrorCount()}
    """.trimIndent()

    AlertDialog.Builder(this)
        .setTitle("FastZap Metrics")
        .setMessage(message)
        .setPositiveButton("OK", null)
        .setNeutralButton("Reset") { _, _ -> metrics.reset() }
        .show()
}
```

## Testing Checklist

After integration, test these scenarios:

### Basic Functionality
- [ ] Single channel plays correctly
- [ ] Video surface renders properly
- [ ] Audio plays correctly
- [ ] No memory leaks on repeated zaps

### Channel Zapping
- [ ] Channel up/down with D-pad
- [ ] Rapid zapping (press up/down quickly)
- [ ] Jump to channel number
- [ ] Select from channel list
- [ ] Select from EPG

### Pre-buffering
- [ ] Second channel zap is faster than first
- [ ] Pre-buffer hit rate >70% with linear navigation
- [ ] No audio glitches during pre-buffered zap
- [ ] No visual artifacts during surface swap

### Edge Cases
- [ ] Switch between live and VOD
- [ ] Switch between different stream types (HLS/RTSP/DASH)
- [ ] Handle network errors gracefully
- [ ] Handle invalid stream URLs
- [ ] Low memory device behavior

### Performance
- [ ] Zap time <300ms for pre-buffered channels
- [ ] Zap time <2s for cold start channels
- [ ] Memory usage stays within acceptable range
- [ ] No frame drops during playback

## Migration Timeline

### Phase 1: Setup (Day 1)
- Add UnifiedPlayerManager to DI
- Update surface binding in MainActivity
- Test basic playback

### Phase 2: Core Integration (Day 2-3)
- Replace playChannel() calls
- Add pre-buffer queue updates
- Test D-pad navigation

### Phase 3: Optimization (Day 4-5)
- Add smart prediction
- Tune pre-buffer strategies for different UI contexts
- Add performance monitoring

### Phase 4: Testing & Tuning (Day 6-7)
- User testing with various channel counts
- Tune buffer profiles for different networks
- Optimize memory usage
- Fix any issues

### Phase 5: Production Release (Day 8+)
- Add setting to enable/disable FastZap
- Document for users
- Monitor crash reports
- Iterate based on feedback

## Rollback Plan

If FastZap causes issues:

1. Add setting toggle:
```kotlin
// In SettingsManager
val fastZapEnabled = booleanPreferencesKey("fast_zap_enabled")
```

2. Conditional usage:
```kotlin
if (settings.fastZapEnabled) {
    unifiedPlayerManager.setZapMode(ZapMode.FAST_ZAP)
} else {
    unifiedPlayerManager.setZapMode(ZapMode.STANDARD)
}
```

3. Or completely fallback to PlayerManager:
```kotlin
// Comment out UnifiedPlayerManager injection
// @Inject lateinit var unifiedPlayerManager: UnifiedPlayerManager

// Use original PlayerManager
@Inject
lateinit var playerManager: PlayerManager
```

## Support

For issues or questions:
1. Check FASTZAP_README.md
2. Review Timber logs for "FastZap" tag
3. Check metrics with `getZapMetrics().logSummary()`
4. Report issues with metrics included
