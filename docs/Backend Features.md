# Sivnox — v0.1 feature inventory

**Status:** active. Derived from `Build_Sheet-Filled.md`.
**Rule:** a feature is in v0.1 only if it appears in this file or in the Build
Sheet's `R1`–`R52` list. Everything in the old 128-feature list that is not here
is deferred or out of scope.

Use the checkboxes as the build tracker. The source column says where the work
comes from; the owner column says which process owns the state.

---

## 1. Frontend surfaces (`R1`–`R52`)

All are Quickshell/QML unless marked otherwise. Every closed surface sits behind
`Loader { active: … }`.

### 1.1 Bar

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R1 | Workspaces | Caelestia QML | Quickshell / Hyprland | ☐ |
| R2 | Active window | Caelestia QML | Quickshell / Hyprland | ☐ |
| R3 | Clock / date | Caelestia QML | Quickshell | ☐ |
| R4 | System tray | Caelestia QML | Quickshell | ☐ |
| R5 | Battery | Caelestia QML | Quickshell / UPower | ☐ |
| R6 | Network | Caelestia QML | Quickshell / NetworkManager | ☐ |
| R7 | Bluetooth | Caelestia QML | Quickshell / BlueZ | ☐ |
| R8 | Volume | Caelestia QML | Quickshell / PipeWire | ☐ |
| R9 | Brightness | Caelestia QML | `brightnessctl` | ☐ |
| R10 | Notifications badge | Caelestia QML | Quickshell `NotificationServer` | ☐ |
| R11 | Keyboard layout | Caelestia QML | Quickshell / Hyprland | ☐ |
| R12 | Power button | Caelestia QML | Quickshell | ☐ |

### 1.2 Launcher

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R13 | App search | Caelestia QML | Quickshell | ☐ |
| R14 | Mode prefixes (calc, web, actions) | Caelestia QML | Quickshell | ☐ |

Rust rewrite of the launcher is triggered only if `T5` (>100 ms cold-open) or
`L10` (>15 MB RSS) fails.

### 1.3 Notifications

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R15 | Popups | Quickshell `NotificationServer` | Quickshell | ☐ |
| R16 | History | Quickshell | Quickshell | ☐ |
| R17 | DND | Quickshell | Quickshell | ☐ |
| O21 | Notification actions | Quickshell | Quickshell | ☐ if provided |

### 1.4 OSD

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R18 | Volume OSD | Own QML | Quickshell | ☐ |
| R19 | Brightness OSD | Own QML | Quickshell | ☐ |
| R20 | Microphone OSD | Own QML | Quickshell | ☐ |
| R21 | Keyboard layout OSD | Own QML | Quickshell | ☐ |

### 1.5 Lock and session

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R22 | Lock screen | Caelestia QML, edited | Quickshell + PAM | ☐ |
| R23 | Idle / suspend | `hypridle` | external | ☐ |
| R24 | Session menu | Caelestia QML | Quickshell | ☐ |
| O23 | Lock-screen theming | Caelestia | Quickshell | ☐ |
| O24 | Greeter (SDDM) | kept, not modified | external | ☐ |

### 1.6 Wallpaper

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R25 | Set / persist | Own QML | Quickshell | ☐ |
| R26 | Picker + live preview | Caelestia model | Quickshell | ☐ |
| R27 | Thumbnails (128×128) | Caelestia logic → Rust | `sivnox thumb` | ☐ |
| R28 | Video first-frame | `ffmpeg` one-shot | `sivnox frame` | ☐ |

Wallpaper is drawn natively by Quickshell layer-shell. No `swww`, no
`hyprpaper`, no `mpvpaper`.

### 1.7 Theme

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R29 | Colour extraction | Own Rust (minimal) | `sivnox colours` | ☐ |
| R30 | Token cache | Own Rust | `sivnox colours` | ☐ |
| R31 | Apply tokens to shell | Own QML (`Theme.qml`) | Quickshell | ☐ |
| R32 | Fan-out to apps | Caelestia engine → Rust | `sivnox theme apply` | ☐ |
| O18 | Post-hook after theme change | Caelestia engine | `sivnox theme apply` | ☐ |
| O19 | Named scheme catalog | own | `sivnox theme use` | ☐ |

