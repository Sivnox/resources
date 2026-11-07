# Sivnox — Build Sheet

**Status:** all decisions locked. Build from this.
**License:** GPL-3.0.
**Scope:** Hyprland only. Laptop daily driver.
**Reference repos:** Caelestia (bar, power session, lock screen, autostart, Hyprland Lua config, theme fan-out engine), Ryoku (clipboard, screenshot). Both GPL-3.0.

---

## 1. Locked decisions

| #       | Decision                     | Value                                                                                                                                  |
| ------- | ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| L1      | License                      | GPL-3.0                                                                                                                                |
| L2      | Shell supervisor             | `systemd --user`                                                                                                                       |
| L3      | Shell config format          | TOML (user-edited), JSON (daemon-written state)                                                                                        |
| L4      | Frontend                     | Quickshell/QML, Caelestia project structure                                                                                            |
| L5      | Surface pattern              | Every surface behind `Loader { active: … }`                                                                                            |
| L6      | Bar                          | From Caelestia, edited                                                                                                                 |
| L7      | Power / session menu         | From Caelestia, edited                                                                                                                 |
| L8      | Lock screen                  | From Caelestia, edited                                                                                                                 |
| L9      | Autostart                    | Caelestia's mechanism                                                                                                                  |
| L10     | Launcher                     | Caelestia QML; Rust alternative only if >15 MB RSS or >100 ms cold-open                                                                |
| L11     | Notification server          | Quickshell `NotificationServer`                                                                                                        |
| L12     | Wallpaper state owner        | Frontend (preview + commit); daemon does not own it                                                                                    |
| L13     | Clipboard                    | Ryoku, ported to Rust daemon                                                                                                           |
| L14     | Screenshot                   | Ryoku, QML overlay + `grim`                                                                                                            |
| L15     | Theme fan-out                | Caelestia engine, Python → Rust. In scope.                                                                                             |
| L16     | Colour extractor             | One-shot CLI (`sivnox colours <path>`); minimal shape now, algorithm quality deferred                                                  |
| L17     | Mode policy                  | Warm for light wallpaper, dark for dark wallpaper. Auto-toggle in TOML like Caelestia; manual choice persists across wallpaper change. |
| L18     | Keyring prompt               | Deferred. Plan: daemon owns `org.gnome.keyring.SystemPrompter` when it returns.                                                        |
| L19     | Per-monitor                  | In protocol from day one; v0.1 tested on one monitor                                                                                   |
| L20     | Grey night (A10)             | Guide, not gate. Function may override.                                                                                                |
| L21     | Rust footprint               | Clipboard daemon, colour extractor, CLI, theme fan-out, Hyprland config glue. Nothing else.                                            |
| L22     | External tools stay external | `grim`, `wl-copy`, `wl-paste`, `hypridle`, `brightnessctl`, `hyprsunset`                                                               |
| **L23** | **Hyprland configuration**   | **Caelestia's complete Lua-based tree, edited.                                                                                         |
|         |                              |                                                                                                                                        |

---

## 2. Required surfaces — laptop baseline

