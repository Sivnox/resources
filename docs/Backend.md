# Sivnox backend — v2 plan

**Status:** active. Derived from `Build_Sheet-Filled.md`.
**Scope:** the small Rust surface only. The shell is Quickshell/QML.
**License:** GPL-3.0.

The old v1 plan — one control-plane daemon owning wallpaper, scheme, config,
services, and IPC — is obsolete. `Build_Sheet-Filled.md` locks a different
shape:

- **One long-lived daemon:** `sivnoxd`, clipboard only.
- **One-shot CLIs:** colour extraction, theme fan-out, thumbnails, first frames,
  config parsing, and frontend action wrappers.
- **No general IPC bus.** Quickshell owns the shell and the wallpaper.
- **No wallpaper daemon, no renderer daemon, no scheme daemon, no service
  daemons.**

See `Frontend.md` for the QML half and `Interfaces.md` for the exact command and
socket contracts.

---

## 1. Process model

```
systemd --user
├── sivnox-shell.service    qs -c sivnox                 (Quickshell, all surfaces)
├── sivnoxd.socket          socket activation             (clipboard picker)
└── sivnoxd.service         sivnoxd                        (clipboard daemon)

one-shot
├── sivnox colours <path>          → token JSON + cache
├── sivnox theme apply <path>      → fan-out + post-hook
├── sivnox thumb <path>            → 128×128 cache path
├── sivnox frame <path>            → first-frame cache path
├── sivnox config dump|check       → normalized JSON / validation
├── sivnox open <surface>          → frontend IPC
├── sivnox osd <kind> [value]      → frontend IPC
├── sivnox lock|session|screenshot → frontend IPC
└── sivnox doctor                  → dependency/config check
```

`L2` puts supervision in `systemd --user`. `L21` limits the Rust footprint to
this list. `P5` names `sivnoxd.service` and `sivnox-shell.service`; a
`sivnoxd.socket` is the recommended way to satisfy "clipboard starts on first
picker open" without a boot-time daemon.

---

## 2. Crate layout

```
sivnox/
├── Cargo.toml                     workspace
├── crates/
│   ├── sivnox/                    CLI binary: colours, theme, thumb, frame,
│   │                              config, open, osd, lock, session, screenshot,
│   │                              doctor
│   ├── sivnoxd/                   clipboard daemon binary
│   ├── sivnox-core/               shared library: paths, config schema,
│   │                              content hash, atomic write, token types,
│   │                              template rendering, sqlite helpers
│   └── sivnox-tests/              integration tests
├── shell/                         Quickshell/QML tree (Caelestia structure)
├── hypr/                          managed Hyprland Lua tree
├── data/
│   ├── templates/                 /usr/share/sivnox/templates
│   └── schemes/                   named TokenSet fallback
├── systemd/                       user units
└── docs/
```

The crate boundaries are deliberately small. `sivnox-core` holds code shared by
the CLI and daemon; it is not a framework and has no async runtime requirement.

**Dependencies:** `serde`, `serde_json`, `toml`, `blake3`, `image`,
`fast_image_resize` (or `image`'s resize), `rusqlite` (bundled SQLite or system
SQLite), `clap` (CLI only), and `anyhow`/`thiserror` for errors. No async runtime
is required. The clipboard watcher uses a child process and blocking I/O.

---

## 3. `sivnoxd` — clipboard daemon

The only long-lived Rust process. See `Clipboard.md` for the full contract.

### Responsibilities

- Watch the Wayland clipboard via `wl-paste --watch`.
- Store text entries in SQLite at `~/.local/share/sivnox/clipboard.db`.
- Deduplicate by content hash.
- Enforce `clipboard.max_entries` and `clipboard.max_bytes`.
- Preserve pinned entries across eviction.
- Serve the picker over `$XDG_RUNTIME_DIR/sivnox/clipboard.sock`.
- Copy a selected entry back with `wl-copy`.

### Lifecycle

- Started by `sivnoxd.socket` on the first picker connection, or manually.
- Restarts on crash with `systemd` backoff.
- Holds one SQLite connection in WAL mode.
- Supervises the `wl-paste` child; restarts it with backoff if it exits.

### Non-responsibilities

- No clipboard UI. The picker is QML.
- No image/binary entry rendering in v0.1. The entry `kind` column exists so it
  can be added later.
- No network, no sync, no history export beyond the database.

---

## 4. `sivnox colours` — colour extractor

One-shot. See `Theme.md` for the algorithm and token schema.

```
sivnox colours <path> [--mode auto|warm|dark] [--json]
```

- Reads `theme.mode` and `theme.manual_mode` when `--mode` is absent.
- Decodes to 64×64, discards the full bitmap, computes tokens.
- Writes `~/.cache/sivnox/colours/<hash>.json`.
- Prints the token JSON on stdout when `--json` is used.
- Exit 0 on success; exit 2 on unreadable/unsupported input; exit 3 on cache
  write failure (tokens are still printed).

The cache key is `blake3(file bytes) + extractor_ver`. The cache is bounded at
5 MB / 200 entries, LRU by mtime.

---

## 5. `sivnox theme apply` — theme fan-out

One-shot. See `Theme.md`.

```
sivnox theme apply <path> [--json]
sivnox theme use <name> [--json]
```

- Runs on wallpaper commit only.
- Extracts or reads cached tokens.
- Renders `/usr/share/sivnox/templates/*` with `{{ key }}` substitution.
- Writes app configs atomically.
- Runs the optional post-hook at `~/.config/sivnox/hooks/post-theme`.
- Prints `{ "ok": [...], "failed": [...] }` when `--json` is used.

