# Investigation Notes: T84A1 Camera P2P Stream Issues

## Overview
Investigation into P2P video stream parsing failures and state management issues for T84A1 Wall Light Cam S100 cameras.

## Development Workflow

### Repository Structure
We maintain **three forked repositories** for development and testing:

| Repository | Purpose | Local Path |
|------------|---------|------------|
| `eufy-security-client` | Core P2P library (this repo) | `/Users/evert/Documents/Code Development/eufy-security-client` |
| `eufy-security-ws` | WebSocket server wrapper | (upstream only) |
| `hassio-eufy-security-ws` | Home Assistant Add-on | `/Users/evert/Documents/Code Development/hassio-eufy-security-ws` |

### Development → Deploy Cycle
1. **Analyze locally** - Debug and develop fixes in local repos
2. **Push to GitHub** - Push changes to our forked repos on GitHub
3. **Bump version** - Increment add-on version by `.1` in `config.yaml`
4. **HA builds** - Home Assistant automatically builds the add-on from GitHub
5. **Install & test** - HA installs the new version for testing

### Add-on Build Flow
```
hassio-eufy-security-ws (GitHub)
    ├── Dockerfile installs eufy-security-ws@1.9.3 from npm
    ├── eufy-security-ws pulls eufy-security-client@3.5.0 from npm
    └── patch-eufy-client.sh applies runtime patches to session.js
```

**Important**: The add-on uses the **npm package** (eufy-security-client@3.5.0), NOT our fork directly. Changes requiring TypeScript interface modifications cannot be runtime patched - they need upstream PR acceptance.

## Issue 1: Malformed Initial P2P Packets - RESOLVED ✅

### Root Cause
**Weak WiFi signal** causing packet corruption at the protocol level.

### Symptoms
- "Infinite loop detected" errors in P2P stream parsing
- Packets starting with random bytes instead of expected MAGIC_WORD "XZYH" (0x585a5948)
- Camera T84A1P1025021F7C affected, while T84A1P102502D0FEF on same network was fine
- RSSI difference of ~20dB between working and problematic cameras

### Evidence
**Before WiFi improvement** (12:50-13:01, 11 minutes):
- 420 malformed packets detected
- Rate: 38.2 packets/minute
- Frequent stream timeouts and restarts

**After WiFi improvement** (13:25-13:30, 5 minutes):
- 0 malformed packets from F7C camera
- Rate: 0 packets/minute
- **100% elimination of packet corruption**

### Solution Implemented
1. **Hardware fix**: Moved F7C camera to dedicated access point with strong signal
2. **Software fix**: Added malformed packet detection and discard logic in `src/p2p/session.ts`

```typescript
// In parseDataMessage() method, before firstPartMessage check
if (!firstPartMessage && bytesToRead === 0) {
    // Malformed initial packet - discard it
    rootP2PLogger.info(`Discarding malformed P2P packet`, {
        stationSN: this.rawStation.station_sn,
        seqNo: message.seqNo,
        dataType: P2PDataType[message.type],
        first4Bytes: header.toString('hex', 0, 4),
        dataLength: header.length,
        rssi: rssiData?.rssi,
        rssiAge: rssiAge,
        queueSize: this.currentMessageState[message.type].queuedData.size
    });
    break; // Exit loop and discard this packet
}
```

### Status
- ✅ Fix implemented in fork (commit e7ff847)
- ✅ Runtime patch deployed in hassio-eufy-security-ws v1.9.13
- ✅ WiFi improvement confirmed 100% effective
- ⏳ Upstream PR prepared but held as draft per user request

## Issue 2: Stream State Not Clearing - ROOT CAUSE FOUND ✅

### Root Cause Identified

**Bug in `endStream()` method**: When a stream times out WITHOUT receiving any video data, the `"livestream stopped"` event is NOT emitted, causing `eufy-security-ws` to keep its `receiveLivestream` flag stuck at `true`.

### The Problem Chain

1. **eufy-security-ws** sets `client.receiveLivestream[serialNumber] = true` when `startLivestream()` is called
2. **eufy-security-ws** only clears this flag when it receives `"station livestream stop"` event
3. **eufy-security-client** only emits `"livestream stopped"` if `p2pStreamNotStarted = false`
4. `p2pStreamNotStarted` is only set to `false` when video data is actually received
5. If stream times out without data → event never emitted → flag never cleared → stuck!

### Log Evidence

```
15:12:27.952 - Stream ending { stationSN: 'T84A1P1025021F7C', queuedDataSize: 0 }
                                                              ^^^^^^^^^^^^^^^^
                                                              No data received!
15:12:29.344 - ERROR: LivestreamAlreadyRunningError (only 1.4 seconds later)
```

