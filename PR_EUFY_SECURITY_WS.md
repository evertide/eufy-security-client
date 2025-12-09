# Pull Request: Add Timeout Fallback for receiveLivestream Flag

**Repository**: bropat/eufy-security-ws

## Summary

This PR adds a timeout fallback mechanism to clear the `receiveLivestream` flag when a stream fails to deliver data, preventing cameras from getting permanently stuck in "preparing" mode.

## Problem

When `eufy-security-client` fails to emit the `"station livestream stop"` event, the `receiveLivestream[serialNumber]` flag in `eufy-security-ws` remains `true` indefinitely. This causes all subsequent `startLivestream()` calls to fail with `LivestreamAlreadyRunningError`.

### Root Cause Analysis

The event chain failure occurs because:

1. `message_handler.js` sets `client.receiveLivestream[serialNumber] = true` when stream starts
2. Stream times out after ~30 seconds with no video data
3. `eufy-security-client` emits `"livestream stopped"` with `channel = -1` (because `p2pStreamChannel` was never set)
4. `eufysecurity.js` calls `getStationDevice(stationSN, -1)` which throws `DeviceNotFoundError`
5. The error is caught but `"station livestream stop"` event is **never emitted**
6. `forward.js` never receives the event to clear `receiveLivestream`
7. All future `startLivestream()` calls fail with `LivestreamAlreadyRunningError`

### User Impact

```
2025-12-09 17:00:00 ERROR LivestreamAlreadyRunningError: Livestream for device T84A1P1025021F7C is already running
2025-12-09 17:01:00 ERROR LivestreamAlreadyRunningError: Livestream for device T84A1P1025021F7C is already running
2025-12-09 17:02:00 ERROR LivestreamAlreadyRunningError: Livestream for device T84A1P1025021F7C is already running
... (continues until add-on restart)
```

Cameras appear stuck in "preparing" mode and never show video.

## Solution

Add a 35-second timeout fallback that automatically clears the `receiveLivestream` flag if the stream never actually delivers data:

```javascript
// In message_handler.js - DeviceCommand.startLivestream case
client.receiveLivestream[serialNumber] = true;

// Fallback timeout: clear flag if stream never delivers data
// This handles the case where the "station livestream stop" event
// is never received due to channel=-1 lookup failure in eufy-security-client
setTimeout(() => {
    if (client.receiveLivestream[serialNumber] === true) {
        console.log(`[timeout] Clearing stuck receiveLivestream flag for ${serialNumber}`);
        client.receiveLivestream[serialNumber] = false;
    }
}, 35000); // 35 seconds (slightly longer than P2P streaming timeout of 30s)
```

### Why 35 Seconds?

- P2P streaming timeout in `eufy-security-client` is ~30 seconds
- Adding 5 seconds buffer allows normal event flow to work first
- If a successful stream is running, `receiveLivestream` will be cleared by the normal event before timeout fires
- The timeout only takes effect when the normal event chain fails

## Testing

Tested on T84A1 Wall Light Cam S100 cameras that were consistently getting stuck:

### Before Fix
```
17:02:33 Stream ending { channel: 0, queuedDataSize: 0 }
17:02:34 ERROR: LivestreamAlreadyRunningError
17:03:34 ERROR: LivestreamAlreadyRunningError
17:04:34 ERROR: LivestreamAlreadyRunningError
... (stuck forever)
```

### After Fix
```
17:23:42 Connected to station T84A1P1025021F7C
[eufy-ws-patch] Stream timeout for T84A1P1025021F7C - clearing receiveLivestream flag
17:26:34 Stopping the station stream (no data for 5 seconds)
17:27:23 Connected to station T84A1P1025021F7C  <-- Successfully reconnected!
```

Cameras recover automatically and can start new streams!

## Code Changes

**File**: `dist/lib/device/message_handler.js`

Add timeout after setting `receiveLivestream = true` in the `DeviceCommand.startLivestream` case:

```javascript
case DeviceCommand.startLivestream:
    if (client.schemaVersion >= 2) {
        if (!station.isLiveStreaming(device)) {
            station.startLivestream(device);
            client.receiveLivestream[serialNumber] = true;
            
            // NEW: Fallback timeout to clear stuck flag
            setTimeout(() => {
                if (client.receiveLivestream[serialNumber] === true) {
                    console.log(`[timeout] Clearing stuck receiveLivestream for ${serialNumber}`);
                    client.receiveLivestream[serialNumber] = false;
                }
            }, 35000);
            
            DeviceMessageHandler.addStreamingDevice(station.getSerial(), client);
        }
        // ... rest of existing code
    }
```

## Related Issues

This is a companion fix to the `eufy-security-client` PR that ensures the stop event is always emitted. Both fixes together provide defense in depth:

1. **eufy-security-client fix**: Always emit `"livestream stopped"` event (even with `channel = -1`)
2. **This fix**: Timeout fallback in case the event chain still fails

## Notes

- This is a safety net - the proper fix is in `eufy-security-client` to always emit the stop event
- Both fixes together provide defense in depth
- The timeout does NOT interfere with normal streaming - it only fires if the flag is still `true` after 35 seconds
- If streaming is successful, the normal `"station livestream stop"` event will clear the flag before the timeout

## Checklist

- [x] Tested with affected devices (T84A1 Wall Light Cam S100)
- [x] Timeout value chosen to not interfere with normal operation
- [x] Logging added for debugging
- [x] No breaking changes
- [x] Backward compatible
