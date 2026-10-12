# 05: Launcher and overview

## Launcher

Entry: `modules/launcher/Launcher.qml` (`Scope` with `screen`/`active`/
`requestClose`), lazy per monitor (15s hold). Dispatches by
`catalog.json` (`default: hero`, `fallback`, plus `main`, `okshell`,
`kairos` entries). Each variant is a directory with an entrypoint plus a
preview; shared code lives in `shared/` (providers, `lib/` dispatch,
lifecycle, frecency, results, catalogue JS).

- `variants/main`: per-screen panel with search row plus dashboard/grid/
  list; renders `Dispatcher` results only.
- `variants/hero`: full-screen surface (`LauncherSurface`, exclusive
  keyboard focus, frost/shutter visuals) with window rail, dismiss scrim,
  result ledger/drawer, action shelf.
- Providers (`shared/providers/Providers.qml`, 12: actions, apps, calc,
  find, mpris, packages, radio, recent, script, snippets, web, windows)
  feed one `Dispatcher`; the body never queries sources directly.

Visibility: `ShellState.launcherOpen` per monitor; close resets the flag
via `requestClose`. Opens: `toggleSurface("launcher")`, the `launcher`
`CustomShortcut`, the same id on the surface bus, and the in-process rail
`app-launcher` flip.

Keep the controller plus `Providers`/`Dispatcher` routing and the
`launcherOpen` binding; add or replace exactly one `variants/<id>/` and
register it in `catalog.json`. Related: `docs/launcher.md`.

## Overview

Entry: `modules/overview/OverviewSurface.qml` (`Scope` ->
`PanelWindow`, overlay layer) with body `Overview.qml` (desktop strip,
filmstrip cells, drag/commit logic that defers commits to close).
Lazy per monitor (15s hold).

- Availability is capability-gated:
  `available = !Wm.caps.nativeOverview && Wm.caps.windowGeometry`.
  On niri the shell overview stands aside and `Super+Tab` is the
  compositor native overview.
- Visibility: `ShellState.overviewOpen` per monitor; Esc/click-out closes
  via `requestClose`.
- Opens: `toggleSurface("overview")` (native toggle or flag flip),
  the `overview` shortcut, the same bus id.
- Data and verbs all via `Wm`: `workspaces`, `outputByName`,
  `focusWorkspace/focusWindow/moveWindowToWorkspace/closeWindow`.
  Cells: `WorkspaceCell.qml`, `DesktopStrip.qml`, `BrandMark.qml`.

Keep the surface layer/focus/close contract and commit-on-close
deferral; replace only strip/cell visuals.

Both surfaces depend only on `ShellState`, `Wm`, `Tokens`/`Config`/
`Motion`/`Theme`, and `FocusGrab` where exclusive focus is needed. Neither
names a compositor (see `08-wm-seam.md`).
