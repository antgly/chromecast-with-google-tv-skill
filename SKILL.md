---
name: google_tv
description: Cast YouTube and Tubi video content to Chromecast with Google TV via ADB
---

# Chromecast with Google TV control

Use this skill when I ask to cast YouTube or Tubi video content, play or pause Chromecast media playback, or check if the Chromecast is online.

## Setup

This skill runs with system `python3` and `adb` on PATH. No venv required.

- Ensure `python3` and `adb` are available on PATH.
- Use `./run` as a convenience wrapper around `python3 google_tv_skill.py`.

## Capabilities

This skill provides a small CLI wrapper around ADB to control a Google TV device. It exposes the following subcommands:

- status: show adb devices output
- play <query_or_id_or_url>: play content. Prefer providing a YouTube video ID or a provider URL/ID.
- pause: send media pause
- resume: send media play
- doctor: verify dependencies and show cached device info

### Usage examples

`./run status --device 192.168.4.64 --port 5555`

`./run play "7m714Ls29ZA" --device 192.168.4.64 --port 5555`

`./run pause --device 192.168.4.64 --port 5555`

### Device selection and env overrides

- You can pass --device (IP) and --port on the CLI.
- Alternatively, set CHROMECAST_HOST and CHROMECAST_PORT environment variables to override defaults.
- If you provide only --device or only --port, the script will use the cached counterpart when available; otherwise it will error.
- The script caches the last successful IP:PORT to `.last_device.json` in the skill folder and will use that cache if no explicit device is provided.
- IMPORTANT: This skill does NOT perform any port probing or scanning. It will only attempt an adb connect to the explicit port provided or the cached port.

### YouTube handling

- If you provide a YouTube video ID or URL, the skill will launch the YouTube app directly via an ADB intent restricted to the YouTube package.
- The skill attempts to resolve titles/queries to a YouTube video ID using the `yt-api` CLI (on PATH). If ID resolution fails, the skill will report failure.
- You can override the package name with `YOUTUBE_PACKAGE` (default `com.google.android.youtube.tv`).

### Tubi handling

- If you provide a Tubi https URL, the skill will send a VIEW intent with that URL (restricted to the Tubi package).
- If the canonical Tubi https URL is needed, the assistant can look it up via web_search and supply it to this skill.
- You can override the package name with `TUBI_PACKAGE` (default `com.tubitv`).

### Pause / Resume

`./run pause`
`./run resume`
`./run doctor`
`./run doctor --connect --device 192.168.4.64 --port 5555`

### Dependencies

- The script uses only the Python standard library (no pip packages required).
- The script expects `adb` and `yt-api` to be installed and available on PATH.

### Caching and non-destructive defaults

- The script stores the last successful device (ip and port) in `.last_device.json` in the skill folder.
- It will not attempt port scanning; this keeps behavior predictable and avoids conflicts with Google's ADB port rotation.

### Troubleshooting

- If adb connect fails, run `adb connect IP:PORT` manually from your host to verify the current port.
- If adb connect is refused and you're running interactively, the script will prompt you for a new port and update `.last_device.json` on success.

## Implementation notes

- The skill CLI code lives in `google_tv_skill.py` in this folder. It uses subprocess calls to `adb` and to `yt-api` when needed.
- For Tubi URL discovery, the assistant can use web_search to find canonical Tubi pages and pass the https URL to the skill.

