# Changelog — Arrowroot (formerly Virtual Mouse)

All notable changes to this project are documented in this file.

## [v1.3.2] - 2026-11-18
### Fixed
- **Hold-and-drag cursor continuity**: Releasing the held anchor finger now drops `BTN_LEFT` and lets the remaining moving finger continue controlling the cursor seamlessly without requiring you to lift and re-touch.

## [v1.3.1] - 2026-11-18
### Fixed
- **2-finger scroll vs. hold-and-drag conflict**: Scrolling now requires both fingers to move in the same direction with comparable distance, and a finger that was already resting (>=150 ms) before the second lands is treated as a hold anchor, so its jitter no longer locks scroll mode. Hold-and-drag engages immediately in that case.
- **Dock buttons no longer break trackpad gestures**: Fingers resting on L/R/Mode/Hide are excluded from gesture finger counts — holding L while moving the cursor works, no false right-clicks, cursor resumes after the hold-and-drag anchor lifts.
- **Drag dock grip in Trackpad mode**: Grip-zone drags are now forwarded from the daemon to the app (`CB_DOCK_MOVE` / `CB_DOCK_DROP`) and the dock position is saved.
- **Mode / Hide buttons fire on release** instead of on touch-down, and only once all fingers lift.
- **Stuck buttons**: `BTN_LEFT`/`BTN_RIGHT` and hold/scroll state are released on every mode change.
- **RTL layouts**: Dock forced to LTR so daemon hit zones always match the buttons.
- **Daemon robustness**: Handles `POLLERR`/`POLLHUP` (no more CPU spin at `SCHED_FIFO`), resyncs after `SYN_DROPPED`, ignores out-of-range MT slots safely, respects `nohup`'s ignored `SIGHUP`.
- **Lifecycle**: start/stop are serialized (async `killall` can no longer kill a fresh daemon); concurrent `startAsync` callers now get a result; callback socket is closed and packet length reset; `MainActivity` no longer leaks via `onStateChanged`; delayed grab re-checks state; `addView` failures handled.
- **UI**: Dock clamp accounts for dock size; tablet default sensitivity consistent (1.6); status bar icons readable in light theme; Android 13+ notification permission requested.

### Security
- Daemon now **fails closed**: refuses to start without `ARROWROOT_TOKEN` and rejects packets with a zero token; removed the `--token` argv option (visible in `ps`).

### CI
- Releases are created only on tag pushes (pushes to `main` no longer overwrite the latest release); APK name and release notes are derived from `versionName` and this changelog.

## [v1.3.0] - 2026-11-04
### Added
- **Rebranded to Arrowroot**: New identity (*Arrow* cursor + *Root* kernel access), custom adaptive launcher icons (`mipmap-mdpi` through `mipmap-xxxhdpi` and `mipmap-anydpi-v26`), and monochrome vector icons for Quick Settings and notifications.
- **Session-Authenticated Loopback IPC**: Per-session 32-bit cryptographic authentication token (`SecureRandom`) verified on every UDP packet between the JNI client bridge and the root daemon, with `pthread_mutex_t` synchronization on the native client socket.
- **Upgraded Material 3 Dashboard & Settings UI**:
  - Live active mode badge (`IDLE`, `TRACKPAD MODE`, `TOUCH MODE`, `TOUCH • KEYBOARD`) in the hero card.
  - Quick-action buttons on the Home dashboard to **Swap Mode** and **Center Dock** in one tap.
  - 3-column **System Diagnostics** card checking **Superuser**, **Kernel `/dev/uinput`**, and **Overlay** readiness.
  - Structured multi-touch gesture reference cards with visual badges (`1F`, `HOLD`, `2F`, `3F`, `IME`).
  - **Start on Boot** preference toggle wired to `BootReceiver` and **Reset All Settings to Default** action.
- **Persistent & Clamped Floating Dock**: Saves custom dragged dock coordinates across sessions, clamps dock position to screen bounds on orientation changes, and supports **double-tapping the drag grip** to snap back to bottom-center.

