# Sivnox — provenance and licensing register

**Status:** active. Required before any Caelestia or Ryoku code is copied.
**Project license:** GPL-3.0 (`L1`).
**Sources:** Caelestia and Ryoku, both GPL-3.0.

Sivnox ports QML, Lua, and behaviour from two GPL-3.0 projects. This register is
how we keep the provenance honest: every copied file has a source, a commit, and
a notice. A file without a row here should not be copied.

---

## 1. Rules

1. **Every ported file keeps its copyright header and license notice.**
2. **Every ported file gets a row in the register below.**
3. **Record the source repository, path, and commit.**
4. **Do not copy code from a project with an incompatible license.**
5. **The complete corresponding source for Sivnox remains available.**
6. **A port commit says what was copied, from where, and what was changed.**
7. **If a file is heavily rewritten, say so; the origin still gets credit.**
8. **Do not copy a project's architecture wholesale.** Copy the minimum needed
   for the v0.1 scope.

---

## 2. Source projects

| Project | License | What we borrow | What we do not borrow |
|---|---|---|---|
| **Caelestia** | GPL-3.0 | Bar, launcher, lock screen, session/power menu, autostart mechanism, Hyprland Lua tree, theme fan-out engine, project structure | Its Python runtime, its installer, unrelated surfaces |
| **Ryoku** | GPL-3.0 | Clipboard behaviour, screenshot overlay, one-concern-per-file discipline | Its Go daemon, its 20-service sprawl, its installer, its package repo |

Quickshell is a dependency, not copied source. Its license is recorded in the
dependency audit.

---

## 3. Register

Fill one row per copied or substantially derived file. Add rows as the ports
happen; do not wait until W4.

| # | Sivnox path | Source project | Source path | Source commit | License | Changes | Status |
|---|---|---|---|---|---|---|---|
| 1 | `shell/modules/bar/…` | Caelestia | `…` | `…` | GPL-3.0 | path/namespace rename | pending |
| 2 | `shell/modules/launcher/…` | Caelestia | `…` | `…` | GPL-3.0 | path/namespace rename | pending |
| 3 | `shell/modules/lock/…` | Caelestia | `…` | `…` | GPL-3.0 | path/namespace rename | pending |
| 4 | `shell/modules/session/…` | Caelestia | `…` | `…` | GPL-3.0 | path/namespace rename | pending |
| 5 | `hypr/hyprland.lua` + `hypr/modules/` | Caelestia | `hypr/…` | `…` | GPL-3.0 | path/command rewrite (`P6`, `P7`) | pending |
| 6 | `crates/sivnox` theme fan-out | Caelestia | `…` | `…` | GPL-3.0 | Python → Rust | pending `P4` |
| 7 | `shell/modules/clipboard/…` | Ryoku | `…` | `…` | GPL-3.0 | QML edits | pending |
| 8 | `crates/sivnoxd` clipboard | Ryoku | `…` | `…` | GPL-3.0 | Go → Rust | pending |
| 9 | `shell/modules/screenshot/…` | Ryoku | `…` | `…` | GPL-3.0 | QML edits | pending |

The blank source paths and commits are filled during W0/W3/W4. `P4` and `P6` are
the gates for the Caelestia rows.

---

## 4. File header template

For a copied/derived source file:

```
// SPDX-License-Identifier: GPL-3.0-or-later
//
// Derived from <project> (<source path>, commit <sha>).
// Original copyright: <holder>.
// Modified for Sivnox: <one line>.
```

Use `--` comments for Lua, `#` for TOML/shell, and `//` for Rust/QML/JS.

For a new file:

```
// SPDX-License-Identifier: GPL-3.0-or-later
//
// Sivnox — <purpose>.
```

The exact copyright holder is copied from the source file, not guessed. If the
source file has no header, record that in the register and add the project-level
notice.

---

## 5. What is not copied

- Caelestia's Python colour/fan-out runtime is **ported**, not vendored.
- Ryoku's Go daemon is **ported**, not vendored.
- Neither project's installer, package repository, ISO, or ISO tooling is copied.
- No font, wallpaper, icon set, or art is copied unless its license is recorded
  here.
- No third-party QML module is copied; it is added as a dependency and recorded
  in the dependency audit.

---

## 6. Dependency audit

Before v0.1 release, record:

| Dependency | Version | License | Use |
|---|---|---|---|
| Quickshell | v0.1 target | recorded at build | frontend runtime |
| Rust crates | `Cargo.lock` | `cargo deny` | CLI/daemon |
| `grim` | distro | distro | screenshot |
| `wl-copy` / `wl-paste` | distro | distro | clipboard |
| `hypridle` | distro | distro | idle/lock |
| `brightnessctl` | distro | distro | brightness |
| `hyprsunset` | distro | distro | night light |
| `ffmpeg` | distro | distro | video first frame |
| `power-profiles-daemon` | distro | distro | power profiles |

A dependency with a license incompatible with GPL-3.0 distribution is rejected
before it enters the build.

---

## 7. CI checks

- Every source file has an SPDX identifier.
- Every copied file appears in the register.
- `cargo deny check licenses` passes.
- A grep for `caelestia` in the shipped managed tree returns only attribution
  comments (T21).
- The register has no `pending` rows for files that are actually shipped.

---

## 8. Open items

- `P4` — exact Caelestia fan-out files.
- `P6` — exact Caelestia Hyprland paths.
- `P7` — path/command rewrite list.
- Source commits for the Caelestia and Ryoku checkouts are recorded when the
  ports begin.
