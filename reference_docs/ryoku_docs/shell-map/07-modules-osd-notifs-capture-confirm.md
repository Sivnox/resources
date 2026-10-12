# 07: OSDs, notifications, capture, confirm

## OSDs

Entries `modules/osd/`, resident after warm: three `OsdWindow{kind:
volume|mic|brightness}` plus `KeyboardOsdWindow`. Content `Osd.qml`
(device = PipeWire sink/source; value = `OsdFeed.brightness` or clamped
audio volume with boost headroom; threshold icons; 700ms arm grace so
login sync pops no toast; 1s hide timer). Chrome `OsdWindow.qml`
(overlay, click-through, eased progress). `KeyboardOsd` shows the active
`KeyboardLayout.variant` on signature change, top-anchored.

Daemon link: `services/OsdFeed.qml` subscribes `osd` (`{brightness:
{value, seq}}`); volume/mic are PipeWire-direct, no daemon. `Nightlight`
is a separate topic (`toggle/set`) with a deck tile, no OSD surface.

Reusable pair (content + chrome) for any value/icon/flash source; keep
the arm grace. Scale via `Tokens.uiScaleFor`.

## Notifications

Resident `modules/notifications/NotificationPopups.qml` per monitor
(overlay, click-through; hidden for the `nacre` bar style) plus
`NotificationCard.qml`. Source `services/Notifs.qml`
(`NotificationServer`; `tracked`/`history`/`popups` newest-first; dnd
suppresses popups only; ttl rules; same-app dedup; reapers). Column keeps
a local `cards` ListModel reconciled by `sync()` so add/remove/displaced
animate; binding the view directly to `Notifs.popups` would kill the
animations. Card: header/app/time/open/expand/close, summary/body/actions
filtering, countdown canvas.

Closed set (`Notifs` + popups + card); the history panel reuses the card
with `compact: false`.

## Capture (shots + recording chrome)

Split deliberately: `services/Capture.qml` (shot prefs to `capture.json`,
`shoot(all|monitor|window|region)`, `commit` resolving to
`grim -o/-g` or `Recorder --monitor/--region`, one detached `sh` doing
grim + `clip-copy` + shutter sound + notify + beautify handoff) and
`services/Recorder.qml` (video backends `gsr|wf|studio` via
`ryoku-cmd-*` scripts, slurp region pick with layout invalidation,
countdown `startAfter`, pgrep polling 2s live / 30s idle).

Overlays, each independent and droppable singly (20s holds):
`CaptureOverlay` (region drag clamped output-local, >5px commit; window
pick from `Wm` where `caps.windowGeometry`; 4-band cutout; Esc cancels),
`RegionOverlay` (live recording box from `Recorder.regionGeom`),
`CameraOverlay` (self-view bubble, hidden handles while recording),
`KeypressOverlay` (chord display, drag persists placement).

Keep grim + `wl-copy` detached (a managed `Process` reaps the clipboard
daemon). Record borrows `Capture.selecting` geometry, then calls
`Recorder.startAfter`.

## Confirm

`modules/confirm/RyokuConfirmationDialog.qml`, lazy (5s hold), shown only
on `sessionActionMonitor`. Props `action` (`logout|reboot|shutdown`),
`message`, labels; self-masked box (outside clicks no-op, no Esc/outside
dismiss, no fullscreen grab). Source `ShellState.sessionAction*`;
confirm runs `SessionActions.run(a)` then clears.

Pairs with any action/message/run/clear store.
