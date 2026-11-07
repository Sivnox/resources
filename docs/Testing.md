# Sivnox — testing and acceptance

**Status:** active. Derived from `Build_Sheet-Filled.md` (`T1`–`T24`).
**Rule:** no efficiency claim without a number and the command that produced it.

This document is the gate for v0.1. Each test has a procedure and a pass
condition. A test that cannot be reproduced is not a test.

---

## 1. Reference environment

Record once per release:

- Laptop model, CPU, GPU, RAM.
- Hyprland version and Lua support.
- Quickshell version.
- Distro and kernel.
- Display resolution and scale.
- A clean user session with no extra apps running.
- The same wallpaper, config, and font set for every performance run.

Store the baseline table in the repository (for example
`docs/baseline-v0.1.md`) with the date and commit.

---

## 2. Performance measurements

### 2.1 T1 — login to usable desktop

**Pass:** under 5 seconds.

**Procedure:**

1. Reboot.
2. Start timing at the first Hyprland frame (session start).
3. Stop when the bar is painted and interactive.
4. Use `systemd-analyze --user` for service timing and a stopwatch for the
   visual first frame. Record both.
5. Run three times; report the median.

**Notes:** the measurement includes `sivnox-shell.service`, not package
installation.

### 2.2 T2 — idle RSS, bar only

**Pass:** under 60 MB.

**Procedure:**

1. Boot to the desktop; close the launcher, picker, clipboard, and screenshot.
2. Wait 60 seconds for idle.
3. Measure the sum of RSS for `qs` and its direct helper processes:
   `ps -o rss= -C qs | awk '{s+=$1} END {print s/1024 " MB"}'`.
4. Also report `sivnoxd` if running; it should be stopped unless the picker was
   used.
5. Run three times; report the median.

### 2.3 T3 — idle RSS, bar + launcher

**Pass:** under 90 MB.

**Procedure:**

1. Open the launcher and leave it idle for 10 seconds.
2. Measure as in T2.
3. Close the launcher; confirm memory returns near T2 (within 10%).
4. Run three times; report the median.

### 2.4 T4 — idle CPU

**Pass:** under 1% average.

**Procedure:**

1. Boot; close all transient surfaces.
2. Wait 60 seconds.
3. `pidstat -u 1 60` on the Quickshell process (or `top -b -d 1 -n 60`).
4. Report the average over the window, excluding the first 10 seconds.
5. Also report the max single-sample value for context.

### 2.5 T5 — launcher cold-open

**Pass:** under 100 ms. If it fails, trigger `L10` (Rust launcher alternative).

**Procedure:**

1. Ensure the launcher has never been opened since the shell started.
2. Instrument the frontend to log a timestamp when the surface is requested and
   when its first frame is painted.
3. Trigger `sivnox open launcher`.
4. Repeat 10 times with a shell restart between runs; report the median.
5. If the frontend cannot self-instrument yet, use a high-speed screen capture
   and count frames; record the method.

### 2.6 T6 — colour apply on wallpaper change

**Pass:** under 200 ms.

**Procedure:**

1. Open the wallpaper picker.
2. Select a wallpaper with a different palette.
3. Commit.
4. Measure from commit to the shell's `Theme.qml` reporting the new tokens
   applied.
5. Repeat with a cold colour cache and a warm cache; report both.
6. The target applies to the warm cache. A cold cache may show fallback tokens
   first and final tokens later; record the total.

---

### 2.7 Warning measurements (not gates)

Record these for context; they do not fail the release on their own.

- **Picker-open RSS:** open the wallpaper picker, wait for thumbnails, measure
  the same way as T2. The grid must not hold full-resolution images.
- **Clipboard-open RSS:** open the clipboard picker with 100 entries.
- **Notification burst:** send 20 notifications in 5 seconds; record the peak.
- **Wallpaper scrub:** preview 20 wallpapers in 10 seconds; record the peak and
  confirm the persisted path is unchanged.

A warning measurement that grows with the number of items is a leak signal and
becomes a bug.

---

## 3. Functional tests

### T7 — clipboard pin survives eviction

**Pass:** yes.