| # | Surface | Need | Source | Notes |
|---|---|---|---|---|
| R1 | Bar — workspaces | v0.1 | Caelestia QML | |
| R2 | Bar — active window | v0.1 | Caelestia QML | |
| R3 | Bar — clock / date | v0.1 | Caelestia QML | |
| R4 | Bar — system tray | v0.1 | Caelestia QML | Quickshell tray module |
| R5 | Bar — battery | v0.1 | Caelestia QML | UPower via Quickshell |
| R6 | Bar — network | v0.1 | Caelestia QML | NetworkManager via Quickshell |
| R7 | Bar — bluetooth | v0.1 | Caelestia QML | BlueZ via Quickshell |
| R8 | Bar — volume | v0.1 | Caelestia QML | PipeWire via Quickshell |
| R9 | Bar — brightness | v0.1 | Caelestia QML | `brightnessctl` |
| R10 | Bar — notifications badge | v0.1 | Caelestia QML | binds to NotificationServer |
| R11 | Bar — keyboard layout | v0.1 | Caelestia QML | |
| R12 | Bar — power button | v0.1 | Caelestia QML | opens session menu |
| R13 | Launcher — app search | v0.1 | Caelestia QML | Rust rewrite only if measured |
| R14 | Launcher — mode prefixes | v0.1 | Caelestia QML | calc, web, actions |
| R15 | Notifications — popups | v0.1 | Quickshell `NotificationServer` | |
| R16 | Notifications — history | v0.1 | Quickshell | |
| R17 | Notifications — DND | v0.1 | Quickshell | |
| R18 | OSD — volume | v0.1 | Own QML | |
| R19 | OSD — brightness | v0.1 | Own QML | |
| R20 | OSD — mic | v0.1 | Own QML | |
| R21 | OSD — keyboard layout | v0.1 | Own QML | |
| R22 | Lock screen | v0.1 | Caelestia QML, edited | `WlSessionLock` + PAM |
| R23 | Idle / suspend | v0.1 | `hypridle` | external |
| R24 | Session menu | v0.1 | Caelestia QML | lock, suspend, reboot, shutdown |
| R25 | Wallpaper — set / persist | v0.1 | Own QML | commit writes path file |
| R26 | Wallpaper — picker + live preview | v0.1 | Caelestia model | frontend owns preview state |
| R27 | Wallpaper — thumbnails | v0.1 | Caelestia logic → Rust | cached, 128×128 |
| R28 | Wallpaper — video first-frame | v0.1 | `ffmpeg` one-shot | on-demand for focused item |
| R29 | Theme — colour extraction | v0.1 | **Own Rust** (minimal shape) | `sivnox colours <path>` |
| R30 | Theme — token cache | v0.1 | Own Rust | `~/.cache/sivnox/colours/<hash>.json` |
| R31 | Theme — apply tokens to shell | v0.1 | Own QML (`Theme.qml`) | |
| R32 | Theme — fan-out to apps | v0.1 | **Caelestia engine → Rust** | app list = Caelestia's |
| R33 | Clipboard — history | v0.1 | **Ryoku → Rust daemon** | |
| R34 | Clipboard — pins | v0.1 | **Ryoku → Rust daemon** | pins survive eviction |
| R35 | Clipboard — size limit / eviction | v0.1 | **Ryoku → Rust daemon** | default 100 entries / 1 MB |
| R36 | Clipboard — picker UI | v0.1 | Ryoku QML, edited | |
| R37 | Screenshot — region select | v0.1 | Ryoku QML + `grim` | |
| R38 | Screenshot — live dimensions | v0.1 | Ryoku QML | |
| R39 | Screenshot — save / copy | v0.1 | `grim` + `wl-copy` | |
| R40 | Media — now playing | v0.1 | Quickshell MPRIS | |
| R41 | Media — controls | v0.1 | Quickshell MPRIS | |
| R42 | Media — keyboard media keys | v0.1 | Hyprland keybind | |
| R43 | Network — wifi connect | v0.1 | Quickshell NetworkManager | |
| R44 | Bluetooth — pair / connect | v0.1 | Quickshell BlueZ | |
| R45 | Brightness — adjust | v0.1 | `brightnessctl` | |
| R46 | Power profiles | v0.1 | `power-profiles-daemon` via Quickshell | |
| R47 | Night light toggle | v0.1 | `hyprsunset` | external |
| R48 | Transparency / blur | v0.1 | Own QML | |
| R49 | Config file | v0.1 | `Config.qml` reading TOML | |
| R50 | Config live reload | v0.1 | `Config.qml` file watcher | |
| R51 | Keybind integration | v0.1 | Hyprland Lua config | Sivnox keybinds added to hypr-user.lua |
| R52 | Autostart | v0.1 | Caelestia mechanism | systemd --user |

---

## 3. Hyprland configuration (Caelestia Lua tree)

This is a separate deliverable from the shell. It ships as managed files in `~/.config/hypr/`, edited through Sivnox's override layer.

### 3.1 Structure

Caelestia's Hyprland config is **fully Lua-based** (Hyprland 0.55+). Three layers:

| Layer | Path | Edit? | Purpose |
|---|---|---|---|
| **Core** | `~/.config/hypr/hyprland.lua` + `~/.config/hypr/modules/` | **No** — managed by Sivnox, overwritten on update | Loads modules, defines the base config |
| **Variables** | `~/.config/sivnox/hypr-vars.lua` | **Yes** — user override file | Overrides Caelestia's defaults: browser, terminal, gaps, borders, rounding, keybinds, etc. |
| **User** | `~/.config/sivnox/hypr-user.lua` | **Yes** — user custom file | Custom window rules, layer rules, binds not covered by variables |

### 3.2 What Sivnox takes from Caelestia

The complete `~/.config/hypr/` tree:
- `hyprland.lua` — main entry, sources modules
- `modules/` — keybinds, window rules, decorations, animations, input, monitors, workspace rules, layer rules, autostart, gestures
- `variables.lua` — reference list of overridable variables (read-only; copy to `~/.config/sivnox/hypr-vars.lua` as starting point)