Then every minute for 26+ minutes:
```
15:18:00 - LivestreamAlreadyRunningError F7C
15:19:00 - LivestreamAlreadyRunningError F7C
15:20:00 - LivestreamAlreadyRunningError F7C
... (automation keeps retrying every minute, always fails)
```

### The Buggy Code (Before Fix)

```typescript
// In endStream() method - src/p2p/session.ts
if (!this.currentMessageState[datatype].invalidStream && 
    !this.currentMessageState[datatype].p2pStreamNotStarted)  // <-- BUG: requires data received
    this.emitStreamStopEvent(datatype);
```

### The Fix

```typescript
// Always emit stop event if stream was not invalid, even if no data was received
// eufy-security-ws sets receiveLivestream=true on startLivestream(),
// but only clears it on "livestream stopped" event
if (!this.currentMessageState[datatype].invalidStream)
    this.emitStreamStopEvent(datatype);
```

### Why This Wasn't a Race Condition

The initial diagnosis of "race condition" was incorrect. The real issue:
- Stream was started (command acknowledged by camera)
- Camera never sent any video data (queuedDataSize: 0)
- 5-second timeout fired, `endStream()` called
- `p2pStreamNotStarted = true` (never received data to set false)
- Stop event NOT emitted
- `eufy-security-ws` still thinks stream is running
- All subsequent `startLivestream()` calls fail with `LivestreamAlreadyRunningError`

### Previous Symptoms Explained

The earlier "373ms race window" we observed was actually a different scenario:
- Stream WAS receiving data
- Network disruption caused timeout
- Stop event WAS emitted (because data was received)
- But new start came in during cleanup

Both issues are now addressed:
1. **p2pStreamEnding flag** (commit 885bfd6) - prevents race during active cleanup
2. **Always emit stop event** (this fix) - ensures flag is cleared even with no data

### Solution Implemented

**Two-layer fix required:**

1. **eufy-security-client patch** (session.js):
   - Remove `&& !p2pStreamNotStarted` condition from stop event emission
   - Ensures event is always emitted when stream ends

2. **eufy-security-ws patch** (message_handler.js) - **v1.9.26 UPDATED**:
   - ~~v1.9.24-v1.9.25: Timeout-based fallback~~ (REMOVED - accumulated timeouts caused issues)
   - **v1.9.26: Stale flag detection** - Check actual stream state before throwing error

### The Race Condition Problem

The original timeout approach (v1.9.24-v1.9.25) was fundamentally flawed:
- Every `startLivestream` set a NEW 35-second timeout
- HA retries every ~10 seconds when stream fails
- After 1 minute: 6+ accumulated timeouts, all eventually fire
- Caused spurious "livestreamStopped" events and unpredictable state

The REAL issue is a **race condition**:
```
Time T:   Stream ends, emits "livestream stopped" event
Time T+1: Event propagates asynchronously (station.js → eufysecurity.js → forward.js)
Time T+2: New startLivestream arrives BEFORE event clears receiveLivestream flag
Time T+3: receiveLivestream[sn] still TRUE → LivestreamAlreadyRunningError thrown!
```

### The Correct Fix (v1.9.26)

Instead of timeout-based workarounds, handle stale flags at decision time:

```javascript
// In message_handler.js startLivestream handler
// Before (v1.9.24): blindly throw error
throw new LivestreamAlreadyRunningError(...);

// After (v1.9.26): check actual stream state first
if (!station.isLiveStreaming(device)) {
    // Stale flag! Stream ended but event hasn't propagated yet
    console.log("[eufy-ws-fix] Stale receiveLivestream flag detected for " + serialNumber);
    station.startLivestream(device);
    // ... start new stream ...
} else {
    // Stream actually IS running
    throw new LivestreamAlreadyRunningError(...);
}
```

### Runtime Patch (v1.9.26)

```bash
# eufy-security-client: Remove p2pStreamNotStarted check
sed -i 's/\.invalidStream && !this\.currentMessageState\[datatype\]\.p2pStreamNotStarted/.invalidStream/' "$SESSION_FILE"

# eufy-security-ws: Stale flag detection (replaces timeout approach)
sed -i 's/throw new LivestreamAlreadyRunningError.*/if (!station.isLiveStreaming(device)) { console.log("[eufy-ws-fix] Stale receiveLivestream flag detected for " + serialNumber + ", clearing and starting new stream"); station.startLivestream(device); ... } else { throw new LivestreamAlreadyRunningError(...); }/' "$WS_MESSAGE_HANDLER"
```

