# Sivnox — current plan

**Status:** active execution plan. Derived from `Build_Sheet-Filled.md`.
**Scope:** Hyprland-only shell for a laptop daily driver.
**Definition of done:** `T1`–`T24` pass on the reference laptop.

The Build Sheet locks the decisions. This document turns them into a week-by-week
sequence and a pre-coding checklist. If this plan and the Build Sheet disagree,
the Build Sheet wins.

---

## 1. The shape in one screen

```
W0  Hyprland Lua tree + first pixels
W1  Notifications, OSD, session, lock, network/BT, media, idle
W2  Wallpaper picker + colour extractor + thumbnails
W3  Clipboard daemon + screenshot + night light + power profiles
W4  Theme fan-out + config complete + systemd + acceptance
```

- **Frontend:** Quickshell/QML, Caelestia project structure.
- **Only daemon:** `sivnoxd` (clipboard).
- **One-shot Rust:** `sivnox colours`, `sivnox theme apply`, `sivnox thumb`,
  `sivnox frame`, `sivnox config`, frontend wrappers.
- **Hyprland:** Caelestia Lua tree, managed core + two user override files.
- **Supervisor:** `systemd --user`.
- **License:** GPL-3.0.

---

## 2. Open items and when they close

Do not start the W0 copy-and-rewrite until P1, P2, P5, P6, and P7 are answered.
P4 is a W4 planning gate, not a W0 gate. Each answer goes in the named document.

| ID | Item | Decision needed | When | Document |
|---|---|---|---|---|
| P1 | Frame location | Is the rounded frame inside the Caelestia bar file, or a separate surface? Keep it if inside; defer if separate. | W0 | `Frontend.md` |
| P2 | `hypridle` config | Sivnox ships a default; the user may override. Confirm the file path and whether it lives under `~/.config/sivnox/` or `~/.config/hypr/`. | W0 | `Hyprland.md` |
| P3 | Post-hook default path | **Closed:** `~/.config/sivnox/hooks/post-theme`; empty `theme.post_hook` disables it. | closed | `Theme.md` |
| P4 | Caelestia fan-out file list | Identify the exact Python files and adapters to port. | W4 planning | `Theme.md` |
| P5 | systemd unit names | `sivnox-shell.service`, `sivnoxd.service`; `sivnoxd.socket` recommended for clipboard activation. Confirm unit names. | W0 | `Interfaces.md` |
| P6 | Caelestia Hyprland paths | Verify `hypr/hyprland.lua` + `hypr/modules/` in the Caelestia source. | W0, before copy | `Hyprland.md` |
| P7 | Path rewrite list | Every `~/.config/caelestia/` → `~/.config/sivnox/`, every `caelestia` command → `sivnox`. | W0, before copy | `Hyprland.md` |

### Additional pre-coding checklist

- [ ] Create the repository skeleton: `crates/`, `shell/`, `hypr/`, `data/`,
      `systemd/`, `docs/`.
- [ ] Add GPL-3.0 `LICENSE` and a `NOTICE` that names Caelestia and Ryoku as
      GPL-3.0 sources.
- [ ] Open `Provenance.md` and record every copied file before it is committed.
- [ ] Add a `rustfmt.toml`, `clippy` gate, and `qmllint` gate.
- [ ] Decide the Quickshell invocation: `qs -c sivnox` with a config directory
      that matches the Caelestia structure.
- [ ] Freeze the token schema in `Theme.md`.
- [ ] Freeze the CLI command names in `Interfaces.md`.
- [ ] Freeze the clipboard socket operations in `Clipboard.md`.
- [ ] Write the baseline measurement script described in `Testing.md`.
- [ ] Confirm the reference laptop has the required external tools.

No Rust daemon is written before W3. `sivnox config` and `sivnox doctor` can be
scaffolded in W0 because `Config.qml` needs them.

---

## 3. W0 — Hyprland config and first pixels

**Goal:** a Hyprland session that loads a Sivnox-managed Lua config and shows a
bar. No Rust daemon, no wallpaper picker, no clipboard.

### Deliverables

1. **Caelestia Hyprland Lua tree integrated**
   - Copy `hypr/hyprland.lua` and `hypr/modules/` into the repository's managed
     tree.
   - Rewrite `~/.config/caelestia/` → `~/.config/sivnox/`.
   - Rewrite Caelestia command names → `sivnox`.
   - Install to `~/.config/hypr/`.
