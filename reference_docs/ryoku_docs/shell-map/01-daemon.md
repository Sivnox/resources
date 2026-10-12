# 01: Daemon (`ryoku-shell`)

One Go program, `ryoku/shell/ipc/`. As `ryoku-shell daemon` it supervises the
UI and owns all side effects. As `ryoku-shell <verb>` it is a thin client that
forwards one line to the daemon socket and prints the reply. See
`ryoku/shell/ipc/main.go` (`sockPath`, `sendCommand`, `usage`).

## Socket

`$XDG_RUNTIME_DIR/ryoku-shell.sock`, mode 0700, one reply line (`ok` or
`err ...`). Three framings share it (`daemon.go:handle`):

- one-shot verbs: `dispatch()` (`daemon.go:1034`).
- `subscribe <topic>`: long-lived stream, immediate last frame then updates.
- `call <topic>.<method> <json>`: RPC with echoed id.
- plus `clip-ingest`/`clip-copy` (clipboard bytes ride the body, never argv)
  and `keyring-respond` (secret on a second line, never `/proc` cmdline).

Standalone client-side verbs that never touch the daemon: `theme catalog`,
`matugen-preview <img>`, `icons ...`, `browser-host` (`main.go`).

## Lifecycle

`runDaemon` (`daemon.go:250`): `wm.Open()`, stale-takeover check via the
`signature` verb (only a provably other-session incumbent is displaced),
exclusive `.lock`, listen, `setupQmlImportPath`, `runtimeJanitor`,
`bootstrap`, accept loop. `bootstrap` (`daemon.go:429`) starts, in order:
settings, keypress, clipboard, tray, weather, music, calendar, power
profiles, network, nightlight (+ sun watcher), OSD, keyring prompter,
session, polkit, updates, paint worker, stage, ryogami watcher, matugen
knob watcher, LED worker, WM watcher, audio/power watchers, widget gate,
idle park, sleep/wake, then `startComponents`.

Supervisor: exactly one persistent component, `shell` (`daemon.go:31`),
launched as `qs -c shell` (or `-p <dir>` under `RYOKU_SHELL_DIR`).
`ensure`/`supervise` restart with exponential backoff, reap strays first
(double shell is worse than a hard kill), park on gates, log per-component
to `~/.local/state/ryoku/surfaces/<name>.log`. `reload` waits for a fresh
pid that stays up 3s and reports the tail line on failure.

## Routing surfaces

`surfaceCommands` (`daemon.go:978`) maps every bare surface id
(`launcher`, `overview`, `clipboard`, `stash`, `screenshot`, `bar-toggle`,
...) plus `menu <id>` to `("shell","shell","openSurface")` via `route()`
(`daemon.go:1011`). `dispatch` resolves the active monitor itself
(`activeMonitor`, warm from the WM watcher) and calls
`shellIpc("openSurface", mon, id)`, which runs
`qs ... ipc call shell openSurface <mon> <id>` with retries (`actions.go`:
`qsSelect`, `ipcCall`, `shellIpc`). QML lands it on
`IpcHandler{target:"shell"}` in `shell.qml` (see `02-shell-root.md`).

Direct verbs in `dispatch` (besides routed surfaces): `menu close`,
`bar <edge> <toggle|reveal|hide>` and `bar <list|catalog|move|show|...>`
(`barCLI`), `dock ...`, `voice` (voxtype toggle + mic surface),
`barstyle <id>` (patches settings store), `lock` / `lock status` /
`lock session` / `lock-quiesce` / `unlock-prepare`, `suspend`
(+ `transaction` flow) / `suspend-cancel`, `sleep-ready`, `signature`,
`audio up|down|mute`, `brightness up|down`, `hub open [section]|close`,
`stage <verb>` (see `06-...`), `theme <name>` / `theme remove <id>`
(patches `theme.theme`), `gtk apply <light|dark>`, `plugin <id>`,
`plugins reload` (no-op ack), `state <name> 0|1` (idle-park visibility),
`sound <event>`, `reload|status|ping|quit`.

## Topics and calls (owned here, fed by workers)

| Topic | Publisher | QML readers |
|---|---|---|
| `settings` | `settings.go` store, sole writer of `~/.config/ryoku/shell.json` | `Config`, `Dock`, Hub `Settings` |
| `wm` | `wmclient.go` folds provider frames | `Ryoku.Ui` `Wm` singleton |
| `clipboard` | `clipboard.go` (100-entry history, `wl-paste --watch`, keeper) | `services/Clipboard.qml` |
| `tray` | `tray.go` (StatusNotifier watcher/host) | `services/Tray.qml` |
| `osd` | brightness seq (`OsdFeed`) | `modules/osd/` |
| `nightlight` | nightlight worker | `services/Nightlight.qml` |
| `stage` | stage worker (`stageStatusJSON`, verbs above) | `StageBackend.qml` |
| music/weather/calendar/network/power/polly/keyring/... | matching `*.go` workers | matching `services/*.qml` |

`statestream.go` is the framing (`registerTopic`, `publish` with
byte-identical dedup, `registerCall`/`serveSubscription`/`serveCalls`).
`control.go` holds reference verbs (`menuClose`, `barToggle`, `audio`,
`brightness`, `hub*`). `qsruntime.go` prunes dead quickshell runtime dirs;
it serves nothing.

## Workers that matter for a fork

- `settings.go`: typed schema for `general|theme|bars|menus|notifications|
  wallpaper`, passthrough for the rest; `patch` (flock, re-read, validate,
  persist, broadcast) and `settings.patch`/`settings.reset` calls.
- `theming.go`/`matugen.go`: `paintWorker` on `paintSig`: wallpaper tone,
  static-vs-follows palette, `colors.json`, templates, border recolor,
  compositor reload where `configReload`. `matugen.json` knobs are inputs,
  never outputs.
- `ryogami.go`: client of `ryogami.sock`; `currentWall()` feeds paint;
  `depth set/clear` feeds the cut pipeline.
- Sleep/lock (`sleepwake.go`, `lock.go`, `prompter.go`, `polkit.go`):
  fail-closed login1 transaction, qylock handoff, keyring prompter, polkit
  agent. Keep as a unit; the lid/idle side is `hypridle` + `ryoku-clamshell`
  (see `docs/compositors.md`).

Next: `02-shell-root.md` (the QML half of the bus), `03-services.md`
(feed catalogue), `08-wm-seam.md` (compositor side of `wm.act`).
