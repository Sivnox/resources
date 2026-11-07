# SIVNOX — THE LAWS

**A constitution for a GPL-3.0, Hyprland-only desktop shell.**
**Version 2 — amended against `Build_Sheet-Filled.md`.**

This document is the *why*. `Build_Sheet-Filled.md` is the *what*. When they
disagree, the Build Sheet wins, and this document is wrong until amended.

**The rule for amending a law:** a law may be amended only when building against
it has produced a concrete, demonstrated problem — not a hypothetical one. "It
would be easier if…" is not grounds. "The shell crashed because of it" is.

**The test for any decision:** does it uphold every law below? If it violates
one, it is not Sivnox — no matter how convenient.

---

## Article 0 — How to use this document

- Read this before adding a surface, a daemon, a file, or a dependency.
- Cite the law in the change description when a decision is non-obvious.
- The veto list in Article XI is the first line of defence.
- A repeated violation is grounds to revisit the law, not to ignore it.

---

## Article I — Identity

### Law 1.1 — Sivnox is the grey night, inhabited, not performed.

Sivnox is the hour that is neither dark nor light. It is still, aware, warm, and
composed. It does not glow, pulse, breathe, or announce itself. It sits between
opposites and is at peace there.

**Enforcement:** any feature whose *purpose* is to be noticed violates this. Any
accent that competes for attention violates this. Any animation that exists for
delight rather than feedback violates this.

### Law 1.2 — Nothing competes for attention. Everything is placed, not floated.

No element hovers, glows, or demands focus. Surfaces rest on surfaces. Hierarchy
is *nearness*, not loudness. The shell assumes a person who can see.

**Enforcement:** if a surface has no anchor (edge, bar, neighbour), it does not
exist. If a colour exists to be noticed rather than to signal, it is removed.
The explicit exception is the transient surface: notifications, OSD, launcher,
popups, and the lock screen are anchored to a screen edge, the bar, or the
session state they interrupt. They do not float free of context.

### Law 1.3 — Stillness is the default. Motion is an event.

The shell is composed, not animated. Transitions are swift, precise, and final.
The clock ticks; nothing else moves without cause.

**Enforcement:** every animation must have a state change behind it. Idle motion
is a bug.

### Law 1.4 — The signal is small.

One accent, used only for state and active elements. Restraint is not the absence
of signal — it is the *scarcity* of it, which is what makes it signal at all.

**Enforcement:** if two elements in the same view share the accent colour, one is
wrong. The accent is never decorative.

### Law 1.5 — Light is borrowed, never made.

Nothing in Sivnox emits light. It reflects. Colours come from the wallpaper, the
hour, and the user — Sivnox presents them in ink and bone. It is a surface, not a
source.

**Enforcement:** no glow, no bloom, no ambient colour that isn't derived from a
source. If it can't be traced to the wallpaper or the palette, it isn't there.

### Law 1.6 — The grey night is a guide, not a gate.

`L20` is explicit: function may override the aesthetic. A control that is
unreadable, an animation that hides a state change, or a surface that cannot be
used is a defect even if it is "on brand".

**Enforcement:** when function overrides the identity, say so in the change
description and record the reason. Do not use this law as a shortcut for
decoration.

---

## Article II — Shape

### Law 2.1 — Quickshell is the shell.

One Quickshell process, started as `qs -c sivnox` under `sivnox-shell.service`.
It renders every visible surface: bar, launcher, notifications, OSD, session
menu, lock screen, wallpaper, clipboard picker, and screenshot overlay.

**Enforcement:** a surface that runs in a second QML process needs a written
exception. v0.1 has none.

### Law 2.2 — The frontend owns the view and the wallpaper.

`L12` is explicit. Quickshell owns wallpaper preview and commit, and it writes
`~/.config/sivnox/wallpaper.json` on commit. It also owns all ephemeral view
state: hover, focus, scroll, animation, which surface is open.

**Enforcement:** no wallpaper daemon, no `swww`, no `hyprpaper`, no `mpvpaper`.
No Rust component reads or writes wallpaper state except as a path or a
thumbnail.

### Law 2.3 — There is exactly one long-lived Rust daemon.

`sivnoxd` owns clipboard history, pins, eviction, and the clipboard database. It
is the only daemon we write.

**Enforcement:** a second long-lived process is a veto. A one-shot CLI is not a
daemon. If a feature needs continuous work, first ask whether Quickshell already
provides it.

