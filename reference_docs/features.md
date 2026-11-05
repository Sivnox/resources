# Ryoku feature inventory

A categorized inventory of every feature the repo ships, with a one-line
description and the concerned files. Use this to pick and choose what to carry
into your own shell. Paths are relative to the repo root.

Third-party applications (kitty, neovim, nautilus, mpv, chromium, zen, …) are
**not** listed here — they are pulled from Arch/AUR by the installer and the
umbrella package, and their source lives elsewhere.

---

## 1. Shell core & control plane

| Feature | Description | Files |
|---|---|---|
| Shell daemon (`ryoku-shell`) | Single Go control plane: Unix socket, one-line verbs, topic pub/sub, call RPC, surface supervisor (`qs -c shell`), clipboard/tray/weather/music/calendar/power/network/nightlight workers, sleep/wake guard, qylock handoff | `ryoku/shell/ipc/*.go` |
| Shell IPC client | `ryoku-shell <verb>` thin client forwarding to the daemon socket; standalone verbs (`theme catalog`, `matugen-preview`, `icons`, `browser-host`) | `ryoku/shell/ipc/main.go` |
| Quickshell shell root | Single in-process QML instance rendering every surface (bar, dock, launcher, overview, wallpaper, desktop, stage, visualizer, OSDs, notifications, capture, confirm) with per-monitor ShellState slices | `ryoku/shell/quickshell/shell/shell.qml` |
| Surface toggle bus | `toggleSurface(id)` single mapping reached both by Hyprland global-shortcuts and by `ryoku-shell <id>` spawns (niri) | `shell.qml:453`, `shell.qml:513` |
| Shell state | Per-monitor screen slices (launcherOpen, overviewOpen, clipboardOpen, barRevealed, visualizerOverlay, sessionAction…) | `ryoku/shell/quickshell/shell/services/ShellState.qml` |
| Live config | `~/.config/ryoku/shell.json` watched live by QML Config singleton; daemon settingsStore is sole writer with validation and broadcast | `services/Config.qml`, `ipc/settings.go` |
| Service registry | ~50 QML singletons (audio, network, battery, clipboard, tray, notifs, weather, calendar, nightlight, polkit, keyring, stage, dock…) via subscribe/call sockets | `services/*.qml`, `services/qmldir` |
| Surface lazy loading | `LazyLoader` overlays born closed, revealed one tick after build, destroyed after a hold timer (15s toggles, 20s flows) | `shell.qml:280-444` |
| Reload cover | `ryoku-reload-cover` begin/finish protocol so `ryoku reload` does not show a black screen | `shell/scripts/ryoku-reload-cover`, `shell.qml:72-131` |

## 2. Bar & frame

| Feature | Description | Files |
|---|---|---|
| Frame bar | Four-edge rail (top/left/bottom/right) with overlay PanelWindow, input-mask union, fullscreen hiding, per-monitor reveal | `modules/bar/Frame.qml`, `modules/bar/Bar.qml`, `modules/bar/FrameRail.qml` |
| Frame menu host | Single-open menu manager driving all bar popouts; voice, keyring, polkit, music, bluetooth, battery, network, sysmon, audio, screenshot islands | `modules/bar/FrameMenuManager.qml`, `modules/bar/popouts/Popout.qml` |
| Bar styles (sumi) | Built-in monochrome rail scene | `modules/bar/barstyles/sumi/` |
| Bar styles (qsbar) | Full widget catalog, panels (popout bodies), control center, IPC router | `modules/bar/barstyles/qsbar/` |
| Bar styles (iris) | Python-based bar style with singletons (system, theme, i18n), timers, widgets, quickactions | `modules/bar/barstyles/python/` |
| Bar styles (kairos / nomarchy / orpheus) | Additional packaged bar style scenes | `modules/bar/barstyles/kairos/`, `nomarchy/`, `orpheus/` |
| Bar products catalogue | `BarProducts` singleton mapping style id → scene URL; drives style switching | `services/BarProducts.qml` |
| Dock | Per-monitor dock surface opposite the band edge; reveal state machine (autohide / pointer / peek strip / dragging / menu); context menu hover previews | `modules/dock/DockSurface.qml`, `services/Dock.qml`, `services/DockPreview.qml` |

## 3. Compositors & WM seam

