## 1.0.0-dev.1 (2026-10-09)

### 🐛 Bug Fixes

* add patches-list markers to README and refresh target note ([0da55d1](https://github.com/aioue/morphe-spotify-capture-patches/commit/0da55d1adbcd548e43ef30ad4bc3cd5c4d75bb3d))

### ✨ New Features

* Spotify capture patch bundle for Morphe (RootlessJamesDSP) ([6412766](https://github.com/aioue/morphe-spotify-capture-patches/commit/6412766f9955799d7e1063725d087dacb429f2ef)), closes [#2095](https://github.com/aioue/morphe-spotify-capture-patches/issues/2095)

# Changelog

All notable changes to this patch bundle are documented here.

## Unreleased

### Fixed

- Patching no longer fails when Spotify already passes `ALLOW_CAPTURE_BY_ALL` to `setAllowedCapturePolicy` (e.g. 9.1.88.2204 `AudioDriver.constructAudioAttributes`).

### Added

- Morphe patch **Allow internal audio capture (RootlessJamesDSP)** for `com.spotify.music`:
  - `android:allowAudioPlaybackCapture="true"` on `<application>`
  - Rewrites `setAllowedCapturePolicy` policy constants to `ALLOW_CAPTURE_BY_ALL` in `com.spotify.*` methods

### Tested on device (2026-10-09)

- Spotify 9.1.88.2204 (versionCode 146816068), stock APK from `/sdcard/Download/`
- Morphe Manager 1.34.0, Patcher 1.15.1, bundle 1.0.1 on Pixel 8a (Android 17, API 37)
- Patcher log: `Allow internal audio capture (RootlessJamesDSP) succeeded` — no exception, no call-site count
- Patched APK installed after uninstalling stock (signature change); app installs
- Patched APK crashed at cold start: Patcher's `STRIP_FAST` dex mode left 14 of 30 dexes with `class_defs_size=0` but a nonzero `class_defs_off` (header and map_list), which ART rejects — `ClassNotFoundException: com.spotify.music.SpotifyApplication` on every launch. Confirmed via `dexdump` class-def check and `Failure to verify dex file ... Offset(1395604) should be zero when size is zero for class-defs` in logcat.
- Post-fix (not the bundle): zeroed the stale offsets in the 14 empty dexes, recomputed dex SHA-1/adler32, re-signed with a separate keystore, reinstalled — two consecutive cold starts clean, `LoginActivity` resumes, empty crash buffer. Upstream Morphe Patcher bug report still to file.

### Listen result (stock and patched)

- RootlessJamesDSP 1.6.14 captures Spotify's playback session but the output is **silence** (EQ on/off, post-gain changes make no difference; Spotify's stream is muted by RLJDSP while captured stream is empty)
- Root cause: stock Spotify playback runs on native **AAudio** via `liborbit-jni-spotify.so`, which resolves `AAudioStreamBuilder_setAllowedCapturePolicy` at runtime (`dlopen`/`dlsym`). The Java-side `constructAudioAttributes` constant this bundle rewrites is not the stream that carries audio, so bytecode patching cannot restore capture.
- Revoking Spotify's `BLUETOOTH_CONNECT`/`BLUETOOTH_SCAN` (RLJDSP issue #94 workaround) did not change the result.

### DSP-path A/B verification (2026-10-09, fixed APK installed)

- Positive control: Morphe-patched YouTube (Java `AudioTrack` path) plays **through** RLJDSP — post-gain changes are audible, so the DSP engine, effect insert, and output path all work on this device.
- Spotify A/B reproduced twice: RLJDSP toggle drops the A2DP HAL output from ~-12 dB to -120 dB (digital silence) within one second (`dumpsys media.audio_flinger` signal power history); Spotify's own track stays active with frames ready, and RLJDSP's DP effect on session 513 stays enabled — the effect mutes the original and the captured stream is empty, matching the AAudio root cause above.
- Spatializer ruled out: spatializer `mLevel: NONE`, zero active spatialized tracks, player `isSpatialized=false`; the SPATIALIZER-type output thread is just the normal A2DP mixer path on this device.
- RLJDSP log signature during the silent state: `BaseSessionDatabase: Session 0 skipped (uid=10341; package=com.spotify.music)` — its session watcher skips Spotify's player entirely.

### Known limits

- Playback that uses **AAudio only** (no Java `AudioAttributes` path) is not fixable with this patch. Current Spotify builds use this path (`liborbit-jni-spotify.so`), so the bundle is a control verification only, not a working capture fix.
- Patched Spotify is re-signed; uninstall stock Spotify before installing.
- Anti-tamper or integrity checks on newer Spotify builds may prevent startup without separate patches (out of scope).