### Law 2.4 — Everything else Rust is one-shot.

`L21` limits the Rust footprint to the clipboard daemon, the colour extractor,
the CLI, theme fan-out, and Hyprland config glue. One-shot tools are allowed to
be processes because they run on a state change, not on every frame or
keypress.

**Enforcement:** no per-frame or per-hover process spawn. A one-shot command may
run on commit, on first pick, on config change, or on an explicit keybind.

### Law 2.5 — Quickshell owns the protocol feeds.

Notifications, UPower, NetworkManager, BlueZ, PipeWire, MPRIS, tray, clock, and
Hyprland workspace state are read by Quickshell from the services that already
own them. `L11` locks Quickshell's `NotificationServer` as the notification
owner.

**Enforcement:** do not add `zbus`, `notify-rust`, or a Rust network/audio
monitor. Reimplementing a protocol feed is a veto.

### Law 2.6 — External tools stay external.

`grim`, `wl-copy`, `wl-paste`, `hypridle`, `brightnessctl`, `hyprsunset`,
`ffmpeg`, and `power-profiles-daemon` keep their jobs (`L22`). Sivnox orchestrates;
it does not absorb them.

**Enforcement:** if a tool already owns the pixels or the hardware, call it and
move on. Do not reimplement a screenshot codec or an idle inhibitor.

### Law 2.7 — `systemd --user` supervises the session.

`L2` is explicit. `sivnox-shell.service` runs the frontend; `sivnoxd.service`
runs the clipboard daemon. A crash is recovered by restart, not by a custom
supervisor inside the shell.

**Enforcement:** no system-level unit, no root, no setuid helper. If the shell
dies, `systemd` restarts it with backoff.

---

## Article III — State

### Law 3.1 — One writer per file.

Every state file has exactly one writer. Readers do not rewrite.

| File | Writer |
|---|---|
| `~/.config/sivnox/config.toml` | the user |
| `~/.config/sivnox/wallpaper.json` | Quickshell on commit |
| `~/.local/share/sivnox/clipboard.db` | `sivnoxd` |
| `~/.cache/sivnox/colours/*.json` | `sivnox colours` |
| `~/.cache/sivnox/thumbs/*` | `sivnox thumb` |
| `~/.cache/sivnox/frames/*` | `sivnox frame` |
| `~/.config/hypr/hyprland.lua` + `modules/` | Sivnox install/update |

**Enforcement:** any read-modify-write in a different component is a bug.

### Law 3.2 — State is small and typed. Surfaces are pure functions of it.

State holds paths, scalars, small enums, and text tokens — never images, decoded
buffers, or open handles. Quickshell renders from state; it does not invent
authoritative state that no file or service knows about.

**Enforcement:** a state struct or QML property that holds a full-resolution
bitmap is a violation.

### Law 3.3 — Writes are atomic and durable.

Config, wallpaper, caches, and the clipboard database are written with
temp-file-plus-rename in the same directory, followed by `fsync` of the file and
the directory. SQLite uses WAL.

**Enforcement:** a crash mid-write leaves the previous valid file, not a
half-written one.

### Law 3.4 — Config is user-owned TOML.

`~/.config/sivnox/config.toml` is sparse: an omitted key uses the compiled
default. The user is the only writer. Invalid TOML is renamed to `.broken`, the
error is logged, and the shell runs on defaults.

**Enforcement:** no daemon writes config. No config format requires a special
tool to read.

### Law 3.5 — Every cache is bounded.

Colour cache: 5 MB / 200 entries. Thumbnail cache: 50 MB / 1000 entries. Frame
cache: 50 MB / 1000 entries. Clipboard: 100 entries / 1 MB by default, both
configurable.

**Enforcement:** an unbounded `Vec`, `HashMap`, or directory is a bug. Eviction
is oldest-first; pinned clipboard entries never evict.

### Law 3.6 — Caches are content-addressed.

A cache key includes the content hash and the extractor or thumbnail version. A
same-size, same-mtime edit must not return a stale result.

**Enforcement:** if a cache key is only `(path, mtime, size)`, it is wrong.

### Law 3.7 — No full-resolution bitmap in Rust memory.

Colour extraction decodes to 64×64 in one pass and discards the full image.
Thumbnails are 128×128. Video frames are extracted at display-thumbnail size via
`ffmpeg`, on demand for the focused item only.

**Enforcement:** if a full bitmap enters a Rust process, it is a bug. Quickshell
may render the wallpaper at display resolution because it owns pixels; Rust may
not.