| Feature | Description | Files |
|---|---|---|
| WM provider contract | `caps | state | watch | act | apply | defaults | schema | binds | outputs | session | environment` verbs over `ryoku-wm-<name>` binaries | `wm/client.go`, `wm/caps.go`, `wm/action.go`, `wm/detect.go` |
| Hyprland provider | Lua-authored config (`hyprland.lua` + modules), live config eval, global shortcuts, focus grab, plugins, screen shader, submap, special workspace | `wm/hyprland/*.go`, `ryoku/hyprland/*.lua`, `ryoku/hyprland/modules/` |
| niri provider | KDL-authored config (`config.kdl` include chain), file-only config, native overview, no plugin system | `wm/niri/*.go`, `ryoku/niri/*.kdl` |
| wayfire provider | wayfire.ini + portals config | `ryoku/wayfire/` |
| Compositor switching | `ryoku wm use <name>` with preview (unhonored report), install/remove variant packages, kept settings survive | `wm/reclaim.go`, `cli/internal/updater/`, `hub/backend/wm.go` |
| QML compositor bridges | Hyprland-only `GlobalShortcutBridge`, `FocusGrabBridge` loaded only when `caps` allows | `wm/hyprland/qml/` |
| Capability-gated QML | `Wm` singleton asks `caps.*`, never compositor names; `Wm.caps.nativeOverview` stands shell overview aside on niri | `ui/Singletons/Wm.qml` |

## 4. Hub (Ryoku Settings)

| Feature | Description | Files |
|---|---|---|
| Settings GUI | FloatingWindow with grouped nav rail, global fuzzy search, live editors for displays, appearance, device lighting, lockscreen, animations, input, keybinds, rules, autostart, environment, shell, widgets | `hub/quickshell/` |
| Hub backend | Go data plane (`ryoku-hub`): reads keybind legend from live Hyprland config, generates compositor overlays (`settings.lua/kdl`, `rebinds.lua/kdl`), persists TOML state, speaks OpenRGB SDK for per-device lighting | `hub/backend/*.go` |
| Neutral desktop store | `~/.config/ryoku/desktop.json` single writer; provider `Apply` writes generated compositor halves; preview without persist; cursor theme apply | `hub/backend/desktop.go` |
| Lock skin | Sets qylock theme and SDDM greeter theme (copies into `/usr/share/sddm/themes/ryoku`, writes `99-ryoku.conf`) | `hub/backend/lock.go` |
| Display outputs | Display editor builds layout; `Client.ApplyOutputs` persists + applies live; preview for file-only compositors | `hub/backend/outputs.go` |
| GPU tuning | Per-profile driver scripts, nvidia-open vs legacy branch, `ryoku-gpu mode` (offload/sync/vfio), `ryoku-power` battery ceiling | `hub/backend/gpu*.go`, `system/hardware/drivers/` |
| Lighting | OpenRGB / QMK keyboard and mouse per-device lighting | `hub/backend/lighting.go`, `openrgb.go`, `qmk.go` |
| Themes / schemes | Download/apply colour schemes, `theme.json`, `colors.json` authoring, GTK blanking for theme-apps off | `hub/backend/schemes.go`, `matugen.go` |
| Rice import / publish | Import exported desktops, publish to riyostore catalogue, undo/redo | `hub/backend/rice*.go`, `import*.go`, `ricepublish.go` |
| Audio / CPU tune | `cputune`, power-profiles-daemon bridge over daemon socket | `hub/backend/cputune.go`, `daemonclient.go` |
| Compositor control page | Compositor schema rows rendered generically; binds cheatsheet section | `hub/backend/wm.go`, `CompositorControl.qml` |

## 5. Theming & appearance

| Feature | Description | Files |
|---|---|---|
| Wallpaper-driven palette | matugen generates `~/.cache/ryoku/colors.json` on every wallpaper/scheme change; fans into kitty, Hyprland, GTK, Qt, btop, share-picker | `shell/matugen/*`, `ipc/matugen.go`, `ipc/theming.go` |
| Theme apps toggle | matugen fans palette into GTK/GUI apps when on; stock when off | `shell/matugen/apps.toml`, `hub/backend/schemes.go` |
| Colors / scheme QML | `Scheme` singleton reads `colors.json` + `theme.json`; accent/lstar ramps | `services/Scheme.qml`, `ui/Singletons/` |
| Brand type stack | Space Grotesk, Rubik VF, Readex Pro, Inter, JetBrains Mono Nerd, Noto | `system/packages/base.packages`, `release/packages/otf-space-grotesk/…` |
| Ryodecors decor art | 1-bit bone-on-transparent decor baked by `bin/art/ryodither`, kept current by doctor | `assets/ryodecors/`, `bin/art/ryodither` |

