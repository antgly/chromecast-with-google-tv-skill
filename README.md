# Chromecast with Google TV Skill

CLI tooling for controlling a Chromecast with Google TV over ADB. It plays YouTube or Tubi directly, and falls back to Google TV global search for other streaming apps.

## Requirements

- macOS/Linux host with `adb` and `uv` on PATH
- Python 3.11+
- Optional: `yt-api` CLI for resolving YouTube titles to video IDs

## Quick start

```bash
./run status --device 192.168.4.64 --port 5555
./run play "7m714Ls29ZA" --device 192.168.4.64 --port 5555
./run play "family guy" --app hulu --season 3 --episode 4 --device 192.168.4.64 --port 5555
./run pause --device 192.168.4.64 --port 5555
./run resume --device 192.168.4.64 --port 5555
```

## Commands

- `status`: show `adb devices` output
- `play <query_or_id_or_url>`: play via YouTube, Tubi, or global-search fallback
- `pause`: send media pause
- `resume`: send media play

## Device selection

The CLI accepts `--device` (IP) and `--port` (ADB port).

- If only one of `--device` or `--port` is provided, the other is pulled from cache when available.
- If neither is provided, the tool uses the last successful device from `.last_device.json`.
- If no cache exists, it attempts ADB mDNS discovery and uses the first result.
- No port scanning is performed. Only explicit, cached, or mDNS-provided ports are tried.

## Content routing

1. If `play` looks like a YouTube ID or URL, it launches YouTube directly.
2. If it looks like a Tubi URL, it launches Tubi directly.
3. Otherwise it tries to resolve the query to a YouTube ID with `yt-api`.
4. If that fails and `--app`, `--season`, and `--episode` are provided, it uses global search.

## Global search fallback

Global search is implemented in [play_show_via_global_search.py](play_show_via_global_search.py). It expects the device to already be connected by the main CLI and only runs UI automation.

Use it by providing `--app`, `--season`, and `--episode` to `play`:

```bash
./run play "family guy" --app hulu --season 3 --episode 4 --device 192.168.4.64 --port 5555
```

## Environment variables

- `CHROMECAST_HOST`: default device IP
- `CHROMECAST_PORT`: default ADB port
- `YOUTUBE_PACKAGE`: YouTube app package override (default `com.google.android.youtube.tv`)
- `TUBI_PACKAGE`: Tubi app package override (default `com.tubitv`)

## Caching

The last successful device is stored in `.last_device.json` in this folder:

```json
{"ip": "192.168.4.64", "port": 5555}
```

## Tests

```bash
uv run test_google_tv_skill.py
uv run test_google_tv_skill.py -v
uv run test_google_tv_skill.py TestYouTubeIDExtraction
```

## Troubleshooting

- If `adb connect` fails, verify the current port: `adb connect IP:PORT`.
- If connection is refused while running interactively, the CLI will prompt for a new port and update the cache on success.
