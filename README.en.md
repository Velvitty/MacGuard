[한국어](README.md)&emsp;&emsp;English

# MacGuard

A browser app that displays a PIN lock screen and produces alarms and logs in response to input, tab changes, network changes and charging-state changes. It also includes camera capture, location display and PWA configuration.

**MacGuard's lock exists inside a web page.** It does not replace macOS login locking or operating-system access control, and does not provide protection that survives browser termination, force-quitting or device sleep.

## Features

- Register a six-digit PIN on first use, change it and unlock the interface.
- Alarms for repeated incorrect PINs and disallowed keys, clicks or scrolling while locked.
- Configurable detection of loss of focus, resizing, online/offline changes and charging changes.
- Web Audio alarms and embedded audio playback.
- Camera-permission-based event photos and previews.
- Browser geolocation or IP-based location display.
- Event logs, counters and `change.log` download.
- Web App Manifest and Service Worker configuration for PWA use.

## Technology

| Technology | Purpose |
| --- | --- |
| HTML / CSS / vanilla JavaScript | Dashboard, PIN interface, state and events |
| Web Audio API | Alarms and feedback |
| MediaDevices / Canvas 2D | Camera video and JPEG capture |
| Geolocation API | Location with browser permission |
| Fetch / ipapi.co | IP location fallback when location permission cannot be used |
| Battery Status API | Charging-state detection in supporting browsers |
| Page Visibility / DOM events | Tab visibility, input and resize detection |
| localStorage | PIN, settings, counters and photos |
| Service Worker / Cache API / Manifest | Selected cached assets and installation metadata |

No application server, API key, package installation or build is required. Web fonts load from Google Fonts.

## Run

Use **HTTPS or localhost** for the camera, geolocation and Service Worker. Download and extract **Code → Download ZIP**, then start a local server in the extracted directory. With Git and Python 3:

```sh
git clone https://github.com/Velvitty/MacGuard.git
cd MacGuard
python3 -m http.server 8000 --bind 127.0.0.1
```

- App: <http://localhost:8000/index.html>
- Guide: <http://localhost:8000/howto.html>

Supporting browsers may offer installation into a standalone window. Installation and individual API availability depend on the browser.

## Workflow

1. Enter the same six-digit numeric PIN twice on first use.
2. Continue from the introduction to the app or settings.
3. Enable the desired detectors. Grant camera and location permissions if using those features.
4. Check device volume and audio playback, then lock. The lock screen appears after the countdown.
5. Enter the registered PIN to unlock. When an alarm is displayed, enter the PIN in its dismissal interface as well.
6. Review events and photos in the dashboard and download logs you want to retain.

## Detection scope

| Detector | Behavior and conditions |
| --- | --- |
| Incorrect PIN | Alarm after at least three failed unlock attempts |
| Keyboard / mouse | Disallowed keys, clicks and scrolling while locked |
| Tab / window state | Depends on settings and browser event delivery |
| Network | Browser `online` / `offline` events, not monitoring of a particular Wi-Fi device |
| Charging | Requires `navigator.getBattery()` |
| Closing | Requests a `beforeunload` confirmation; cannot force the browser to stay open |
| Camera | Requires permission and an available camera |
| Location | Browser geolocation, with IP fallback when disabled or unsuccessful |

## Repository and components

| Path | Role |
| --- | --- |
| `index.html` | UI, alarms, PIN, camera, location and logging |
| `howto.html` | Introduction and usage guide |
| `manifest.json` | PWA name, entry point and icons |
| `sw.js` | Static asset caching |
| `icons/icon-192x192.png`, `icons/icon-512x512.png` | Installation icons |
| `README.md`, `README.en.md` | Korean and English documentation |

The main functions are `tryUnlock()` and `lockNow()` for locking and unlocking; `triggerSecurityAlert()` for alarms; `initCamera()` and `captureIntruder()` for camera initialization and capture; `fetchLocation()` and `ipLocation()` for location lookup; and `addLog()` and `saveChangeLog()` for log management.

| Component | Implementation |
| --- | --- |
| UI / state | `state.isLocked`, `state.introActive` and overlay CSS classes manage the introduction, dashboard, lock screen and alarm display |
| PIN | Input interfaces and comparison functions for registration, change, unlock and alarm dismissal |
| Detectors | DOM, battery and network events call `triggerSecurityAlert(reason)` |
| Alarm audio | Oscillator → WaveShaper → Gain, plus embedded MP3 |
| Camera | `<video>` stream → hidden `<canvas>` → JPEG Data URL |
| Location | One `getCurrentPosition()` request, then `ipLocation()` on failure |
| Logs | `addLog()`, on-screen list, in-memory `changeLog`, `saveChangeLog()` |
| PWA | Manifest metadata and static-file caching in `sw.js` |

