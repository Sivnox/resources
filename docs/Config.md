# Sivnox — configuration and state

**Status:** active. Derived from `Build_Sheet-Filled.md` (`L3`, §8, §9).
**User config:** `~/.config/sivnox/config.toml` (TOML, sparse overrides).
**Daemon/frontend state:** JSON, SQLite, and caches (separate files).
**Rule:** one writer per file. The user writes config; components write only
their own state.

---

## 1. Config model

### 1.1 Location

```
~/.config/sivnox/config.toml
```

- Owned by the user, mode `0600`, directory `0700`.
- Sparse: an omitted key uses the compiled default.
- Watched by `Config.qml` for live reload.
- Parsed only by `sivnox config dump` / `sivnox config check` (Rust `toml`).
- Never written by a daemon or by the shell.

### 1.2 Validation

- Unknown keys are ignored with a warning (forward compatibility) or rejected;
  v0.1 chooses **warn and ignore** so a newer config does not break an older
  shell.
- Wrong types are rejected.
- Out-of-range values are clamped to the nearest valid value and logged.
- Invalid TOML:
  1. `sivnox config dump` exits 2 with the parse location.
  2. `Config.qml` keeps the last good config and shows the error in the log.
  3. The install/doctor path may rename the file to `config.toml.broken` and
     copy a default; this is never silent.
- A missing file is valid: all defaults apply.

### 1.3 Reload

- Human edits are debounced (250 ms) by `Config.qml`.
- A valid change applies live; no shell restart (`T19`).
- A change to `hypr-vars.lua` or `hypr-user.lua` requires `hyprctl reload` and is
  handled by Hyprland, not `Config.qml`.

---

## 2. Full v0.1 schema

This is the canonical example. Defaults are shown; omit any key to use the
default.

```toml
# ~/.config/sivnox/config.toml
# Sparse overrides. Omit a key to use the compiled default.

[bar]
position = "top"        # top | bottom
height = 32             # 24..64
modules_left = ["launcher", "workspaces"]
modules_center = ["clock"]
modules_right = ["tray", "media", "volume", "brightness", "battery", "network", "bluetooth", "notifications", "power"]

[theme]
mode = "auto"           # auto | warm | dark
manual_mode = ""        # "" | "warm" | "dark" — persists across wallpaper change if set
warm_shift = true       # shift extracted colours toward warm ink/bone
apps = ["kitty", "gtk", "qt", "btop", "cava", "fuzzel", "papirus"]
post_hook = ""          # path to a script run after fan-out; "" disables

[wallpaper]
directory = "~/Pictures/Wallpapers"
monitor_min_resolution = "1920x1080"
video = false           # v0.1: disabled; frame preview only

[clipboard]
max_entries = 100       # 1..10000
max_bytes = 1048576     # 1024..104857600
pins_persist = true

[notifications]
timeout_ms = 5000       # 0..60000
dnd = false

[osd]
enabled = true
position = "center"     # center | top | bottom

[session]
idle_timeout_s = 300    # 0 disables idle lock
lock_on_lid_close = true

[hyprland]
vars_file = "~/.config/sivnox/hypr-vars.lua"
user_file = "~/.config/sivnox/hypr-user.lua"
```

---

## 3. Key reference

### 3.1 `[bar]`

| Key | Type | Default | Valid values | Notes |
|---|---|---|---|---|
| `position` | string | `"top"` | `top`, `bottom` | The bar edge and the layer-rule reservation. |
| `height` | integer | `32` | 24–64 | Logical pixels. |
| `modules_left` | string[] | see example | module names | Order matters. |
| `modules_center` | string[] | see example | module names | |
| `modules_right` | string[] | see example | module names | |

Known modules: `launcher`, `workspaces`, `clock`, `tray`, `media`, `volume`,
`brightness`, `battery`, `network`, `bluetooth`, `notifications`, `power`,
`keyboard`. An unknown module is ignored with a warning.

### 3.2 `[theme]`

| Key | Type | Default | Valid values | Notes |
|---|---|---|---|---|
| `mode` | string | `"auto"` | `auto`, `warm`, `dark` | See `Theme.md`. |
| `manual_mode` | string | `""` | `""`, `warm`, `dark` | Non-empty overrides `auto` and persists. |
| `warm_shift` | bool | `true` | | Shift hues toward ink/bone. |
| `apps` | string[] | see example | adapter names | `T18` requires `kitty` and `gtk`. |
| `post_hook` | string | `""` | path or `""` | Empty disables. |

### 3.3 `[wallpaper]`

| Key | Type | Default | Valid values | Notes |
|---|---|---|---|---|
| `directory` | string | `"~/Pictures/Wallpapers"` | path | `~` is expanded. |
| `monitor_min_resolution` | string | `"1920x1080"` | `WxH` | Filter below this. |
| `video` | bool | `false` | | v0.1 only previews a first frame. |

### 3.4 `[clipboard]`

| Key | Type | Default | Valid values | Notes |
|---|---|---|---|---|
| `max_entries` | integer | `100` | 1–10000 | |
| `max_bytes` | integer | `1048576` | 1024–104857600 | |
| `pins_persist` | bool | `true` | | Pins never evict when true. |

### 3.5 `[notifications]`

