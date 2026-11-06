# How compositors launch and manage the keyboard process

Research for [#3](https://github.com/asluizh/tessera/issues/3). Checked against upstream `master`/`main` on Nov 5, 2026.

## Answer

- **KWin launches the keyboard itself.** The keyboard is a child process of `kwin_wayland`, connected through an inherited socket (`WAYLAND_SOCKET`). Only that connection may bind `zwp_input_method_v1` and `zwp_input_panel_v1`. A copy started any other way gets no IM v1 and can't work as the KWin keyboard.
- **KWin chooses the keyboard from a .desktop file.** The Virtual Keyboard KCM lists every application whose .desktop file has `X-KDE-Wayland-VirtualKeyboard=true` and writes the chosen file's path to `kwinrc [Wayland] InputMethod`. KWin reads that file's `Exec=` and runs it. KWin watches `kwinrc`, so changing the setting restarts the keyboard live.
- **KWin restarts the keyboard only after a crash exit**, meaning death by a signal. It gives up after 5 crashes that each come within 20 s of the previous one. A normal exit with any exit code, including Rust's default panic exit (code 101), is not restarted.
- **wlroots compositors (Sway, Hyprland, ...) launch nothing.** The user starts the keyboard with `exec` or `exec_always` (Sway), `hl.exec_cmd` or the older `exec-once` (Hyprland), a systemd user unit, or XDG autostart. Each seat accepts only one IM v2 client. A second one gets `zwp_input_method_v2.unavailable`.
- **Phosh starts its keyboard through gnome-session or systemd.** The keyboard has an XDG autostart .desktop file with `X-GNOME-Provides=inputmethod` and `X-GNOME-AutoRestart=true`. Stevia also ships a systemd unit with `Type=dbus`, `BusName=sm.puri.OSK0` and `Restart=on-failure`. Single-instance is enforced by owning the D-Bus name `sm.puri.OSK0`.
- **Implication for single-instance handling:** neither protocol makes a second copy fail cleanly, so Tessera has to enforce single-instance itself. See [Implications for Tessera](#implications-for-tessera).

## KWin (Plasma 6)

### Selection: the KCM and `X-KDE-Wayland-VirtualKeyboard`

- The KCM's model queries `KApplicationTrader` for every service with `X-KDE-Wayland-VirtualKeyboard` set to true. It adds a "None" entry at the top. [`src/kcms/virtualkeyboard/kcmvirtualkeyboard.cpp`](https://invent.kde.org/plasma/kwin/-/blob/master/src/kcms/virtualkeyboard/kcmvirtualkeyboard.cpp) (`VirtualKeyboardsModel` constructor)
- The value stored is the absolute path of the .desktop file, found with `QStandardPaths::locate(QStandardPaths::ApplicationsLocation, <desktopEntryName>.desktop)`, so the file must be installed under `share/applications/`. (Same file, `DesktopFileNameRole`.)
- It is stored in `kwinrc`, group `[Wayland]`, key `InputMethod`, of type Path. [`virtualkeyboardsettings.kcfg`](https://invent.kde.org/plasma/kwin/-/blob/master/src/kcms/virtualkeyboard/virtualkeyboardsettings.kcfg)
- The visibility mode (`VirtualKeyboardMode`, with the legacy `VirtualKeyboardEnabled`) is stored in the same group. The KCM sets it through D-Bus: `org.kde.KWin` `/VirtualKeyboard`, interface `org.kde.kwin.VirtualKeyboard`, which has the properties `available`, `mode`, `active`, `visible` and `activeClientSupportsTextInput` and the methods `forceActivate()` and `willShowOnActive()`. [`src/virtualkeyboard_dbus.h`](https://invent.kde.org/plasma/kwin/-/blob/master/src/virtualkeyboard_dbus.h), [`src/inputmethod.cpp`](https://invent.kde.org/plasma/kwin/-/blob/master/src/inputmethod.cpp) (constructor)
- The .desktop files in the wild are minimal:
  - Plasma Keyboard: `Exec=plasma-keyboard`, `X-KDE-Wayland-VirtualKeyboard=true`, `NoDisplay=true`. [`src/org.kde.plasma.keyboard.desktop`](https://invent.kde.org/plasma/plasma-keyboard/-/blob/master/src/org.kde.plasma.keyboard.desktop)
  - Maliit: `Exec=maliit-keyboard`, `X-KDE-Wayland-VirtualKeyboard=true`, `NoDisplay=true`. [`com.github.maliit.keyboard.desktop`](https://github.com/maliit/keyboard/blob/master/com.github.maliit.keyboard.desktop)

### Startup and live change

- `ApplicationWayland::startSession()` creates a `KConfigWatcher` on `kwinrc`. If `kwin_wayland --inputmethod <cmd>` was given, that command is used. Otherwise KWin calls `refreshSettings([Wayland], {"InputMethod"})`. [`src/main_wayland.cpp`](https://invent.kde.org/plasma/kwin/-/blob/master/src/main_wayland.cpp)
- `refreshSettings` opens the path with `KDesktopFile` and passes `desktopGroup().readEntry("Exec")` to `InputMethod::setInputMethodCommand()`. The same function runs on every later `kwinrc` change to `InputMethod`, so switching keyboards in the KCM takes effect immediately. (Same file.)
- `setInputMethodCommand()` does nothing if the command is unchanged. Otherwise it calls `startInputMethod()`, which first calls `stopInputMethod()`: SIGTERM, then a wait, then SIGKILL if the process hasn't exited, then the IM connection is destroyed. An empty command ("None") just stops the keyboard. [`src/inputmethod.cpp`](https://invent.kde.org/plasma/kwin/-/blob/master/src/inputmethod.cpp)
- KWin splits `Exec` with `KShell::splitArgs` and runs the first word as the program. It does no `.desktop` field-code expansion, so `Exec=` must not contain `%u`, `%f` and similar codes. (Same file, `startInputMethod`.)
- KWin starts the keyboard only once the session starts, which is after the Workspace exists. (Same files.)

### What KWin passes to the process

From `InputMethod::startInputMethod()` in [`src/inputmethod.cpp`](https://invent.kde.org/plasma/kwin/-/blob/master/src/inputmethod.cpp):

- The environment is `kwinApp()->processStartupEnvironment()` plus:
  - `WAYLAND_SOCKET=<fd>`: one end of a socketpair from `WaylandServer::createInputMethodConnection()`. `FD_CLOEXEC` is cleared in the child just before exec, and the parent closes its copy.
  - `QT_QPA_PLATFORM=wayland`
  - `MALIIT_ENABLE_ANIMATIONS=0` (KWin animates the panel itself)
  - `WAYLAND_DEBUG=1`, only if KWin itself runs with `KWIN_IM_WAYLAND_DEBUG=1`
- Arguments are whatever `Exec=` contains. KWin adds none.
- The child's stderr is forwarded to KWin's own stderr, so it ends up in the journal (`QProcess::ForwardedErrorChannel`).

### Privilege: only the launched connection gets IM v1

The Wayland server's `allowInterface()` returns true for every global when `client == waylandServer()->inputMethodConnection()`. For every other client it hides `zwp_input_panel_v1` and `zwp_input_method_v1`. [`src/wayland_server.cpp`](https://invent.kde.org/plasma/kwin/-/blob/master/src/wayland_server.cpp)

Two consequences:

- Starting the binary by hand (from a terminal, autostart or systemd) gives a client with no IM v1. Plasma Keyboard detects this and exits with code 1 and the message: "Cannot run plasma-keyboard standalone. You can enable it in Plasma's System Settings app, on the 'Virtual Keyboard' page." [`src/main.cpp`](https://invent.kde.org/plasma/plasma-keyboard/-/blob/master/src/main.cpp)
- Privilege belongs to that one socket, not to the process. The keyboard must create its panel surface and IM objects on the connection taken from `WAYLAND_SOCKET`. A second connection opened through `WAYLAND_DISPLAY`, for example by a toolkit that opens its own, is unprivileged. libwayland and wayland-rs (`Connection::connect_to_env`) both prefer `WAYLAND_SOCKET` when it is set.

### Crash and exit handling

From the `QProcess::finished` handler and `init()` in [`src/inputmethod.cpp`](https://invent.kde.org/plasma/kwin/-/blob/master/src/inputmethod.cpp):

- KWin restarts the keyboard only on `QProcess::CrashExit`, meaning the process was killed by a signal (SIGSEGV, SIGABRT and so on).
- Each crash increments a counter and restarts a 20 s single-shot timer that resets the counter when it fires. KWin restarts on crashes 1 to 4. On the 5th crash in a row, each within 20 s of the previous one, it logs "Input Method keeps crashing, please fix" and stops.
- A normal exit with any exit code is not restarted. This includes `exit(1)` and the exit code 101 that a Rust panic produces under the default `panic = "unwind"`. The keyboard stays dead until the user changes the KCM setting again or KWin restarts. `panic = "abort"` produces SIGABRT, which counts as a crash and is restarted.
- When the screen is about to lock, KWin hides the panel. It does not stop the process.

## wlroots family and Hyprland

### No compositor-managed launch

None of these compositors has a setting that names an on-screen keyboard. The keyboard is an ordinary client the user starts:

- **Sway:** `exec <cmd>` runs once at startup. `exec_always <cmd>` runs again after every `reload`, which spawns duplicates unless the program refuses a second copy. [`sway/sway.5.scd`](https://github.com/swaywm/sway/blob/master/sway/sway.5.scd)
- **Hyprland:** the current wiki documents the Lua config, where you call `hl.exec_cmd(...)` inside `hl.on("hyprland.start", ...)`. Older hyprlang configs use `exec-once`. [`configuring/core/autostart.md`](https://github.com/hyprwm/hyprland-wiki/blob/main/content/configuring/core/autostart.md)
- **Hyprland and systemd:** Hyprland now starts `hyprland-session.target` and `graphical-session.target` itself. [`configuring/extra/systemd.md`](https://github.com/hyprwm/hyprland-wiki/blob/main/content/configuring/extra/systemd.md)
- **UWSM:** under UWSM, XDG autostart is handled by systemd, and the wiki recommends shipping user units bound to `graphical-session.target` instead of exec lines. [`useful-utilities/uwsm.md`](https://github.com/hyprwm/hyprland-wiki/blob/main/content/useful-utilities/uwsm.md)
- **wvkbd:** no unit, no D-Bus and no single-instance logic. It is shown and hidden by signals (SIGUSR1 hides, SIGUSR2 shows, SIGRTMIN toggles), so `pkill -SIGRTMIN wvkbd` is the control path. `--hidden` starts it hidden. [`README.md`](https://github.com/jjsullivan5196/wvkbd/blob/master/README.md), [`main.c`](https://github.com/jjsullivan5196/wvkbd/blob/master/main.c)

### One IM v2 client per seat

- **Sway:** if the seat already has an input method, the new one is sent `unavailable` and Sway logs "Attempted to connect second input method to a seat". [`sway/input/text_input.c`](https://github.com/swaywm/sway/blob/master/sway/input/text_input.c)
- **Hyprland:** `CInputMethodRelay::onNewIME` does the same and logs "Cannot register 2 IMEs at once!". [`src/managers/input/InputMethodRelay.cpp`](https://github.com/hyprwm/Hyprland/blob/main/src/managers/input/InputMethodRelay.cpp)
- **wvkbd** ignores `unavailable` (its handler is empty), so a duplicate wvkbd keeps running without auto-show. (`main.c`)
- **Fcitx5 and IBus** compete for the same slot, so whichever binds first wins.

Nothing prevents a second layer-shell surface or virtual-keyboard client. A duplicate Tessera that ignored `unavailable` would still draw a second keyboard that can type.

## Phosh (phoc)

- **Squeekboard** ships an XDG autostart-style .desktop file with `OnlyShowIn=Phosh;`, `X-GNOME-Autostart-Phase=Panel`, `X-GNOME-Provides=inputmethod` and `X-GNOME-AutoRestart=true`. [`data/sm.puri.Squeekboard.desktop.in.in`](https://gitlab.gnome.org/World/Phosh/squeekboard/-/blob/main/data/sm.puri.Squeekboard.desktop.in.in)
  - It owns its D-Bus name with `G_BUS_NAME_OWNER_FLAGS_NONE`. If it loses the name it logs "Is Squeekboard already running?" and calls `exit(1)`, unless `SQUEEKBOARD_DEBUG=force_show` is set.
  - A missing `zwp_input_method_manager_v2` only produces a warning. A missing `zwp_virtual_keyboard_manager_v1` is fatal. [`src/server-main.c`](https://gitlab.gnome.org/World/Phosh/squeekboard/-/blob/main/src/server-main.c)
- **Stevia** ships the same kind of .desktop file, with `Exec=phosh-osk-stevia --allow-replacement` and `X-GNOME-HiddenUnderSystemd=true`. [`data/sm.puri.OSK0.desktop.in.in`](https://gitlab.gnome.org/World/Phosh/stevia/-/blob/main/data/sm.puri.OSK0.desktop.in.in)
  - It also ships a systemd user unit, `mobi.phosh.OSK.service`, with `Type=dbus`, `BusName=sm.puri.OSK0`, `Restart=on-failure`, `RestartSec=1s`, `PartOf=mobi.phosh.OSK.target` and `After=gnome-session-initialized.target`. [`data/mobi.phosh.Stevia.service.in`](https://gitlab.gnome.org/World/Phosh/stevia/-/blob/main/data/mobi.phosh.Stevia.service.in), [`data/meson.build`](https://gitlab.gnome.org/World/Phosh/stevia/-/blob/main/data/meson.build)
  - `data/meson.build` installs no `dbus-1/services` activation file, so the keyboard is started by the session, not by D-Bus activation.
- **Replacement in Stevia:** `--allow-replacement` and `--replace` map to `G_BUS_NAME_OWNER_FLAGS_ALLOW_REPLACEMENT` and `G_BUS_NAME_OWNER_FLAGS_REPLACE`, so a new keyboard can take `sm.puri.OSK0` over from the running one. [`src/phosh-osk-stevia.c`](https://gitlab.gnome.org/World/Phosh/stevia/-/blob/main/src/phosh-osk-stevia.c)

## Implications for Tessera

1. **One binary, two launch modes.** If `WAYLAND_SOCKET` is set and the IM v1 globals show up on that connection, we were started by KWin. Otherwise we are a self-started IM v2 client. On KWin without IM v1 globals, we should exit with Plasma Keyboard's kind of message ("enable Tessera in System Settings > Virtual Keyboard"). On KWin that can still mean `tesseractl` talking to the KWin-launched instance.
2. **All Wayland objects go on the inherited connection under KWin.** The UI toolkit (Slint) must render on the connection taken from `WAYLAND_SOCKET`, not open its own. This constrains component 3 (Surface).
3. **Ship a KWin .desktop file:** `X-KDE-Wayland-VirtualKeyboard=true`, `NoDisplay=true`, no field codes in `Exec`, installed in `share/applications/`. Don't let the same file act as XDG autostart on Plasma, or a second, unprivileged copy will start at login.
4. **Under KWin, prefer abort over unwind on panic.** With `panic = "abort"`, or a panic hook that aborts, a panic counts as a crash and KWin restarts the keyboard. Deliberate fatal errors should exit normally, so KWin doesn't burn its 5 restarts.
5. **Single-instance on IM v2 compositors and phoc is ours to enforce.** The compositor only sends `unavailable`.
   - Own a well-known D-Bus name for Tessera's own API (and `sm.puri.OSK0` on phoc), using Squeekboard's "exit if the name is taken" behaviour or Stevia's `--replace` semantics.
   - Treat `zwp_input_method_v2.unavailable` as fatal, with a clear message that another IM or OSK holds the seat.
   - This makes `exec_always` and repeated `exec-once` lines harmless.
6. **Ship a systemd user unit for wlroots and Hyprland**, modelled on Stevia: `Type=dbus`, `Restart=on-failure`, `PartOf=` and `After=graphical-session.target`. Document `exec-once`, `hl.exec_cmd` and `exec` lines as the simple alternative. D-Bus activation adds nothing: nobody on these desktops activates OSKs by bus name, and Phosh doesn't either.
7. **Under KWin, `tesseractl show` can call `org.kde.KWin /VirtualKeyboard forceActivate()`** rather than only toggling our own surface. Mode and enabled state also live in that D-Bus interface.
