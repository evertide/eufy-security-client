# P2P Stream Issues: Investigation Status

## Summary

Investigation and fix development for T84A1 Wall Light Cam S100 P2P streaming issues.

## Development Workflow

### Repository Structure
| Repository | Purpose | GitHub |
|------------|---------|--------|
| `eufy-security-client` | Core P2P library | evertide/eufy-security-client |
| `eufy-security-ws` | WebSocket server | (upstream only) |
| `hassio-eufy-security-ws` | Home Assistant Add-on | evertide/hassio-eufy-security-ws |

### Deploy Cycle
1. **Analyze locally** → Debug in local repos
2. **Push to GitHub** → Commit to our forks
3. **Bump version** → Increment by `.1` in `config.yaml`
4. **HA builds** → Automatic from GitHub
5. **Install & test** → HA installs new version

### Important Limitation
The add-on uses **npm package** (eufy-security-client@3.5.0), NOT our fork.  
Runtime patches via `sed` can only modify JavaScript, not TypeScript interfaces.

## Issues Overview

| Issue | Status | Runtime Patchable | PR Ready |
|-------|--------|-------------------|----------|
| 1. Malformed Packets | ✅ FIXED | ✅ Yes | ✅ Yes |
| 2. State Management | 🔧 IN PROGRESS | ❌ No (needs interface) | ⏸️ Partial |

## Issue 1: Malformed P2P Packets - FIXED ✅

**Problem**: "Infinite loop detected" errors causing stream crashes

**Root Cause**: Weak WiFi signal (-70 dBm+) causing packet corruption

**Evidence**:
- Before WiFi fix: 420 malformed packets / 11 min (38.2/min)
- After WiFi fix: 0 malformed packets / 5 min
- **100% elimination** with strong signal

**Solution**: Detect and discard malformed initial packets (first 4 bytes ≠ MAGIC_WORD)

**Status**: 
- ✅ Fixed in fork (commit e7ff847)
- ✅ Runtime patched in add-on v1.9.13
- ✅ PR description ready

## Issue 2: Stream State Management - IN PROGRESS 🔧

**Problem**: `LivestreamAlreadyRunningError` causing cameras to stay in "preparing" mode

**Initial Diagnosis**: Race condition between stream timeout and client restart (373ms window)

**Initial Fix**: Added `p2pStreamEnding` flag (commit 885bfd6)

**Current Status** (December 9, 2025):
- ❌ After camera reboot, BOTH cameras stuck in "preparing" mode
- ❌ 6x `LivestreamAlreadyRunningError` (increased from 3)
- ❌ Error occurs 6+ seconds after endStream() - NOT a timing race
- ❌ State not clearing properly after disconnect/reconnect

**Investigation Ongoing**:
- Initial race condition fix may be incomplete
- Larger state synchronization issue between P2P and station layers
- Need to trace full reconnect sequence

**Limitation**: Cannot runtime patch - requires TypeScript interface changes

## Repositories Updated

### eufy-security-client Fork
**Repository**: https://github.com/evertide/eufy-security-client
**Branch**: master

**Commits**:
- e7ff847 - Malformed packet fix ✅
- 885bfd6 - Race condition fix (partial) 🔧
- Documentation updates

**Changes**:
1. `src/p2p/interfaces.ts` - Added `p2pStreamEnding` flag
2. `src/p2p/session.ts` - Malformed packet detection + race condition fix
3. `INVESTIGATION_NOTES.md` - Complete analysis
4. `PR_DESCRIPTION.md` - Ready for upstream (Issue 1 only)
5. `SUMMARY.md` - This file

### hassio-eufy-security-ws Add-on
**Repository**: https://github.com/evertide/hassio-eufy-security-ws
**Version**: 1.9.13

**Runtime Patches Applied**:
1. Malformed packet detection ✅
2. WiFi RSSI tracking ✅
3. Connection close diagnostics ✅
4. Stream end diagnostics ✅
5. Race condition detection logging ✅

## Testing Results

### Malformed Packet Testing ✅
| Condition | Duration | Malformed Packets | Status |
|-----------|----------|-------------------|--------|
| Weak WiFi | 11 min | 420 (38.2/min) | ❌ Failing |
| Strong WiFi | 5 min | 0 (0/min) | ✅ Perfect |

### State Management Testing 🔧
| Scenario | Before | After | Status |
|----------|--------|-------|--------|
| Normal operation | OK | OK | ✅ |
| Network disruption | 3 errors | - | Was ⚠️ |
| Camera reboot | - | 6 errors, stuck | ❌ NEW ISSUE |

## Upstream PR Status

### Issue 1 (Malformed Packets) - Ready ✅
- Clean fix, no interface changes
- Can be runtime patched
- Well tested, 100% improvement
- PR description in `PR_DESCRIPTION.md`

### Issue 2 (State Management) - Blocked 🔧
- Initial fix may be incomplete
- Larger issue discovered after camera reboot
- Cannot runtime patch (needs interface changes)
- Need full understanding before PR

## Next Steps

1. **Investigate** why cameras stuck in "preparing" mode
2. **Trace** full P2P reconnect sequence
3. **Find** where state is not being cleared
4. **Fix** the root cause in fork
5. **Test** thoroughly before PR
6. **Submit** Issue 1 PR to upstream (can be done independently)

## Key Learnings

1. **WiFi Quality Critical**: Weak signal = packet corruption
2. **T84A1 Quirks**: No CMD_WIFI_CONFIG (can't track RSSI)
3. **State Issues**: More complex than initially thought
4. **Runtime Patching**: Limited - can't modify TypeScript interfaces

---

**Status**: Issue 1 complete, Issue 2 needs more investigation
**Date**: December 9, 2025
**Maintainer**: @evertide
