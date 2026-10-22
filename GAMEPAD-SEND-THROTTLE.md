# Gamepad send throttle (local patch)

This branch adds a client-side rate limit for gamepad state packets sent to the host.
It exists to work around periodic stutter/frame-pacing hitches caused by controllers
that poll much faster than the stream frame rate (observed with the built-in controller
of Windows handhelds, e.g. ROG Xbox Ally X).

## Why

* Moonlight batches axis events for the same pad, but has no send-rate limit
  (`moonlight-qt#1156`). With a joystick present, `SDL_WaitEventTimeout()` degrades to a
  1 ms poll on the thread that also presents video frames, so a 1000 Hz pad can produce
  ~1000 `LiSendMultiControllerEvent()` calls per second.
* Sunshine processes all input on a single worker thread and issues one synchronous
  ViGEm IOCTL per Xbox 360 packet (the DS4 path coalesces, the XB360 path does not),
  so a high packet rate turns into frame-pacing spikes on the host.
* Handheld Companion "fixes" the same symptom because its controller loop runs at a
  fixed 8 ms (125 Hz). This patch reproduces only that part, without HidHide, without a
  virtual pad / kernel driver, and without touching Armoury Crate SE.

## Behaviour

| Setting | Result |
|---|---|
| 62.5 Hz (16 ms) | at most every 16 ms (lowest selectable rate) |
| 125 Hz (8 ms) — default in this branch | axis/trigger state sent at most every 8 ms |
| 250 Hz (4 ms) | at most every 4 ms |
| 500 Hz (2 ms) | at most every 2 ms |
| 1000 Hz (1 ms) | at most every 1 ms |
| Unlimited (0) | original upstream behaviour, one send per evdev batch |

Guarantees:

* Button and trigger **edges** are still sent immediately, so click latency is unchanged.
* The **trailing state is always delivered**. If an axis change lands inside the window,
  a one-shot SDL timer pushes a user event and the main thread flushes the pending state,
  so the host can never be left with a stale stick position (which would look like a
  permanently drifting camera).
* The timer callback only calls `SDL_PushEvent()`; gamepad state is only touched from the
  event loop.
* Mouse, keyboard, touch, gyro/accel, rumble, adaptive triggers and LEDs are untouched.

## Files changed

| File | Change |
|---|---|
| `app/settings/streamingpreferences.h` | `gamepadSendIntervalMs` property, member, signal |
| `app/settings/streamingpreferences.cpp` | `gamepadsendintervalms` key, `qBound(0, …, 100)`, default 8 |
| `app/streaming/input/input.h` | `SDL_CODE_GAMEPAD_STATE_FLUSH`, three `GamepadState` fields, handler API |
| `app/streaming/input/input.cpp` | read the preference in the ctor, cancel timers in the dtor |
| `app/streaming/input/gamepad.cpp` | `queueGamepadState()` / `cancelPendingGamepadFlush()` / `flushPendingGamepadState()` / timer callback, axis path switched to the throttle, button edges reset the window, device removal cancels the timer |
| `app/streaming/session.cpp` | dispatch `SDL_CODE_GAMEPAD_STATE_FLUSH` to `flushPendingGamepadState()` |
| `app/gui/SettingsView.qml` | "Controller update rate" combo box in Gamepad Settings |

Upstream intent (if this is ever proposed as a PR): default to `0` so behaviour is
unchanged for everyone else; this branch keeps `8` as the local default.

## Build (Windows)

Verified recipe. Requirements: **Qt 6.11 is the officially supported version, but 6.10.3
builds and runs fine**; the toolchain below can be assembled **without administrator
rights**.

### 1. MSVC + Windows SDK without Visual Studio