### 1.8 Clipboard

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R33 | History | Ryoku → Rust daemon | `sivnoxd` | ☐ |
| R34 | Pins | Ryoku → Rust daemon | `sivnoxd` | ☐ |
| R35 | Size limit / eviction | Ryoku → Rust daemon | `sivnoxd` | ☐ |
| R36 | Picker UI | Ryoku QML, edited | Quickshell | ☐ |

### 1.9 Screenshot

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R37 | Region select | Ryoku QML + `grim` | Quickshell + external | ☐ |
| R38 | Live dimensions | Ryoku QML | Quickshell | ☐ |
| R39 | Save / copy | `grim` + `wl-copy` | external | ☐ |

### 1.10 Media and system

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R40 | Now playing | Quickshell MPRIS | Quickshell | ☐ |
| R41 | Media controls | Quickshell MPRIS | Quickshell | ☐ |
| R42 | Media keys | Hyprland keybind | Hyprland | ☐ |
| R43 | Wi-Fi connect | Quickshell NetworkManager | Quickshell | ☐ |
| R44 | Bluetooth pair/connect | Quickshell BlueZ | Quickshell | ☐ |
| R45 | Brightness adjust | `brightnessctl` | external | ☐ |
| R46 | Power profiles | `power-profiles-daemon` | Quickshell / service | ☐ |
| R47 | Night light toggle | `hyprsunset` | external | ☐ |
| R48 | Transparency / blur | Own QML | Quickshell | ☐ |

### 1.11 Config and integration

| ID | Feature | Source | Owner | v0.1 |
|---|---|---|---|---|
| R49 | Config file | `Config.qml` reading TOML | user + Quickshell | ☐ |
| R50 | Config live reload | `Config.qml` watcher | Quickshell | ☐ |
| R51 | Keybind integration | Hyprland Lua | Hyprland | ☐ |
| R52 | Autostart | Caelestia mechanism + systemd | systemd user | ☐ |

---

## 2. Rust components

| Component | Features | Est. lines | Risk |
|---|---|---|---|
| `sivnoxd` | SQLite history, pins, eviction, watcher, picker socket, `wl-copy` select | ~600 | low |
| `sivnox colours` | 64×64 extract, mode policy, OKLCH ramps, contrast, cache | ~150 now / ~500 later | low |
| `sivnox theme apply` | Template renderer, app adapters, atomic writes, post-hook | ~600–800 | **medium** |
| `sivnox thumb` | 128×128 JPEG cache | ~150 | low |
| `sivnox frame` | `ffmpeg` first-frame wrapper + cache | ~50 | none |
| `sivnox config` | TOML parse, defaults merge, validation, JSON output | ~200 | low |
| `sivnox` CLI | Subcommand dispatch, JSON/exit codes, frontend wrappers | ~200 | low |
| Hyprland glue | Path rewrite, managed-tree sync, variable seeding | ~50 | none |

**Total Rust: roughly 1,850 lines.** Do not port the QML surfaces.

---

## 3. Hyprland configuration

| Feature | Source | Test |
|---|---|---|
| Complete Lua tree in `~/.config/hypr/` | Caelestia `hypr/hyprland.lua` + `modules/` | T21 |
| Variable override layer | `~/.config/sivnox/hypr-vars.lua` | T22 |
| User override layer | `~/.config/sivnox/hypr-user.lua` | T23 |
| Path rewrite | `~/.config/caelestia/` → `~/.config/sivnox/` | T21 |
| Command rewrite | `caelestia` → `sivnox` | T21 |
| Bar layer rule | Hyprland layer rule | T24 |
| Keybinds for Sivnox surfaces | `hypr-user.lua` | T23 |
| Idle / lid policy | `hypridle` + Sivnox default | T12 |
| Dev validation | `hyprvalidate` | T21 |

