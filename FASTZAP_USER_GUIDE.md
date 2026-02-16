# FastZap User Guide

## What is FastZap?

**FastZap** is an instant channel zapping feature that makes changing channels nearly instant (<300ms) by intelligently pre-buffering your next likely channel in the background.

### Traditional Channel Switching
```
User presses DOWN → Stop current → Load new channel → Play
                    ────────────── 1-3 seconds ──────────────
```

### FastZap Channel Switching
```
User presses DOWN → Instant switch (already buffered!)
                    ────────── <300ms ──────────
```

## Should I Enable FastZap?

### ✅ Enable FastZap If:
- ✓ Your device has **2GB+ RAM**
- ✓ Your IPTV subscription allows **multiple connections**
- ✓ You want the **fastest possible** channel zapping
- ✓ You frequently channel surf (up/down navigation)

### ❌ Disable FastZap If:
- ✗ Your IPTV subscription **limits to 1 connection**
- ✗ Your device has **less than 2GB RAM**
- ✗ You see **"Max connections exceeded"** errors
- ✗ You want to **minimize memory usage**
- ✗ You want to **save battery** (mobile devices)

## FastZap Modes

### 1. AUTO Mode (Recommended)
**What it does:**
- Automatically enables FastZap on devices with 2GB+ RAM
- Automatically disables on low-memory devices
- Smart detection based on device capabilities

**Use this if:**
- You're not sure which mode to choose
- You want the app to decide for you
- You switch between multiple devices

### 2. Always On (Performance)
**What it does:**
- Forces FastZap enabled regardless of device
- Uses ~200MB memory
- Opens 2 stream connections (one active, one pre-buffering)

**Use this if:**
- You have a high-end Android TV box (4GB+ RAM)
- Your IPTV allows multiple simultaneous connections
- You want maximum performance

⚠️ **Warning:** Some IPTV providers limit connections to 1. If you see "Max connections exceeded" errors, switch to "Always Off" mode.

### 3. Always Off (Single-Connection Safe)
**What it does:**
- Forces standard single-player mode
- Uses ~80MB memory
- Only opens 1 stream connection at a time
- Channel zap time: 1-3 seconds

**Use this if:**
- Your IPTV subscription **limits to 1 connection**
- You see "Too many connections" or "Max streams exceeded" errors
- You have a low-memory device (Fire Stick Lite, old Android TV)
- You want to minimize memory/battery usage

## Common IPTV Connection Limits

| Provider Type | Typical Limit | FastZap Safe? |
|---------------|---------------|---------------|
| Premium IPTV | 2-5 connections | ✅ Yes |
| Budget IPTV | 1 connection | ❌ No - Use "Always Off" |
| Free IPTV trials | 1 connection | ❌ No - Use "Always Off" |
| Family/Multi-room | 3+ connections | ✅ Yes |

**How to check:** Contact your IPTV provider or check your account dashboard for "Max simultaneous streams" or "Max connections" setting.

## Memory Usage Comparison

```
┌─────────────────────────────────────────────────┐
│                                                  │
│  Standard Mode (Always Off):                    │
│  ├─ Single player:      ~80 MB                  │
│  └─ Total:              ~80 MB                  │
│                                                  │
│  FastZap Mode (Always On):                      │
│  ├─ Player A (Active):  ~80 MB                  │
│  ├─ Player B (Buffer):  ~80 MB                  │
│  ├─ Pre-buffer queue:   ~40 MB                  │
│  └─ Total:              ~200 MB                 │
│                                                  │
│  Memory Saved (Always Off): ~120 MB             │
│                                                  │
└─────────────────────────────────────────────────┘
```

## Device Recommendations

### High-End Devices (4GB+ RAM)
**Recommended:** Always On
- Nvidia Shield TV
- Chromecast with Google TV (4K)
- Mi Box S
- Fire TV Cube
- High-end Android TV boxes

### Mid-Range Devices (2-4GB RAM)
**Recommended:** AUTO
- Fire TV Stick 4K
- Chromecast with Google TV (HD)
- Mid-range Android TV boxes
- Most Android tablets

### Low-End Devices (<2GB RAM)
**Recommended:** Always Off
- Fire TV Stick Lite
- Old Chromecast devices
- Budget Android TV boxes
- Old Android phones/tablets

## Troubleshooting

### "Too Many Connections" / "Max Streams Exceeded" Error

**Cause:** Your IPTV provider limits simultaneous connections, and FastZap tries to open 2 streams (one active, one pre-buffering).

**Solution:**
1. Go to **Settings** → **Player** → **FastZap Mode**
2. Select **"Always Off"**
3. Restart the app