### 3.3 What Sivnox edits

**Path changes:** all `~/.config/caelestia/` references → `~/.config/sivnox/`

> all caelestia specific commands are migrated to sivnox

### 3.4 Tools

| Tool | Purpose | When |
|---|---|---|
| `hyprconf2lua` | Convert legacy `hyprland.conf` to `.lua` | Only if importing an existing non-Lua config |
| `hyprvalidate` | Validate Lua syntax against Hyprland API schema | After every edit to `hypr-vars.lua` or `hypr-user.lua` |

These are dev tools, not runtime dependencies.

### 3.5 What this gives Sivnox

- A working Hyprland config from day one (window rules, gestures, animations, keybinds, monitors, workspace rules) — no reinvention
- A clean override layer that survives Caelestia upstream updates
- A place for Sivnox-specific rules without touching managed files

---

## 4. Optional surfaces — deferred

| #   | Surface                      | Need                                                                              | Source           |
| --- | ---------------------------- | --------------------------------------------------------------------------------- | ---------------- |
| O1  | Overview                     | defer                                                                             | —                |
| O2  | Window switcher              | defer                                                                             | —                |
| O3  | Dock                         | defer                                                                             | —                |
| O4  | Dashboard / control center   | defer                                                                             | —                |
| O5  | Desktop widgets              | defer                                                                             | —                |
| O6  | Calendar                     | defer                                                                             | —                |
| O7  | Weather                      | defer                                                                             | —                |
| O8  | System monitor               | defer                                                                             | —                |
| O9  | Visualizer                   | defer                                                                             | —                |
| O10 | Stage / parallax             | defer                                                                             | —                |
| O11 | Special workspaces           | defer                                                                             | —                |
| O12 | Window resizer               | defer                                                                             | —                |
| O13 | Emoji picker                 | defer                                                                             | —                |
| O14 | Screenshot annotation        | defer                                                                             | —                |
| O15 | Screen recording             | defer                                                                             | —                |
| O16 | Settings GUI                 | defer                                                                             | —                |
| O17 | Frame (rounded border)       | **check Caelestia source** — if inside bar file, keep; if separate surface, defer |                  |
| O18 | Post-hook after theme change | **yes**, ~30 lines                                                                | Caelestia engine |
| O19 | Named scheme catalog         | v0.1 (needed as extractor fallback)                                               | own              |
| O20 | Multi-monitor rendering      | protocol yes, test one monitor                                                    |                  |
| O21 | Notification actions         | v0.1 if Quickshell provides                                                       | Quickshell       |
| O22 | Update notifier              | defer                                                                             | —                |
| O23 | Lock-screen theming          | v0.1 (part of lock screen)                                                        | Caelestia        |
| O24 | Greeter (SDDM)               | kept                                                                              | —                |

---

## 5. Daemon map

| Daemon | Exists | Owns | Start | Language |
|---|---|---|---|---|
| Clipboard | **yes** | history, pins, eviction | on first picker open | **Rust** |
| Colour extractor | **one-shot CLI** | extraction, cache | invoked on wallpaper commit | **Rust** |
| Theme fan-out | **one-shot CLI** | writing app configs, post-hook | invoked on wallpaper commit | **Rust** |
| Keyring prompt | **deferred** | `org.gnome.keyring.SystemPrompter` | — | — |
| Wallpaper | no | — | — | — |
| Notification server | no | Quickshell owns | — | — |
| Network / BT / Audio / MPRIS | no | Quickshell owns | — | — |
| Shell supervisor | no | `systemd --user` | boot | — |

**Daemons to write: one — clipboard.** Everything else is one-shot CLI or Quickshell-native.

---

## 6. Rust conversion map

| Component | Current source | Port to | Lines (est.) | Risk |
|---|---|---|---|---|
| Clipboard daemon | Ryoku (Go) | Rust | ~600 | low |
| Wallpaper thumbnail cache | Caelestia (Python) | Rust | ~150 | low |
| Video first-frame | Caelestia (Python) | `ffmpeg` + thin Rust wrapper | ~50 | none |
| Colour extractor | Caelestia (Python) | Rust (minimal) | ~150 now, ~500 later | low |
| Theme fan-out engine | Caelestia (Python) | Rust | ~600–800 | **medium** — largest port |
| Template renderer | Caelestia (Python) | Rust | ~60 | low |
| CLI client | Caelestia (Python) | Rust | ~200 | low |
| Hyprland config glue | Caelestia (Lua) | Lua (keep) + Rust path rewrite | ~50 | none |

