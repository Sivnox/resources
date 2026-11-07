# Sivnox — Hyprland configuration contract

**Status:** active. Derived from `Build_Sheet-Filled.md` (`L23`, §3, `R51`,
`T21`–`T24`).
**Base:** Caelestia's complete Lua-based Hyprland tree, edited.
**Scope:** Hyprland 0.55+ with Lua configuration support.

The Hyprland configuration is a separate deliverable from the QML shell. It
ships as managed files in `~/.config/hypr/`, with two user-owned override files
under `~/.config/sivnox/`.

---

## 1. Three layers

| Layer | Path | Edit? | Owner | Purpose |
|---|---|---|---|---|
| **Core** | `~/.config/hypr/hyprland.lua` + `~/.config/hypr/modules/` | **No** | Sivnox | Loads modules and defines the base config. Overwritten on update. |
| **Variables** | `~/.config/sivnox/hypr-vars.lua` | **Yes** | user | Overrides Caelestia's defaults: browser, terminal, gaps, borders, rounding, keybinds, and similar. |
| **User** | `~/.config/sivnox/hypr-user.lua` | **Yes** | user | Custom window rules, layer rules, and binds not covered by variables. |

The core tree is the only managed part. The override files are seeded once and
never rewritten by Sivnox.

### 1.1 Load order

```
hyprland.lua
├── modules/*.lua                 managed core
├── ~/.config/sivnox/hypr-vars.lua   user variables
└── ~/.config/sivnox/hypr-user.lua   user rules and binds
```

The exact `require`/`dofile` mechanism follows Caelestia's tree. W0 verifies the
entry point and the module names (`P6`).

---

## 2. Source tree to copy

`P6` must verify the exact Caelestia paths before the copy. The Build Sheet
assumes:

```
hypr/
├── hyprland.lua               # main entry, sources modules
├── modules/                   # one concern per file
│   ├── keybinds
│   ├── window rules
│   ├── decorations
│   ├── animations
│   ├── input
│   ├── monitors
│   ├── workspace rules
│   ├── layer rules
│   ├── autostart
│   └── gestures
└── variables.lua              # reference list of overridable variables
```

If Caelestia's actual layout differs, record the difference here and update
`P6` before copying. Do not invent a parallel tree.

---

## 3. W0 copy-and-rewrite procedure

### 3.1 Copy

1. Copy `hyprland.lua` and `modules/` into the repository's managed tree
   (for example `hypr/`).
2. Copy `variables.lua` as the reference for `hypr-vars.lua`.
3. Do not copy Caelestia's apps, services, or installer.

### 3.2 Path rewrite (`P7`)

Search the copied tree for:

- `~/.config/caelestia/` → `~/.config/sivnox/`
- `$HOME/.config/caelestia/` → `$HOME/.config/sivnox/`
- `/home/*/.config/caelestia/` → the user's `~/.config/sivnox/`
- `caelestia` command names → `sivnox`
- Caelestia socket or IPC paths → Sivnox equivalents, if any remain
- Caelestia asset paths → Sivnox asset paths

The copy is not done until a case-insensitive search for `caelestia` in the
managed tree returns no runtime references. Comments that credit the source are
allowed and encouraged.

### 3.3 Seed the override files

- `hypr-vars.lua` is seeded from `variables.lua` with the defaults Sivnox wants.
  If it already exists, leave it alone.
- `hypr-user.lua` is seeded with the Sivnox keybinds and a header explaining
  that it is user-owned.

### 3.4 Install

- Managed files install to `~/.config/hypr/`.
- Override files install to `~/.config/sivnox/`.
- The install/update path overwrites managed files only.
- The user's override files are never clobbered.

This is not a `materialize` framework. It is a small managed-file sync.

### 3.5 Validate

- `hyprvalidate` passes on the managed tree and both override files.
- `hyprctl reload` succeeds.
- `T21`–`T24` pass.

`hyprconf2lua` is used only if an existing non-Lua config is imported. Neither
tool is a runtime dependency.

---

## 4. The variables contract

`hypr-vars.lua` is the supported override surface. It should cover the knobs a
user is likely to change without editing managed code:

| Group | Examples |
|---|---|
| Applications | terminal, browser, file manager, launcher |
| Layout | gaps in/out, border size, rounding, active/inactive border colours |
| Input | keyboard layout, repeat rate, touchpad, tap-to-click |
| Monitors | scale, transform, position, workspace assignment |
| Decorations | blur, opacity, shadow, dim |
| Animations | enabled, speed, curve |
| Keybinds | application launchers, Sivnox surfaces, media keys |
| Workspaces | count, names, default assignments |
| Idle | lock timeout, suspend timeout, lid behaviour |
| Autostart | extra user commands |

The exact variable names are Caelestia's. The reference list is
`variables.lua`; `hypr-vars.lua` is the user-editable copy.

### 4.1 Rules

- A variable with no override uses the Caelestia/Sivnox default.
- A user override wins.
- The managed tree must not read a user file by an undocumented path.
- A variable that changes at runtime must be reloadable (`T22`).

---

## 5. Sivnox keybinds

