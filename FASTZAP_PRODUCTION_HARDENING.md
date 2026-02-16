# FastZap Production Hardening Plan

## Executive Summary

FastZap is architecturally sound (9/10 rating) but requires production hardening before shipping to address:
1. ✅ **RTSP multi-connection protection** (FIXED)
2. ⏳ Memory pressure monitoring (TODO)
3. ⏳ Surface lifecycle edge cases (TODO)
4. ⏳ Low-end device stress testing (TODO)
5. ⏳ ANR watchdog logging (TODO)

---

## ✅ COMPLETED: RTSP Protection (Priority 1)

### Problem
RTSP streams use persistent TCP sessions. Pre-buffering creates 2 simultaneous connections:
```
Player A: rtsp://server.com/channel1 (active session)
Player B: rtsp://server.com/channel2 (pre-buffer session)
→ Server sees 2 connections from same IP
→ Triggers "Max connections exceeded"
```

### Solution Implemented
```kotlin
// In FastZapPlayerManager.startPreBuffering()
val streamType = detectStreamType(nextChannel.streamUrl)
if (streamType == StreamType.RTSP) {
    Timber.w("RTSP stream detected - skipping pre-buffer")
    zapMetrics.recordPreBufferSkipped(channelId, "RTSP")
    return@launch  // Skip pre-buffer, use cold start instead
}
```

### Impact
- ✅ RTSP streams no longer trigger multi-connection errors
- ✅ FastZap still works for RTSP (just uses cold start ~1-2s instead of instant)
- ✅ HLS/DASH streams continue to use pre-buffering (<300ms)
- ✅ Metrics track skipped pre-buffers

### Files Modified
- `FastZapPlayerManager.kt` - Added RTSP detection + skip logic
- `ZapMetrics.kt` - Added `recordPreBufferSkipped()` method
- Compiles cleanly ✅

---

## ⏳ TODO: Memory Pressure Monitoring (Priority 2)

### Problem
Dual-player system uses ~200MB. On low-memory devices (<2GB RAM):
- Risk of OOM crashes
- Garbage collection pressure
- Decoder fallback to software
- System may kill background processes

### Current Protection
```kotlin
// AUTO mode checks total RAM at startup
if (totalMemoryMB < 2048) {
    useFastZap = false  // Disable on <2GB devices
}
```

**But:** This is static check. Need **runtime** monitoring.

### Solution to Implement
```kotlin
class MemoryPressureMonitor(private val context: Context) {
    private val activityManager = context.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager

    fun startMonitoring(onPressure: (MemoryPressureLevel) -> Unit) {
        // Register ComponentCallbacks2 to receive memory trim events
        context.registerComponentCallbacks(object : ComponentCallbacks2 {
            override fun onTrimMemory(level: Int) {
                when (level) {
                    ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL -> {
                        onPressure(MemoryPressureLevel.CRITICAL)
                    }
                    ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW -> {
                        onPressure(MemoryPressureLevel.HIGH)
                    }
                    ComponentCallbacks2.TRIM_MEMORY_RUNNING_MODERATE -> {
                        onPressure(MemoryPressureLevel.MODERATE)
                    }
                }
            }

            override fun onConfigurationChanged(config: Configuration) {}
            override fun onLowMemory() {
                onPressure(MemoryPressureLevel.CRITICAL)
            }
        })
    }
}

enum class MemoryPressureLevel {
    NORMAL, MODERATE, HIGH, CRITICAL
}
```

### Integration
```kotlin
// In FastZapPlayerManager.init()
private val memoryMonitor = MemoryPressureMonitor(context)

init {
    memoryMonitor.startMonitoring { level ->
        when (level) {
            MemoryPressureLevel.CRITICAL, MemoryPressureLevel.HIGH -> {
                Timber.w("High memory pressure - releasing buffering player")
                releaseBufferingPlayer()
                // Don't fully disable FastZap, just release buffer player
            }
            else -> {}
        }
    }
}
```

### Expected Impact
- Automatic graceful degradation under memory pressure
- Prevents OOM crashes
- Releases buffering player temporarily
- Can re-enable when pressure subsides

### Estimated Time
- Implementation: 2-3 hours
- Testing: 2-3 hours on low-end devices
- Total: 4-6 hours

---

## ⏳ TODO: Surface Lifecycle Hardening (Priority 3)

### Problem
Surface swapping has edge cases:
- Activity recreation (screen rotation)
- PiP transitions
- HDMI disconnect/reconnect
- App backgrounding
- DRM surface loss

