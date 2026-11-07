# Sivnox frontend — Quickshell/QML contract

**Status:** active. Derived from `Build_Sheet-Filled.md` (`L4`, `L5`, `L11`,
`L12`, `L19`, `R1`–`R52`).
**Process:** `qs -c sivnox` under `sivnox-shell.service`.
**Source base:** Caelestia project structure, edited.

This document is the frontend half of the contract. It defines the process
model, the directory shape, the surface lifecycle, the config and theme
adapters, the wallpaper transaction, and the per-monitor rule. It does not
redesign the visual language; that is `Constitution.md` and the Caelestia QML.

---

## 1. Process model

- One Quickshell process: `qs -c sivnox`.
- Started by `sivnox-shell.service` (`systemd --user`), not by `exec-once` in
  Hyprland.
- Renders one scene, one per-monitor slice per output.
- Holds all ephemeral view state and the wallpaper transaction.
- Reads config through `Config.qml`; reads tokens through `Theme.qml`.
- Never talks to a Rust control daemon. There isn't one.
- Talks to `sivnoxd` only for clipboard picker operations, over a local socket.
- Calls one-shot `sivnox` commands for colour extraction, theme fan-out,
  thumbnails, first frames, config parsing, and frontend actions.

```
sivnox-shell.service
└── qs -c sivnox
    ├── shell.qml                  root, per-monitor scope, surface bus
    ├── Config.qml                 TOML → normalized JSON
    ├── Theme.qml                  frozen token adapter
    ├── modules/
    │   ├── bar/
    │   ├── launcher/
    │   ├── notifications/
    │   ├── osd/
    │   ├── session/
    │   ├── lock/
    │   ├── wallpaper/
    │   ├── clipboard/
    │   └── screenshot/
    ├── services/                  Quickshell-native feeds and IPC helpers
    ├── components/                shared primitives
    └── utils/                     shared JavaScript
```

The exact Caelestia file names are preserved where possible. W0 verifies the
tree (`P6`) before the copy.

---

## 2. Surface lifecycle

`L5` is absolute: **every closed surface is behind `Loader { active: … }`**.

### 2.1 Persistent surfaces

| Surface | Loader | Notes |
|---|---|---|
| Bar | active after shell start | One per monitor. Always visible unless fullscreen hides it. |
| Notification server | active at start | The server owns the D-Bus name; the popup/history Loaders are separate. |

### 2.2 Transient surfaces

| Surface | Default | Opened by | Closed by |
|---|---|---|---|
| Launcher | `active: false` | `sivnox open launcher`, keybind | Escape, selection, focus loss |
| Notification popups | `active: false` | notification arrival | timeout, dismiss |
| Notification history | `active: false` | badge click, `sivnox open notifications` | Escape, click-out |
| OSD | `active: false` | `sivnox osd …`, volume/brightness/mic keys | timer |
| Session menu | `active: false` | power button, `sivnox open session` | selection, Escape |
| Lock screen | `active: false` | `sivnox lock`, `hypridle` | successful PAM auth |
| Wallpaper picker | `active: false` | `sivnox open wallpaper` | commit, cancel, Escape |
| Clipboard picker | `active: false` | `sivnox open clipboard` | selection, Escape |
| Screenshot overlay | `active: false` | `sivnox screenshot` | capture, Escape |

A surface that is closed but still instantiated is a bug. Do not "hide" a
surface by setting opacity or `visible: false` while keeping the object alive;
unload it.

### 2.3 Surface bus

One in-process surface bus maps a surface id to its Loader and its monitor. It is
the only place that opens a surface.

```
keybind / CLI ──> sivnox open <surface> ──> Quickshell IPC ──> surfaceBus.open(id, monitor)
```

Rules:

- The bus resolves the active monitor unless the request names one.
- Opening a surface closes the previous transient surface on the same monitor,
  unless the surface is explicitly stackable (notifications, OSD).
- The bus never holds authoritative state beyond "which surface is open".
- The bus is frontend view state. It is not written to disk.

The exact Quickshell IPC handler names are settled in W0. The stable external
contract is `sivnox open <surface>`; QML internals may change.

---

## 3. `Config.qml`

### 3.1 Responsibilities

- Watch `~/.config/sivnox/config.toml`.
- Call `sivnox config dump` on start and on change.
- Hold the normalized effective config as read-only properties.
- Never write `config.toml`.
- On parse failure, surface the error and keep the last good config; the shell
  renames the invalid file to `.broken` via the one-shot command and logs it.

### 3.2 Why a Rust parser

TOML is the user format (`L3`). A second TOML parser in JavaScript would drift
from the validator. `sivnox config dump` uses the Rust `toml` crate, merges the
sparse user file over compiled defaults, and returns JSON. `Config.qml` only
consumes JSON.

