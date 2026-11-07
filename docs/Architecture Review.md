# Sivnox — architecture review and decision log

**Status:** active decision record. Superseded concerns are closed by
`Build_Sheet-Filled.md`; remaining risks are tracked here.
**Scope:** Hyprland-only shell. v0.1 is bar + launcher + notifications + OSD +
lock/session + wallpaper + clipboard + screenshot + theme fan-out.

This document used to be a critique of the daemon-centric v1 design. That design
is gone. What follows is:

1. how each old blocker was resolved,
2. the architecture that replaced it,
3. the risks that remain before coding, and
4. the mitigations.

Where this document disagrees with `Build_Sheet-Filled.md`, the Build Sheet
wins.

---

## 1. The old design in one paragraph

The v1 plan had one Rust control-plane daemon owning wallpaper state, a preview
lease, scheme extraction, config, lazy services, and a custom three-primitive
IPC bus. Quickshell was a subscriber. The review found that this centralized too
much, duplicated protocol feeds, and contradicted itself on preview leases,
frontend view state, notifications, and compositor dispatch.

The Build Sheet replaces it with a **frontend-first** design: Quickshell owns the
shell and the wallpaper, one small Rust daemon owns clipboard, and one-shot
Rust tools do bounded jobs.

---

## 2. Closed blockers

| # | Old blocker | Resolution | Locked by |
|---|---|---|---|
| 1 | Preview lease vs. the 5 ms client: a short-lived CLI preview self-cancelled on disconnect. | There is no lease daemon. The frontend owns preview/commit. CLI calls are short-lived and cannot hold a preview. | L12, R25, R26 |
| 2 | "The frontend owns nothing" is false for a GUI: hover, focus, scroll, animation, and open surfaces are view state. | The frontend owns view state **and** wallpaper transaction state. Rust owns only clipboard. | L12 |
| 3 | Notifications cannot be lazy: they must own `org.freedesktop.Notifications` continuously. | Quickshell's `NotificationServer` owns the name. No Rust notification service. | L11 |
| 4 | `wm.dispatch` violated the no-external-process law by spawning `hyprctl` per action. | There is no daemon dispatch path. Keybinds live in Hyprland Lua and call Sivnox/frontend commands. | L23, R51 |
| 5 | No surface-request path: keybind → daemon → ??? | Keybind → `sivnox open <surface>` → Quickshell IPC. No daemon in the middle. | R51, T23 |
| 6 | QML cannot compile against a Rust `sivnox-proto` crate. | There is no Rust wire crate for QML. The contract is the CLI, the clipboard socket, and the file schemas. | L21 |
| 7 | The notification topic contradicted the bounded-queue law. | Quickshell owns notifications end to end; no custom topic. | L11 |
| 8 | Duplicating Quickshell's services in Rust. | Quickshell owns notifications, UPower, NetworkManager, BlueZ, PipeWire, MPRIS, tray, clock, and Hyprland state. | L11, L21 |
| 9 | The clock as a 1 Hz IPC topic. | The clock is local QML. No `clock.tick`. | L4 |
| 10 | Single daemon = single failure domain. | The only daemon is clipboard. A clipboard crash cannot take down the bar or notifications. The frontend is supervised by `systemd`. | L2, L21 |
| 11 | Single Quickshell process = visual failure domain. | `systemd` restarts `sivnox-shell.service`; wallpaper and config are on disk, so the bar can re-render. | L2, T11 |
| 12 | Video wallpapers have no renderer. | Video playback is out of scope. Only first-frame preview via `ffmpeg`. | R28, O-list |
| 13 | "No full-resolution bitmap" is false as specified. | Extractor decodes to 64×64; thumbnails are 128×128; frames are scaled by `ffmpeg`. Quickshell, which owns pixels, may render at display resolution. | R27, R28, R29 |
| 14 | No frontend performance budget. | v0.1 has explicit budgets: T1–T6. | T1–T6 |
| 15 | Materialize is overbuilt for a shell. | No materialize framework in v0.1. Managed Hyprland files are rewritten at install/update; config is sparse. | L23, Config |
| 16 | No supervisor/startup contract. | `systemd --user` units for the shell and clipboard daemon; wallpaper and config are reloaded from disk. | L2, P5 |
| 17 | Multi-monitor state is missing. | The frontend model is per-monitor from day one; v0.1 tests one monitor. | L19 |
| 18 | Cache key is not a content hash. | Caches are keyed by content hash + version. | R30 |
| 19 | Atomic is not durable. | Temp + rename + `fsync` file and directory; SQLite WAL. | Constitution 3.3 |
| 20 | Theme contradiction: tokens opaque vs. a full `siv_*` engine. | The token set is frozen in `Theme.md`; the extractor emits it; the frontend applies it; fan-out renders templates. | R29–R32 |
| 21 | Accessibility is absent. | 4.5:1 contrast enforced; focus/selection visible without colour; keyboard navigation on surfaces. | Theme |
| 22 | No QML tests or visual regression. | `qmllint`, IPC probes, and the acceptance procedure in `Testing.md`. | Testing |
| 23 | Transient surfaces violate the "nothing floats" identity. | Transient surfaces are an explicit exception: notifications, OSD, launcher, and popups are anchored to a screen edge or the bar. | Constitution 1.2 |
| 24 | Version churn (Quickshell, Hyprland IPC). | No version pins in v0.1; use distro defaults. Pin later if drift breaks builds. | Build Sheet §7 |
| 25 | Missing app/session action verbs. | `sivnox open`, `sivnox osd`, `sivnox lock`, `sivnox session`, `sivnox screenshot`. | Interfaces |
| 26 | Backend-first with no visual feedback for months. | W0 builds the Hyprland config and first pixels; Rust starts at W2. | W0–W4 |
| 27 | Token fan-out precedes frontend token design. | `Theme.md` freezes the token schema before the fan-out port. | Theme |
| 28 | Notifications are in the MVP but are the hardest service. | Quickshell's `NotificationServer` is used in W1; no Rust D-Bus implementation. | L11, W1 |
| 29 | No per-monitor phase. | Per-monitor fields exist in the model from day one. | L19 |