UI, CSS, events, audio and storage are combined in `index.html`. There is no server authentication, account system, remote-control API or database.

## State transitions and PIN policy

| State | Entry / transition | Detection |
| --- | --- | --- |
| Introduction | `init()` sets `introActive = true`; registration opens if no PIN exists | Ordinary detectors return during the introduction |
| Dashboard | Open settings or successfully unlock; `isLocked = false` | Lock-only detection stops; settings, logs and photos are available |
| Preparing to lock | `lockNow()` animations and countdown | Keys are blocked; the alarm function returns while the countdown blocker is active |
| Locked | Countdown completes; `isLocked = true` and the lock screen appears | Alarms follow each detector's toggle and event conditions |
| Alarm visible | `triggerSecurityAlert()` opens an overlay and plays audio | Managed separately from the lock state; further events can update counters and content |
| Alarm dismissed | Correct PIN in `dismissAlert()` | Stops the overlay and sound; unlocking still requires `tryUnlock()` |

An immediate `lockNowFast()` function exists, but the buttons and startup flow call `lockNow()`. The displayed countdown digits do not represent the exact duration of all preparation animations.

- PIN fields strip non-digits and retain at most six digits. Registration requires two matching six-digit entries.
- Unlock checks on six digits or Enter. A nonempty incorrect entry counts as a failure.
- `failCount` tracks failures in the current run and resets to 0 on successful unlock. Three or more failures trigger `WRONG_PIN`.
- `tog_wrongpass` controls the short incorrect-PIN feedback sound; **it does not disable the three-failure alarm**.
- Changing a PIN requires the current PIN, a new six-digit PIN, a matching confirmation and a value different from the old PIN. Three failed current-PIN checks trigger an alarm and locking.
- An incorrect alarm-dismissal PIN creates feedback and a log but does not increment `failCount` or cumulative `mg_fails`. There is no server-side lockout or retry delay.

## Event conditions and defaults

Ordinary lock events require `isLocked = true` and `introActive = false`. Toggle changes immediately update the settings object and `mg_settings`.

| Setting / event | Default | Behavior |
| --- | --- | --- |
| `tog_wrongpass` | On | Short incorrect-PIN feedback |
| `tog_capture` | On | Permits capture at explicit capture call sites, including incorrect PINs and charging changes |
| `tog_power` | On | Battery `chargingchange`; alarm and optional photo |
| `tog_gps` | On | Tries geolocation; falls back to IP location when disabled, unsuccessful or unsupported |
| `tog_resize` | Off | Records the window dimensions and triggers an alarm on every `resize` event |
| `tog_blur` | On | Hidden tabs, focus loss, mouse movement and pointer leaving the window |
| `tog_close` | On | Browser leave confirmation through `beforeunload` |
| `tog_network` | On | Alarms on `online` and `offline` |
| Disallowed keys | No separate toggle | Allows digits, Backspace, Delete, Tab, left/right arrows, Home, End and Enter; alarms on other keys |
| Click / scroll | No separate toggle | Alarms on `mousedown` and `wheel`, except when the alarm overlay is already open |

Mouse movement threshold is `min(windowWidth, windowHeight) × 0.075`, measured as straight-line distance from the reference point. Movement and pointer-exit detectors share a three-second cooldown. The implemented distance is **7.5% of the shorter side**, despite a comment saying 15%.

`beforeunload` merely requests confirmation; it does not require a successful PIN check before the browser can close. `CLOSE` and `KIOSK` message names do not imply forced fullscreen or OS shutdown prevention. There is no `fullscreenchange` detector or Fullscreen API invocation.

## Stored data and retention

localStorage belongs to the page's **origin: protocol, host and port**. Different domains, HTTP/HTTPS and localhost ports do not share it.

| Key | Format / use |
| --- | --- |
| `mg_pin` | Plaintext six-digit string |
| `mg_fails` | Integer string; cumulative unlock / PIN-change failures |
| `mg_alerts` | Integer string; handled alarm calls |
| `mg_photos` | Integer string; cumulative captures, not necessarily retained-photo count |
| `mg_settings` | JSON object `{ tog_wrongpass, tog_capture, tog_power, tog_gps, tog_resize, tog_blur, tog_close, tog_network }`; boolean values for the eight `tog_*` settings above |
| `mg_logcfg` | JSON object controlling enabled log categories; all twelve default to `true` |
| `mg_capturedPhotos` | JSON array `[{ url, time, reason }]`: recent photo Data URLs, localized time strings and capture reasons |
| `mg_email` | Residual JSON setting read at initialization, default `{}`; not used to send mail |

Log category keys are `log_access`, `log_alert`, `log_photo`, `log_pin`, `log_power`, `log_network`, `log_resize`, `log_blur`, `log_close`, `log_location`, `log_sys` and `log_cfg`. Disabling a category suppresses both on-screen and downloadable records for it; it does not disable detection.

