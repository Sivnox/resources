# 02: Shell root (`shell.qml`) and shared state

Single resident instance `qs -c shell`. File:
`ryoku/shell/quickshell/shell/shell.qml`, root `ShellRoot` at `:54`.

## Per-monitor stack

`Variants{model:ShellState.screens}` with one `Scope` per screen carrying
`st = ShellState.forScreen(modelData)` (`:165`). Order reads top to bottom;
Wayland layers decide real stacking. Inside each scope:

1. `Wallpaper` (always on) feeding `Desktop` (widgets/stage host).
2. `OverviewBackdrop` (lazy; only if `Wm.caps.overviewBackdrop` and enabled).
3. `Visualizer` (lazy; only if enabled or placing; suppressed while the
   desktop hosts it).
4. `Frame` (resident bar; reads reveal from the slice; parks the record
   island beside the dock band).
5. `DockSurface` (lazy; only if `Dock.cfg("enabled")`).
6. `DockMenuOverlay` (lazy, 2s hold).
7. Toggle overlays: launcher, overview, clipboard (lazy, 15s hold each).
8. Resident-after-warm OSDs (`volume`, `mic`, `brightness`, keyboard) and
   `NotificationPopups` (warm = 1.5s timer, `:63`).
9. Per-flow overlays (20s hold): region (recorder), capture, camera,
   keypress.
10. `RyokuConfirmationDialog` (5s hold; only on the owning monitor).

## Lazy-loader pattern

Every overlay follows one shape (comment at `:274`): born closed,
`activeAsync: open || hold.running`, `onItemChanged: Qt.callLater(showNow =
open)` so the reveal is driven by an active-change one tick after build,
`onOpenChanged` restarts the hold on close. Destruction happens only while
closed, after the slide-out finished. Holds: 15s toggles, 20s flows, 5s
confirm, 2s dock menu.

## One toggle mapping, two arrivals

`toggleSurface(id)` (`:453`) is the single definition of every surface
transition: `barToggle` flips `st.barRevealed`; `launcher`/`clipboard`
flip their flags; `overview` calls `Wm.toggleOverview()` on
`caps.nativeOverview` else flips `st.overviewOpen`; `visualizer` toggles
persisted enabled; `visualizer-overlay`/`visualizer-place` flip slice
state; `quicksettings`/`wallpaper-menu`/`stash`/`screenshot`/`compress`/
`install` go onto the frame menu bus via `requestSurfaceActive`.

Both arrival paths end in that one function:

- Hyprland: `CustomShortcut{name:... onPressed: toggleSurface(...)}`
  over the global-shortcuts protocol.
- niri (no such protocol): keybind spawns `ryoku-shell <id>`, daemon
  routes to `shellIpc("openSurface", mon, id)`, which arrives as
  `ShellState.surfaceRequested` and is routed through the same
  `toggleSurface` (`Connections` at `:513`). Frame-menu ids never match
  there, so each surface opens exactly once.

`IpcHandler{target:"shell"}` (`:715`) is the daemon-facing channel:
`openSurface`/`closeSurface`, `keyringPrompt`/`keyringHide`, voice
show/off/hide, `pluginPopout`, `bar`, `closeAllMenus`, `sessionConfirm`.
A second `IpcHandler{target:"visualizer"}` (`:748`) handles Hub-placed
drag positioning; `target:"keypresses"` (`:762`) exposes overlay state.

Root machinery ported from the old pill root: `Devices.restore` +
caffeine/game-mode re-arm on boot, keep-awake external inhibitor plus a
compositor `IdleInhibitor`, dnd binding onto `Notifs`, 1x1 keyboard-focus
bounce window, self-view cleared when capture ends, polkit/stash surface
routing.

## `ShellState` (`services/ShellState.qml`)

The per-monitor bus. `screens` deduped by output name; `forScreen(screen)`
and `forActive()` (focused output via `Wm`, fallback first instance);
`requestSurface(id, mon, ctx)` / `closeSurface` signals plus
`requestSurfaceActive`; session-action state
(`sessionAction/Monitor/Message/Positive`, `askSessionAction`,
`clearSessionAction`); `focusRestoreRequested` for the bounce window.
Each monitor slice persists `launcherOpen`, `overviewOpen`,
`clipboardOpen`, `barRevealed` (default true), `visualizerOverlay`,
`visualizerPlacing`, OSD/notification visibility.

## `Wm` (`ryoku/ui/Singletons/Wm.qml`)

Capability-shaped compositor view. Consumers ask `caps.*`, never which
compositor runs. Presentation lists come from Wayland protocols
(ext-workspace windowsets, toplevels); the residue (focused output,
per-window workspace, focus order, geometry, caps, workspace model)
rides the daemon `wm` topic over a `subscribe wm` socket with
version-gated section copies. API: `outputs`, `workspaces`,
`focusedWorkspace`, `windows`, `focusedWindow`, `focusedOutput`,
`overviewOpen`, `keyboardLayout(s)`; verbs `focusWindow/closeWindow/
focusWorkspace/cycleWorkspace/moveWindowToWorkspace/...`,
`toggleOverview`, `reloadConfig`, `setScreenShader`,
`setWorkspaceLayout`, `outputByName/workspaceByName/
windowsOnWorkspace/outputHasFullscreen`. Gated internally on caps, so QML
never branches on compositor names (the `Ryoku.Wm.Hyprland` bridges for
global shortcuts and focus-grab are capability-loaded).

## `Config` and `Tokens`

`services/Config.qml` watches `~/.config/ryoku/shell.json` (plus
`theme.json`, `brand.json`) with `FileView watchChanges` and reloads
live; writes go through the daemon socket (`settings.patch`), with
`setScreenShader` also mirrored to `Wm`. `Ryoku.Ui` `Tokens.qml` is the
single place for look: role/layer chain (named scheme, wallpaper,
base), per-monitor UI scale and bar/widget visibility, file watches on
theme/shell/palette files.

Next: `03-services.md` (every singleton), `04`-`07` (surfaces),
`08-wm-seam.md` (provider contract behind `Wm`).