### One Build Sheet clarification

Build Sheet §9 introduces the state table as "daemon-written", but `L12` makes
the frontend the owner of wallpaper state and `R25` says the commit writes the
path file. The two are reconciled as follows:

- `wallpaper.json` is written by Quickshell on commit, because the frontend owns
  the wallpaper transaction.
- Every other state file has exactly one writer (`sivnoxd` for the clipboard
  database, the one-shot tools for their caches).
- The user is the only writer of `config.toml` and the Hyprland override files.

This is a wording correction, not a new decision. It keeps the "one writer per
file" rule intact.

---

## 3. The replacement architecture

```
Hyprland (Lua config, keybinds, layer rules)
      │
      ├── keybind ──> sivnox open <surface> ──> Quickshell IPC
      │
      └── layer-shell surfaces
              │
        qs -c sivnox  (sivnox-shell.service)
        ├── bar
        ├── launcher
        ├── notifications (NotificationServer)
        ├── OSD
        ├── session menu
        ├── lock screen (WlSessionLock + PAM)
        ├── wallpaper (layer-shell, preview/commit)
        ├── clipboard picker ──> clipboard.sock ──> sivnoxd
        └── screenshot overlay

one-shot Rust
├── sivnox colours <path>      → colour cache + tokens
├── sivnox theme apply <path>  → templates + app configs + post-hook
├── sivnox thumb <path>        → 128×128 cache
├── sivnox frame <path>        → ffmpeg first-frame cache
├── sivnox config dump|check   → TOML → JSON
└── sivnox doctor              → dependency/config check

external tools
├── grim, wl-copy/wl-paste
├── hypridle
├── brightnessctl, hyprsunset
├── ffmpeg
└── power-profiles-daemon
```

### Why this shape

- **The shell is the product.** Keeping the surfaces in one QML process gives
  one look, one motion language, and one place to apply tokens.
- **Quickshell already owns the protocols.** Notifications, UPower,
  NetworkManager, BlueZ, PipeWire, MPRIS, tray, and Hyprland state are provided
  by Quickshell modules. Reimplementing them in Rust would add code, latency,
  and duplicate state.
- **Wallpaper is a view.** Quickshell draws it on a layer-shell surface, so
  preview and commit are simply frontend state plus one atomic file write.
- **Rust is used where it buys something.** Clipboard history needs a database
  and a long-lived watcher. Colour extraction, thumbnails, first frames, and
  template fan-out are bounded CPU/IO jobs that should not live in the QML
  process.
- **`systemd --user` supervises.** A crash restarts the process; the shell does
  not need a custom supervisor.

---

## 4. Remaining risks

### Risk 1 — Caelestia Hyprland path drift (medium)

`P6` and `P7` are open until W0. The Build Sheet assumes Caelestia ships
`hypr/hyprland.lua` + `hypr/modules/`. If the upstream tree moves or uses
different command names, the copy and rewrite checklist changes.

**Mitigation:** verify the exact tree at the start of W0 (`P6`), keep the rewrite
list in `Hyprland.md`, and make `T21`–`T24` the gate. A search for `caelestia` in
the managed tree must be clean.

### Risk 2 — Caelestia theme fan-out is the largest port (medium-high)

`L15` and `R32` require a Rust port of the Caelestia engine (~600–800 lines).
The exact adapter list is `P4`. Template rendering is easy; app-specific
adapters, atomic writes, and post-hook behaviour are where the bugs will be.

**Mitigation:** freeze the token schema first (`Theme.md`). Port the renderer and
one adapter, then add adapters one at a time. `T18` requires at least kitty and
GTK on commit. Fan-out is one-shot, so a failure cannot hang the shell.