### Current Implementation
```kotlin
fun setSurface(surface: Surface?) {
    currentSurface = surface
    activePlayer?.setVideoSurface(surface)
}
```

**Issue:** No validation, no state tracking.

### Solution to Implement
```kotlin
class SurfaceManager {
    private var currentSurface: Surface? = null
    private var surfaceState: SurfaceState = SurfaceState.DESTROYED

    enum class SurfaceState {
        CREATED, ATTACHED, DETACHED, DESTROYED
    }

    fun setSurface(surface: Surface?) {
        when {
            surface == null && currentSurface != null -> {
                // Surface being destroyed
                Timber.d("Surface destroyed")
                detachPlayers()
                surfaceState = SurfaceState.DESTROYED
            }
            surface != null && currentSurface == null -> {
                // New surface created
                Timber.d("Surface created")
                currentSurface = surface
                attachPlayers(surface)
                surfaceState = SurfaceState.ATTACHED
            }
            surface != currentSurface -> {
                // Surface changed (e.g., PiP transition)
                Timber.d("Surface changed")
                detachPlayers()
                currentSurface = surface
                attachPlayers(surface)
            }
        }
    }

    private fun attachPlayers(surface: Surface) {
        if (!surface.isValid) {
            Timber.e("Attempted to attach invalid surface!")
            return
        }
        activePlayer?.setVideoSurface(surface)
        surfaceState = SurfaceState.ATTACHED
    }

    private fun detachPlayers() {
        activePlayer?.clearVideoSurface()
        surfaceState = SurfaceState.DETACHED
    }
}
```

### Expected Impact
- Prevents crashes from invalid surface usage
- Handles Activity recreation gracefully
- Proper PiP support
- Better logging for debugging

### Estimated Time
- Implementation: 3-4 hours
- Testing: 4-6 hours (requires device testing)
- Total: 7-10 hours

---

## ⏳ TODO: Low-End Device Stress Testing (Priority 4)

### Devices to Test

| Device | RAM | CPU | Why |
|--------|-----|-----|-----|
| Fire Stick Lite | 1GB | Amlogic quad | Absolute minimum |
| Fire Stick 4K (2018) | 1.5GB | Amlogic S905 | Common low-end |
| Mi Box S | 2GB | Amlogic S905X | Mid-range baseline |
| Chromecast HD | 2GB | Amlogic S905X | Google reference |

### Test Scenarios
1. **Cold Start Test**
   - Load 10,000 channels
   - Load 50,000 EPG entries
   - Enable FastZap
   - Monitor memory usage
   - Check for crashes

2. **Sustained Zapping Test**
   - Zap through 100 channels rapidly
   - Monitor decoder pressure
   - Check for frame drops
   - Check for audio glitches

3. **Background Stress Test**
   - Open EPG grid view
   - Enable FastZap
   - Background app (Chrome playing YouTube)
   - Check for OOM kills

4. **Long-Running Test**
   - Play channel for 2 hours
   - FastZap enabled
   - Check for memory leaks
   - Check GC pressure

### Success Criteria
- No crashes on 2GB+ devices
- Graceful degradation on <2GB devices
- <5% frame drop rate on sustained playback
- Memory usage stays within 300MB ceiling

### Estimated Time
- Per device: 4-6 hours
- Total (4 devices): 16-24 hours

---

## ⏳ TODO: ANR Watchdog (Priority 5)

### Problem
If player operations block main thread >5s, Android shows ANR dialog.

### Solution to Implement
```kotlin
object AnrWatchdog {
    private val handler = Handler(Looper.getMainLooper())
    private var isRunning = false

    fun start() {
        if (BuildConfig.DEBUG) {
            isRunning = true
            scheduleCheck()
        }
    }

    private fun scheduleCheck() {
        val startTime = SystemClock.elapsedRealtime()

        handler.postDelayed({
            val elapsed = SystemClock.elapsedRealtime() - startTime
            if (elapsed > 5000) {  // >5s = ANR risk
                Timber.e("ANR RISK: Main thread blocked for ${elapsed}ms")
                // Log stack trace
                Thread.getAllStackTraces().forEach { (thread, stack) ->
                    if (thread.name == "main") {
                        Timber.e("Main thread stack: ${stack.joinToString("\n")}")
                    }
                }
            }

            if (isRunning) {
                scheduleCheck()
            }
        }, 1000)
    }
}
```