See `Hyprland.md`.

---

## 4. Delivery and systemd

| Feature | Detail |
|---|---|
| `sivnox-shell.service` | `systemd --user`, runs `qs -c sivnox`, restart on failure |
| `sivnoxd.service` | `systemd --user`, runs the clipboard daemon |
| `sivnoxd.socket` | socket activation for the clipboard picker (recommended; confirm `P5`) |
| License | GPL-3.0, notices for Caelestia/Ryoku ports |
| Templates | `/usr/share/sivnox/templates/` |
| Defaults | `/usr/share/sivnox/` (config example, schemes) |
| User files | `~/.config/sivnox/`, `~/.local/share/sivnox/`, `~/.cache/sivnox/` |

No installer, no package repository, no ISO in v0.1.

---

## 5. External tools

| Purpose | Tool | Required |
|---|---|---|
| Screenshot | `grim` | yes |
| Clipboard | `wl-copy`, `wl-paste` | yes |
| Idle / suspend | `hypridle` | yes |
| Brightness | `brightnessctl` | yes |
| Night light | `hyprsunset` | yes |
| Video first frame | `ffmpeg` | yes |
| Power profiles | `power-profiles-daemon` | yes |
| Wallpaper display | Quickshell layer-shell | native |
| Legacy config conversion | `hyprconf2lua` | dev only |
| Lua validation | `hyprvalidate` | dev only |

No version pins in v0.1; use distro defaults. Pin later if drift breaks builds.

---

## 6. Deferred (`O1`–`O24`)

- O1 Overview
- O2 Window switcher
- O3 Dock
- O4 Dashboard / control center
- O5 Desktop widgets
- O6 Calendar
- O7 Weather
- O8 System monitor
- O9 Visualizer
- O10 Stage / parallax
- O11 Special workspaces
- O12 Window resizer
- O13 Emoji picker
- O14 Screenshot annotation
- O15 Screen recording
- O16 Settings GUI
- O17 Frame — confirm `P1`; defer if it is a separate surface
- O20 Multi-monitor rendering — protocol/model only, test one monitor
- O22 Update notifier

O18, O19, O21, O23, O24 are in v0.1 as listed above.

Additional deferred items that are not surfaces:

- **Keyring prompt (`L18`)** — deferred. When it returns, it extends `sivnoxd`
  to own `org.gnome.keyring.SystemPrompter`; it is not a second daemon.
- **Video wallpaper playback** — deferred; first-frame preview only.
- **Multi-monitor rendering** — protocol/model only; test one monitor.
- **Materialize/doctor framework** — not built; `sivnox doctor` is a dependency
  check.
- **Settings GUI, installer, package repository, ISO** — deferred.

---

## 7. Acceptance mapping

| Test | Feature |
|---|---|
| T1 | Login time |
| T2 | Bar memory |
| T3 | Bar + launcher memory |
| T4 | Idle CPU |
| T5 | Launcher cold-open |
| T6 | Colour apply |
| T7–T8 | Clipboard |
| T9–T10 | Wallpaper preview/commit |
| T11 | Shell restart |
| T12 | Lock on lid close |
| T13 | Volume OSD |
| T14 | Notification |
| T15–T16 | Wi-Fi and Bluetooth |
| T17 | Screenshot dimensions |
| T18 | Theme fan-out |
| T19 | Config live reload |
| T20 | User-only processes |
| T21–T24 | Hyprland Lua config |

Procedures are in `Testing.md`.

---

## 8. Not in the old feature list anymore

The previous `Backend Features.md` contained 128 items across a daemon-centric
architecture: socket buses, per-topic ring buffers, a wallpaper lease, a scheme
service, lazy Rust services, materialize, doctor reconcilers, and a 128-test
suite. Those belong to the superseded design.

If a feature from that list is still wanted, it must be added to the Build Sheet
and then reintroduced here with an owner and a test.