2. **Override layers**
   - `~/.config/sivnox/hypr-vars.lua` seeded from `variables.lua`.
   - `~/.config/sivnox/hypr-user.lua` with Sivnox keybinds and a place for user
     rules.
3. **Quickshell shell root**
   - `shell.qml` with the Caelestia structure.
   - `Config.qml` reading TOML through `sivnox config dump`.
   - `Theme.qml` with the frozen token interface (defaults until W2).
   - One bar with clock, workspaces, battery, and volume (`R1`, `R2`, `R3`,
     `R5`, `R8`).
   - One `Loader` popup — the launcher — opened by a keybind (`T23`). The full
     launcher search lands in W1; W0 only proves the Loader and the keybind.
4. **Hyprland layer rule** reserving bar space.
5. **`sivnox-shell.service`** scaffolded as a `systemd --user` unit so the shell
   is supervised from the first week.
6. **Validation**
   - `hyprvalidate` passes on the managed tree and both override files.
   - `hyprctl reload` succeeds.

### Exit tests

- `T21`: Hyprland loads the Lua config without errors.
- `T22`: a value in `hypr-vars.lua` changes live.
- `T23`: a keybind in `hypr-user.lua` opens the popup.
- `T24`: the layer rule reserves space for the bar.
- A config change in `config.toml` applies without a shell restart.

### Not in W0

Notifications, wallpaper picker, clipboard, screenshot, theme fan-out, lock
screen, OSD, systemd units beyond the shell unit, and any Rust daemon.

---

## 4. W1 — the daily-driver surfaces

**Goal:** the shell can be used for a day without the wallpaper or clipboard
features.

### Deliverables

- Notifications: Quickshell `NotificationServer`, popups, history, DND,
  notification badge (`R10`, `R15`–`R17`, `O21`).
- OSD: volume, brightness, microphone, keyboard layout (`R18`–`R21`).
- Session menu: lock, suspend, reboot, shutdown (`R24`).
- Lock screen: `WlSessionLock` + PAM, themed from the current token set
  (`R22`, `O23`).
- Network and Bluetooth popups: Wi-Fi connect, Bluetooth pair/connect
  (`R43`, `R44`).
- Media bar widgets and media keys (`R40`–`R42`).
- `hypridle` wired for idle and lid close (`R23`, `T12`).
- Bar complete: tray, network, Bluetooth, brightness, keyboard layout, power
  button (`R4`, `R6`, `R7`, `R9`, `R11`, `R12`).
- Launcher: app search and mode prefixes (`R13`, `R14`).
- `sivnox open`, `sivnox osd`, `sivnox lock`, `sivnox session` wrappers.

### Exit tests

- Lid close → lock screen (`T12`).
- Volume key → OSD (`T13`).
- `notify-send` → notification arrives and dismisses (`T14`).
- Wi-Fi connect via menu (`T15`).
- Bluetooth pair via menu (`T16`).
- Launcher cold-open under 100 ms (`T5`); if it fails, trigger `L10`.

---

## 5. W2 — wallpaper and colour

**Goal:** a wallpaper picker that previews and commits, and a shell that
recolours from the wallpaper.

### Deliverables

- Quickshell wallpaper layer-shell surface (`R25`).
- Picker with live preview and commit/cancel (`R26`).
- `~/.config/sivnox/wallpaper.json` atomic write on commit.
- `sivnox thumb` → 128×128 cache (`R27`).
- `sivnox frame` → `ffmpeg` first-frame cache for focused video (`R28`).
- `sivnox colours` → 64×64 extraction, mode policy, token cache (`R29`, `R30`).
- `Theme.qml` applies tokens to the shell (`R31`).
- The picker calls `sivnox colours` on commit and applies the result.

### Exit tests

- Wallpaper preview reverts on cancel (`T9`).
- Commit survives a shell restart (`T10`).
- Colour apply within 200 ms (`T6`).
- Thumbnails and frames are cached and bounded.
- `sivnox colours` is deterministic and cache-hit is faster than cache-miss.

---

## 6. W3 — clipboard and capture

**Goal:** clipboard history with pins, and a screenshot overlay.

### Deliverables

- `sivnoxd` clipboard daemon:
  - SQLite at `~/.local/share/sivnox/clipboard.db`.
  - `wl-paste --watch` ingestion.
  - dedup, pins, eviction at 100 entries / 1 MB.
  - picker socket at `$XDG_RUNTIME_DIR/sivnox/clipboard.sock`.
  - `wl-copy` on select.
