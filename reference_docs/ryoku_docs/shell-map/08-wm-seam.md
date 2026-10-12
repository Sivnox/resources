# 08: Compositor seam (`ryoku/wm/`)

One desktop, multiple window managers. The shell, Hub, CLI, and lockscreen
are the same code everywhere; only a provider underneath differs (today
Hyprland and niri). Nothing outside `ryoku/wm/` may name a compositor
(enforced by `bin/ryoku-dev-verify-wm-isolation`). Consumers ask
capabilities (`Wm.caps.*`, `Client.Can`), never names. Full contract:
`docs/compositors.md`; walkthrough: `docs/adding-a-window-manager.md`.

## Provider verbs

One binary, `ryoku-wm-<name>`, shipped by `ryoku-desktop-<name>`:

| Verb | Answers |
|---|---|
| `caps` | capability manifest, workspace model, owned files, packages |
| `state` | one snapshot: outputs, workspaces, windows, keyboard |
| `watch` | stream of frames, one JSON object per line |
| `act <id>` | one neutral action (`window.close`, `workspace.focus`, ...) |
| `apply <store>` | write the compositor config from the neutral store |
| `defaults` | provider baseline subtree of the store |
| `schema` | exclusive settings rows, for the Hub renderer |
| `binds` | compositor-only keybinds resolved against the store |
| `outputs <layout>` | apply a display layout |
| `session` | `wayland-session` desktop entry |
| `environment <pid>` | export the opaque session handle |

`apply --preview` writes nothing and reports unhonored settings (used by
`ryoku wm use` and the Hub before a switch). Hyprland adds `plugins`;
niri declares no plugin system in `caps`.

## Client (`ryoku/wm/client.go`)

`Open()` resolves the active provider without running it; `OpenNamed`
targets a compositor the caller is not in. `Caps` probes once and caches
(failures re-probe). `Can` gates affordances. `Act` checks the action
capability first (`ErrUnsupported`, typed). `Apply` hands the store by
path; `DryRun` previews any provider; `Preview` is live-only, gated on
`liveConfigEval`. `State`, `Defaults` (raw), `Schema` (raw rows), `Binds`
(single legend: shared catalogue resolved plus exclusives), `Session`
(raw entry), `Plugins` (gated), `Watch/WatchKinds` (NDJSON, caller owns
restart), `ApplyOutputs` (gated on `monitorConfig`).

## Capabilities that shape the shell

Shared: `animations focusHistory keyboardLayoutSwitch layerRules
monitorConfig nightLight outputPower paletteBorder sessionExit touchpadToggle
windowFloat windowRules windowWorkspaceMap workspaceMoveToOutput workspaces`.
Hyprland-only: `configReload cursorSet focusGrab globalShortcuts
liveConfigEval plugins screenShader specialWorkspace submap tiledLayout
windowGeometry`. niri-only: `nativeOverview`. Consequences live in the
shell as caps checks: native overview stands the shell overview aside;
missing `windowGeometry` removes window picking; missing
`globalShortcuts` routes keybinds through `ryoku-shell <verb>` spawns;
file-only config applies on save instead of live preview; lid/panel
handoff differs per compositor (Hyprland binds lid to `ryoku-clamshell`,
niri sends switch events). Nightlight, touchpad, border-follows-palette,
output cycle/enable, `window.summon`, and game mode each ride named
actions documented in `docs/compositors.md`.

## Config homes and switching

`ryoku/wm/detect.go` is the only mapping of compositor to directory
(`hypr/`, `niri/`), entry (`hyprland.lua` vs `config.kdl` include chain
with generated `settings`/`rebinds` plus seeds and `user` last-word),
and portal backend. `ryoku wm use <name>` previews (carries, unhonored,
kept settings), installs `ryoku-desktop-<name>` as a plain transaction
(rollback-able), and never deletes `wm.<name>.*` settings. Leaf scripts a
provider owns (`ryoku-monitor`, `ryoku-workspace`, provider cursor tools)
live in that provider payload; anything called by bare name on every
compositor lives in `ryoku/shell/scripts/` or `system/hardware/`.

## Adding a window manager (checklist)

1. `ryoku/wm/<name>/` implementing the verbs; mirror the nearest
   provider; pin emitted argv in a test.
2. Register in `ryoku/wm/detect.go` (name, env handle, config dir, seeds).
3. `ryoku/<name>/` shipped config payload.
4. Its `schema` (exclusive rows = its whole Hub presence).
5. `release/packages/ryoku-desktop-<name>/` (provides-compositor,
   Xwayland, screencast deps).
6. Offer it in the installer compositor question.

New compositor knowledge = new capability or `caps` field, never a name
test elsewhere.