### Risk 3 — Clipboard SQLite and watcher lifecycle (medium)

The clipboard daemon is the only long-lived Rust process. It must supervise
`wl-paste`, deduplicate, evict, and serve a picker without losing pins or
corrupting the database.

**Mitigation:** WAL mode, one connection, content-hash dedup, `busy_timeout`,
and integration tests for restart/eviction. The daemon starts on first picker
use via `sivnoxd.socket`; if that unit is rejected, start it from the picker and
document the change in `P5`.

### Risk 4 — Quickshell memory and cold-open (medium)

`T2` (<60 MB bar) and `T3` (<90 MB bar + launcher) are the real budgets. One
eager singleton can blow them.

**Mitigation:** every closed surface behind `Loader`, no eager service
construction, thumbnails instead of full images in grids, and a baseline
measurement before optimisation. If `T5` fails, `L10` triggers the Rust launcher
alternative.

### Risk 5 — Wallpaper rendering memory (medium)

Quickshell draws the wallpaper at display resolution. A 4K image plus a preview
plus thumbnails can exceed the budget if handled carelessly.

**Mitigation:** one displayed image per monitor, preview uses the same decoded
surface, thumbnails only in the picker, and frames only for the focused item.
`T2`/`T3` are measured with the picker closed; add a picker-open measurement to
`Testing.md` as a warning, not a gate.

### Risk 6 — TOML parsing in QML (low-medium)

Build Sheet locks TOML for user config but does not specify the parser. A second
TOML implementation in JavaScript would be a maintenance and correctness risk.

**Mitigation:** `Config.qml` watches the file and calls `sivnox config dump`,
which uses the Rust `toml` crate and returns normalized JSON. One parser, one
set of validation rules. Invalid TOML becomes `.broken` + defaults.

### Risk 7 — Shell restart and state recovery (medium)

Systemd restarts the shell after a crash. The bar must re-render, wallpaper must
reappear, and config must reload.

**Mitigation:** wallpaper commit is atomic; `Config.qml` reads on start; the bar
is rebuilt from Quickshell services. `T10` and `T11` are the gate.

### Risk 8 — License and provenance (high if ignored)

Caelestia and Ryoku are GPL-3.0. Copying their QML, Lua, or Go code makes Sivnox
GPL-3.0 (which `L1` accepts). Missing notices would still be a compliance bug.

**Mitigation:** keep a `NOTICE`/header on every copied file, record the source
and commit in `Provenance.md`, and ship the complete corresponding source.

### Risk 9 — Multi-monitor retrofit (low for v0.1, high later)

`L19` requires per-monitor fields now, but v0.1 only tests one monitor. A
singleton with a hardcoded output would be expensive to undo.

**Mitigation:** make monitor a parameter in surface requests, bar models, and
wallpaper state; test one monitor; add a second-monitor test post-v0.1.

---

## 5. What we deliberately accept

- **One QML process is a visual failure domain.** Systemd restart and on-disk
  state are the mitigation. Splitting surfaces into separate processes is not
  worth the memory in v0.1.
- **One Rust daemon is a clipboard failure domain.** The rest of the shell keeps
  working if it dies.
- **No settings GUI.** TOML is the interface in v0.1. This is a deliberate
  trade: fewer surfaces, faster delivery.
- **No materialize/doctor framework.** Managed Hyprland files are replaced at
  update; user files are never touched. `sivnox doctor` is a dependency check,
  not a reconciler.
- **Video wallpaper is deferred.** First-frame preview is enough for the picker.

---

## 6. What Sivnox borrows

| Source | Borrow | Do not borrow |
|---|---|---|
| Caelestia | Bar, launcher, lock, session menu, Hyprland Lua tree, theme fan-out engine, project structure | Its Python runtime, its config format assumptions |
| Ryoku | Clipboard behaviour, screenshot overlay, one-concern-per-file discipline | Its Go daemon, its 20-service sprawl, its installer |
| Quickshell | NotificationServer, UPower, NetworkManager, BlueZ, PipeWire, MPRIS, tray, layer-shell | — |
| Waybar/mako | Graceful degradation: the bar keeps rendering last-known state | Their per-tool config sprawl |

---

## 7. Verdict

The architecture is now small enough to build:

- **Frontend-first.** W0 is Hyprland config and first pixels, not a daemon.
- **One daemon, one job.** Clipboard only.
- **One-shot Rust.** Colour, fan-out, assets, config, wrappers.
- **Quickshell owns the protocols it already implements.**
- **The Build Sheet is the contract.** The remaining work is execution and
  closing `P1`–`P7`.

The risk has moved from "we are building the wrong control plane" to "we are
porting Caelestia and Ryoku carefully, and we must hit the frontend memory
budgets." That is a better risk to have.