**Total Rust: ~1,850 lines. Hyprland config stays Lua.**

Do not port: bar, launcher, lock screen, power session, OSD, notifications, network/BT/audio menus, media. Those are QML and stay QML.

---

## 7. External tools

| Purpose | Tool | Required | Type |
|---|---|---|---|
| Screenshot capture | `grim` | yes | runtime |
| Clipboard copy / paste | `wl-copy` / `wl-paste` | yes | runtime |
| Idle / suspend | `hypridle` | yes | runtime |
| Brightness | `brightnessctl` | yes | runtime |
| Night light | `hyprsunset` | yes | runtime |
| Video first-frame | `ffmpeg` | yes | runtime |
| Power profiles | `power-profiles-daemon` | yes | runtime |
| Wallpaper display | Quickshell layer-shell | native | runtime |
| Convert legacy config | `hyprconf2lua` | no | dev |
| Validate Lua config | `hyprvalidate` | no | dev |

No version pins in v0.1; use distro defaults. Pin later if drift breaks builds.

---

## 8. Config schema (TOML, `~/.config/sivnox/config.toml`) example

User-edited. Sparse overrides — omit a key, get the compiled default.

```toml
[bar]
position = "top"        # top | bottom
height = 32
modules_left = ["launcher", "workspaces"]
modules_center = ["clock"]
modules_right = ["tray", "media", "volume", "brightness", "battery", "network", "bluetooth", "notifications", "power"]

[theme]
mode = "auto"           # auto | warm | dark
manual_mode = ""        # "" | "warm" | "dark" — persists across wallpaper change if set
warm_shift = true       # shift extracted colours toward warm ink/bone
apps = ["kitty", "gtk", "qt", "btop", "cava", "fuzzel", "papirus"]
post_hook = ""          # path to a script run after fan-out

[wallpaper]
directory = "~/Pictures/Wallpapers"
monitor_min_resolution = "1920x1080"
video = false           # v0.1: disabled; frame preview only

[clipboard]
max_entries = 100
max_bytes = 1048576
pins_persist = true

[notifications]
timeout_ms = 5000
dnd = false

[osd]
enabled = true
position = "center"

[session]
idle_timeout_s = 300
lock_on_lid_close = true

[hyprland]
vars_file = "~/.config/sivnox/hypr-vars.lua"
user_file = "~/.config/sivnox/hypr-user.lua"
```

**Rules:** watched for changes; invalid TOML → `.broken` + log + run defaults.

---

## 9. State schema (JSON / SQLite / cache)

Daemon-written. User does not edit.

| File | Contents |
|---|---|
| `~/.config/sivnox/wallpaper.json` | `{ "path": "…", "is_video": false }` — atomic write on commit |
| `~/.local/share/sivnox/clipboard.db` | SQLite: entries, pins, timestamps, sizes |
| `~/.cache/sivnox/colours/<hash>.json` | `{ "tokens": {…}, "extractor_ver": 1 }` |
| `~/.cache/sivnox/thumbs/<hash>.jpg` | 128×128 thumbnails |
| `~/.cache/sivnox/frames/<hash>.png` | video first-frames |

**Rules:** one writer per file; atomic writes; cache bounded (colours 5 MB / 200 entries; thumbs and frames 50 MB / 1000 each); no file the user cannot read.

---

## 10. Theme tokens

**Frozen key set (v1):**

| Key | Role |
|---|---|
| `siv_ink` | primary text, borders on light |
| `siv_bone` | base light surface |
| `siv_grey` | middle — metadata, disabled, dim |
| `siv_signal` | accent — state, active |
| `siv_ink_dim` | secondary text on light |
| `siv_ink_soft` | tertiary text |
| `siv_bone_dim` | secondary light surface |
| `siv_grey_deep` | border on dark |
| `siv_signal_hover` | accent hover state |

**Minimal extractor (v0.1):**
1. Decode → 64×64 downsample → discard full-res
2. Median luminance → light/dark
3. Respect `theme.mode` / `theme.manual_mode`; if `auto`, use wallpaper lightness
4. Pick dominant hue, clamp to warm/dark band (if `warm_shift = true`)
5. Derive four base tokens with fixed offsets
6. Derive ramps in OKLCH
7. Enforce 4.5:1 contrast for text-on-surface
8. Write cache, exit

**Fan-out:** Caelestia engine → Rust. Templates in `/usr/share/sivnox/templates/`. Runs on commit only. Post-hook after. App list from `theme.apps`.

---

