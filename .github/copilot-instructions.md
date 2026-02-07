# Google TV Skill - AI Agent Guidelines

## Project Purpose
Python skill for controlling Chromecast with Google TV via ADB. Supports:
- Direct YouTube playback (ID/URL extraction + optional yt-api resolution)
- Tubi URL launching
- Fallback global search for other streaming apps (Hulu, Max, Disney+, etc.)
- Pause/resume media control and device status checks

## Architecture: Two-Script Design Pattern

**Separation of Concerns** - the codebase maintains clear boundaries:

- **`google_tv_skill.py`** (Primary CLI): Handles ALL device connection logic
  - ADB connection management, retry with exponential backoff
  - Device discovery: explicit IP:PORT → cache → mDNS → interactive prompt
  - Content routing logic (YouTube → Tubi → global search fallback)
  - Exit codes: 0=success, 2=connection error, 3=resolution failure, 4=adb command error

- **`play_show_via_global_search.py`** (UI Automation Helper): Assumes pre-connected device
  - Called via `uv run` with already-verified device address
  - Only orchestrates UI keypress sequences via ADB
  - No connection discovery; receives device as `--device` argument
  - Never modifies `.last_device.json`

**Key Principle**: Do not add SDK discovery or connection logic to the helper; keep it focused on UI automation.

## Device Connection Strategy (Non-Destructive)

`ensure_connected()` implements intelligent fallback chain:

1. **Explicit IP + Port** (from CLI or env vars)
2. **Cached Device** (if partial args provided, e.g. only IP)
3. **mDNS Service Discovery** (fallback if no cache; discovers first visible device)
4. **Interactive Port Prompt** (only if stdin.isatty() on connection refused)
5. **Cached Fallback Retry** (if primary fails, try cache before giving up)

**Non-Destructive Design**: Does NOT perform port scanning. Will only attempt explicit ports or mDNS results. Interactive prompts only appear on explicit connection refused + tty.

**Caching**: Last successful IP:PORT stored in `.last_device.json` (JSON format: `{"ip": "192.168.x.x", "port": 5555}`). Cache enables quick reconnection without rediscovery.

## Content Resolution Flow

```
play <query> [--app <name> --season N --episode N]
  ↓
1. Try extract_youtube_id() - detects YouTube IDs and URLs (YouTube, youtu.be, shorts, embeds)
   ├─ Success → launch_youtube_intent() via ADB
   └─ Failure → next
2. Check looks_like_tubi() - minimal pattern match (https://tubitv.com/...)
   ├─ Matches → handle_tubi() with VIEW intent
   └─ No match → next
3. Try resolve_youtube_id_with_yt_api() - subprocess call to yt-api CLI
   ├─ Success → launch_youtube_intent()
   └─ Failure → next
4. Global Search Fallback - requires --app, --season, --episode
   ├─ Valid args → launch_global_search_show() → delegate to helper
   └─ Missing args → exit 1 with usage hint
5. Failure → exit 3 (resolution failed)
```

The resolution chain is designed to be lenient upfront. Only require --season/--episode when none of the primary methods succeed.

## Key Constants & Defaults

| Item | Value | Notes |
|------|-------|-------|
| YouTube Package | `com.google.android.youtube.tv` | Override via `YOUTUBE_PACKAGE` env |
| Tubi Package | `com.tubitv` | Override via `TUBI_PACKAGE` env |
| Device Override Env | `CHROMECAST_HOST`, `CHROMECAST_PORT` | Checked in main() |
| ADB Timeout | 10 seconds | Applied to all subprocess.run() calls |
| Connect Attempts | 3 (with 0.5s → 1s → 2s backoff) | Short exponential backoff |
| Cache File | `.last_device.json` | Relative to skill directory |

## YouTube ID/URL Extraction Patterns

`extract_youtube_id()` recognizes:
- Direct 11-char IDs (regex: `[A-Za-z0-9_-]{6,}`)
- YouTube URLs: `youtube.com/watch?v=ID`, `youtu.be/ID`, `/shorts/ID`, `/live/ID`, `/embed/ID`
- Fragment URLs: `youtube.com#watch?v=ID`

All URL parsing is defensive; malformed URLs return `None`.

## Global Search UI Automation (Helper Script)

The helper receives fully-qualified device name (from `adb devices`) and orchestrates keypresses:

