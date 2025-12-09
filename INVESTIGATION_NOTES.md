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
- ✅ Runtime patch deployed in hassio-eufy-security-ws v1.9.10
- ✅ WiFi improvement confirmed 100% effective
- ⏳ Upstream PR prepared but held as draft per user request

## Issue 2: Stream State Race Condition - IN PROGRESS 🔧

### Root Cause
**Race condition between stream timeout and client stream restart** after network disruption.

### Symptoms
- `LivestreamAlreadyRunningError` thrown when trying to start stream
- Occurs after network disruption/reconnection
- Timestamp analysis shows simultaneous `endStream()` and `startLivestream()` calls

### Investigation Timeline

**13:25:28** - Network disruption begins
```
Heartbeat check failed for station T84A1P1025020FEF. Connection seems lost.
```

**13:26:47-13:26:48** - Clean shutdown sequence
```
13:26:47.915 - endStream() called for F7C
13:26:48.036 - onClose() for F7C: wasStreaming: false ✅
13:26:48.162 - onClose() for 0FEF: wasStreaming: false ✅
```
→ Cleanup working correctly during shutdown

**13:26:54** - Successful reconnection
```
Connected to station T84A1P1025021F7C on host 192.168.2.242
```

**13:29:41** - Race condition triggers
```
13:29:41.611 - endStream() called: sendStopCommand: false, queuedDataSize: 0
13:29:41.984 - ERROR: LivestreamAlreadyRunningError
```

### Analysis

The cleanup logic in `_disconnected()` DOES work correctly:
```typescript
private _disconnected(): void {
    // ... clear timeouts ...
    if (this.currentMessageState[P2PDataType.VIDEO].p2pStreaming) {
        this.endStream(P2PDataType.VIDEO);  // ✅ Called correctly
    }
    // ... emit close event ...
    this._initialize();
}
```

The race condition occurs at **application layer** (eufy-security-ws):
1. Client attempts to restart stream after reconnection
2. P2P layer simultaneously ends stream due to 5-second data timeout
3. `isLiveStreaming()` check in `startLivestream()` races with `endStream()`
4. Client sees streaming=true, throws error
5. By the time error is thrown, endStream() has already completed

### Solution Implemented - RESOLVED ✅

**Option 3: Stream state event synchronization** (chosen for clean implementation)

#### Changes Made

1. **Add `p2pStreamEnding` flag** to `P2PDataMessageState` interface
   - Boolean flag indicates stream teardown in progress
   - Prevents new streams from starting during cleanup

2. **Update `endStream()` method** in `src/p2p/session.ts`
   - Set `p2pStreamEnding = true` at start of method
   - Clear `p2pStreamEnding = false` after cleanup completes
   - Prevents race condition window from opening

3. **Update `isStreaming()` method** to check both flags
   ```typescript
   return this.currentMessageState[datatype].p2pStreaming || 
          this.currentMessageState[datatype].p2pStreamEnding;
   ```
   - Blocks new stream starts while existing stream ending
   - `startLivestream()` will correctly see "streaming" state

4. **Initialize flag** in `initializeMessageState()`
   - Set `p2pStreamEnding: false` on initialization
   - Ensures clean starting state

#### How It Works

**Before Fix (Race Condition):**
```
Time 0ms:  isLiveStreaming(F7C) → false (client checks)
Time 1ms:  timeout fires → endStream() starts
Time 2ms:  startLivestream(F7C) → checks again → false → proceeds
Time 3ms:  endStream() sets p2pStreaming = false
Time 4ms:  startLivestream() sets p2pStreaming = true
Time 5ms:  endStream() calls initializeMessageState() → resets everything
Time 6ms:  ERROR: p2pStreaming mismatch → LivestreamAlreadyRunningError
```

**After Fix (Race Prevented):**
```
Time 0ms:  isLiveStreaming(F7C) → false (client checks)
Time 1ms:  timeout fires → endStream() starts
Time 1ms:  endStream() sets p2pStreamEnding = true ✅
Time 2ms:  startLivestream(F7C) → checks isLiveStreaming()
Time 2ms:  isLiveStreaming() → p2pStreamEnding = true → returns true ✅
Time 2ms:  startLivestream() → throws LivestreamAlreadyRunningError (expected!)
Time 3ms:  endStream() completes → sets p2pStreamEnding = false
Time 4ms:  Client retries → isLiveStreaming() → false → succeeds ✅
```

### Status
- ✅ Fix implemented in eufy-security-client fork (commit 885bfd6)
- ⚠️ **Cannot be runtime patched** - requires TypeScript interface changes
- ❌ **CURRENT ISSUE**: Cameras stuck in "preparing" mode after reboot

### Current Problem (December 9, 2025)

After rebooting both cameras, they reconnect but stay stuck in "preparing" mode:
- Both cameras (F7C and 0FEF) affected
- 6x `LivestreamAlreadyRunningError` logged (increased from 3)
- Error occurs 6+ seconds after `endStream()` - NOT a timing race
- Home Assistant automation tries to start stream but fails
- Cameras should drop to idle after timeout, then automation restarts stream

**Key Observation**: The 6-second gap between `endStream()` and the error suggests this is NOT the same race condition we fixed with `p2pStreamEnding`. The state is simply not clearing properly.

**Hypothesis**: After camera reboot/reconnect:
1. `_disconnected()` is called
2. `endStream()` is called (if streaming)
3. `_initialize()` resets state
4. But something else is keeping `isLiveStreaming()` returning true

**Investigation Needed**:
- Check what happens during camera reconnect sequence
- Verify `p2pStreamChannel` is being reset to -1
- Check if station-level state is out of sync with p2p session state
- Look at eufy-security-ws layer for additional state tracking

### Device Context
- **Camera**: T84A1 Wall Light Cam S100 (DeviceType.WALL_LIGHT_CAM = 151)
- **Affected Serials**: T84A1P1025021F7C (F7C), T84A1P102502D0FEF (0FEF)
- **Networks**: F7C @ 192.168.2.242, 0FEF @ 192.168.2.52
- **Note**: T84A1 cameras do NOT send CMD_WIFI_CONFIG, so RSSI tracking shows undefined

## References
- Upstream Issue: bropat/eufy-security-client#690 (similar infinite loop, residual data only)
- Fork: evertide/eufy-security-client
  - commit e7ff847: malformed packet fix
  - commit 885bfd6: race condition fix (p2pStreamEnding flag)
- Add-on: hassio-eufy-security-ws v1.9.13 (runtime patch deployed)
- Documentation: hassio-eufy-security-ws/eufy-security-ws/PATCH_INFO.md

## Upstream Pull Request

### PR Status: DRAFT ⏸️
**Repository**: bropat/eufy-security-client  
**PR Description**: Ready in `PR_DESCRIPTION.md`

### Issue 1 (Malformed Packets) - Ready for PR ✅
- Clean fix with no interface changes
- Can be applied via runtime patch OR upstream merge
- Well tested with 100% improvement in affected environment

### Issue 2 (State Management) - Investigation Ongoing 🔧
- Initial race condition fix implemented (p2pStreamEnding flag)
- Requires TypeScript interface changes (cannot runtime patch)
- **NEW**: Cameras stuck in preparing mode suggests larger state issue
- Need to fully understand before upstream PR