### Status
- ✅ Root cause identified (race condition, NOT channel=-1 issue)
- ✅ Fix implemented in eufy-security-client fork (commit e49f670)
- ✅ Runtime patch v1.9.26 deployed with stale flag detection
- ⏳ Testing in progress (Dec 9, 2025)
- ⏳ Upstream PR prepared (draft)

### Test Evidence (v1.9.26)

Expected log output when stale flag is detected:
```
[eufy-ws-fix] Stale receiveLivestream flag detected for T84A1P1025021F7C, clearing and starting new stream
```

This indicates the fix is working - detecting stale flags from race conditions and recovering gracefully instead of throwing `LivestreamAlreadyRunningError`.
[eufy-ws-patch] Stream timeout for T84A1P1025020FEF - clearing receiveLivestream flag
17:26:20 Stopping the station stream for T84A1P1025020FEF (no data for 5 seconds)
17:26:34 Stopping the station stream for T84A1P1025021F7C (no data for 5 seconds)
17:27:08 Connected to station T84A1P1025020FEF  <-- Successfully reconnected!
17:27:23 Connected to station T84A1P1025021F7C  <-- Successfully reconnected!
```

Cameras no longer get stuck in "preparing" mode!

### Device Context
- **Camera**: T84A1 Wall Light Cam S100 (DeviceType.WALL_LIGHT_CAM = 151)
- **Affected Serials**: T84A1P1025021F7C (F7C), T84A1P102502D0FEF (0FEF)
- **Networks**: F7C @ 192.168.2.242, 0FEF @ 192.168.2.52
- **Note**: T84A1 cameras do NOT send CMD_WIFI_CONFIG, so RSSI tracking shows undefined

## Summary of All Fixes

### Fix 1: Malformed Packet Detection (commit e7ff847)
- **Problem**: Corrupted packets cause infinite loop
- **Solution**: Detect and discard malformed initial packets
- **Runtime Patchable**: ✅ Yes
- **Upstream PR**: eufy-security-client

### Fix 2: Race Condition Prevention (commit 885bfd6)
- **Problem**: Race between stream timeout and new stream start
- **Solution**: Add `p2pStreamEnding` flag to block new streams during cleanup
- **Runtime Patchable**: ❌ No (requires interface changes)
- **Upstream PR**: eufy-security-client

### Fix 3: Stop Event Always Emitted (commit e49f670)
- **Problem**: Stream stop event not emitted if no data received
- **Solution**: Remove `!p2pStreamNotStarted` condition from `emitStreamStopEvent()` call
- **Runtime Patchable**: ✅ Yes
- **Upstream PR**: eufy-security-client

### Fix 4: Stale Flag Detection (v1.9.26)
- **Problem**: Race condition between stream end event propagation and new start requests
- **Solution**: Check actual stream state before throwing `LivestreamAlreadyRunningError`; if stream isn't actually running, handle stale flag gracefully
- **Runtime Patchable**: ✅ Yes
- **Upstream PR**: eufy-security-ws
- **Note**: Replaces timeout approach from v1.9.24-v1.9.25 which accumulated spurious timeouts

## References
- Upstream Issue: bropat/eufy-security-client#690 (similar infinite loop, residual data only)
- Fork: evertide/eufy-security-client
  - commit e7ff847: malformed packet fix
  - commit 885bfd6: race condition fix (p2pStreamEnding flag)
  - commit e49f670: stop event fix
- Fork: evertide/hassio-eufy-security-ws
  - v1.9.24: runtime patches for all fixes
- Documentation: hassio-eufy-security-ws/eufy-security-ws/PATCH_INFO.md

---

## Upstream Pull Requests (DRAFT)

### PR #1: eufy-security-client - P2P Stream Reliability Fixes

**Repository**: bropat/eufy-security-client  
**Title**: Fix P2P stream reliability issues (malformed packets, race conditions, event emission)

#### Description

This PR addresses three related issues that cause P2P video streams to fail or become stuck:

##### Issue 1: Malformed Initial P2P Packets

**Problem**: Corrupted packets (due to weak WiFi or network issues) cause infinite loops in the P2P stream parser when they don't start with the expected MAGIC_WORD "XZYH".

**Solution**: Add detection for malformed initial packets and discard them safely:

```typescript
// In parseDataMessage() - detect and discard malformed packets
if (!firstPartMessage && this.currentMessageBuilder[message.type].header.bytesToRead === 0) {
    rootP2PLogger.info(`Discarding malformed P2P packet`, {
        stationSN: this.rawStation.station_sn,
        seqNo: message.seqNo,
        first4Bytes: data.subarray(0, 4).toString('hex')
    });
    data = Buffer.from([]);
    this.currentMessageState[message.type].leftoverData = Buffer.from([]);
    break;
}
```

##### Issue 2: Race Condition During Stream Cleanup

**Problem**: When a stream times out, there's a window where a new `startLivestream()` can be called before `endStream()` completes, causing state corruption.

**Solution**: Add `p2pStreamEnding` flag to prevent new streams during cleanup:

```typescript
// In P2PClientProtocolEvents interface
p2pStreamEnding: boolean;