| Key | Type | Default | Valid values | Notes |
|---|---|---|---|---|
| `timeout_ms` | integer | `5000` | 0–60000 | 0 = no auto-dismiss. |
| `dnd` | bool | `false` | | Do not disturb. |

### 3.6 `[osd]`

| Key | Type | Default | Valid values | Notes |
|---|---|---|---|---|
| `enabled` | bool | `true` | | |
| `position` | string | `"center"` | `center`, `top`, `bottom` | |

### 3.7 `[session]`

| Key | Type | Default | Valid values | Notes |
|---|---|---|---|---|
| `idle_timeout_s` | integer | `300` | 0–86400 | 0 disables idle lock. |
| `lock_on_lid_close` | bool | `true` | | |

### 3.8 `[hyprland]`

| Key | Type | Default | Valid values | Notes |
|---|---|---|---|---|
| `vars_file` | string | `"~/.config/sivnox/hypr-vars.lua"` | path | Read by the managed Lua tree. |
| `user_file` | string | `"~/.config/sivnox/hypr-user.lua"` | path | Read by the managed Lua tree. |

---

## 4. Defaults and compilation

- Compiled defaults live in `sivnox-core` alongside the schema.
- `sivnox config dump` merges the sparse user file over the defaults and returns
  the complete effective config.
- `Config.qml` consumes only the merged JSON.
- A default change in code is a behaviour change; record it in the commit.
- The defaults are the same on every machine unless the user overrides them.

### 4.1 Why not JSON for config?

`L3` locks TOML for user-edited config and JSON for daemon-written state. TOML
gives comments and a readable sparse file; JSON is better for machine-written
state. Do not mix the two roles.

---

## 5. State files

State is written by the component that owns it. The user does not edit state.

| File | Owner | Format | Written when |
|---|---|---|---|
| `~/.config/sivnox/wallpaper.json` | Quickshell | JSON | Wallpaper commit |
| `~/.local/share/sivnox/clipboard.db` | `sivnoxd` | SQLite | Clipboard change |
| `~/.cache/sivnox/colours/<hash>.json` | `sivnox colours` | JSON | Colour extraction |
| `~/.cache/sivnox/thumbs/<hash>.jpg` | `sivnox thumb` | JPEG | Thumbnail request |
| `~/.cache/sivnox/frames/<hash>.png` | `sivnox frame` | PNG | Frame request |

### 5.1 `wallpaper.json`

```json
{ "path": "/home/user/Pictures/Wallpapers/night.jpg", "is_video": false }
```

- Only commit writes it.
- Preview and cancel never touch it.
- On start, a missing or invalid file falls back to the first valid wallpaper in
  the configured directory; if none exists, a neutral default.

### 5.2 Cache bounds

| Cache | Max size | Max entries | Eviction |
|---|---|---|---|
| Colours | 5 MB | 200 | oldest mtime |
| Thumbs | 50 MB | 1000 | oldest mtime |
| Frames | 50 MB | 1000 | oldest mtime |

`doctor`/the owning command prunes over-limit caches; no background daemon does
it.

### 5.3 Atomicity

- Non-SQLite: temp file, `fsync`, rename, `fsync` directory.
- SQLite: WAL, one writer, `busy_timeout`.
- A crash leaves the previous valid file.

---

## 6. Permissions

| Path | Mode |
|---|---|
| `~/.config/sivnox/` | `0700` |
| `~/.config/sivnox/config.toml` | `0600` |
| `~/.local/share/sivnox/` | `0700` |
| `~/.config/sivnox/clipboard.db` | `0600` |
| `~/.cache/sivnox/` | `0700` |
| Cache files | `0600` |
| `$XDG_RUNTIME_DIR/sivnox/` | `0700` |
| `clipboard.sock` | `0600` |

Files are user-owned and readable without special tools.

---

## 7. User override files

### 7.1 `hypr-vars.lua`

- Seeded from Caelestia's `variables.lua`.
- User-owned; never overwritten.
- Read by the managed Hyprland tree.
- See `Hyprland.md` §4.

### 7.2 `hypr-user.lua`

- Seeded with Sivnox keybinds and a header.
- User-owned; never overwritten.
- Read by the managed Hyprland tree.
- See `Hyprland.md` §5.

### 7.3 `hooks/post-theme`

- Default path for the fan-out post-hook.
- Executable script.
- Receives the nine token values in the environment.
- Empty `theme.post_hook` disables it.
- See `Theme.md` §5.4.

---

## 8. Migration policy

- v0.1 has an implicit config version. A `schema_version` key is added only
  when a migration is needed.
- A future key rename ships a migration that reads the old key, writes the new
  one, and logs. It never silently drops a user setting.
- Cache versions (`extractor_ver`, thumbnail version) invalidate by key, not by
  deleting the user's files.
- Clipboard `PRAGMA user_version` gates schema migration.

---

## 9. Testing

- `sivnox config dump` on the example returns a complete config.
- A missing key uses the default.
- An out-of-range value is clamped and logged.
- Invalid TOML exits 2 and does not crash `Config.qml`.
- A live edit applies without restart (`T19`).
- Every state file is written by exactly one component.
- A simulated crash mid-write leaves the previous file intact.
- Permissions match §6.

The acceptance procedure is in `Testing.md`.