- Clipboard picker UI (`R33`–`R36`).
- Screenshot overlay: region select, live dimensions, save/copy (`R37`–`R39`).
- Night light toggle (`R47`).
- Brightness widget (`R45`).
- Power profiles widget (`R46`).

### Exit tests

- Clipboard pin survives eviction (`T7`).
- Unpinned entry drops oldest-first at the limit (`T8`).
- Screenshot region shows live dimensions (`T17`).
- Clipboard daemon restarts without losing the database.
- No clipboard data leaves the user's machine.

---

## 7. W4 — theme fan-out, config, systemd, acceptance

**Goal:** the shell is a daily driver and the v0.1 gate is green.

### Deliverables

- `sivnox theme apply`: Rust port of the Caelestia fan-out engine.
- Template renderer and at least kitty, GTK, and btop adapters; the rest of the
  app list as `P4` allows (`R32`, `T18`).
- Post-hook at `~/.config/sivnox/hooks/post-theme` (`O18`, `P3`).
- Named scheme catalog and `sivnox theme use` fallback (`O19`).
- Config complete: validation, live reload, transparency options
  (`R48`–`R50`).
- Keybinds added to `hypr-user.lua` (`R51`).
- `sivnox-shell.service`, `sivnoxd.service`, `sivnoxd.socket` (`R52`, `P5`).
- `sivnox doctor` dependency/config check.
- Baseline table recorded in the repository.

### Exit tests

- All `T1`–`T24` pass.
- Kitty, GTK, and btop recolour on wallpaper commit (`T18`).
- Config change applies without restart (`T19`).
- All daemons and files run as the user (`T20`).
- `systemd` restarts the shell after `kill` (`T11`).

---

## 8. Acceptance matrix

The full procedure is in `Testing.md`.

| Group | Tests |
|---|---|
| Performance | T1 login, T2 bar RSS, T3 bar + launcher RSS, T4 idle CPU, T5 launcher cold-open, T6 colour apply |
| Clipboard | T7 pins, T8 eviction |
| Wallpaper | T9 preview revert, T10 commit persistence |
| Recovery | T11 shell restart |
| Session | T12 lid lock, T13 OSD, T14 notification |
| Connectivity | T15 Wi-Fi, T16 Bluetooth |
| Capture/theme/config | T17 screenshot, T18 fan-out, T19 config reload |
| Hygiene | T20 user-only |
| Hyprland | T21 config loads, T22 variable override, T23 keybind, T24 layer rule |

`T5` failure triggers the Rust launcher alternative (`L10`). No other test has a
pre-approved alternative; a failure is fixed, not waived.

---

## 9. Post-v0.1 queue

Not started until v0.1 is green. These are the deferred items from `O1`–`O24`
plus obvious follow-ups:

1. Multi-monitor rendering (`O20`).
2. Settings GUI (`O16`).
3. Overview and window switcher (`O1`, `O2`).
4. Dock and dashboard (`O3`, `O4`).
5. Desktop widgets, calendar, weather, system monitor (`O5`–`O8`).
6. Visualizer and stage (`O9`, `O10`).
7. Special workspaces and window resizer (`O11`, `O12`).
8. Emoji picker, screenshot annotation, screen recording (`O13`–`O15`).
9. Update notifier (`O22`).
10. Keyring prompt (`L18`), extending `sivnoxd` rather than adding a daemon.
11. Video wallpaper playback; named-scheme editor; rice import/export.

---

## 10. Working agreements

- **Build Sheet first.** If a task is not in the Build Sheet or this plan, add
  it to the Build Sheet before building it.
- **One daemon.** Do not add a second long-lived process.
- **Quickshell owns protocols.** Do not add a Rust feed for something Quickshell
  already provides.
- **Measure before claiming.** Every performance statement has a number and a
  command in `Testing.md`.
- **GPL notices.** Every ported file names its source.
- **Every surface behind a `Loader`.** Code review checks this first.
- **Update docs with code.** A contradiction left behind is a bug.

---

## 11. Next action

Close `P1`–`P7`, scaffold the repository and license, then start W0:

> copy Caelestia's Hyprland Lua tree, rewrite paths, create `hypr-vars.lua` and
> `hypr-user.lua`, scaffold the Quickshell shell, and get the first bar on
> screen.

No backend daemon, no wallpaper picker, and no clipboard work until W2/W3.