---

## Article IV — The frontend

### Law 4.1 — Every surface lives behind a `Loader`.

`L5` is explicit. The bar is always active, but it is still a loaded surface.
Launcher, notifications, OSD, session, lock, wallpaper picker, clipboard picker,
and screenshot are `active: false` until their state says otherwise.

**Enforcement:** a surface that can be closed but is still instantiated is a
bug. `Loader` boundaries are the first thing code review checks.

### Law 4.2 — Follow the Caelestia project structure.

`L4` locks Quickshell/QML with the Caelestia project structure. Keep its modules,
services, components, and utilities in the same shape. Edit the surfaces we need;
do not invent a parallel shell layout.

**Enforcement:** a new top-level directory needs a reason. Caelestia conventions
win unless they conflict with this document.

### Law 4.3 — Per-monitor from day one.

`L19` requires the model and protocol shape to be per-monitor even though v0.1
is tested on one monitor. A surface request carries a monitor; a bar has a
monitor; a wallpaper has a monitor.

**Enforcement:** a singleton that assumes `monitor 0` is a bug. Do not add
multi-monitor rendering to v0.1, but do not design it out.

### Law 4.4 — Wallpaper is a frontend transaction.

Preview changes the displayed wallpaper without touching disk. Commit writes
`wallpaper.json` atomically. Cancel restores the persisted path. A shell restart
reads the persisted file.

**Enforcement:** only commit writes the file. Preview and cancel never do.

### Law 4.5 — Notifications belong to Quickshell.

`L11` locks Quickshell's `NotificationServer` as the owner of
`org.freedesktop.Notifications`. Popups, history, DND, and actions render from
that server.

**Enforcement:** no Rust notification server. No duplicate D-Bus name owner.

### Law 4.6 — Lock and session are frontend surfaces.

The lock screen uses `WlSessionLock` and PAM. The session menu offers lock,
suspend, reboot, and shutdown. Idle and lid behaviour are delegated to
`hypridle`.

**Enforcement:** no ad-hoc `systemctl` or `loginctl` spawn from a QML binding on
every frame. Session actions are explicit commands.

### Law 4.7 — Theme reaches the shell through `Theme.qml`.

The shell reads the frozen token set and applies it through one adapter,
`Theme.qml`. Surfaces read that adapter, never a raw cache file.

**Enforcement:** a hardcoded colour outside the token adapter is a bug.

---

## Article V — Theme

### Law 5.1 — The token key set is frozen.

The v1 keys are `siv_ink`, `siv_bone`, `siv_grey`, `siv_signal`, `siv_ink_dim`,
`siv_ink_soft`, `siv_bone_dim`, `siv_grey_deep`, and `siv_signal_hover`.

**Enforcement:** adding or renaming a key is a schema change and requires a
version bump plus a migration note.

### Law 5.2 — The extractor is a one-shot service, not an owner.

`sivnox colours <path>` computes tokens and writes a cache entry. It does not
apply a theme, start a daemon, or hold state beyond the cache.

**Enforcement:** the extractor is a pure function of `(path, mode, extractor_ver)`.
If it needs more, it reopens the path; it never keeps the bitmap.

### Law 5.3 — Fan-out runs on commit only.

Preview does not rewrite app configs. Commit extracts (or reuses cached tokens),
renders templates, writes app configs, then runs the optional post-hook.

**Enforcement:** a template write on preview is a bug. Fan-out is one-shot and
must report `{ ok, failed }`.

### Law 5.4 — Templates are plain key/value substitution.

Templates live in `/usr/share/sivnox/templates/` and use `{{ key }}`. The app
list comes from `theme.apps`.

**Enforcement:** no app-specific logic in the shell. An adapter is a template
path plus a target path.

### Law 5.5 — The post-hook is optional and bounded.

The default path is `~/.config/sivnox/hooks/post-theme`. It receives the tokens
in the environment, has a timeout, and its output is logged. An empty path
disables it.

**Enforcement:** a hanging hook must not hang the shell; it runs in the one-shot
fan-out process.

### Law 5.6 — Contrast is not optional.

The token derivation enforces at least 4.5:1 for text on its surface. The
accent is signal, not decoration. Focus and selection remain visible without
colour alone.

**Enforcement:** a token set that fails contrast is rejected and the named-scheme
fallback is used.

---

## Article VI — Hyprland

