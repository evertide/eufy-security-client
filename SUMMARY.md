# P2P Stream Issues: Investigation Complete ✅

## Summary

Comprehensive investigation and fix for T84A1 Wall Light Cam S100 P2P streaming issues, addressing both immediate problems and underlying root cause.

## Issues Identified & Resolved

### 1. Malformed P2P Packets - FIXED ✅

**Problem**: "Infinite loop detected" errors causing stream crashes

**Root Cause**: Weak WiFi signal (-70 dBm or worse) causing packet corruption

**Evidence**:
- Before WiFi fix: 420 malformed packets / 11 minutes (38.2/min)
- After WiFi fix: 0 malformed packets / 5 minutes
- **100% elimination** with strong signal

**Solution Implemented**:
- Detect malformed initial packets (first 4 bytes ≠ MAGIC_WORD)
- Discard and log with diagnostics (RSSI, queue size, rssiAge)
- No crash, clean recovery

**Status**: ✅ Fixed in fork (commit e7ff847), runtime patched in add-on v1.9.10

### 2. Stream State Race Condition - FIXED ✅

**Problem**: `LivestreamAlreadyRunningError` after network disruption

**Root Cause**: Race between stream timeout cleanup and client restart

**Timeline**:
```
13:29:41.611 - endStream() called (5s timeout)
13:29:41.984 - LivestreamAlreadyRunningError (373ms race window)
```

**Solution Implemented**:
- Add `p2pStreamEnding` flag to block new streams during teardown
- Prevents race condition by making state transition atomic
- Client retry succeeds after cleanup completes

**Status**: ✅ Fixed in fork (commit 885bfd6), documented for upstream

## Repositories Updated

### eufy-security-client Fork
**Repository**: https://github.com/evertide/eufy-security-client
**Branch**: master
**Latest Commit**: 867abc0

**Changes**:
1. `src/p2p/interfaces.ts` - Added `p2pStreamEnding` flag
2. `src/p2p/session.ts` - Malformed packet detection + race condition fix
3. `INVESTIGATION_NOTES.md` - Complete analysis and testing results
4. `PR_DESCRIPTION.md` - Ready for upstream submission

**Commits**:
- e7ff847 - Malformed packet fix
- 885bfd6 - Race condition fix  
- 867abc0 - Documentation

### hassio-eufy-security-ws Add-on
**Repository**: https://github.com/evertide/hassio-eufy-security-ws
**Branch**: master
**Latest Commit**: e05229a
**Version**: 1.9.10

**Changes**:
1. `eufy-security-ws/patch-eufy-client.sh` - Runtime patches v1.9.10
2. `eufy-security-ws/PATCH_INFO.md` - Complete documentation
3. Patches applied: Malformed packet detection + RSSI tracking + diagnostics

**Commits**:
- 975ddf9 - RSSI tracking v1.9.10
- e05229a - Race condition documentation

## Testing Results

### Malformed Packet Testing
| Condition | Duration | Malformed Packets | Rate | Status |
|-----------|----------|-------------------|------|--------|
| Weak WiFi | 11 min | 420 | 38.2/min | ❌ Failing |
| Strong WiFi | 5 min | 0 | 0/min | ✅ Perfect |
| **Improvement** | - | **-100%** | **-100%** | **✅ Resolved** |

### Race Condition Testing
- Network disruption: Clean disconnect ✅
- Reconnection: Successful with no errors ✅  
- Stream restart: No race condition ✅
- Normal operation: No impact ✅

## Upstream PR Status

**Fork Ready**: ✅ All changes committed and pushed
**PR Draft**: ✅ Complete description in PR_DESCRIPTION.md
**Submission**: ⏸️ Held per user request

**When to submit**:
- User gives approval
- Can be submitted immediately - all documentation ready
- PR includes both fixes with complete testing evidence

## Files Ready for Review

### In eufy-security-client Fork
1. `PR_DESCRIPTION.md` - Complete PR text ready to copy
2. `INVESTIGATION_NOTES.md` - Detailed technical analysis
3. `src/p2p/interfaces.ts` - Interface changes
4. `src/p2p/session.ts` - Implementation

### In hassio-eufy-security-ws
1. `PATCH_INFO.md` - User-facing documentation
2. `patch-eufy-client.sh` - Runtime patches v1.9.10

## Recommendations

### For T84A1 Cameras
1. ✅ Ensure WiFi signal strength -60 dBm or better
2. ✅ Use dedicated 2.4GHz AP on clear channel
3. ✅ Monitor malformed packet rate as signal quality proxy
4. ⚠️ Note: T84A1 cameras don't send CMD_WIFI_CONFIG (no real-time RSSI)

### For Production Use
1. ✅ Current runtime patches (v1.9.10) work well for malformed packets
2. ⚠️ Race condition requires TypeScript changes (can't runtime patch)
3. ✅ eufy-security-ws retry logic handles race condition gracefully
4. 📋 Recommend using fork version for best results

## Next Steps

1. **User Testing**: Monitor stability with current patches
2. **Upstream PR**: Submit when user approves
3. **Long-term**: Switch to official version once merged upstream

## Key Learnings

1. **WiFi Quality Critical**: Weak signal = packet corruption at protocol level
2. **T84A1 Quirks**: No CMD_WIFI_CONFIG messages (unlike other Eufy devices)
3. **Race Conditions**: Network disruptions expose timing issues
4. **Diagnostics Matter**: RSSI/queue logging essential for root cause analysis

---

**Status**: All work complete, ready for upstream submission
**Date**: December 9, 2025
**Maintainer**: @evertide
