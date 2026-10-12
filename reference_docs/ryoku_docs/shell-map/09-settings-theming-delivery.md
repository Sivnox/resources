# 09: Settings, theming, delivery

## Who writes what

- `~/.config/ryoku/shell.json`: daemon `settingsStore` only
  (`ryoku/shell/ipc/settings.go`). Hub and QML send `settings.patch`/
  `settings.reset`; the store validates (`general|theme|bars|menus|
  notifications|wallpaper`, rest passthrough), persists, broadcasts on
  the `settings` topic, and schedules retheme/font-apply. QML `Config`
  watches the file and retunes live, no reload.
- `~/.config/ryoku/desktop.json`: Hub backend owns (`ryoku/hub/backend/
  desktop.go`, `withDesktopLock`); provider `Apply` writes the generated
  compositor halves; `previewDesktop` drafts without persisting;
  `wm.<provider>` exclusives split from neutral `desktop.*` via provider
  `Defaults()`.
- `~/.config/ryoku/{visualizer,widgets,matugen}.json`, `theme.json`,
  `brand.json`: daemon or Hub writes per file (matugen knobs are inputs;
  the daemon is the sole palette renderer).
- `~/.config/ryoku/hub.toml`: Hub's own UI prefs (`config.go`).
- Login shell: `shellpref.go` (loader lines in rc files, `ryoku-prompt`
  file, `usermod -s` via pkexec, session env sync, `desktop.env SHELL`,
  re-apply + reload).

## Theming pipeline (wallpaper-driven)

Daemon `paintWorker`: wallpaper tone always; static theme or
matugen-follows palette into `~/.cache/ryoku/colors.json`; render
`matugen/config.toml` (core: kitty, hypr colors, btop, share-picker,
qt6ct, kde) plus `apps.toml` only when Theme-apps is on (gtk, discord,
obs, zed, telegram, steam, terminals, ...); reload (gsettings
color-scheme/accent/gtk, kitty USR1, QML `theme.reload` nudge);
recolor borders via `act decoration.borderColors` unless the store pins
a fixed color; compositor reload where `configReload`. QML reads
`colors.json` (`Scheme.qml` files, `qsbar/Theme.qml`, visualizer scheme,
browser bridge). Hub fixed schemes author `colors.json` + `theme.json`
then reuse the same renderer; Hub never renders palettes itself.

## Delivery (repo to system, one way)

- `ryoku materialize` (`ryoku/cli/internal/updater/`): lays
  `/usr/share/ryoku/config` into `~/.config`. Shipped files clobber;
  seeds (`monitors/gpu/keyboard/user`, editor configs, `nvim/*`) seed
  only; hand-edited shipped files fork to `~/.config/ryoku/user_edits/`
  before clobber (except `quickshell/*` and live-owned); previous
  manifest pruned (`quickshell/` converged directly); `overlayUserEdits`
  last; missing generated halves re-applied best-effort. User files
  (`user.lua`, `user.fish`) untouched.
- `ryoku update`: snapshot, packages (Ryoku set; `--system` adds
  system/AUR/flatpak), materialize, reload, doctor, finalize. Checkout
  boxes use channel-pull + deploy instead; packaged boxes hand off to
  the new binary `--stage2`.
- `ryoku doctor`: convergent reconcilers for stateful drift (boot,
  portals, config tree, plugins, border pin, lockscreen, shells, ...).
- `ryoku/shell/deploy.sh` (dev loop): builds/installs `ryoku-shell`,
  `ryoku-wm-*`, scripts, ryogami + wall-ui seed, Hub/Rashin/CLI,
  privileged helpers, QML modules (`Ryoku.Blobs` Qt-stamped, `Ryoku.Ui`,
  i18n, `PluginKit`, `FrameBars`, `Ryoku.Wm.Hyprland`), `~/.config/
  quickshell/*`, portals, compositor payload swap (active tree staged +
  `mv`; inactive trees laid + applied), app configs, generated re-emit,
  user-edits overlay, live cutover (lid guard, qylock stage, quiesce,
  shell restart, `config.reload`, idle/clamshell/ryogami) or log-out
  notice.
- Packaging: `release/packages/` builds the signed `[ryoku]` repo;
  `ryoku-desktop` pins components and lays the base config. Installer
  adds the repo, installs the umbrella, `materialize` seeds the user.

Live machines are deployment targets only; edits flow repo -> package ->
`materialize`/`doctor`, never back. Normative details: `docs/updates.md`,
`docs/cli.md`, `docs/development.md`.