### Law 6.1 — The Hyprland config is Caelestia's Lua tree, managed.

`L23` is explicit. `~/.config/hypr/hyprland.lua` and `~/.config/hypr/modules/`
are managed by Sivnox and overwritten on update.

**Enforcement:** a user edit to a managed file may be lost. User changes belong
in the override layers.

### Law 6.2 — There are three config layers.

| Layer | Path | Owner |
|---|---|---|
| Core | `~/.config/hypr/hyprland.lua` + `modules/` | Sivnox |
| Variables | `~/.config/sivnox/hypr-vars.lua` | the user |
| User | `~/.config/sivnox/hypr-user.lua` | the user |

**Enforcement:** a managed file that reads a user file by an undocumented path is
a bug. The override contract is documented in `Hyprland.md`.

### Law 6.3 — Caelestia paths and commands are rewritten.

Every `~/.config/caelestia/` reference becomes `~/.config/sivnox/`, and every
Caelestia command becomes its Sivnox equivalent. No Caelestia namespace survives
in the shipped tree.

**Enforcement:** the W0 copy is not done until a search for `caelestia` in the
managed tree is clean.

### Law 6.4 — `hyprconf2lua` and `hyprvalidate` are development tools.

They are not runtime dependencies and are not shipped in v0.1.

**Enforcement:** the shell never calls them at runtime.

### Law 6.5 — The bar reserves its space.

A layer rule reserves space for the bar so windows do not sit under it.

**Enforcement:** `T24` is the test.

---

## Article VII — Efficiency

### Law 7.1 — The budgets are the product.

v0.1 targets (`T1`–`T6`): login under 5 s, bar under 60 MB, bar + launcher under
90 MB, idle CPU under 1%, launcher cold-open under 100 ms, colour apply under
200 ms.

**Enforcement:** a change that regresses a budget needs a measurement and a
decision, not an opinion.

### Law 7.2 — Nothing starts until it is needed.

Surfaces are behind loaders. Services are Quickshell-native and lazy, with one
carve-out: the notification server owns `org.freedesktop.Notifications`
continuously (Law 4.5). The clipboard daemon starts on first picker use (socket
activation). Colour and fan-out are one-shot.

**Enforcement:** a surface or daemon that starts at boot without a reason is a
bug. The notification server is the only always-on service, and its reason is
the protocol name.

### Law 7.3 — Efficiency is measured, not asserted.

Every claim comes from the procedure in `Testing.md`. No "feels lighter".

**Enforcement:** a performance claim without a number is rejected.

### Law 7.4 — Optimise Quickshell before Rust.

The frontend dominates memory and frame time. Rust is small by design (`L21`).
Do not spend days shaving bytes off a 10 MB CLI while the shell is over budget.

**Enforcement:** the baseline table decides where effort goes.

### Law 7.5 — The Rust footprint stays small.

Clipboard daemon, colour extractor, CLI, theme fan-out, thumbnails, first
frames, config parsing, and Hyprland glue. Nothing else.

**Enforcement:** a new Rust component needs a written exception and a line in
`Backend.md`.

---

## Article VIII — Control

### Law 8.1 — If a behaviour exists, it has a command.

Every action the shell can take is reachable through a keybind, a surface, or
`sivnox <command>`. No hidden actions.

**Enforcement:** `sivnox --help` lists every command. A command that does not
appear does not exist.

### Law 8.2 — The user can see and cap the resources.

Config exposes clipboard limits and theme app list. Caches are bounded. The
clipboard database is inspectable with ordinary SQLite tools.

**Enforcement:** a new cache or queue ships with a bound and a config key.

### Law 8.3 — Config is single-writer, live-watched, validated.

`Config.qml` watches `config.toml`. One parser validates it. Invalid input is
renamed to `.broken`, logged, and defaults are used.

**Enforcement:** no second config reader with different rules.

### Law 8.4 — State is plain and local.

Config is TOML, wallpaper is JSON, clipboard is SQLite, caches are JSON/images.
All are in the user's home, readable without special tools.

**Enforcement:** no binary blob, no database the user cannot inspect, no
`/etc/sivnox`.

### Law 8.5 — The source is GPL-3.0.

Sivnox is GPL-3.0 because it ports Caelestia and Ryoku code. Ported files keep
their copyright notices. The complete corresponding source is available.

**Enforcement:** a copied file without a notice is a bug.

---

## Article IX — Sovereignty

### Law 9.1 — The shell runs as the user, in the user's session.