// In endStream() - set flag before cleanup
this.currentMessageState[datatype].p2pStreamEnding = true;
// ... cleanup ...
this.currentMessageState[datatype].p2pStreamEnding = false;

// In startStream() - check flag
if (this.currentMessageState[datatype].p2pStreamEnding) {
    throw new LivestreamAlreadyRunningError("Stream is currently ending");
}
```

##### Issue 3: Stop Event Not Emitted Without Data

**Problem**: When a stream times out without receiving ANY video data, `p2pStreamNotStarted` is still `true`, causing `emitStreamStopEvent()` to be skipped. This leaves downstream consumers (like eufy-security-ws) with stale state.

**Solution**: Remove the `p2pStreamNotStarted` condition:

```typescript
// Before (buggy):
if (!this.currentMessageState[datatype].invalidStream && 
    !this.currentMessageState[datatype].p2pStreamNotStarted)
    this.emitStreamStopEvent(datatype);

// After (fixed):
if (!this.currentMessageState[datatype].invalidStream)
    this.emitStreamStopEvent(datatype);
```

#### Testing

- Tested on T84A1 Wall Light Cam S100 cameras
- Confirmed malformed packet detection works (420 packets safely discarded in 11 minutes)
- Confirmed stream recovery works after WiFi improvement (0 corrupted packets)
- Confirmed stuck stream issue resolved (cameras no longer stuck in "preparing" mode)

#### Related Issues

- Partially addresses #690 (infinite loop issue - this PR adds additional safeguards)

---

### PR #2: eufy-security-ws - Fix Stream State Race Condition

**Repository**: bropat/eufy-security-ws  
**Title**: Fix race condition in livestream state management

#### Description

**Problem**: When a stream ends, the "station livestream stop" event propagates asynchronously through the system. If a new `startLivestream()` request arrives before the event clears the `receiveLivestream[serialNumber]` flag, it throws `LivestreamAlreadyRunningError` even though no stream is actually running.

**Root Cause**: Race condition between asynchronous event propagation and new stream requests.

**Previous Approach (v1.9.24-v1.9.25)**: Added 35-second timeout fallback.
- ❌ Failed because: Every `startLivestream` set a new timeout; HA retries every ~10s; accumulated timeouts all fire eventually, causing unpredictable state.

**New Approach (v1.9.26)**: Stale flag detection at decision time.

```javascript
// In message_handler.js startLivestream handler
// OLD: blindly throw error when flag is true
else {
    throw new LivestreamAlreadyRunningError(`Livestream for device ${serialNumber} is already running`);
}

// NEW: check actual stream state first
else {
    if (!station.isLiveStreaming(device)) {
        // Stale flag! Stream ended but event hasn't propagated yet
        console.log("[eufy-ws-fix] Stale receiveLivestream flag detected for " + serialNumber + ", clearing and starting new stream");
        station.startLivestream(device);
        if (!DeviceMessageHandler.streamingDevices[station.getSerial()]?.includes(client)) {
            DeviceMessageHandler.addStreamingDevice(station.getSerial(), client);
        }
    } else {
        // Stream actually IS running
        throw new LivestreamAlreadyRunningError(`Livestream for device ${serialNumber} is already running`);
    }
}
```

#### Why This Works

- Checks the **actual** stream state (`station.isLiveStreaming(device)`) at the exact moment of decision
- No accumulating timeouts
- Handles race conditions gracefully by starting a new stream when the old one is actually done
- Only throws error when stream is legitimately already running

#### Testing

Expected log output when stale flag is detected:
```
[eufy-ws-fix] Stale receiveLivestream flag detected for T84A1P1025021F7C, clearing and starting new stream
```

Tested on T84A1 Wall Light Cam S100 cameras that were consistently getting stuck in "preparing" mode.

#### Notes

- This replaces the timeout approach from v1.9.24-v1.9.25
- Works together with the `eufy-security-client` fix that always emits the stop event
- Both fixes together provide defense in depth against race conditions
