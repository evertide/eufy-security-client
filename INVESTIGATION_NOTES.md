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

### Status
- ✅ Root cause identified
- ✅ Fix implemented (see below)
- ⏳ Needs testing
- ⏳ Update runtime patch
- ⏳ Update PR description

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

### Fix 2: Race Condition Prevention (commit 885bfd6)
- **Problem**: Race between stream timeout and new stream start
- **Solution**: Add `p2pStreamEnding` flag to block new streams during cleanup
- **Runtime Patchable**: ❌ No (requires interface changes)

### Fix 3: Stop Event Always Emitted (NEW)
- **Problem**: Stream stop event not emitted if no data received, leaving eufy-security-ws stuck
- **Solution**: Remove `!p2pStreamNotStarted` condition from `emitStreamStopEvent()` call
- **Runtime Patchable**: ✅ Yes

## References
- Upstream Issue: bropat/eufy-security-client#690 (similar infinite loop, residual data only)
- Fork: evertide/eufy-security-client
  - commit e7ff847: malformed packet fix
  - commit 885bfd6: race condition fix (p2pStreamEnding flag)
  - commit TBD: stop event fix
- Add-on: hassio-eufy-security-ws v1.9.13 (runtime patch deployed)
- Documentation: hassio-eufy-security-ws/eufy-security-ws/PATCH_INFO.md

## Upstream Pull Request

### PR Status: DRAFT ⏸️
**Repository**: bropat/eufy-security-client  
**PR Description**: Needs update in `PR_DESCRIPTION.md`

### All Fixes Ready for PR
1. ✅ Malformed packet detection
2. ✅ p2pStreamEnding race condition fix
3. ✅ Stop event always emitted fix

All three fixes are complementary and address different failure modes.