### Fixed
- **Eliminated `chmod 666 /dev/uinput` Vulnerability**: Removed world-writable permission changes on `/dev/uinput` and enforced `0700` (`root:root`) permissions on `/data/local/tmp/vmouse_daemon`.
- **Async-Signal-Safe Daemon Teardown**: Changed `signal_handler` in `vmouse_daemon.c` to only flip a `volatile sig_atomic_t` flag and updated `RootMouseController.stop()` to send `SIGTERM` before fallback `SIGKILL`, ensuring `EVIOCGRAB` and `UI_DEV_DESTROY` always execute cleanly.
- **Activated 1px WindowInsets IME Sensor & Fixed Window Leak**: Invoked `initImeSensorWindow()` in `OverlayService.onCreate()` for instant soft-keyboard detection and removed `imeSensorView` in `onDestroy()`.
- **Fixed Orphan Root Shell in `startRootImeMonitor()`**: Tied the root IME monitor loop to `pidof vmouse_daemon` so it exits immediately when the daemon terminates.
- **Aligned Dock Hit-Testing Proportions**: Matched `classify_dock_zone()` thresholds (`10%`, `33%`, `62%`, `84%`) in `vmouse_daemon.c` to the exact button widths in `layout_overlay_controls.xml` (both phone and `sw600dp` tablet layouts).
- **Multi-Finger Tap Cursor Stability**: Prevented single-finger cursor movement during multi-finger taps (`gesture_max_fingers > 1`) when fingers lift asynchronously, eliminating cursor drift during 2-finger right-click and 3-finger mode-swap taps.
- **Direct Touchscreen Detection (`INPUT_PROP_DIRECT`)**: Prioritized input devices advertising `INPUT_PROP_DIRECT` in `open_touchscreen_device()`.
- **Removed Duplicate Layout Files**: Removed redundant copies of `activity_main.xml` in `layout-land` and `layout-sw600dp`.

## [v1.2.0] - 2026-11-01
### Added
- **Quick Settings (QS) Toggle Tile**: Added a single-tap Quick Settings tile (`VirtualMouseTileService`) in the notification shade to start and stop the pointer service from anywhere.
- **Material You (Material 3) + Monet Dynamic Theme**: UI overhaul with Android 12+ wallpaper-based Dynamic Colors (`DynamicColors`).
- **2-Tab Navigation (`Home` & `Settings`)**: Split the main screen into a Home dashboard and a dedicated Settings tab.
- **1-Finger Hold + 2nd-Finger Drag Gesture**: Hold 1st finger on the screen to lock Left Mouse Button (`BTN_LEFT=1`) down, and move a 2nd finger to drag the cursor across the screen (with multi-stroke support).
- **Bi-Directional 3-Finger Tap Mode Swap**: Tap with 3 fingers anywhere on the screen to switch between **Trackpad Mode** and **Normal Touch Mode** in both directions.
- **Auto-Touch Mode on Soft Keyboard (IME)**: Automatically switches to **Touch Mode** when the soft keyboard appears and restores **Trackpad Mode** when it closes.
- **Hide Floating Mouse Buttons (`✕`)**: Added an on-dock `✕` button, a Settings switch, and a notification action to hide or restore the floating dock.

### Fixed
- **Multi-Touch Slot Overwrite Bug (`ABS_X`/`ABS_Y`)**: Removed legacy single-touch `ABS_X`/`ABS_Y` fallbacks in `handle_touch_events()`.
- **Display Context Crash on Android 11+**: Replaced `context.display` in `OverlayService` with `DisplayManager.getDisplay(Display.DEFAULT_DISPLAY)`.

## [v1.1.0] - 2026-10-31
### Added
- **Kernel `EVIOCGRAB` Touchscreen Capture**: Eliminated full-screen overlay window so `/dev/uinput` hardware clicks reach apps directly.
- **Zero-Latency C JNI Loopback UDP Bridge (`127.0.0.1:29876`)**: Sub-2-microsecond IPC between the app and root daemon.

## [v1.0.0] - 2026-10-30
### Added
- Initial release with `/dev/uinput` hardware mouse daemon (`EV_REL` + `BTN_LEFT`/`BTN_RIGHT`/`BTN_MIDDLE`) and floating overlay controls.
