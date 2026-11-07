# Sivnox — interfaces and file contracts

**Status:** active. Derived from `Build_Sheet-Filled.md` (`L21`, `P5`, §5, §7,
§9).
**Purpose:** the stable boundaries between the QML shell, the Rust tools, and the
user. If a component needs to reach around these, the contract is wrong.

---

## 1. Binaries

| Binary | Kind | Runs | Purpose |
|---|---|---|---|
| `sivnox` | one-shot CLI | on demand | Extract colours, fan out themes, build assets, parse config, ask the frontend to act, run diagnostics. |
| `sivnoxd` | daemon | `systemd --user` / socket activation | Clipboard history, pins, eviction, picker socket. |

There are no other Sivnox binaries in v0.1. No `sivnox-wallpaper`, no
`sivnox-scheme`, no `sivnox-ipc`.

---

## 2. `sivnox` CLI

### 2.1 Global rules

- `sivnox <command> [args]`.
- Help: `sivnox --help`, `sivnox <command> --help`.
- Machine output is JSON on stdout when `--json` is passed.
- Human diagnostics go to stderr.
- No command starts a daemon except `sivnoxd` (not exposed as a subcommand).
- No command performs network I/O.
- All commands are safe to run repeatedly.

### 2.2 Commands

#### `sivnox colours <path>`

Extract a token set.

```
sivnox colours ~/Pictures/Wallpapers/night.jpg --json
```

- Output: token JSON (`Theme.md` §3.2).
- Cache: `~/.cache/sivnox/colours/<hash>.json`.
- Options: `--mode auto|warm|dark`.
- Exit: 0 success, 2 bad input, 3 cache failure.

#### `sivnox theme apply <path>`

Fan out tokens to app configs and run the post-hook.

```
sivnox theme apply ~/Pictures/Wallpapers/night.jpg --json
```

- Output: `{ "ok": [...], "failed": [...] }`.
- Runs on commit only.
- Options: `--no-hook`.
- Exit: 0 if at least one adapter succeeded, 1 if all failed.

#### `sivnox theme use <name>`

Apply a named scheme from `~/.config/sivnox/schemes/`.

- Output: same shape as `theme apply`.
- Used as a fallback when extraction is unavailable.

#### `sivnox thumb <path>`

Build or read a 128×128 thumbnail.

```
sivnox thumb ~/Pictures/Wallpapers/night.jpg
```

- Output: absolute path to the cached JPEG.
- Cache: `~/.cache/sivnox/thumbs/<hash>.jpg`.
- Options: `--size 128`.
- Exit: 0 success, 2 bad input, 3 cache failure.

#### `sivnox frame <path>`

Extract the first frame of a video/GIF.

```
sivnox frame ~/Pictures/Wallpapers/loop.mp4
```

- Output: absolute path to the cached PNG.
- Cache: `~/.cache/sivnox/frames/<hash>.png`.
- Uses `ffmpeg` with an explicit scale; never emits a full-resolution frame.
- Exit: 0 success, 2 bad input or missing `ffmpeg`, 3 cache failure.

#### `sivnox config dump`

Print the effective config as JSON for `Config.qml`.

```
sivnox config dump
```

- Merges `config.toml` over compiled defaults.
- Exit: 0 valid, 2 invalid TOML (stderr explains), 4 file missing (defaults are
  still printed; the shell decides whether that is an error).

#### `sivnox config check`

Validate `config.toml` without printing the merged config.

- Exit: 0 valid, 2 invalid.

#### `sivnox open <surface>`

Ask the running frontend to open a surface.

```
sivnox open launcher
sivnox open session
sivnox open clipboard
```

Surfaces: `launcher`, `notifications`, `session`, `wallpaper`, `clipboard`,
`screenshot`. The bar is always present. Unknown surface → exit 2.

#### `sivnox osd <kind> [value]`

Show an OSD.

```
sivnox osd volume 42
sivnox osd brightness +5
sivnox osd mic mute
sivnox osd layout us
```

Kinds: `volume`, `brightness`, `mic`, `layout`. The value is passed to the
frontend; the frontend clamps it.

#### `sivnox lock`

Lock the session through the frontend's `WlSessionLock` surface.

#### `sivnox session <action>`

Actions: `lock`, `suspend`, `reboot`, `shutdown`, `logout`. Each is an explicit
command; no QML binding spawns `systemctl` or `loginctl` on a timer.

#### `sivnox screenshot [region|window|output]`

Open the screenshot overlay. Default: `region`.

#### `sivnox doctor`

Check dependencies and local state. Prints a report, exits non-zero if a
required tool or file is missing. Does not repair silently.

### 2.3 Frontend IPC delegation

`sivnox open`, `osd`, `lock`, `session`, and `screenshot` are wrappers. They
delegate to the running Quickshell instance (`qs ipc -c sivnox …` or the
Caelestia equivalent). The exact handler names are internal to the frontend and
settled in W0. The CLI command names are the stable contract.

If the frontend is not running, they exit 5 with a clear message. They do not
start it.

---

## 3. Clipboard socket

The full request/reply contract is in `Clipboard.md` §5. Summary:

| Property | Value |
|---|---|
| Path | `$XDG_RUNTIME_DIR/sivnox/clipboard.sock` |
| Mode | `0600` |
| Framing | one JSON request line → one JSON reply line |
| Version | `v: 1` |
| Ops | `list`, `get`, `select`, `pin`, `unpin`, `delete`, `clear_unpinned`, `status` |

Only `sivnoxd` listens. Only the QML picker and `sivnox` diagnostics connect.

---

## 4. Files and writers