Camera capture uses the video's actual dimensions, falling back to 320×240 if unavailable. It mirrors the image, adds the reason and time at the bottom, and encodes a JPEG at quality `0.8`.

- Runtime photo list: at most 20 recent photos.
- Persistent list restored after refresh: at most 10 recent photos.
- Dashboard thumbnails: at most six; click to enlarge.
- Not every alarm takes a photo. Explicit capture paths calling `captureIntruder()` include incorrect unlock PINs, charging changes and three or more failed PIN-change checks.
- There is no quota-exceeded recovery for photo/JSON storage, so saving can fail when browser storage is full.

`addLog()` retains the newest 100 entries in the visible list but does not impose a count limit on the downloadable `changeLog` array. The UTF-8 `change.log` format is `[time] [level] [category] message`. File times come from UTC `toISOString()`, with the `Z` omitted; screen times use local device time.

Clearing the visible log removes DOM entries without emptying `changeLog`. Reloading loses in-memory logs. Reset functions differ: `resetApp()` deletes stored photos, but `confirmResetApp()` does not remove `mg_capturedPhotos`. Do not assume every reset action removes all photos.

## Permissions, audio and caching

| Feature | Requirements |
| --- | --- |
| Camera | Secure context and permission; `getUserMedia({ video: true })`, no microphone request |
| Location | Secure context, permission and Geolocation support; one-shot lookup |
| IP location | External connection to `ipapi.co`; may run when location permission is unavailable or GPS is disabled |
| Alarm audio | Web Audio support, user-gesture activation, audible device/tab volume and permission to play audio |
| Battery | Detector is registered only if `navigator.getBattery` exists |
| Installation / cache | Manifest and Service Worker support on HTTPS or localhost |

The first click, key press or touch attempts audio activation with a silent buffer. Alarms cycle through descending (1200→200Hz), air-raid (400↔1200Hz), rising (200→1400Hz) and alternating (600↔900Hz) sirens, alongside a looping embedded MP3. PIN dismissal stops timers and created audio nodes. The app cannot force device volume upward or unmute the device.

The Service Worker cache is `macguard-v3`, precaching only `./index.html` and `./manifest.json`:

1. Installation uses `cache.addAll()` and `skipWaiting()`.
2. Activation deletes caches with other names and calls `clients.claim()`. The deletion is not limited to a MacGuard prefix, so other caches on the same origin may be affected.
3. Requests use `caches.match()` first and the network second. Network responses are not dynamically added to the cache.

Offline availability of the guide, icons, fonts, IP location or a fallback for `/` is not guaranteed. Coordinate HTML deployment with cache-version and Service Worker updates because cached HTML is served before the newest network copy.

The manifest declares start URL `./index.html`, scope `.`, `standalone` display, `landscape` orientation and 192×192 / 512×512 PNG icons. An installed window still follows the browser environment’s detection, permission and lifecycle rules.

## Privacy and security boundaries

- The PIN is stored unencrypted in the current site’s `localStorage`. Do not reuse an OS or service password.
- Settings, alarm/failure/capture counters and recent photos are stored in the browser. The capture list retains at most 20 photos in memory and the 10 most recent photos in persistent storage.
- Event logs accumulate in memory during the session. Download logs you need to keep; deleting site data can remove the PIN, settings and photos.
- IP lookup requests `https://ipapi.co/json/`. Disabling the location toggle can still lead to this fallback request.
- The app does not upload photos or logs to a remote server. The email-notification function is empty and sends no email.

The lock is implemented with JavaScript and DOM in the same browser. It does not prevent access through developer tools or site-data editing. There is no PIN hashing, server verification or tamper-resistant logging. After refresh, the introduction's route to settings opens the dashboard without reauthenticating the saved PIN, so this is not persistent access control.

Turning off camera capture controls later capture/initialization calls but does not stop already-open media tracks; there is no `stop()` call for those tracks. Close the page or revoke browser permission to end camera access reliably. Continued detection or recording in the background, during sleep or after a tab closes, device motion sensing and remote lost-device tracking are not implemented.

## Troubleshooting

- If the camera fails, check HTTPS/localhost and site permissions.
- If charging detection fails, check Battery Status API support; it is not available in every macOS browser.
- Backgrounding or sleep may restrict alarms and detection.
- The Service Worker precaches only `index.html` and `manifest.json`. Complete offline operation including the guide, icons and external fonts is not guaranteed.
- If updates do not appear, inspect the site's cache and Service Worker. Clearing site data may also delete saved PINs, settings and photos.
- Introductory marketing text may describe a broader scope than the implementation. Use the conditions above when evaluating actual behavior.

## License

The app displays Copyright © 2026 Velvitty (Yeon). All rights reserved. There is no separate `LICENSE` file. Ask the repository owner about redistribution, commercial use or additional permissions.