This forces single-connection mode, which only opens one stream at a time.

### App Crashes or Freezes

**Cause:** Device doesn't have enough memory for FastZap's dual-player system.

**Solution:**
1. Go to **Settings** → **Player** → **FastZap Mode**
2. Select **"Always Off"**
3. Clear app cache (Settings → Apps → PureFusionIPTV → Clear Cache)
4. Restart the app

### FastZap Isn't Fast

**Symptoms:** Channel switching still takes 1-3 seconds despite FastZap being enabled.

**Possible causes:**
1. **First channel switch:** First zap is always cold start (1-3s). Second+ zaps should be instant.
2. **Random channel jumps:** FastZap pre-buffers sequential channels (up/down). Random jumps won't be pre-buffered.
3. **Slow network:** Pre-buffering needs time to download. Wait 1-2 seconds after opening app.

**Solution:**
- Test by pressing DPAD_DOWN twice. Second zap should be <300ms.
- Navigate channels sequentially (up/down) for best performance.
- Ensure stable internet connection (5+ Mbps).

### High Battery Drain (Mobile Devices)

**Cause:** FastZap pre-buffers streams in the background, using more CPU/network.

**Solution:**
1. For mobile devices, use **"Always Off"** mode
2. Or use **AUTO** mode (automatically disables on low-memory devices)

## Settings Location

1. Open **PureFusionIPTV**
2. Navigate to **Settings** → **Player Settings**
3. Scroll to **"FastZap"** section
4. Toggle **"Enable FastZap"** on/off
5. Select **FastZap Mode:**
   - AUTO (recommended)
   - Always On (performance)
   - Always Off (single-connection safe)

## Performance Metrics

### With FastZap (Always On)
- First channel zap: 1-2 seconds (cold start)
- Second+ zaps: <300ms (instant!)
- Memory usage: ~200MB
- Connections: 2 simultaneous

### Without FastZap (Always Off)
- All channel zaps: 1-3 seconds
- Memory usage: ~80MB
- Connections: 1 only

## FAQ

### Q: Does FastZap work with VOD (movies/series)?
**A:** FastZap is optimized for live TV channel surfing. For VOD, it provides minimal benefit since users don't rapidly switch between movies.

### Q: Will FastZap use more internet data?
**A:** Yes, slightly. FastZap pre-buffers the next channel in the background (~5-10 seconds of video). On a 5 Mbps stream, this is ~3-6 MB extra per channel zap. If you channel surf heavily, this can add up.

**Recommendation:** Use "Always Off" on metered/capped connections.

### Q: Can I check if my IPTV allows multiple connections?
**A:** Yes:
1. Play a channel on PureFusionIPTV
2. Try playing the SAME playlist on another device/app
3. If second device shows "Max connections exceeded", your IPTV limits to 1 connection
4. Use "Always Off" mode

### Q: Does FastZap work with Xtream Codes playlists?
**A:** Yes! FastZap works with all playlist types (M3U, Xtream Codes, local files).

### Q: My device has 3GB RAM but FastZap isn't enabled?
**A:** Check your FastZap Mode setting:
- If set to "Always Off", it's forced disabled
- If set to "AUTO", it should auto-enable
- Try setting to "Always On" manually

### Q: Can I switch modes without restarting the app?
**A:** Changing FastZap mode requires the player to reinitialize. Current playback will stop briefly, but you don't need to restart the entire app.

## Best Practices

### For Maximum Performance
1. Use **"Always On"** mode
2. Navigate channels sequentially (up/down arrows)
3. Use Favorites list (FastZap learns your patterns)
4. Ensure stable 5+ Mbps internet

### For Maximum Compatibility
1. Use **"Always Off"** mode
2. Works with all IPTV providers (including 1-connection limits)
3. Lower memory usage (works on all devices)
4. Single stream connection only

### For Smart Balance
1. Use **"AUTO"** mode (recommended)
2. App decides based on device capabilities
3. Best for most users
4. Automatically adapts to your device

## Reporting Issues

If FastZap causes problems:

1. Switch to "Always Off" mode to verify it's FastZap-related
2. Note your device model and RAM
3. Note your IPTV provider (if comfortable sharing)
4. Report issue to: https://github.com/Eliminater74/PureFusionIPTV/issues

Include:
- Device model
- Android version
- RAM amount
- FastZap mode used
- Error message (if any)

---

**FastZap is designed to give you TiviMate-class instant channel zapping while respecting your IPTV provider's connection limits and your device's capabilities.** Choose the mode that best fits your situation!