`sivnox-shell.service` and `sivnoxd.service` are `systemd --user` units. No root,
no setuid, no system unit, no privileged helper.

**Enforcement:** the binaries refuse to run as root. Packaging ships user units
only.

### Law 9.2 — Nothing phones home. Ever.

No telemetry, analytics, crash reporting, update checks, licence pings, or
network calls. The only network activity is the user's own apps and the system
services they use.

**Enforcement:** the Rust binaries link no HTTP client. CI checks for network
capability.

### Law 9.3 — No locks, gates, or tiers.

The whole source is the whole product. No feature flag hides behaviour, no
"pro" tier, no activation.

**Enforcement:** a licence or tier check on a behaviour path is a veto.

### Law 9.4 — The user owns the files.

Config, state, cache, and database live in the user's home and are user-owned.
`/usr/share` ships defaults and templates only.

**Enforcement:** no symlink from the user's config into `/usr/share`.

### Law 9.5 — The user can replace any component.

The frontend can be replaced by another Quickshell tree. The colour extractor
can be replaced by another command that writes the same token shape. The
clipboard daemon can be replaced by another process that speaks the same socket.

**Enforcement:** the contracts in `Interfaces.md` are the boundary. A component
that reaches around them is a bug.

---

## Article X — Process

### Law 10.1 — Build in the order of the Build Sheet.

`W0` through `W4` in `Build_Sheet-Filled.md`. No week starts until the previous
week's exit test passes.

**Enforcement:** a PR that adds a W2 surface during W0 does not merge.

### Law 10.2 — Every acceptance test has a procedure.

`T1`–`T24` are defined in `Testing.md`. A test that cannot be reproduced is not
a test.

**Enforcement:** a "pass" without a procedure is not a pass.

### Law 10.3 — Docs change with the code.

If an implementation choice is not in the Build Sheet, record it in the
appropriate contract document before or with the change.

**Enforcement:** a PR that leaves a contradiction behind is rejected.

### Law 10.4 — Port with notices, not with enthusiasm.

Copy the minimum from Caelestia and Ryoku. Keep GPL notices. Do not fork their
architecture wholesale.

**Enforcement:** a ported file is reviewed for scope and licence header.

### Law 10.5 — No new daemon without a decision.

The only daemon is `sivnoxd`. A new daemon needs a measured reason and a Build
Sheet amendment.

**Enforcement:** the veto list applies.

### Law 10.6 — No unmeasured efficiency claim.

Show the number and the command that produced it.

**Enforcement:** a performance claim in a commit message includes the measurement.

---

## Article XI — The Veto

The following are always rejected, regardless of convenience:

1. A second long-lived daemon beyond `sivnoxd`.
2. A wallpaper daemon, or `swww`/`hyprpaper`/`mpvpaper` in the runtime.
3. A general control-plane IPC bus for surfaces, wallpaper, or services.
4. Reimplementing a protocol feed Quickshell already provides.
5. A Rust notification server.
6. A full-resolution bitmap held in Rust memory.
7. A second writer to a state file.
8. An unbounded cache, queue, or database.
9. A network call, telemetry, or crash reporter.
10. Root, setuid, or a systemd system unit.
11. Hidden state or an action with no command.
12. A feature tier, licence check, or activation.
13. Two accents in one view.
14. A Material Design artifact: tonal palette systems, elevation, ripple.
15. A surface that floats without an anchor.
16. A config format the user cannot read and edit.
17. A performance claim without a measurement.
18. A copied file without its GPL notice.
19. A v0.1 feature from the deferred list (`O1`–`O24`) without a Build Sheet
    amendment.
20. A per-keypress helper that does real work on the hot path. The thin
    `sivnox` wrapper that forwards one request to the running frontend is
    allowed; a script that starts a daemon, parses config, or spawns several
    tools is not.
21. A second source of truth for wallpaper state.
22. A theme designed in Rust rather than extracted and applied by the frontend.
23. A symlink from user config into `/usr/share`.

---

## The catechism

> **Quickshell renders the shell and owns the view.**
>
> **Rust does bounded jobs: one clipboard daemon and one-shot tools.**
>
> **The wallpaper is a frontend transaction. The config is the user's.**
>
> **The grey night is inhabited, not performed — still, placed, warm, and quiet.**
>
> **GPL-3.0. User-owned. No network. No root. No hidden state.**

---

**Amended into force for v0.1. Amend only by demonstration, never by preference.**