## 6. Wallpaper & visuals

| Feature | Description | Files |
|---|---|---|
| Ryogami wallpaper daemon | Wallpaper state, transitions, live video, depth surface, catalog + thumbnails; shell reads via `ryogami.sock` topic | `ryoku/shell/ryogami/daemon/`, `modules/wallpaper/Wallpaper.qml` |
| Wallpaper picker UI | `Super+W` wall-ui selector writing through ryogami daemon | `ryoku/shell/ryogami/wall-ui/` |
| Overview backdrop | Blurred wallpaper for compositor overview, capability-gated (`overviewBackdrop`) | `modules/wallpaper/OverviewBackdrop.qml` |
| Stage (cut-out / parallax) | Wallpaper layers with depth, front placement, music bounce, idle drift; rendered inside Desktop; verbs via `ryoku-shell stage <verb>` | `modules/stage/`, `ipc/stage.go` |
| Visualizer (cava spectrum) | Up to 4 boxes with geometry/rotation/overhang, bars or line styles, cava + python wavecapture, shared Spectrum, Placer drag surface | `modules/visualizer/` |
| Desktop widget layer | Bottom widget surface hosting clock, calendar, music, all-in-one, stats, weather, notes, third-party plugins; glass sampling mirror | `modules/desktop/Desktop.qml`, `services/Registry.qml` |

## 7. Launcher & search

| Feature | Description | Files |
|---|---|---|
| App launcher | Search row + dashboard/grid/list results from 12 providers (actions, apps, calc, find, mpris, packages, radio, recent, script, snippets, web, windows) via one Dispatcher | `modules/launcher/` |
| Launcher variants | `main`, `hero` (full-screen with window rail, action shelf), `okshell`, `kairos` registered in `catalog.json` | `modules/launcher/variants/*` |
| Global fuzzy search | Hub-level search across settings rows | `hub/quickshell/` |

## 8. Notifications & OSD

| Feature | Description | Files |
|---|---|---|
| Notification column | Per-monitor toast column with slide animations, lifespan countdown, actions, dnd suppression, history vs popups split | `modules/notifications/`, `services/Notifs.qml` |
| OSDs | Volume / mic / brightness toasts with threshold icons and arm grace; keyboard layout OSD on signature change | `modules/osd/`, `services/OsdFeed.qml` |

## 9. Capture & recording

| Feature | Description | Files |
|---|---|---|
| Screenshot (Capture) | Shots (all/monitor/window/region), grim + wl-copy detached, shutter sound, notify, beautify handoff to waifu2x | `services/Capture.qml`, `modules/capture/CaptureOverlay.qml` |
| Screen recording | gpu-screen-recorder / wf-recorder backends, region pick via slurp, countdown, studio mode, pgrep polling | `services/Recorder.qml`, `shell/scripts/ryoku-cmd-*` |
| Region / camera / keypress overlays | Per-flow overlays (region box, self-view camera bubble, chord display) with 20s holds | `modules/capture/` |

## 10. Media & music

