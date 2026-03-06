# IDE Click-to-Focus Design

**Date:** 2026-02-20
**Status:** Approved

## Problem

When peon-ping runs inside an IDE's embedded terminal (Cursor, VS Code, Windsurf, Zed), clicking the notification overlay does nothing. `_mac_terminal_bundle_id()` returns empty because `TERM_PROGRAM` is `vscode` (not a standalone terminal), so the click handler never gets registered.

The `_mac_ide_pid()` function already detects IDE ancestor PIDs and passes them as argv[6] to `mac-overlay.js`, but that value is currently unused ("reserved for future").

## Solution

When `bundle_id` is empty, fall back to deriving the bundle ID from the IDE ancestor PID using `lsappinfo` (macOS built-in). This populates `bundle_id` for both the overlay and terminal-notifier notification paths with no additional changes to those paths.

## Changes

### 1. `peon.sh` — Add `_mac_bundle_id_from_pid()`

```bash
_mac_bundle_id_from_pid() {
  local pid="$1"
  [ -z "$pid" ] || [ "$pid" = "0" ] && return
  lsappinfo info -only bundleid -app pid="$pid" 2>/dev/null \
    | grep -o '"[^"]*"' | tr -d '"'
}
```

### 2. `peon.sh` — Fallback logic in `send_notification()`

After computing `bundle_id` and `ide_pid`, if `bundle_id` is empty and `ide_pid > 0`, set `bundle_id` from `_mac_bundle_id_from_pid(ide_pid)`.

### 3. `mac-overlay.js` — PID-based NSRunningApplication fallback

When `bundleId` is empty but `idePid > 0`, use `NSRunningApplication.runningApplicationWithProcessIdentifier_()` to activate the IDE app. Belt-and-suspenders fallback for cases where `lsappinfo` might fail.

### 4. Tests

- BATS: mock `lsappinfo` returning a bundle ID for a given PID
- BATS: `_mac_bundle_id_from_pid` with pid=0 returns empty
- Overlay: verify `ide_pid` activates app when `bundle_id` is empty

## Files Touched

| File | Change |
|---|---|
| `peon.sh` | Add `_mac_bundle_id_from_pid()`, fallback in `send_notification()` |
| `scripts/mac-overlay.js` | PID-based activation fallback |
| `tests/mac-overlay.bats` | IDE click-to-focus tests |
| `tests/setup.bash` | Mock `lsappinfo` |

## Scope Exclusions

- No attempt to switch to the terminal panel within the IDE (just brings window to front)
- No new dependencies (`lsappinfo` is macOS built-in)
- No changes to Windows/Linux paths
