# Sivnox — v0.1 scope

**Status:** active. Derived from `Build_Sheet-Filled.md`.
**License:** GPL-3.0.
**Platform:** Hyprland (0.55+, Lua configuration) on a laptop.
**Definition of done:** every test in `T1`–`T24` passes on the reference laptop.

---

## 1. What Sivnox is

Sivnox is a complete, opinionated Hyprland desktop shell that a person can use as
a daily driver after a single setup. It is not a distribution, an installer, a
settings GUI, or a general-purpose control plane.

The shell has three parts:

| Part | Runs as | Owns |
|---|---|---|
| **Quickshell frontend** | `qs -c sivnox` under `sivnox-shell.service` | Every visible surface, wallpaper preview/commit, notifications, and all read-only protocol feeds. |
| **`sivnoxd`** | `systemd --user` | Clipboard history, pins, eviction, and the clipboard SQLite database. |
| **`sivnox`** | one-shot CLI | Colour extraction, theme fan-out, thumbnails, first frames, config parsing, and frontend action wrappers. |

External tools own their own work: `grim`, `wl-copy`/`wl-paste`, `hypridle`,
`brightnessctl`, `hyprsunset`, `ffmpeg`, and `power-profiles-daemon`.

---

## 2. In scope — v0.1

### 2.1 Frontend surfaces

All of `R1`–`R52` are in scope for v0.1:

- **Bar:** workspaces, active window, clock/date, system tray, battery, network,
  Bluetooth, volume, brightness, notification badge, keyboard layout, power
  button.
- **Launcher:** app search and mode prefixes (calculator, web, actions).
- **Notifications:** popups, history, DND, actions where Quickshell provides
  them.
- **OSD:** volume, brightness, microphone, keyboard layout.
- **Lock and session:** `WlSessionLock` + PAM lock screen; session menu with
  lock, suspend, reboot, shutdown.
- **Wallpaper:** set/persist, picker with live preview, 128×128 thumbnails,
  video first-frame preview.
- **Theme:** colour extraction, token cache, apply tokens to the shell, fan-out
  to external apps.
- **Clipboard:** history, pins, size limit/eviction, picker UI.
- **Screenshot:** region select with live dimensions, save/copy.
- **Media:** now playing, controls, media keys.
- **System:** Wi-Fi connect, Bluetooth pair/connect, brightness adjust, power
  profiles, night light, transparency/blur.
- **Configuration:** `config.toml`, live reload, keybind integration, autostart.

### 2.2 Rust backend

| Component | Scope |
|---|---|
| `sivnoxd` | Clipboard database, watcher, pins, eviction, picker socket. |
| `sivnox colours` | 64×64 extraction, mode policy, frozen token set, cache. |
| `sivnox theme apply` | Rust port of the Caelestia fan-out engine, templates, post-hook. |
| `sivnox thumb` | 128×128 thumbnail cache. |
| `sivnox frame` | Video first-frame cache via `ffmpeg`. |
| `sivnox config` | TOML parse/validate, normalized JSON for `Config.qml`. |
| `sivnox open/osd/lock/session/screenshot` | Thin wrappers that ask the running frontend to act. |
| Hyprland glue | Path rewrite and managed-file sync; no runtime daemon. |

### 2.3 Hyprland configuration

- Caelestia's complete Lua-based tree in `~/.config/hypr/`, managed by Sivnox.
- User override layers in `~/.config/sivnox/hypr-vars.lua` and
  `~/.config/sivnox/hypr-user.lua`.
- Sivnox keybinds added to `hypr-user.lua`.
- Layer rule reserves space for the bar.
- `hypridle` wired for lid/idle behaviour.

### 2.4 Delivery

- `sivnox-shell.service` and `sivnoxd.service` as `systemd --user` units.
- GPL-3.0 licensing and notices for ported Caelestia/Ryoku code.
- No package repository, installer, or ISO in v0.1.

---

## 3. Out of scope — v0.1

### 3.1 Deferred surfaces

`O1`–`O24` from the Build Sheet, including:

- Overview, window switcher, dock, dashboard/control center.
- Desktop widgets, calendar, weather, system monitor.
- Visualizer, stage/parallax, special workspaces, window resizer.
- Emoji picker, screenshot annotation, screen recording.
- Settings GUI, update notifier, greeter work.
- Multi-monitor rendering beyond protocol/model readiness.

