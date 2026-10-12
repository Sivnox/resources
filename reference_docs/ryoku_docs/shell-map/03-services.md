# 03: Services (the singleton catalogue)

`ryoku/shell/quickshell/shell/services/`, manifest `qmldir`
(`module shell.services`). ~50 singletons: every surface reads these, no
surface owns its own copy. `ServiceLoader` constructs the eager set at
load (`ShellState`, `ScreenTime`, `Keypresses`, `KeyboardLayout`); the
rest construct on first binding.

## Socket pattern (every daemon-fed singleton)

Two `Quickshell.Io.Socket`s to `ryoku-shell.sock`:

- `sub`: writes `subscribe <topic>`, parses newline-delimited frames.
- `ctl`: queues `call <topic>.<method> <json>` (see `01-daemon.md`).

Examples: `Clipboard.qml` (`subscribe clipboard`, calls
`copy/delete/clear/star`), `Tray.qml` (`subscribe tray`, calls
`activate/contextMenu/scroll/...`), `Wm.qml` (`subscribe wm`),
`Weather`, `Holidays`, `Network`, `OsdFeed` (`subscribe osd`),
`Nightlight` (`subscribe nightlight`, calls `toggle/set`).

## Catalogue by job

- Bus/state: `ShellState` (per-monitor surface bus, see `02-...`),
  `Events`, `Flags` (`keepAwake`, `gameMode`, `dnd`), `Session` /
  `SessionActions` (logout/reboot/shutdown runners), `Toggles`.
- Look: `Theme`, `Scheme` (watches `colors.json`), `Motion` (durations),
  `Icons`, `Needle`.
- Compositor/window: `Wm` (in `Ryoku.Ui`, see `02-...`), `KeyboardLayout`,
  `Devices` (restore/probe displays), `ScreenTime` (per-app accrual off
  `Wm.focusedWindow`), `AppLaunch`, `Dock`, `DockPreview`,
  `MusicPreview`.
- Audio/media: `Audio` (PipeWire volumes), `AudioBars`, `Equalizer`,
  `Media`/`Music` (players), `VoiceBars`, `MicMute` state inside OSD
  path, `Camera`, `Keypresses` (evdev overlay stream).
- Data: `Battery`, `Network`, `BluetoothDiscovery`/`BtLink`, `Weather`,
  `Holidays`, `Calendar` state, `Updates`, `Sysinfo`/`StatsFeed`,
  `PowerProfiles`, `Nightlight`, `OsdFeed`.
- Capture/productivity: `Clipboard`, `Capture` (shot prefs + `shoot` +
  `commit` to grim/Recorder), `Recorder` (video backends + region pick +
  countdown), `Stash` (feature sidebar tasks + auth step-aside),
  `BarProducts` (bar-style scene catalogue), `Keyring`/`Polkit`
  (auth islands driven by daemon topics).
- Config: `Config` (shell.json live view + daemon writes, see `02-...`).

## Fork guidance

- Keep the subscribe/call shape verbatim; it is the only QML-to-daemon
  contract. New data = new topic + calls in the daemon, new singleton
  mirroring the pattern, never a second socket convention.
- `Perf` (freeze/saver/gaming/lowPower) gates expensive work globally:
  spectrum analysis, idle waves, wallpaper fade. New animated surfaces
  must honor it.
- `Tokens.uiScaleFor` is the per-monitor scale every overlay uses.
  `Notifs` owns toast lifetime; `Clipboard` owns history (100 entries,
  hash dedup). Do not reimplement either per surface.

Next: `04`-`07` show which services each surface consumes.
