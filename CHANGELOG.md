# Changelog

All notable changes to this project will be documented in this file.

The format is based on Keep a Changelog (https://keepachangelog.com/en/1.1.0/)
and this project adheres to Semantic Versioning (https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-02-08

### Added

- Added `pair` command to run `adb pair` with explicit `--device` and `--pair-port` arguments.
- Added optional `--code` support with interactive prompt fallback for pairing code entry.
- Added unit tests for pairing helpers and `pair` command flow.

### Changed

- Documented that pairing uses a dedicated pairing port and does not auto-connect or update cache.
- `play`, `pause`, and `resume` now offer interactive pairing when ADB connect fails due to authentication/unpaired state.
- Adjusted pairing prompt timing to run after normal reconnect fallbacks, avoiding pair prompts for generic connectivity/IP/port drift issues.

## [1.0.3] - 2026-02-08

### Documentation

- Added an explicit execution policy to `SKILL.md` requiring command execution for pause/resume/status/play intents.
- Documented that responses must not claim success unless command exit code is 0.
- Documented failure handling requirements to return real command errors and next corrective steps.

## [1.0.2] - 2026-02-08

### Documentation

- Tuned `SKILL.md` trigger metadata for reliability by requiring only `adb` and `uv` at selection time.
- Added explicit Chromecast/Google TV intent-to-command mappings in `SKILL.md` to improve skill routing.
- Clarified conditional runtime dependencies: `yt-api` is only needed for YouTube query resolution and `scrcpy` only for global-search fallback.

## [1.0.1] - 2026-02-08

### Documentation

- Updated runtime requirements in `README.md` to require `adb`, `scrcpy`, `yt-api`, and `uv`.
- Updated `SKILL.md` metadata and setup/dependency notes to include `yt-api` installation and PATH requirements.
- Clarified fallback behavior notes in `SKILL.md`.

## [1.0.0] - 2026-02-07

### Fixed

- Improved ADB command execution and error handling in global search.
- Handled subprocess timeout in the global search helper.

### Documentation

- Updated `SKILL.md` description.
- Added `scrcpy` requirement and install helper metadata.

## [0.1.0] - 2026-02-06

- Initial release.