`vs_buildtools.exe` requires elevation, so this build used
[msvc-wine's `vsdownload.py`](https://github.com/mstorsjo/msvc-wine), which pulls the
MSVC toolset and the Windows SDK straight from Microsoft's VS channel:

```powershell
python vsdownload.py --dest E:\MSVC --major 17 --accept-license `
    --architecture x64 --host-arch x64 --language en-US `
    --with-sdk yes --with-msvc yes --with-asan no --with-atl no `
    --with-msbuild no --with-devcmd no --with-dia no
```

Result used here: MSVC `14.44.35207`, Windows SDK `10.0.26100.0`.

### 2. Qt

```powershell
python -m pip install aqtinstall
python -m aqt install-qt windows desktop 6.10.3 win64_msvc2022_64 -O E:\Qt
```

Notes:

* `aqt` 3.3.0 cannot fetch Qt **6.11+** (the repository changed to per-arch folders such as
  `qt6_6113_msvc2022_64`). Use 6.10.3, or Qt's official installer.
* If a mirror stalls, download the archive by hand from
  `.../qt6_6103/qt6_6103/qt.qt6.6103.win64_msvc2022_64/<ver><module>-...7z`,
  verify it against the matching `.sha1` and unpack it into `.../6.10.3/msvc2022_64`.

### 3. Dependency libraries

`setup-deps.ps1` uses `curl.exe`, which fails on some machines with
`CRYPT_E_NO_REVOCATION_CHECK`. Use `Invoke-WebRequest` instead (or `curl --ssl-no-revoke`):

```powershell
Invoke-WebRequest 'https://github.com/moonlight-stream/moonlight-qt-deps/releases/download/v19/windows-x64.zip' -OutFile deps.zip
Expand-Archive deps.zip -DestinationPath libs\windows -Force
```

### 4. Build

**The source tree must not contain spaces** — `nmake` fails to apply qmake's inference
rules for paths like `...\VSCode Workstation\...`, which shows up as
`LNK1181: cannot open input file 'release\callbacks.obj'`.

```powershell
# INCLUDE / LIB / PATH must point at the MSVC and SDK folders from step 1
New-Item -ItemType Directory -Force E:\moonlight-build | Out-Null
cd E:\moonlight-build
qmake.exe E:\moonlight-qt\moonlight-qt.pro -config release -spec win32-msvc
nmake /NOLOGO release
```

Output: `E:\moonlight-build\app\release\Moonlight.exe`.

### 5. Portable package

```powershell
windeployqt --release --no-translations Moonlight.exe   # DLLs/plugins
# libs\windows\lib\x64 holds the runtime DLLs (SDL2, FFmpeg, opus, ...) - copy them too
Copy-Item E:\moonlight-qt\libs\windows\lib\x64\*.dll .
Copy-Item E:\moonlight-build\AntiHooking\release\AntiHooking.dll .
# QML modules + their plugins (windeployqt --qmldir fails on this Qt: qmlimportscanner error)
robocopy E:\Qt\6.10.3\msvc2022_64\qml\QtQuick qml\QtQuick /E
robocopy E:\Qt\6.10.3\msvc2022_64\qml\QtQml   qml\QtQml   /E
Copy-Item E:\Qt\6.10.3\msvc2022_64\bin\Qt6*.dll .   # skip the *d.dll debug builds
```

## Verified on this machine

| Check | Result |
|---|---|
| `qmake` + `nmake release` | exit 0, 0 compile errors, 1 benign `LNK4291` (Qt6EntryPoint) |
| Patch present in the binary | strings `gamepadsendintervalms` and `gamepadSendIntervalMs` found |
| App starts from the packaged folder | process stays alive, window created (`Moonlight`) |
| Runtime log | no errors; only the upstream `ToolTip.toolTip.contentWidth` warning from Qt 6.10 |
| Stutter fix | **not verifiable here** — needs the handheld, a stream, and a 1000 Hz controller |

Runtime log location: `%TEMP%\Moonlight-<epoch>.log`.

## Verify

1. Set `min_log_level = verbose` in `sunshine.conf` on the host and restart Sunshine.
2. Stream from the handheld, push the right stick continuously for 20 s, then count
   controller packets:

```powershell
$log = 'C:\Program Files\Sunshine\config\sunshine.log'
(Select-String -Path $log -Pattern '--begin controller packet--').Count
```

Record the value with "Unlimited" and with "125 Hz". Expect roughly an 8x reduction.
3. Flick and instantly release the stick: the host-side stick must return to centre
   (this is the trailing-state guarantee).
4. Compare button latency between "Unlimited" and "125 Hz"; it should be identical.
5. Regression: mouse, keyboard, touch, rumble, adaptive triggers, external gamepad.

## Rollback

* In-app: set **Controller update rate** to **Unlimited** (no rebuild needed).
* Or restore `Moonlight.exe.bak`.

## Known limitations / unverified

* The built-in controller's actual report rate on Windows has not been measured here.
  If it already reports at ~125 Hz, this patch will not change anything (step 2 above
  will show it immediately).
* Sub-8 ms opposite-direction flicks are coalesced (same as any 125 Hz pipeline). Use
  250/500 Hz if that feels sluggish.
* If the host is presenting a DS4 virtual pad, Sunshine already coalesces, so the gain
  is smaller.
* Built with Qt 6.10.3 rather than the officially supported 6.11; the only observed
  difference is the upstream `ToolTip.toolTip.contentWidth` warning at startup.