| Feature | Description | Files |
|---|---|---|
| Media keys / players | playerctl, Mpris providers, now-playing widget following ryotunes | `services/Music.qml`, `services/Media.qml` |
| Visualizer audio | cava pipewire source, waveform via python wavecapture | `modules/visualizer/Spectrum.qml`, `Waveform.qml` |
| Voice dictation | Super+` tap-to-toggle Voxtype transcription, mic wave surface, wtype injection | `ipc/daemon.go voice()`, `services/VoiceBars.qml`, `system/packages/aur.packages` (voxtype-bin) |

## 11. Communications & phone

| Feature | Description | Files |
|---|---|---|
| Phone integration | kdeconnect, scrcpy, android-tools, avahi, v4l2loopback — enabled via Ryoku Hub | `system/packages/phone.packages`, `apps/` |

## 12. System tools & utilities

| Feature | Description | Files |
|---|---|---|
| CLI (`ryoku`) | `update`, `rollback`, `snapshots`, `status`, `materialize`, `reload`; orchestrates pacman, yay, snapper | `ryoku/cli/*.go` |
| GlazePKG (gpk) | Ryoku package manager | `release/packages/gpk/`, `apps/` |
| RyoManager | System monitor and task manager | `release/packages/ryomanager/`, `ryoku/apps/ryomanager/` (if present) |
| Prowl | Agent knowledge-vault index tooling | `release/packages/prowl/`, `.prowl/` |
| System info | `ryoku-sysinfo`, `ryoku-profile-stats` | `shell/scripts/` |
| App launcher helpers | `ryoku-app`, `ryostage`, stash `.sh` helpers | `shell/scripts/` |
| LocalSend / stash | LAN file sharing, file stash, compress/install file pickers | `shell/scripts/stash-*`, `localsend.sh` |
| Fastfetch | System summary for launcher | `apps/fastfetch/` |
| Terminal shell configs | fish (recommended), zsh (oh-my-zsh), bash (blesh) stacks | `apps/fish/`, `apps/zsh/`, `apps/terminal-shell/` |

## 13. Gaming & performance

| Feature | Description | Files |
|---|---|---|
| Game mode | `decoration.gameMode` live-config-eval strip decorations + reload; power-profiles boost | `ipc/daemon.go`, `shell.qml syncGameMode` |
| Gamescope / gamemode / mangohud | Nested micro-compositor, perf governor, FPS overlay | `system/packages/base.packages` |
| Controller support | xpadneo-dkms (Bluetooth Xbox), game-devices-udev (27 vendors, battery reporting), dualsensectl opt-in | `system/packages/base.packages`, `release/packages/` |
| Power | `ryoku-power` battery charge ceiling + PCIe link power; `ryoku-idle` laptop-gated hypridle; `ryoku-hw-laptop` shared laptop detector | `system/hardware/power/`, `docs/power.md` |

## 14. Containers & virtualization

| Feature | Description | Files |
|---|---|---|
| Docker / Cobalt engine | Privileged door with polkit grant, single `ryoku-cobalt` container, no passthrough | `system/containers/`, `release/packages/docker*` |
| QEMU / VM | qemu-desktop, edk2-ovmf, virglrenderer, looking-glass, quickemu, spice-gtk (opt-in via doctor) | `system/packages/base.packages`, `ryoku-desktop` optdepends |
| ryovm | Quickemu-based VM wrapper | `apps/ryovm/` |

## 15. AI & agent OS (Rashin)

| Feature | Description | Files |
|---|---|---|
| Rashin daemon | Local knowledge vault at `~/.local/share/ryoku/rashin/`, embedded dashboard on `127.0.0.1:3600`, Hermes bridge over ACP | `ryoku/rashin/backend/` |
| Rashin terminal | `rashin` command: natural language → command plan with fish prompt weave | `ryoku/rashin/` |
| Rashin Hub page | Enable, one-click Hermes setup, open dashboard | `hub/quickshell/pages/RashinPage.qml` |

## 16. Updates, delivery & packaging

| Feature | Description | Files |
|---|---|---|
| ryoku update | Snapper pre-snapshot, pacman + AUR, materialize, reload, post-snapshot | `cli/internal/updater/update.go` |
| ryoku materialize | Lay `/usr/share/ryoku/config` → `~/.config`; clobber owned, seed absent, fork hand-edits to `user_edits`, prune dropped | `cli/internal/updater/materialize.go` |
| ryoku doctor | Convergent reconcilers for stateful drift (boot, portals, config tree, plugins, border pin, lockscreen, shells, rice emblem…) | `cli/internal/doctor/*.go` |
| Deploy (dev) | `deploy.sh`: build binaries, install QML modules (`Ryoku.Blobs`, `Ryoku.Ui`, `PluginKit`, `FrameBars`, `Ryoku.Wm.Hyprland`), lay compositor payload, app configs, live cutover | `ryoku/shell/deploy.sh` |
| Signed [ryoku] repo | `build-repo.sh` runs makepkg, signs artifacts, repo-adds into `out/` | `release/packages/*`, `release/repo/` |
| Umbrella package | `ryoku-desktop` version-pins all monorepo components + compositor virtual | `release/packages/ryoku-desktop/PKGBUILD` |
| i18n | 35-language catalog, extractor/translator (`ryoku-i18n`), QML `I18n` singleton, Go runtime | `ryoku/i18n/` |

## 17. Installer & ISO

| Feature | Description | Files |
|---|---|---|
| Installer TUI | Bubble Tea wizard: keyboard, locale, tz, network, hardware profile, graphics, WM, browser, shell, apps, disk, layout, user, encryption; safety gates (BIOS, Secure Boot, live-medium, wipe ack, online) | `installation/tui/*.go` |
| Installer backend | Bash orchestrator: preflight → partition → filesystems → mount → pacstrap → configure → bootloader → post (AUR, finalize, snapshots); `RYOKU_*` contract | `installation/backend/ryoku-install`, `installation/backend/lib/*.sh` |
| Disk strategies | `whole` (wipe, ESP + Btrfs root) and `alongside` (dual-boot, 2 GiB boot + XBOOTLDR/dedicated ESP, chainload existing OS) | `installation/backend/lib/disk.sh` |
| Offline install | ISO bakes full package closure into `file:// [offline]` repo; no network required | `installation/iso/offline-repo.sh`, `installation/backend/lib/offline.sh` |
| CachyOS edition | CachyOS kernel layer, mirrors, settings, gaming meta; replaces `linux`/`linux-headers`/`yay-bin` | `system/packages/cachyos.packages`, `cachyos.replaces`, `installation/backend/lib/cachyos.sh` |
| Live ISO | archiso profile, autologin to cage+foot+ryoku-tui, systemd-boot/syslinux entries, mkarchiso | `installation/iso/` |
| no-ISO installer | `ryoku-shell-installer/` — curl-able bootstrap converting existing Arch to Ryoku | `ryoku-shell-installer/` |

## 18. Lockscreen & greeter

| Feature | Description | Files |
|---|---|---|
| qylock lockscreen | Quickshell lockscreen via `WlSessionLock` (same on Hyprland and niri), clockwork theme | `ryoku/lockscreen/qylock/` |
| SDDM greeter | Wayland greeter (weston kiosk) rendering qylock skin; setup installs theme + `99-ryoku.conf` | `ryoku/lockscreen/sddm/`, `hub/backend/lock.go` |
| Fail-closed sleep | Daemon owns login1 suspend transaction, qylock-before-suspend, output wake, lighting restore | `ipc/sleepwake.go`, `ipc/lock.go` |

## 19. Accessibility & input

| Feature | Description | Files |
|---|---|---|
| Keyboard layout switch | `keyboardLayoutSwitch` cap; xkb layout feed; keyboard OSD | `wm/caps.go`, `services/KeyboardLayout.qml`, `modules/osd/KeyboardOsdWindow.qml` |
| Touchpad toggle | `input.touchpad on|off|toggle`; FN key drive; state file on niri | `wm/caps.go` (`touchpadToggle`) |
| Keep-awake / caffeine | External `systemd-inhibit` bridge + compositor IdleInhibitor | `shell.qml syncCaffeine`, `shell/scripts/ryoku-cmd-caffeine` |
| Fingerprint | pam-fprint-grosshack PAM module for lock/greeter touch-or-type | `system/packages/aur.packages` (pam-fprint-grosshack) |

## 20. Networking & connectivity

| Feature | Description | Files |
|---|---|---|
| NetworkManager + iwd | Default WiFi backend; wpa_supplicant alternate for WPA3-SAE | `system/packages/base.packages` |
| Connections kill switch | nftables fail-closed firewall | `system/packages/base.packages` (nftables) |
| Bluetooth | BlueZ, openrgb lighting, Broadcom `.hcd` firmware | `system/packages/base.packages`, `release/packages/broadcom-bt-firmware` |
| LocalSend | LAN file stash sharing | `shell/scripts/localsend.sh`, `aur.packages` |

---

### How to use this list

Each feature above maps to files you can copy or reference. Ryoku's cardinal rule is **one concern per file, one source of truth** — if a feature needs something that already exists, reuse it rather than duplicating. The compositor seam (`ryoku/wm/`) is the cleanest boundary: anything compositor-specific lives behind `caps`/`act`/`apply`, so you can drop in a different window manager without touching the shell.

For the build/packaging side, see `release/packages/` (31 PKGBUILDs), `system/packages/` (base/hardware/dev/aur sets), and `installation/` (TUI + backend + ISO).