- Waits for Series Overview UI (60s timeout)
- Navigates to Seasons → target season → target episode
- Triggers "Open in <app>" dialog
- Minimal timing: transition waits (3s), focus waits (0.5s)

**Troubleshooting patterns**:
- View mismatch → timeout expiry (device might be on different UI)
- Keypress batch limits (10 presses per 0.1s interval) to avoid buffer overflow on slow devices
- No recovery for mid-automation connection loss

## Testing & Validation

### Unit Tests

A comprehensive test suite covers core logic without requiring device connectivity:

```bash
uv run test_google_tv_skill.py           # Run all tests
uv run test_google_tv_skill.py -v        # Run with verbose output
uv run test_google_tv_skill.py TestYouTubeIDExtraction  # Run specific test class
```

**Test Coverage** (47 unit tests):
- **YouTube ID extraction**: Direct IDs, youtube.com/watch, youtu.be, /shorts/, /live/, /embed/, fragment URLs
- **YouTube ID validation**: Valid/invalid formats, length constraints, special characters
- **Tubi detection**: URL patterns, case-insensitivity, partial matching
- **Connection refused detection**: Various error message patterns
- **Cache operations**: Save, load, invalid JSON, missing fields, port type validation
- **Package name resolution**: Defaults, environment overrides, whitespace stripping
- **Video ID finding**: Recursive JSON traversal (videoId, video_id, id keys)

All tests use Python's `unittest` module (no external dependencies). Tests use `unittest.mock` to patch file I/O and environment variables.

### Manual Integration Testing

Test command-line scenarios with a real device:

```bash
./run status --device 192.168.4.64 --port 5555
./run play "7m714Ls29ZA" --device 192.168.4.64 --port 5555  # Direct YouTube ID
./run play "family guy" --app hulu --season 3 --episode 4 --device 192.168.4.64 --port 5555
./run pause --device 192.168.4.64 --port 5555
```

**Cache testing**:
```bash
echo '{"ip": "192.168.4.64", "port": 5555}' > .last_device.json
./run status  # Should use cached values
```

### Untested Scenarios

The following require device interaction and are validated manually:
- ADB connection management (handled by `adb_connect()` with retry logic)
- mDNS device discovery
- Interactive port prompts (requires tty)
- UI automation via `play_show_via_global_search.py` (keypress sequences, timing)
- ADB intent execution (YouTube/Tubi app launching)

## Common Modification Patterns

**Adding a new streaming app fallback**:
1. Add pattern detection in `play_cmd()` (e.g., `looks_like_netflix()`)
2. Import package name as env override
3. Route to `launch_global_search_show()` with app name and args

**Changing timeout behavior**:
- `ADB_TIMEOUT_SECONDS` is module constant; affects `run_adb()`, `adb_shell()`
- Global search helper has separate `ADB_TIMEOUT_SECONDS` (same value for consistency)

**Modifying connection retry strategy**:
- Edit `adb_connect()` retry loop and backoff formula
- Consider impact on mDNS fallback timing in `ensure_connected()`

## Dependencies & Runtime

- **Python 3.11+** (via PEP 723 inline script metadata)
- **System binaries**: `adb`, `uv` (on PATH)
- **Optional**: `yt-api` CLI for YouTube ID resolution
- **No pip packages** - uses Python standard library only

`run` bash wrapper validates `uv` and `adb` availability before delegating to `google_tv_skill.py`.

## Environment Variables & Configuration

| Env Var | Purpose | Example |
|---------|---------|---------|
| `CHROMECAST_HOST` | Default device IP | `192.168.4.64` |
| `CHROMECAST_PORT` | Default ADB port | `5555` |
| `YOUTUBE_PACKAGE` | YouTube app package override | `com.google.android.youtube.tv` |
| `TUBI_PACKAGE` | Tubi app package override | `com.tubitv` |

Env vars are checked in `main()` after argparse if CLI args are empty.

## Intent Construction Patterns

ADB intent launch format:
```
am start -a android.intent.action.VIEW -d <URL> [-p <package>]
```

- YouTube: restricts intent to YouTube package (makes YouTube TV the preferred handler)
- Tubi: restricts intent to Tubi package
- Global search: launched via `android.search.action.GLOBAL_SEARCH`, no package restriction (Google TV system dialog)