The Sivnox keybinds live in `hypr-user.lua` (`R51`, `T23`). They call the stable
CLI wrappers, not QML internals.

| Action | Bind (example) | Command |
|---|---|---|
| Launcher | `SUPER + Space` | `sivnox open launcher` |
| Session menu | `SUPER + Escape` | `sivnox open session` |
| Wallpaper picker | `SUPER + W` | `sivnox open wallpaper` |
| Clipboard | `SUPER + V` | `sivnox open clipboard` |
| Screenshot region | `Print` | `sivnox screenshot region` |
| Lock | `SUPER + L` | `sivnox lock` |
| Volume up/down/mute | `XF86Audio*` | `sivnox osd volume …` |
| Brightness up/down | `XF86MonBrightness*` | `sivnox osd brightness …` |
| Microphone mute | `XF86AudioMicMute` | `sivnox osd mic …` |
| Media keys | `XF86AudioPlay` etc. | `playerctl` or Quickshell MPRIS |

The exact bindings may follow Caelestia's defaults. The commands are the
contract. A keybind that spawns an ad-hoc script is a bug; add the action to
`sivnox` first.

---

## 6. Layer rule for the bar

The bar is a layer-shell surface. A Hyprland layer rule reserves space so tiled
windows do not sit under it (`T24`).

Requirements:

- Reserve the bar height on the configured edge.
- Respect `[bar] position` (`top` or `bottom`) and `height`.
- Do not reserve space when the bar is hidden on fullscreen.
- The rule lives in the managed tree or the user layer, not in QML.

The exact Caelestia layer rule is kept if it already does this. Verify with
`hyprctl layers` and a tiled window at W0.

---

## 7. Autostart and supervision

`L2` and `L9` put supervision in `systemd --user`, not `exec-once`.

- `sivnox-shell.service` starts `qs -c sivnox`.
- `sivnoxd.service` starts the clipboard daemon.
- Hyprland's autostart module must not also start the shell, or two shells will
  fight for the layer surfaces. `L9` keeps Caelestia's autostart mechanism for
  non-shell helpers; the shell itself is the systemd unit.
- If Caelestia's mechanism starts the shell directly, replace that entry with
  the systemd unit; do not run both.
- Hyprland-specific helpers that are not shell surfaces may use `exec-once`.
- If Hyprland restarts, systemd keeps the shell alive; the shell reloads config
  and wallpaper from disk.

---

## 8. Idle and lid policy (`P2`)

`hypridle` owns idle and lid behaviour (`R23`, `T12`).

- Sivnox ships a default `hypridle` config.
- The user may override it.
- `P2` decides the path:
  - **Recommended:** `~/.config/sivnox/hypridle.conf`, launched by
    `sivnox-shell.service` or a dedicated user unit with
    `hypridle -c`.
  - Alternative: `~/.config/hypr/hypridle.conf`, if Caelestia already owns that
    path.
- Default policy: lock after `session.idle_timeout_s`; lock on lid close if
  `session.lock_on_lid_close`; suspend after a longer timeout.
- `hypridle` calls `sivnox lock`, not an ad-hoc lock binary.
- A failed lock must not leave the session unlocked and silent; log it and
  retry once.

Close `P2` at W0 and record the chosen path here.

---

## 9. Managed-file policy

| File | On update |
|---|---|
| `~/.config/hypr/hyprland.lua` | overwrite |
| `~/.config/hypr/modules/*` | overwrite |
| `~/.config/sivnox/hypr-vars.lua` | leave alone; seed only if absent |
| `~/.config/sivnox/hypr-user.lua` | leave alone; seed only if absent |
| `~/.config/sivnox/config.toml` | leave alone; seed only if absent |
| `~/.config/sivnox/hypridle.conf` | leave alone; seed only if absent |

A user who edits a managed file may lose the edit. The header of every managed
file says so and points to the override layer.

---

## 10. Tools

| Tool | Purpose | When |
|---|---|---|
| `hyprconf2lua` | Convert a legacy `hyprland.conf` to Lua | Only when importing an existing config |
| `hyprvalidate` | Validate Lua syntax against the Hyprland API schema | After every edit to vars/user files |

Both are development tools. They are not runtime dependencies and are not
shipped in v0.1.

---

## 11. Acceptance

| Test | Requirement |
|---|---|
| T21 | The Hyprland Lua config loads without errors. |
| T22 | A variable in `hypr-vars.lua` changes live after reload. |
| T23 | A keybind in `hypr-user.lua` opens the launcher. |
| T24 | The layer rule reserves space for the bar. |

Additional checks:

- `rg -i caelestia` on the managed tree returns only attribution comments.
- `hyprctl reload` is clean.
- A missing user override file does not stop the session; defaults apply.
- A malformed user override file is reported and the previous config remains.

---

## 12. Open items

| ID | Item | Close by |
|---|---|---|
| P2 | `hypridle` config path and ownership | W0 |
| P6 | Exact Caelestia Hyprland file paths | W0, before copy |
| P7 | Complete path/command rewrite list | W0, before copy |

Record the answers in this document when closed.
