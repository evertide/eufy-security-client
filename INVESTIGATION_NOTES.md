# Investigation Notes: T84A1 Camera P2P Stream Issues

## Overview
Investigation into P2P video stream parsing failures and state management issues for T84A1 Wall Light Cam S100 cameras.

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

### Potential Solutions

#### Option 1: Add state synchronization lock (P2P layer)
- Add mutex/lock around streaming state changes
- Prevent `startLivestream()` and `endStream()` from executing simultaneously
- Pro: Fixes race at protocol level
- Con: Requires careful deadlock prevention

#### Option 2: Retry logic with state validation (Application layer)
- Catch `LivestreamAlreadyRunningError` in eufy-security-ws
- Wait brief period and retry if state was transitioning
- Pro: Non-invasive to core library
- Con: Doesn't fix root cause

#### Option 3: Stream state event synchronization
- Add "stream ending" event before actual end
- Block new stream starts while ending in progress
- Pro: Clean separation of concerns
- Con: Requires API changes

#### Option 4: Add "stale check" to isLiveStreaming()
- Check if stream is actually active (recent packets)
- Return false for stale/ending streams
- Pro: Minimal code change
- Con: Adds complexity to state check

### Device Context
- **Camera**: T84A1 Wall Light Cam S100 (DeviceType.WALL_LIGHT_CAM = 151)
- **Affected Serial**: T84A1P1025021F7C
- **Network**: 192.168.2.242:29721
- **Note**: T84A1 cameras do NOT send CMD_WIFI_CONFIG, so RSSI tracking shows undefined

### Next Steps
1. Decide on solution approach
2. Implement fix in eufy-security-client
3. Test with network disruption scenarios
4. Update runtime patch if needed
5. Update upstream PR with both fixes

## References
- Upstream Issue: bropat/eufy-security-client#690 (similar infinite loop, residual data only)
- Fork: evertide/eufy-security-client (commit e7ff847 for malformed packet fix)
- Add-on: hassio-eufy-security-ws v1.9.10 (runtime patch deployed)
- Documentation: hassio-eufy-security-ws/eufy-security-ws/PATCH_INFO.md
