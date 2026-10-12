# Shell map: how the Ryoku desktop works

A fork-oriented map of the shell: one daemon plus one QML process, the
surface bus between them, and the compositor seam underneath. Start here,
then follow the links.

## Process model

```
keybind / click
  |-- Hyprland global-shortcut --> CustomShortcut in shell.qml --> toggleSurface
  |-- niri (no protocol) -------> `ryoku-shell <id>` --> daemon --> shellIpc --> same toggleSurface
daemon `ryoku-shell daemon` (ryoku/shell/ipc/)
  |-- supervises `qs -c shell` (single QML process)
  |-- owns clipboard, tray, wallpaper hooks, theming, sleep guard
  `-- publishes topics on $XDG_RUNTIME_DIR/ryoku-shell.sock
QML `qs -c shell` (ryoku/shell/quickshell/shell/shell.qml)
  |-- one Scope per monitor, all surfaces in-process
  `-- reads daemon topics via subscribe/call sockets (see 03-services.md)
compositor via ryoku-wm-<name> providers (see 08-wm-seam.md)
```

Rules that hold everywhere: QML renders, Go decides (`docs/conventions.md`).
Nothing outside `ryoku/wm/` names a compositor. `shell.json` has one writer
(the daemon settings store).

## Reading order for a fork

1. `02-shell-root.md` then `01-daemon.md`: the two halves and the bus.
2. `03-services.md`: the singleton catalogue and the subscribe/call pattern.
3. Module files `04`-`07`: pick the surfaces you keep.
4. `08-wm-seam.md`: what the shell asks of the compositor.
5. `09-settings-theming-delivery.md`: how settings flow and ship.

## Files

- `01-daemon.md`: control plane, socket verbs, topics, supervisor.
- `02-shell-root.md`: `shell.qml` stack, lazy pattern, `toggleSurface`,
  `ShellState`, `Wm`, `Config`, `Tokens`.
- `03-services.md`: all `services/` singletons and daemon feed table.
- `04-modules-bar-dock.md`: Frame/menu bus, bar styles, dock.
- `05-modules-launcher-overview.md`: launcher variants/providers, overview.
- `06-modules-wallpaper-stage-visualizer-desktop.md`: wallpaper/ryogami,
  stage, visualizer, desktop integrator.
- `07-modules-osd-notifs-capture-confirm.md`: OSDs, notifications,
  capture/record overlays, confirm dialog.
- `08-wm-seam.md`: provider verbs, caps gating, `ryoku-wm-*` binaries.
- `09-settings-theming-delivery.md`: Hub writes, matugen pipeline,
  materialize/doctor/update, `deploy.sh`.

Related upstream docs (normative, not duplicated here): `docs/compositors.md`,
`docs/structure.md`, `docs/theming.md`, `docs/stage.md`, `docs/barstyles.md`,
`docs/plugins.md`, `docs/updates.md`, `docs/cli.md`.