### Expected Impact
- Early detection of ANR risks
- Debug logging for troubleshooting
- Prevent user-facing ANR dialogs

### Estimated Time
- Implementation: 1-2 hours
- Testing: 1 hour
- Total: 2-3 hours

---

## 📊 Priority Summary

| Task | Priority | Est. Time | Status | Impact |
|------|----------|-----------|--------|--------|
| RTSP Protection | P1 | - | ✅ Done | Critical |
| Memory Monitoring | P2 | 4-6h | ⏳ TODO | High |
| Surface Hardening | P3 | 7-10h | ⏳ TODO | Medium |
| Device Testing | P4 | 16-24h | ⏳ TODO | High |
| ANR Watchdog | P5 | 2-3h | ⏳ TODO | Low |

**Total Remaining:** ~30-45 hours

---

## 🎯 Recommended Phased Rollout

### Phase 1: Internal Testing (Week 1)
- ✅ RTSP protection (done)
- Implement memory monitoring
- Test on Shield TV / Chromecast 4K (high-end devices)
- Goal: Verify core functionality

### Phase 2: Beta Testing (Week 2)
- Implement surface hardening
- Implement ANR watchdog
- Test on mid-range devices (Mi Box S)
- Limited beta release (50-100 users)
- Goal: Catch edge cases

### Phase 3: Stress Testing (Week 3)
- Test on Fire Stick Lite (worst case)
- 10k channels + 50k EPG stress test
- 24-hour soak test
- Fix any discovered issues
- Goal: Prove stability

### Phase 4: Production Release (Week 4)
- Full release with FastZap enabled (AUTO mode)
- Monitor crash reports
- Monitor user feedback
- Be ready to hotfix if needed
- Goal: Stable release

---

## 🔧 Quick Wins (Can Ship Now)

These are low-risk improvements that don't block release:

1. **Adaptive Pre-buffer Count** (1 hour)
   ```kotlin
   // Reduce pre-buffer count on low memory
   val preBufferCount = if (totalMemoryMB < 3072) 1 else 3
   ```

2. **User-Visible Zap Time** (30 min)
   ```kotlin
   // Show zap time in toast (debug builds)
   if (BuildConfig.DEBUG) {
       Toast.makeText(context, "Zap: ${zapTime}ms", Toast.LENGTH_SHORT).show()
   }
   ```

3. **Connection Limit Warning** (1 hour)
   ```kotlin
   // Detect "Max connections" errors and suggest ALWAYS_OFF mode
   if (error.contains("max connections", ignoreCase = true)) {
       showConnectionLimitDialog()
   }
   ```

---

## 🚀 Minimum Viable Release (MVP)

To ship FastZap **now** with acceptable risk:

**Required:**
- ✅ RTSP protection (done)
- ⏳ Memory monitoring (4-6 hours)
- ⏳ Basic low-end device testing (4-8 hours)

**Optional (post-launch):**
- Surface hardening
- ANR watchdog
- Comprehensive stress testing

**Total MVP Time:** 8-14 hours

**Risk Level:** Medium (AUTO mode provides safety net)

---

## 📝 Documentation Todos

- [ ] Update FASTZAP_README.md with RTSP limitation
- [ ] Add "Known Limitations" section
- [ ] Document memory requirements clearly
- [ ] Add troubleshooting for "Max connections" errors
- [ ] Update changelog with RTSP protection note

---

## 🎓 Lessons Learned

1. **Architectural review is invaluable** - Caught RTSP issue before production
2. **Static checks aren't enough** - Need runtime memory monitoring
3. **Low-end device testing is critical** - Can't assume 4GB RAM
4. **Protocol-specific handling matters** - RTSP ≠ HLS ≠ DASH
5. **Metrics are essential** - Can't fix what you don't measure

---

## 🏁 Conclusion

FastZap is **architecturally excellent (9/10)** but needs **production hardening** before wide release.

**Immediate Next Steps:**
1. ✅ RTSP protection (complete)
2. Implement memory monitoring (4-6 hours)
3. Test on Fire Stick Lite (4-8 hours)
4. Limited beta release (50 users)
5. Monitor + iterate

**Ship Timeline:**
- MVP: 1-2 weeks (with monitoring + basic testing)
- Full hardening: 3-4 weeks (with comprehensive testing)
- Conservative: Start with AUTO mode disabled by default, let users opt-in

**Recommendation:** Ship MVP in 1-2 weeks with AUTO mode enabled, monitor closely, iterate based on real-world data.