```
config.toml ──watch──> sivnox config dump ──JSON──> Config.qml ──bindings──> surfaces
```

### 3.3 Reload behaviour

- File watcher debounces rapid writes.
- A valid change applies live; no shell restart.
- An invalid change is reported once; defaults or last-good values remain.
- `sivnox config check` is the same validator without the frontend.

The full schema is in `Config.md`.

---

## 4. `Theme.qml`

### 4.1 Responsibilities

- Call `sivnox colours <active path> --json` on start and after a commit, and
  consume the returned token JSON.
- Expose the frozen keys to every surface.
- Resolve `theme.mode` / `theme.manual_mode` via the returned tokens, not with
  QML colour maths.
- Never read the cache path directly and never hardcode a colour in a surface.

### 4.2 Token keys

```
siv_ink, siv_bone, siv_grey, siv_signal,
siv_ink_dim, siv_ink_soft, siv_bone_dim,
siv_grey_deep, siv_signal_hover
```

The meaning and derivation are in `Theme.md`.

### 4.3 Flow

```
wallpaper commit
  └── sivnox colours <path>  → cache + tokens
      └── Theme.qml reloads tokens
          └── surfaces rebind (target < 200 ms, T6)

wallpaper commit
  └── sivnox theme apply <path>  → app templates + post-hook
```

Preview does **not** fan out to apps. It may apply tokens to the shell if the
cached tokens already exist; a cache miss does not block the preview.

### 4.4 Mode policy

- `auto`: light wallpaper → warm/bone mode; dark wallpaper → dark mode.
- `warm`: force the warm/bone token set.
- `dark`: force the dark token set.
- `manual_mode` overrides `auto` and persists across wallpaper changes.
- `warm_shift` shifts extracted hues toward the ink/bone band.

The extractor owns the policy; `Theme.qml` only applies the result.

---

## 5. Wallpaper controller

The wallpaper is a frontend transaction (`L12`, `R25`, `R26`).

### 5.1 State

```
persisted : path       // read from ~/.config/sivnox/wallpaper.json
preview   : path|null  // in-memory only
active    = preview ?? persisted
```

### 5.2 Rendering

- A Quickshell layer-shell background surface per monitor.
- The surface draws `active` at the monitor's logical resolution.
- No `swww`, `hyprpaper`, or `mpvpaper`.
- Video playback is disabled in v0.1; the picker shows a first frame only.

### 5.3 Picker

- Lists wallpapers from `wallpaper.directory`.
- Filters by `wallpaper.monitor_min_resolution`.
- Shows 128×128 thumbnails from `sivnox thumb`.
- Shows a first frame for a focused video from `sivnox frame`.
- Selecting an item sets `preview`; the background updates immediately.
- **Commit:** write `wallpaper.json` atomically, set `persisted`, clear
  `preview`, then call `sivnox colours` and `sivnox theme apply`.
- **Cancel:** clear `preview`; the background returns to `persisted`; cached
  persisted tokens are re-applied.
- Escape or closing the picker without commit cancels.

### 5.4 Persistence

`~/.config/sivnox/wallpaper.json`:

```json
{ "path": "/home/user/Pictures/Wallpapers/example.jpg", "is_video": false }
```

Written by Quickshell only, temp + rename + `fsync`. On start, read it; if
missing or unreadable, use the first valid wallpaper in the configured
directory or a neutral default.

---

## 6. Bar

The bar is the Caelestia bar, edited (`L6`). It is one Loader per monitor with
the modules from `config.toml` `[bar]`.

### 6.1 Modules

| Module | Source | Notes |
|---|---|---|
| Workspaces | Hyprland | Quickshell feed |
| Active window | Hyprland | Quickshell feed |
| Clock/date | local | No IPC |
| Tray | Quickshell | |
| Battery | UPower | |
| Network | NetworkManager | |
| Bluetooth | BlueZ | |
| Volume | PipeWire | |
| Brightness | `brightnessctl` | read on demand |
| Notification badge | `NotificationServer` | count of unread |
| Keyboard layout | Hyprland | |
| Power button | session menu | opens `session` surface |

### 6.2 Position and height

From `[bar]` in `config.toml`:

```toml
[bar]
position = "top"   # top | bottom
height = 32
```

A Hyprland layer rule reserves space for the bar (`T24`). The bar must not
overlap tiled windows.

### 6.3 Fullscreen

The bar hides on fullscreen windows. It returns when fullscreen ends. This is
view state; no config write is needed.

### 6.4 Frame (`P1`)