App list comes from `theme.apps`. The exact Caelestia adapter file list is `P4`
and is finalized in W4 planning.

---

## 6. `sivnox thumb` and `sivnox frame` — wallpaper assets

One-shot. Called by the wallpaper picker.

```
sivnox thumb <path> [--size 128]
sivnox frame <path>
```

- `thumb`: decode → 128×128 JPEG at `~/.cache/sivnox/thumbs/<hash>.jpg`.
- `frame`: `ffmpeg -vf scale=... -frames:v 1` → PNG at
  `~/.cache/sivnox/frames/<hash>.png`. On demand for the focused item only.
- Cache key: content hash + size/version.
- Bounds: 50 MB / 1000 entries each.
- No full-resolution bitmap enters the Rust process beyond the decode needed
  for the target size.

The picker never pre-extracts frames for the whole grid.

---

## 7. `sivnox config` — TOML parser and validator

One-shot. See `Config.md`.

```
sivnox config dump   # normalized JSON for Config.qml
sivnox config check  # validate, exit non-zero on error
```

`Config.qml` watches `config.toml` and calls `sivnox config dump` on change. This
keeps one TOML parser (Rust `toml`) and avoids a second TOML implementation in
JavaScript. A parse failure returns a typed error; the shell renames the file to
`.broken`, logs, and applies defaults.

`config dump` merges the sparse user file over compiled defaults and returns the
complete effective config.

---

## 8. `sivnox open/osd/lock/session/screenshot` — frontend wrappers

These are thin one-shot commands that ask the running Quickshell instance to act.
They exist so keybinds and scripts have one stable interface and do not need to
know Quickshell's IPC handler names.

```
sivnox open <surface>
sivnox osd <volume|brightness|mic|layout> [value]
sivnox lock
sivnox session <lock|suspend|reboot|shutdown|logout>
sivnox screenshot [region|window|output]
```

Implementation detail: they delegate to `qs ipc -c sivnox ...` (or the equivalent
Caelestia IPC entry point). The exact handler names are settled in W0 and
documented in `Frontend.md`. They are not daemons and hold no state.

---

## 9. `sivnox doctor` — local health check

One-shot, optional but recommended before W0 acceptance.

Checks:

- `qs` and Quickshell version available.
- `grim`, `wl-copy`, `wl-paste`, `hypridle`, `brightnessctl`, `hyprsunset`,
  `ffmpeg`, `power-profiles-daemon` present.
- `config.toml` parses; report `.broken` files.
- Clipboard database opens and schema version matches.
- Cache directories exist and are within bounds.
- Socket path is not stale.
- Fonts/templates required by the shell are present.

It never repairs silently. It prints a report and exits non-zero on a missing
required dependency.

---

## 10. Hyprland glue

The Hyprland configuration stays Lua (`L23`). Rust's role is limited to:

- Copying the managed tree at install/update time.
- Rewriting `~/.config/caelestia/` to `~/.config/sivnox/` and Caelestia command
  names to Sivnox command names.
- Seeding `hypr-vars.lua` from `variables.lua` when absent.
- Never editing a user's `hypr-vars.lua` or `hypr-user.lua`.

This can be a small `sivnox hypr sync` subcommand or an install script; it is not
a runtime daemon. See `Hyprland.md`.

---

## 11. What the backend does not do

| Not built | Owner |
|---|---|
| Wallpaper rendering or state | Quickshell |
| Notifications | Quickshell `NotificationServer` |
| Bar, launcher, lock, OSD, session, screenshot UI | Quickshell/QML |
| Battery, audio, network, Bluetooth, MPRIS, tray, clock | Quickshell / external services |
| General IPC bus | nobody |
| Settings GUI | deferred (`O16`) |
| Keyring prompt | deferred (`L18`); when it returns it extends `sivnoxd`, not a second daemon |
| Materialize/doctor framework | not in v0.1; `sivnox doctor` is a small dependency check |
| Video wallpaper playback | deferred; frame preview only |
| AI, network, telemetry | forbidden |

---

## 12. Error and logging policy

- One error type per binary; print a human line to stderr and a typed JSON error
  when `--json` is used.
- Logs go to stderr under `systemd`; `journalctl --user -u ...` is the viewer.
- No log file in the user's home in v0.1. No rotation, no telemetry.
- Exit codes:
  - `0` success
  - `1` unexpected internal error
  - `2` bad input / unsupported file
  - `3` I/O or cache failure
  - `4` dependency missing

---

## 13. Implementation order

| Week | Rust work |
|---|---|
| W0 | Scaffold workspace; `sivnox config dump/check`; `sivnox doctor` skeleton. No daemon yet. |
| W1 | Nothing new required; use Quickshell and external tools. |
| W2 | `sivnox colours`, `sivnox thumb`, `sivnox frame`; wire picker. |
| W3 | `sivnoxd` + picker socket; screenshot wrapper. |
| W4 | `sivnox theme apply` (fan-out + templates + post-hook); `sivnox open/osd/lock/session`; systemd units; `doctor` complete. |

The daemon is not written until W3. If W0 or W1 needs clipboard, use `wl-paste`
directly and defer history.

---

## 14. Testing

Unit tests live beside each component. Integration tests live in
`sivnox-tests/`:

- `config dump` round-trips the TOML example from `Config.md`.
- `colours` is deterministic for a fixed image; cache hit is faster than a miss.
- `thumb`/`frame` respect the cache key and bounds.
- clipboard insert, dedup, pin, eviction, and restart.
- socket request/reply for every clipboard op.
- atomic write survives a simulated crash.
- no network syscalls in the binaries (`strace -f -e trace=network`).

Acceptance tests `T1`–`T24` are in `Testing.md`.