## 11. Build order

| Week | Ships | Exit test |
|---|---|---|
| **W0** | Caelestia Hyprland Lua config integrated (`~/.config/hypr/` + `hypr-vars.lua` + `hypr-user.lua`); `shell.qml`, `Config.qml`, `Theme.qml`; one bar (clock, workspaces, battery, volume); one Loader popup | Hyprland loads without errors; keybind opens the popup; config change applies without restart |
| **W1** | Notifications (Quickshell), OSD (volume, brightness, mic), session menu (Caelestia), lock screen (Caelestia), network + BT popups, media bar widgets, `hypridle` wired | Lid close → locks; volume key → OSD; wifi connect via menu |
| **W2** | Wallpaper picker (preview/commit), colour extractor (minimal), `Theme.qml` binding, thumbnails, `sivnox colours` CLI | Wallpaper change recolours the shell within 200 ms |
| **W3** | Clipboard daemon (Rust) + picker, screenshot overlay + `grim`, night light, brightness widget, power profiles widget | Clipboard pins survive eviction; screenshot shows live dimensions |
| **W4** | Theme fan-out (Rust port of Caelestia engine), post-hook, config complete, transparency pass, keybinds added to `hypr-user.lua`, systemd units | Kitty/GTK/btop recolour on wallpaper commit; v0.1 acceptance passes |

---

## 12. Acceptance tests (v0.1)

| # | Test | Pass condition |
|---|---|---|
| T1 | Login to usable desktop | < 5 s |
| T2 | Idle RSS (bar only) | < 60 MB |
| T3 | Idle RSS (bar + launcher) | < 90 MB |
| T4 | Idle CPU | < 1% |
| T5 | Launcher cold-open | < 100 ms |
| T6 | Colour apply on wallpaper change | < 200 ms |
| T7 | Clipboard pin survives eviction | yes |
| T8 | Clipboard drops oldest unpinned at limit | yes |
| T9 | Wallpaper preview reverts on cancel | yes |
| T10 | Wallpaper commit survives shell restart | yes |
| T11 | Kill shell → systemd restarts it, bar re-renders | yes |
| T12 | Lid close → lock screen appears | yes |
| T13 | Volume key → OSD | yes |
| T14 | Notification arrives and dismisses | yes |
| T15 | Wifi connect via menu | yes |
| T16 | Bluetooth pair via menu | yes |
| T17 | Screenshot region shows live dimensions | yes |
| T18 | Theme fan-out writes at least kitty + GTK on commit | yes |
| T19 | Config change applies without restart | yes |
| T20 | All daemons and files run as user, no root | yes |
| **T21** | **Hyprland Lua config loads without errors** | **yes** |
| **T22** | **`hypr-vars.lua` override changes a variable live** | **yes** |
| **T23** | **Keybind in `hypr-user.lua` opens the launcher** | **yes** |
| **T24** | **Layer rule reserves space for the bar** | **yes** |

If T5 fails, trigger L10: write the Rust launcher alternative.

---

## 13. Out of scope (v0.1)

Keyring prompt · overview · window switcher · dock · dashboard · desktop widgets · calendar · weather · system monitor · visualizer · stage · special workspaces · window resizer · emoji picker · screenshot annotation · screen recording · settings GUI · frame (if separate surface) · update notifier · greeter · multi-monitor rendering · AI · telemetry · network features.

---

## 14. Open items (close before W0)

| #      | Item                                                            | Recommendation                                                                              |
| ------ | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| P1     | Frame location — inside Caelestia bar file or separate surface? | Check Caelestia source; if separate, defer                                                  |
| P2     | `hypridle` config — inline in Sivnox or user's own?             | Sivnox ships default, user can override                                                     |
| P3     | Post-hook default path                                          | `~/.config/sivnox/hooks/post-theme`                                                         |
| P4     | Caelestia engine — exact file list to port                      | Identify during W4 planning                                                                 |
| P5     | systemd unit names                                              | `sivnoxd.service`, `sivnox-shell.service`                                                   |
| **P6** | **Exact Caelestia Hyprland file paths**                         | **Verify: `hypr/hyprland.lua` + `hypr/modules/` — confirm during W0**                       |
| **P7** | **Path rewrite list**                                           | **Search-and-replace `~/.config/caelestia/` → `~/.config/sivnox/` in all copied Lua files** |

That's it. Everything else is decided.

---

**Start W0: copy Caelestia's Hyprland Lua tree, rewrite paths, create `hypr-vars.lua` and `hypr-user.lua`, scaffold the Quickshell shell.**