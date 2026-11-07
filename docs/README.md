# Sivnox documentation

**Sivnox is a GPL-3.0, Hyprland-only desktop shell for a laptop daily driver.**
The shell is Quickshell/QML. The Rust surface is deliberately small: one clipboard
daemon, a handful of one-shot CLIs, and the Hyprland config glue.

This directory is the design and implementation contract. Start here, then read
the documents in the order below.

---

## Source of truth

When two documents disagree, the higher row wins:

| Priority | Document | What it settles |
|---|---|---|
| 1 | **`Build_Sheet-Filled.md`** | Locked decisions (`L1`–`L23`), required surfaces (`R1`–`R52`), build order (`W0`–`W4`), acceptance tests (`T1`–`T24`), open items (`P1`–`P7`). |
| 2 | **`SCOPE.md`** | What v0.1 is and is not. |
| 3 | `Frontend.md`, `Hyprland.md`, `Theme.md`, `Clipboard.md`, `Interfaces.md`, `Config.md`, `Provenance.md` | Implementation and compliance contracts derived from the Build Sheet. |
| 4 | `current_plan.md` | Execution order and per-week deliverables. |
| 5 | `Constitution.md` | Identity and engineering laws. Amended to match the Build Sheet. |
| 6 | `Architecture Review.md` | Decision log: how the old architecture was resolved, and what risks remain. |
| 7 | `Backend.md`, `Backend Features.md` | Implementation plan and feature inventory for the small Rust surface. |
| 8 | `Testing.md` | How every acceptance test is measured. |

`../reference_docs/` is **not** a Sivnox specification. It is Ryoku reference
material, kept for borrowing patterns. Do not edit it to match Sivnox.

If a document becomes wrong, fix it in the same change that makes it wrong. A
silent contradiction is a bug.

---

## Reading order

### For a new contributor

1. `Build_Sheet-Filled.md` — the locked product.
2. `SCOPE.md` — the boundary.
3. `Constitution.md` — why the shell looks and behaves the way it does.
4. `Frontend.md` — how the QML shell is organised.
5. `Architecture Review.md` — why the shape changed.

### Before writing code

1. `current_plan.md` — W0 tasks and open items.
2. `Hyprland.md` — the managed Lua tree and override layers.
3. `Frontend.md` — surfaces, loaders, config, wallpaper, notifications.
4. `Theme.md` — token set, extractor, fan-out.
5. `Clipboard.md` — the only long-lived Rust daemon.
6. `Interfaces.md` — CLI, socket, files, versioning.
7. `Config.md` — TOML schema and state files.
8. `Provenance.md` — copied-file register and GPL notices.
9. `Testing.md` — the gate.

---

## Document map

| File | Status | Purpose |
|---|---|---|
| `Build_Sheet-Filled.md` | **canonical** | Locked decisions and v0.1 plan. Do not rewrite without a decision. |
| `SCOPE.md` | active | In-scope and out-of-scope for v0.1. |
| `Constitution.md` | active | Identity, architecture, state, efficiency, and sovereignty laws. |
| `Architecture Review.md` | active | Decision log and remaining risk register. |
| `current_plan.md` | active | W0–W4 execution plan and pre-coding checklist. |
| `Frontend.md` | active | Quickshell/QML architecture and surface contracts. |
| `Hyprland.md` | active | Caelestia Lua tree integration and override layers. |
| `Theme.md` | active | Frozen tokens, extractor, cache, fan-out, post-hook. |
| `Clipboard.md` | active | `sivnoxd` clipboard daemon and picker contract. |
| `Interfaces.md` | active | CLI commands, clipboard socket, state files, versioning. |
| `Config.md` | active | TOML configuration schema, defaults, validation, watchers. |
| `Provenance.md` | active | Copied-file register, GPL notices, dependency audit. |
| `Backend.md` | active | Rust component plan and crate layout. |
| `Backend Features.md` | active | v0.1 buildable feature inventory. |
| `Testing.md` | active | Acceptance test procedures and performance measurement. |

---

## Non-negotiables

- **Hyprland only.** No niri, no Wayfire, no multi-compositor abstraction in v0.1.
- **Quickshell owns the shell.** Bar, launcher, lock, OSD, notifications, session,
  screenshot, clipboard picker, wallpaper rendering.
- **Quickshell owns wallpaper state.** Preview and commit live in the frontend.
  There is no wallpaper daemon and no `swww`.
- **Quickshell owns protocol feeds.** Notifications, UPower, NetworkManager,
  BlueZ, PipeWire, MPRIS, tray, clock, workspaces.
- **One long-lived Rust daemon.** `sivnoxd` owns clipboard history, pins, and
  eviction. Nothing else is a daemon.
- **Everything else Rust is one-shot.** `sivnox colours`, theme fan-out,
  thumbnails, first frames, config parsing, frontend IPC wrappers.
- **`systemd --user` supervises the session.** `sivnox-shell.service` and
  `sivnoxd.service`.
- **GPL-3.0.** Caelestia and Ryoku are both GPL-3.0. Ported code keeps its
  notices and the resulting source stays available.
- **No network, no telemetry, no root.** The shell is a user process and its
  files are the user's.

---

## Conventions for these documents

- Use the Build Sheet IDs (`L*`, `R*`, `W*`, `T*`, `P*`) when referring to a
  decision. It keeps the paper trail short.
- Prefer tables over prose for decisions and paths.
- Mark assumptions explicitly. If a detail is not in the Build Sheet, it belongs
  in `Interfaces.md` or `Config.md` as an implementation choice, not smuggled
  into the Build Sheet.
- No document claims a performance win without a measurement procedure in
  `Testing.md`.