If the rounded frame is part of the Caelestia bar file, keep it. If it is a
separate surface, defer it (`O17`). The W0 check decides; do not build a second
frame surface.

---

## 7. Notifications

`L11` locks Quickshell's `NotificationServer` as the owner of
`org.freedesktop.Notifications`.

- One server instance in the shell.
- Popups, history, and DND are Loader-gated surfaces.
- The bar badge binds to the server's unread count.
- Actions are wired if the Quickshell version provides them (`O21`).
- No Rust notification server, no duplicate D-Bus owner.

---

## 8. Lock and session

### 8.1 Lock screen

- Caelestia lock screen, edited (`R22`, `O23`).
- Uses `WlSessionLock` and PAM.
- Reads `Theme.qml`; the lock surface must remain legible with the current
  token set.
- Opened by `sivnox lock` or `hypridle` on idle/lid.
- A failed auth returns to the lock screen; it never crashes the shell.

### 8.2 Session menu

- Caelestia session menu, edited (`R24`).
- Actions: lock, suspend, reboot, shutdown. Logout if the session manager
  supports it.
- Actions call explicit commands; no polling and no per-frame process spawn.
- `hypridle` owns idle and lid policy (`R23`, `T12`).

---

## 9. Clipboard picker

- QML surface from Ryoku, edited (`R36`).
- Opens with `sivnox open clipboard`.
- Connects to `$XDG_RUNTIME_DIR/sivnox/clipboard.sock`.
- Requests history, pins, and selection; renders text entries.
- Selection asks `sivnoxd` to copy the entry with `wl-copy`.
- The daemon starts on first picker connection via `sivnoxd.socket`.
- The picker never reads the SQLite database directly.

The socket contract is in `Clipboard.md` and `Interfaces.md`.

---

## 10. Screenshot overlay

- Ryoku overlay, edited (`R37`–`R39`).
- Region select with live dimensions.
- Capture via `grim`; save/copy via `grim` + `wl-copy`.
- One-shot commands only; no screenshot daemon.
- Escape cancels; the overlay unloads.

---

## 11. Per-monitor model

`L19` requires the shape now, even though v0.1 tests one monitor.

### 11.1 Rules

- Every surface request carries a `monitor` field.
- The surface bus resolves the active monitor when the field is absent.
- The bar and wallpaper background are created per monitor.
- Notifications and OSD target the focused monitor unless specified.
- No singleton may hardcode `monitor 0`.

### 11.2 v0.1 scope

- Render one monitor correctly.
- Keep the model generic enough for a second monitor without a rewrite.
- Add a second-monitor test post-v0.1 (`O20`).

---

## 12. Frontend action wrappers

These commands are the stable external interface. The QML internals may change;
the command names do not.

| Command | Effect |
|---|---|
| `sivnox open <surface>` | Open a surface on the focused monitor. |
| `sivnox osd <kind> [value]` | Show an OSD. Kinds: volume, brightness, mic, layout. |
| `sivnox lock` | Lock the session. |
| `sivnox session <action>` | Run a session action: lock, suspend, reboot, shutdown, logout. |
| `sivnox screenshot [mode]` | Start the screenshot overlay. Modes: region, window, output. |

Each is a one-shot process that delegates to Quickshell IPC. They hold no state.

---

## 13. Performance discipline

- **Loaders, not visibility.** Unload closed surfaces.
- **No eager singletons.** A service that is not visible at login is created on
  first use.
- **Thumbnails in grids.** Never put a full-resolution image in a picker grid.
- **One wallpaper surface per monitor.** Do not keep old previews decoded.
- **No timer that calls a command.** Subscribe to Quickshell state; do not poll.
- **No per-frame allocation in hot paths.**

Budgets: `T1` login <5 s, `T2` bar <60 MB, `T3` bar + launcher <90 MB, `T4` idle
CPU <1%, `T5` launcher <100 ms, `T6` colour apply <200 ms.

---

## 14. Testing

- `qmllint` clean.
- Every surface opens and closes without leaking an object.
- Every surface is behind a Loader.
- `Config.qml` applies a valid change live and survives an invalid one.
- `Theme.qml` applies tokens from `sivnox colours` on start and on commit.
- Wallpaper preview/cancel/commit behaves as `T9`/`T10`.
- Lock, OSD, notification, launcher, clipboard, and screenshot are exercised by
  `T12`–`T17`.
- The bar still renders last-known state while the daemon is down.

The full procedure is in `Testing.md`.

---

## 15. Open item

**P1 — Frame location.** Check the Caelestia source at W0. If the frame is
inside the bar file, keep it. If it is a separate surface, defer it as `O17`.
Record the answer at the top of this document when closed.