### 3.2 Architectural non-goals

- **No general control-plane daemon.** There is no `sivnoxd` IPC bus for
  wallpaper, scheme, services, or surfaces.
- **No wallpaper daemon.** Quickshell draws the wallpaper on a layer-shell
  background and owns preview/commit.
- **No `swww`, `hyprpaper`, or `mpvpaper`.** Wallpaper display is native
  Quickshell layer-shell.
- **No reimplementation of Quickshell feeds.** UPower, NetworkManager, BlueZ,
  PipeWire, MPRIS, tray, clock, and Hyprland workspace state stay in Quickshell.
- **No materialize/doctor framework.** Managed files are copied/rewritten by the
  installation path; config is sparse and validated on read.
- **No keyring prompt in v0.1.** `L18` defers `org.gnome.keyring.SystemPrompter`.
  When it returns, it extends `sivnoxd`; it does not add a second daemon.
- **No AI, telemetry, analytics, crash reporting, update checks, or network
  calls.**
- **No root, setuid, or systemd system units.**
- **No settings GUI, plugin system, store, ricer, or theme marketplace.**
- **No video wallpaper playback.** Only first-frame preview in v0.1.
- **No multi-compositor abstraction.** Hyprland only.

### 3.3 Ports we do not make

Do not port from Caelestia or Ryoku: bar, launcher, lock screen, session menu,
OSD, notifications, network/Bluetooth/audio menus, or media widgets. Those stay
QML and are copied/edited at the QML layer.

---

## 4. Ownership boundary

| State or action | Owner | Never |
|---|---|---|
| Hover, focus, scroll, animation, open surface | Quickshell | daemon |
| Wallpaper preview, commit, cancel | Quickshell | daemon, `swww` |
| Notification server (`org.freedesktop.Notifications`) | Quickshell | Rust |
| UPower, NetworkManager, BlueZ, PipeWire, MPRIS, tray | Quickshell / the service | Rust |
| Clock and workspace list | Quickshell / Hyprland | Rust |
| Clipboard history, pins, eviction | `sivnoxd` | Quickshell |
| Colour extraction, theme fan-out, thumbnails, frames | one-shot `sivnox` | long-lived daemon |
| Config file | the user | any daemon |
| Wallpaper state file | Quickshell on commit | any daemon |
| Hyprland core config | Sivnox managed files | the user directly |

---

## 5. Definition of done

v0.1 ships when all of the following are true:

1. `T1`–`T24` pass on the reference laptop.
2. Login reaches a usable desktop in under 5 seconds.
3. Idle memory is under 60 MB for the bar and under 90 MB with the launcher.
4. Idle CPU is under 1%.
5. Launcher cold-open is under 100 ms.
6. A wallpaper commit recolours the shell within 200 ms.
7. `kill` of the shell is recovered by `systemd --user`, and the bar re-renders.
8. Clipboard pins survive eviction; unpinned entries drop oldest-first.
9. Wallpaper preview reverts on cancel and commit survives a shell restart.
10. Hyprland loads the Lua tree without errors, and `hypr-vars.lua` changes a
    variable live.

---

## 6. Assumptions

- Hyprland 0.55+ with Lua configuration support is the target.
- Quickshell is available at a version that supports the Caelestia tree and
  `NotificationServer`; v0.1 follows distro defaults rather than pinning.
- Caelestia and Ryoku sources are available under GPL-3.0 for porting.
- One monitor is tested in v0.1. The model and protocol shape remain
  per-monitor-ready.
- The reference laptop has working UPower, NetworkManager, BlueZ, PipeWire,
  and `power-profiles-daemon` integrations through Quickshell.

## 7. Open decisions

`P1`–`P7` in the Build Sheet are closed before W0 unless an item explicitly says
otherwise. Their resolution belongs in `current_plan.md` and the relevant
implementation document:

| ID | Decision | Document |
|---|---|---|
| P1 | Frame location | `Frontend.md` |
| P2 | `hypridle` config ownership | `Hyprland.md` |
| P3 | Post-hook path | `Theme.md` |
| P4 | Caelestia fan-out file list | `Theme.md` |
| P5 | systemd unit names | `Interfaces.md` |
| P6 | Caelestia Hyprland paths | `Hyprland.md` |
| P7 | Path rewrite list | `Hyprland.md` |