| Path | Format | Writer | Reader |
|---|---|---|---|
| `~/.config/sivnox/config.toml` | TOML | user | `sivnox config`, `Config.qml` |
| `~/.config/sivnox/wallpaper.json` | JSON | Quickshell on commit | Quickshell on start |
| `~/.config/sivnox/hypr-vars.lua` | Lua | user | Hyprland |
| `~/.config/sivnox/hypr-user.lua` | Lua | user | Hyprland |
| `~/.config/sivnox/hooks/post-theme` | executable | user | `sivnox theme apply` |
| `~/.config/sivnox/schemes/*.json` | JSON | user | `sivnox theme use` |
| `~/.local/share/sivnox/clipboard.db` | SQLite | `sivnoxd` | `sivnoxd` |
| `~/.cache/sivnox/colours/<hash>.json` | JSON | `sivnox colours` | `sivnox colours` (returns JSON to `Theme.qml`) |
| `~/.cache/sivnox/thumbs/<hash>.jpg` | JPEG | `sivnox thumb` | wallpaper picker |
| `~/.cache/sivnox/frames/<hash>.png` | PNG | `sivnox frame` | wallpaper picker |
| `~/.config/hypr/hyprland.lua` | Lua | Sivnox install/update | Hyprland |
| `~/.config/hypr/modules/*` | Lua | Sivnox install/update | Hyprland |

**One writer per row.** A reader that rewrites a file is a bug.

### 4.1 Atomic write recipe

For every non-SQLite write:

1. Write to a temp file in the same directory.
2. `fsync` the temp file.
3. `rename` over the target.
4. `fsync` the directory.

For SQLite: WAL mode, `synchronous = NORMAL`, one writer.

### 4.2 Permissions

- Directories: `0700`.
- Files: `0600`.
- Sockets: `0600`.
- Executables shipped in `/usr/share` may be `0755`; user-owned files are
  `0600`.

---

## 5. State file shapes

### 5.1 `wallpaper.json`

```json
{
  "path": "/home/user/Pictures/Wallpapers/night.jpg",
  "is_video": false
}
```

- Written only on commit.
- Missing or unreadable → fallback wallpaper, never a crash.
- `is_video` is metadata for the picker; video playback is disabled.

### 5.2 Colour cache

```json
{
  "tokens": {
    "siv_ink": "#cdc4ba",
    "siv_bone": "#0b0b0b",
    "siv_grey": "#958f87",
    "siv_signal": "#e2342a",
    "siv_ink_dim": "#b0a9a0",
    "siv_ink_soft": "#7a756e",
    "siv_bone_dim": "#17171a",
    "siv_grey_deep": "#3a3a3a",
    "siv_signal_hover": "#f05a50"
  },
  "extractor_ver": 1
}
```

### 5.3 Clipboard database

See `Clipboard.md` §3.2. The schema version is `PRAGMA user_version = 1`.

---

## 6. Configuration handshake

`Config.qml` does not parse TOML itself. It runs:

```
sivnox config dump
```

and consumes JSON. The one-shot command:

- reads `~/.config/sivnox/config.toml`,
- merges it over compiled defaults,
- validates types and ranges,
- returns the effective config.

On invalid TOML:

- `sivnox config dump` exits 2 with a message.
- `Config.qml` keeps the last good config, logs once, and asks the user to fix
  the file. The install/doctor path may rename it to `.broken` and copy a fresh
  default; the shell never deletes user input silently.

See `Config.md` for the schema.

---

## 7. Versioning

| Thing | Version | Invalidation |
|---|---|---|
| Colour cache | `extractor_ver` | Bump when the algorithm or token derivation changes. |
| Clipboard DB | `PRAGMA user_version` | Migration on open. |
| Clipboard socket | `v: 1` | Additive ops only within a release. |
| Config | implicit v0.1 | A schema version is added only when a migration is needed. |
| Token key set | v1 | Frozen; a change is a schema change. |

No component promises compatibility with an older Sivnox release. The project
ships as a whole.

---

## 8. systemd units (`P5`)

| Unit | Type | Runs |
|---|---|---|
| `sivnox-shell.service` | user service | `qs -c sivnox` |
| `sivnoxd.service` | user service | `sivnoxd` |
| `sivnoxd.socket` | user socket | clipboard picker activation |

- All are `systemd --user` units.
- No system unit, no root, no setuid.
- `sivnox-shell.service` restarts on failure with a short backoff.
- `sivnoxd.service` requires `sivnoxd.socket`.
- `P5` confirms the names before W0.

---

## 9. Exit codes

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | unexpected internal error / all fan-out adapters failed |
| 2 | bad input, invalid config, unknown surface |
| 3 | I/O or cache failure |
| 4 | required dependency or file missing |
| 5 | frontend not running |

`--json` errors use `{ "ok": false, "error": { "code": "...", "message": "..." } }`.

---

## 10. What is deliberately absent

- No general IPC bus.
- No wallpaper command. The frontend owns wallpaper; the CLI does not set it.
- No notification command. Quickshell owns the server.
- No service start/stop commands. There are no Rust services.
- No keyring prompt. `L18` defers it; when it returns, it extends `sivnoxd`.
- No `status` daemon endpoint. `sivnox doctor` and `journalctl` are enough.
- No network command of any kind.

---

## 11. Testing

- Every command has a `--help` line and a test for success and failure.
- `sivnox config dump` matches the example in `Config.md`.
- `sivnox colours` is deterministic and cache-aware.
- Clipboard socket ops round-trip.
- File permissions are `0600`/`0700`.
- No command opens a network socket (`strace -f -e trace=network`).
- `sivnox doctor` reports a missing tool without crashing.

The acceptance procedure is in `Testing.md`.
