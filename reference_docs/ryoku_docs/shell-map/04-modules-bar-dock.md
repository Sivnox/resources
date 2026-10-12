# 04: Bar (Frame) and dock

## Frame (the bar)

Entry: `modules/bar/Frame.qml` (one `Scope` per monitor, instanced from
`shell.qml`). Inside: `Bar.qml` repeats `FrameRail` over the four edges;
`FrameMenuManager.qml` repeats `FrameMenu` per `Config.normalizedFrameBars`
and is the single-open menu host (mask union, focus lifecycle, keyboard
bounce on dismiss).

- Styles: `barstyles/` holds scenes. `sumi` is the built-in rail
  (`manifest.json` with empty scene); `qsbar` is the full widget/panel
  tree (`core/` catalogue + IPC router, `modules/` widgets, `panels/`
  popout bodies, `controlcenter/`). `Frame.qml` picks the scene via
  `BarProducts.sceneUrl(Config.barStyle)`; non-sumi styles still map the
  overlay menu host. New style = new `barstyles/<id>/` + catalogue entry;
  the menu bus stays untouched.
- Popouts: `popouts/Popout.qml` base plus ~22 content cards. Hard-wired
  islands (voice, keyring, polkit, music, bluetooth, battery, network,
  sysmon, audio, screenshot) plus dock/music/plugin previews and all
  config-driven menus/surfaces from `normalizedFrameBars`.
- Visibility: `ShellState.barRevealed` per monitor ANDed with per-edge
  `Config` reveal flags and `Tokens.barEnabledFor`; hidden on
  `Wm.outputHasFullscreen`; `Frame.qml` unions the input mask.
- Opens: `toggleSurface("barToggle")` flips the slice flag; every menu or
  surface id arrives via `ShellState.surfaceRequested` filtered per
  monitor (`quick-settings`, `wallpaper`, `stash`,
  `quick-settings#capture`, `stash#compress`, `stash#install`), or via
  daemon `IpcHandler shell.bar/openSurface`.

Keep the Frame overlay plus `FrameMenuManager` contract (single open,
menu bus, mask, focus return). Replace only rail widgets, style scenes,
and card bodies.

## Dock

Entry: `modules/dock/DockSurface.qml` (`PanelWindow`, lazy while
`Dock.cfg("enabled")`), band `DockBand`, overlay `DockMenuOverlay` on the
owning monitor. Model: `services/Dock.qml` (`resolve`, pins, `activate`,
`anyFocused`, menu state, `styleOptions`).

- Visibility has no `ShellState` flag: `visible` is the persisted enabled
  knob; `revealed` is a state machine (autohide off, pointer inside,
  nothing focused, dragging, menu here), forced off fullscreen unless the
  pointer is in the peek strip. The shell also steps it back during stage
  widget editing.
- No `toggleSurface` id; enable/persist goes `Dock.setCfg` ->
  `settings.patch dock`. Context menu via `Dock.openMenu` with screen
  ownership; hover previews flow through `DockPreview` into the frame
  menu host. Frame reads the dock band geometry to park the record
  island beside it.

Keep the `Dock` model plus reveal/mask/peek contract; replace only band
item chrome per style.

Related: `docs/barstyles.md` (style authoring), `docs/frame.md` (Blobs
render), `02-shell-root.md` (bus), `08-wm-seam.md` (fullscreen and focus
inputs come from `Wm`/protocols, never compositor branches).