**Procedure:**

1. Set `clipboard.max_entries = 3` in a test config.
2. Copy three distinct text values.
3. Pin the first.
4. Copy three more distinct values.
5. Open the picker.
6. The pinned entry is present; the oldest unpinned entries are gone.

Automate in `sivnox-tests` against the daemon and SQLite.

### T8 — clipboard drops oldest unpinned at limit

**Pass:** yes.

**Procedure:**

1. Set `max_entries = 3`, `pins_persist = false`.
2. Copy four distinct values.
3. The first value is gone; the newest three are present.
4. Repeat with `max_bytes` as the limiting factor.

### T9 — wallpaper preview reverts on cancel

**Pass:** yes.

**Procedure:**

1. Open the wallpaper picker.
2. Select a different wallpaper (preview).
3. Cancel or press Escape.
4. The displayed wallpaper returns to the persisted one.
5. `wallpaper.json` is unchanged.

### T10 — wallpaper commit survives shell restart

**Pass:** yes.

**Procedure:**

1. Commit a wallpaper.
2. Verify `wallpaper.json` has the new path.
3. `systemctl --user restart sivnox-shell`.
4. The wallpaper returns to the committed path.
5. `kill -9` the shell and let systemd restart it; the same happens.

### T11 — kill shell → systemd restarts it, bar re-renders

**Pass:** yes.

**Procedure:**

1. `kill -9 $(pgrep -x qs)`.
2. `systemctl --user status sivnox-shell` shows a restart.
3. The bar returns within 5 seconds.
4. Wallpaper and config reload from disk.

### T12 — lid close → lock screen appears

**Pass:** yes.

**Procedure:**

1. Enable `session.lock_on_lid_close`.
2. Close the lid (or simulate `loginctl lock-session`).
3. The lock screen appears; the session is not accessible behind it.
4. Authenticate; the session returns.

### T13 — volume key → OSD

**Pass:** yes.

**Procedure:**

1. Press a volume key.
2. The volume OSD appears with the current value.
3. The value changes and the OSD updates.
4. The OSD dismisses after its timeout.

Repeat for brightness, mic, and layout.

### T14 — notification arrives and dismisses

**Pass:** yes.

**Procedure:**

1. `notify-send "Sivnox" "test"`.
2. A popup appears with the correct title/body.
3. It dismisses on timeout or click.
4. It appears in history.
5. With DND on, no popup appears but history records it.

### T15 — Wi-Fi connect via menu

**Pass:** yes.

**Procedure:**

1. Open the network popup.
2. Select a known network.
3. Enter credentials if needed.
4. The connection succeeds and the bar reflects it.

### T16 — Bluetooth pair via menu

**Pass:** yes.

**Procedure:**

1. Put a device in pairing mode.
2. Open the Bluetooth popup.
3. Pair and connect.
4. The bar reflects the connected state.

### T17 — screenshot region shows live dimensions

**Pass:** yes.

**Procedure:**

1. Trigger `sivnox screenshot region`.
2. Drag a region.
3. The overlay shows the live `W×H` while dragging.
4. Release saves/copies the correct region.

### T18 — theme fan-out writes at least kitty + GTK on commit

**Pass:** yes.

**Procedure:**

1. Commit a wallpaper with a distinct palette.
2. Check the kitty and GTK target files changed.
3. The values match the current token set.
4. `sivnox theme apply --json` reports both in `ok`.
5. A deliberately bad template appears in `failed` and does not stop the others.

### T19 — config change applies without restart

**Pass:** yes.

**Procedure:**

1. Change `[bar] height` in `config.toml`.
2. Save.
3. The bar height changes within one second.
4. Change an invalid value; the shell logs and keeps the last good config.

### T20 — all daemons and files run as user, no root

**Pass:** yes.

**Procedure:**

1. `ps -o user,pid,cmd -C qs -C sivnoxd`.
2. All processes run as the logged-in user.
3. `systemctl --user status` shows user units; no system unit exists.
4. `find ~/.config/sivnox ~/.local/share/sivnox ~/.cache/sivnox -not -user "$USER"`
   returns nothing.
