# 06: Wallpaper, stage, visualizer, desktop

## Wallpaper (ryogami bridge + painters)

`modules/wallpaper/`: `Wallpaper.qml` per screen owns `WallpaperFrame`,
which subscribes to `$XDG_RUNTIME_DIR/ryogami.sock` (`subscribe
wallpaper`, 2s retry) and exposes `wallpaperUrl` (with revision query),
`fit`, `transition`, video props, `live`, and `reloadReady`. `Backdrop.qml`
paints stills (ping-pong `Image` + `reveal.frag.qsb` / skwd shaders) or
yields to the `ryogami-live` video player; the `Super+W` picker
(`ryoku/shell/ryogami/wall-ui/`) writes through the same daemon the shell
only reads. `OverviewBackdrop.qml` is a separate lazy surface, gated on
`Wm.caps.overviewBackdrop` plus its own config file. `depth`/`depthRev`
on the frame are the cut-pipeline wire (see stage); do not reinterpret.

Go side (`ryoku/shell/ryogami/daemon/`): retained coalesced `wallpaper`
topic (`surface.go`, `topics.go`, `proto.go`); verbs plus JSON-RPC.
`ryoku-shell` bridges it (`ipc/ryogami.go`): `currentWall()` feeds the
paint worker, `depth set/clear` feeds cuts.

Fork: reuse `Wallpaper` + `WallpaperFrame` + `Backdrop` as a unit; feed
`url`/`transition`/video together. `OverviewBackdrop` is droppable.

## Stage (cut-out / parallax, inside Desktop)

No separate surface: `modules/stage/` renders inside `Desktop` (backdrop
z1, visualizer z1.5, layers z2/4, widgets above, compose overlay z60,
menus z90). `Singletons/Config.qml` (`~/.config/ryoku/stage.json`:
quality, edge, shadow, motion, front, follow-mouse, backdrop; drag edits
coalesced 400ms). `StageSession.qml` (widget-edit mode/monitor/selection).
`StageLayer.qml` (depth-to-parallax gain, music bounce, idle drift, busy
ring with cancel). `StageBackend.qml` (subscribes `stage` on
`ryoku-shell.sock`; fire-and-forget verbs `set-effect/refresh/cancel/
add-layer/cut-layer/remove-layer/clear`, coalesced `set-layer`; engine
via `ryostage check/models/install/remove`).

Keep the verb names and the `walls[path]{effect,layers[{out,enabled,
front,depth,label}],subject,background,rev}` shape; depth mode is the
same layer with motion off. Related: `docs/stage.md`.

## Visualizer (cava spectrum)

Entry `modules/visualizer/Visualizer.qml` (`mode: off|desktop|overlay`,
`placing`, `suppressed`; click-through panel; one `VisualizerView` per
`VizItem`). `Singletons/Config.qml` (`visualizer.json`: enabled, fps,
adaptive, up to 4 boxes with geometry/rotation/overhang). `Motion.qml`
(budget governor, attack/decay ticks, idle wave honoring
`Perf.visualizerFrozen`). `Spectrum.qml` (single shared `cava` process,
bars = max over instances). `Waveform.qml` (line styles via python
wavecap). `Scheme.qml` (`colors.json` accent). `Placer.qml` + `EditBar`
(separate overlay surface, since the spectrum never takes pointer).
`InlineVisualizer.qml` (surfaceless repeater the desktop hosts between
parallax layers when `hostsVisualizer`; the sibling surface suppresses
itself).

Keep one shared `Spectrum` or multiple cava processes spawn; keep the
`Visualizer`/`InlineVisualizer` view sharing.

## Desktop (Bottom widget layer + integrator)

Entry `modules/desktop/Desktop.qml` (`Scope` -> `PanelWindow`
`ryoku-widgets`, Bottom, click-through; exclusive keyboard focus only
while editing, with bounce helper). Owns the composition order above
plus glass sampling mirror, per-widget loaders gated on
`reloadReady = Config.ready && Registry.ready`, widget/plugin menus with
snapshot/restore, and placement via `ryoku-plugins-place`.
`Singletons/Config.qml` (`widgets.json`: clock, calendar, music, aio,
stats, weather, notes blocks); `Registry` (plugins.json); `Now`, `Fonts`,
`Theme`/`Scheme`, `Performance` per-module singletons.

Fork `Config` + `Registry` + the Backdrop/Stage/Viz slots together; the z
contract and glass sampling assume one surface.

Cross-links: paint worker and `colors.json` in `09-...`; `Wm` caps that
gate these modules in `08-...`; OSD/notification/capture siblings in
`07-...`.