5. File modes match `Config.md` §6.

### T21 — Hyprland Lua config loads without errors

**Pass:** yes.

**Procedure:**

1. Start Hyprland with the managed tree.
2. `hyprctl reload` prints no errors.
3. The Hyprland log contains no Lua syntax/API errors.
4. `rg -i caelestia` in the managed tree returns only attribution comments.

### T22 — `hypr-vars.lua` override changes a variable live

**Pass:** yes.

**Procedure:**

1. Change a visible variable (for example gaps or border size).
2. `hyprctl reload`.
3. The change is visible.
4. Remove the override; the default returns.

### T23 — keybind in `hypr-user.lua` opens the launcher

**Pass:** yes.

**Procedure:**

1. Press the configured launcher keybind.
2. The launcher opens on the focused monitor.
3. Escape closes it.
4. The keybind calls `sivnox open launcher`, not an ad-hoc script.

### T24 — layer rule reserves space for the bar

**Pass:** yes.

**Procedure:**

1. Open a tiled window.
2. It does not overlap the bar.
3. `hyprctl layers` shows the bar layer with the reserved area.
4. Switch `bar.position` from `top` to `bottom`; the reservation follows.
5. Fullscreen hides the bar and uses the full screen; exiting restores the
   reservation.

---

## 4. Week gates

| Week | Gate |
|---|---|
| W0 | T21–T24; config change applies; bar renders |
| W1 | T5, T12–T16; notifications, OSD, lock, launcher usable |
| W2 | T6, T9, T10; wallpaper preview/commit and recolour |
| W3 | T7, T8, T17; clipboard and screenshot |
| W4 | T1–T4, T11, T18–T20; full v0.1 |

No week starts until the previous gate is green.

---

## 5. CI and static checks

Run on every change:

- `cargo fmt --check`
- `cargo clippy --workspace -- -D warnings`
- `cargo test --workspace`
- `cargo deny check licenses` (or an equivalent GPL-aware check)
- `qmllint` on every QML file
- `shellcheck` on any shell script
- `hyprvalidate` on the managed Lua tree and override examples
- a grep for `caelestia` in the shipped managed tree (attribution comments
  allowed)
- a grep for hardcoded colours outside `Theme.qml` defaults
- `strace -f -e trace=network` on the Rust binaries to prove no network calls
- a file-mode check on created state

The CI does not run Hyprland; T21–T24 are a manual/reference-machine gate.

---

## 6. Unit and integration tests

### Rust

- `config dump` round-trips the example and merges defaults.
- `colours` determinism, cache hit, content-hash invalidation, contrast.
- `thumb`/`frame` cache keys and bounds.
- clipboard insert, dedup, pin, eviction, restart, corruption recovery.
- clipboard socket request/reply for every op.
- atomic write leaves the previous file on failure.
- no network syscalls.

### QML

- `qmllint` clean.
- Each surface opens, closes, and unloads.
- No surface is instantiated while closed.
- `Config.qml` handles valid, invalid, and missing config.
- `Theme.qml` applies tokens from `sivnox colours` and falls back on failure.
- Wallpaper preview/cancel/commit.
- No full-resolution image in a picker grid.

### Shell/Hyprland

- `hyprvalidate` clean.
- `hyprctl reload` clean.
- Keybinds call `sivnox`, not ad-hoc scripts.
- Layer rule reserved area.

---

## 7. Manual test data

Prepare once:

- Three wallpapers with clearly different palettes (warm light, cool dark,
  monochrome).
- One video and one GIF for first-frame preview.
- A Wi-Fi network and a Bluetooth device available for T15/T16.
- A notification source (`notify-send`).
- A clipboard text set for T7/T8.
- A deliberately malformed `config.toml` for T19.
- A deliberately malformed Lua override for T21.

---

## 8. Reporting

A test result records:

- Test ID and commit.
- Machine and session details.
- Command or procedure used.
- Raw number or observation.
- Pass/fail.
- If fail: the smallest reproduction and the suspected component.

Store the v0.1 baseline table in the repository. A regression is a number that
moved in the wrong direction, not a feeling.
