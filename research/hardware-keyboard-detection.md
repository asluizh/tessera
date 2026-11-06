# Prior art: tablet-mode and hardware-keyboard detection

Research for issue #4. Question: how do Stevia, Squeekboard, Plasma Keyboard/KWin and GNOME decide
whether to auto-show an on-screen keyboard, which signals they use, what permissions they need, and
how they behave on a laptop whose built-in keyboard is always present.

All findings are from source read at these commits (shallow clones, November 2026):

| Project | Repo | Commit |
|---|---|---|
| Stevia | https://gitlab.gnome.org/World/Phosh/stevia | `cca2aef2` |
| Squeekboard | https://gitlab.gnome.org/World/Phosh/squeekboard | `27c359c4` |
| Phoc | https://gitlab.gnome.org/World/Phosh/phoc | `1ef2711f` |
| Phosh | https://gitlab.gnome.org/World/Phosh/phosh | `a9690bd8` |
| KWin | https://invent.kde.org/plasma/kwin | `cbf247d0` |
| Plasma Keyboard | https://invent.kde.org/plasma/plasma-keyboard | `86c59719` |
| Mutter | https://gitlab.gnome.org/GNOME/mutter | `432c167e` |
| GNOME Shell | https://gitlab.gnome.org/GNOME/gnome-shell | `7904014f` |
| libinput | https://gitlab.freedesktop.org/libinput/libinput | `5f0ed181` |
| Hyprland wiki | https://github.com/hyprwm/hyprland-wiki | `b1cd0c77` |

## Short answer

Nobody treats "an evdev keyboard exists" as "a hardware keyboard is in use" on laptops, except
Stevia, and Stevia gets it wrong there in exactly the way the ticket fears. The projects that work on
laptops use one of two signals instead:

1. **The tablet-mode switch, when the device has one.** Mutter, KWin and Phosh all treat
   `SW_TABLET_MODE` as authoritative when present.
2. **Fallbacks when there is no switch:** the *last input device used* (KWin, GNOME Shell),
   "no pointer device attached" (Mutter, KWin) or the DMI chassis type (Phosh). No project uses
   keyboard *presence* as the fallback on laptops.

All of them get these signals inside the compositor (libinput), so none needs extra permissions.
A standalone OSK reading `/dev/input` directly needs the `input` group.

## Primary facts by source

### Kernel: `SW_TABLET_MODE`

- `SW_TABLET_MODE` is `0x01` in `include/uapi/linux/input-event-codes.h` ("set = tablet mode"). The
  current state of all switches can be read with the `EVIOCGSW` ioctl
  (`include/uapi/linux/input.h`).
  https://github.com/torvalds/linux/blob/master/include/uapi/linux/input-event-codes.h
- `Documentation/input/event-codes.rst`, EV_SW section: "Upon binding to a device or resuming from
  suspend, a driver must report the current switch state." So the switch has an initial value and
  needs no polling.
  https://www.kernel.org/doc/html/latest/input/event-codes.html#ev-sw

### libinput

- `doc/user/switches.rst`, "Tablet mode switch handling": when `SW_TABLET_MODE` is on, "the
  touchpad and internal keyboard are disabled. If a trackpoint exists, it is disabled too." And,
  importantly: "This handling of tablet mode switches is transparent to the user, no notifications
  are sent and **the device appears as enabled at all times**."
  https://wayland.freedesktop.org/libinput/doc/latest/switches.html#switches-tablet-mode
  - So a compositor or client that counts keyboard devices still sees the built-in keyboard while
    the convertible is folded. Presence alone tells you nothing, even on hardware that has a switch.
- Internal vs external keyboard (`src/evdev.c`, `evdev_tag_keyboard`): a device counts as a
  keyboard only if it has `KEY_Q`..`KEY_P`. It is tagged internal or external from udev
  `ID_INTEGRATION` or from the quirk `AttrKeyboardIntegration`. `quirks/10-generic-keyboard.quirks`
  sets `MatchBus=ps2` to internal and `MatchBus=bluetooth` to external. **USB and I2C keyboards
  are untagged by default**, and many modern laptops (Surface Type Cover, some ASUS models) need
  per-model quirks (`quirks/50-system-asus.quirks`).
- `src/evdev-fallback.c`, `fallback_pair_tablet_mode` (line ~1519): only keyboards tagged internal
  (or trackpoints) are suspended in tablet mode. External keyboards keep working. The quirks
  `ModelTabletModeNoSuspend` and `ModelTabletModeSwitchUnreliable` exist
  (`doc/user/device-quirks.rst`), so the switch is known to be wrong on some hardware.
- libinput has **no public API** for internal or external keyboards. The tag is private. A
  consumer has to read udev `ID_INTEGRATION` itself or repeat the bus heuristic.

### Stevia (Phosh's newer OSK)

- `src/pos-hw-tracker.c`: hardware-keyboard state comes **only** from the Phoc private protocol
  `zphoc_device_state_v1`. `has_hw_kb = !!(capabilities & ZPHOC_DEVICE_STATE_V1_CAPABILITY_KEYBOARD)`
  (line 54). `pos_hw_tracker_get_allow_active` returns `!has_hw_kb` unless the GSetting
  `mobi.phosh.osk ignore-hw-keyboards` is true (`data/mobi.phosh.osk.gschema.xml`: "The OSK
  doesn't show when it detects an attached hardware keyboard. This setting disables this
  detection.").
- `src/pos-app.c` (~line 186-196): the OSK is visible only if the a11y `screen-keyboard-enabled`
  setting allows it, the per-app activation filter allows it, **and** the hw tracker allows it.
- Stevia does not use the tablet-mode switch itself. `protocol/phoc-device-state-unstable-v1.xml`
  exposes `get_tablet_mode_switch`, but Stevia only subscribes to `capabilities`.

### Phoc (the compositor that provides that capability)

- `src/device-state.c`, `phoc_device_state_update_capabilities`: sets `CAPABILITY_KEYBOARD` when
  `phoc_seat_has_hw_keyboard()` is true.
- `src/seat.c:1885` `phoc_seat_has_hw_keyboard`: true if **any** keyboard on the seat passes
  `phoc_input_device_get_is_keyboard`.
- `src/input-device.c:304` `phoc_input_device_get_is_keyboard`: "A physical keyboard should at
  least have a space, enter and a letter". It checks `libinput_device_keyboard_has_key` for
  `KEY_A`, `KEY_ENTER` and `KEY_SPACE`, or the udev `ID_INPUT_KEYBOARD` property. Non-libinput
  devices, such as virtual keyboards, are excluded.
- **Laptop behaviour:** the built-in keyboard always passes this test, and libinput reports it as
  enabled even in tablet mode, so Stevia on Phoc sees "hardware keyboard present" on every laptop
  and convertible at all times. It never auto-shows unless the user turns on
  `ignore-hw-keyboards`. The design targets phones, where any keyboard really is an attached one.

### Phosh and Squeekboard

- Squeekboard has **no detection of its own**. `src/server-context-service.c` binds
  `org.gnome.desktop.a11y.applications screen-keyboard-enabled` to its "enabled" property and
  sends `squeek_state_send_keyboard_present(!enabled)`. In `src/state.rs`, `get_outcome`,
  `Presence::Present` hides the panel unless visibility is forced. "Physical keyboard present" in
  Squeekboard just means that the GSetting is off.
- Phosh flips that GSetting. `src/docked-manager.c:307` sets `screen-keyboard-enabled = !docked`.
  Docked mode turns on when the mode manager's "mimicry" is not phone, tablet or unknown
  (`docked-manager.c:125-128`).
- `src/mode-manager.c` `update_props` (line 116): the device type comes from the systemd-hostnamed
  D-Bus property `org.freedesktop.hostname1 Chassis` (handset, laptop, desktop, convertible,
  tablet, embedded). For `convertible`, the Phoc tablet-mode-switch object decides:
  mimicry is `LAPTOP` unless tablet mode is on (line 160). `laptop` and `desktop` chassis get
  `PHOSH_MODE_HW_KEYBOARD` **by chassis type, not by enumerating devices**. A phone becomes
  "desktop" when an external display **and a pointer** are attached
  (`PHOSH_MODE_DOCKED_PHONE_MASK`, `mode-manager.h:55`). That rule needs no keyboard.

### KWin and Plasma Keyboard

- Plasma Keyboard has no hardware detection of its own. A search for tablet, udev, evdev or
  hardware finds only UI hover tweaks keyed on `Kirigami.Settings.tabletMode`. Show and hide
  policy belongs to KWin.
- `src/inputmethod.cpp:174` `shouldShowOnActive`, with the default mode `NonMouseInput`
  (`inputmethod.h:299`, setting `[Wayland] VirtualKeyboardMode` in kwinrc): auto-show only if
  `input()->touch() == input()->lastInputHandler() || input()->tablet() == input()->lastInputHandler()`.
  Every physical key press calls `input()->setLastInputHandler(this)`
  (`keyboard_input.cpp:281`, `processKey`), and so does every pointer event
  (`pointer_input.cpp`). **KWin's OSK policy uses the last input device, not tablet mode.** Typing
  on the built-in keyboard and then tapping a field shows the OSK, because touch was the last
  input. Clicking a field with the touchpad does not show it.
- KWin's tablet mode (`src/tabletmodemanager.cpp`) is a separate signal, used by the Plasma shell
  and by Kirigami for touch-friendly UI. It is exported on D-Bus as
  `org.kde.KWin.TabletModeManager` at `/org/kde/KWin`, with properties `tabletModeAvailable` and
  `tabletMode` (`tabletmodemanager.h:25-26`). The logic:
  - With a tablet-mode switch, it follows the switch exactly (`TabletModeSwitchEventSpy`).
  - Without one (`TabletModeTouchpadRemovedSpy`, line 68): it is available if a touchscreen exists,
    and on when there is a touchscreen and **no pointer device** (`hasTouch && !hasPointer`).
    Keyboards are not considered. Devices tagged `kwin-ignore-tablet-mode` in udev are skipped
    (`blocksTabletMode`, line 23).
  - The user can override it with kwinrc `[Input] TabletMode=auto|on|off`.

### Mutter and GNOME Shell

- `mutter/src/backends/native/meta-seat-impl.c:1960` `update_touch_mode`:
  no touchscreen, so off; tablet switch present and off, so off; tablet switch present and on,
  so on; **no switch, so `!has_pointer`** ("assume touch-mode is mutually exclusive with
  pointers"). Keyboards play no part.
- `gnome-shell/js/ui/keyboard.js` (`KeyboardManager`, ~line 925-990): the OSK is enabled when
  `screen-keyboard-enabled` is set, or when `seat.get_touch_mode() && lastDeviceIsTouchscreen()`.
  The `last-device-changed` handler **ignores `KEYBOARD_DEVICE`** events, so a key press neither
  enables nor disables the OSK. Only a non-keyboard device (touchscreen versus pointer or stylus)
  changes the "last device".
- Laptop behaviour: a clamshell laptop with a touchpad never enters touch-mode, so there is no
  auto-OSK. A convertible auto-shows only when folded *and* the last non-keyboard input was the
  touchscreen.

### Hyprland (a target compositor)

- Hyprland exposes switches only as bind triggers: `switch:[name]`, `switch:on:[name]`,
  `switch:off:[name]`, with names listed by `hyprctl devices`
  (`content/configuring/core/binds/switches.md`). It has no protocol or IPC state for tablet mode
  or keyboard presence comparable to KWin's D-Bus or Phoc's `zphoc_device_state_v1`. On Hyprland,
  Tessera would have to read evdev itself or have the user wire a bind to `tesseractl`.

### logind and hostnamed

- logind handles the lid switch and power keys (`HandleLidSwitch`, which the Hyprland wiki warns
  can conflict). It does not publish tablet-mode state. hostnamed publishes `Chassis` with no
  permissions needed (session D-Bus read of a system service). That is a static hint, not live
  state.

## Permissions

- Compositor-side signals (KWin D-Bus, Phoc protocol, Mutter touch-mode) need nothing: the
  compositor already owns the devices through logind `TakeDevice`.
- Reading evdev directly: on Fedora, `50-udev-default.rules` sets `SUBSYSTEM=="input", GROUP="input"`
  with mode 0660, and `70-uaccess.rules` grants no `uaccess` ACL to keyboards or switches. Only
  audio, video, DRM, joysticks and similar get it. A user-session OSK therefore needs the user in
  the `input` group, which also gives it the ability to read every keystroke: a keylogger
  capability, and a security trade-off to document. The switch alone (`SW_TABLET_MODE`) usually
  sits on its own small device node (for example "Intel HID switches" or "Tablet Mode Switch"),
  but those nodes are still group `input`.
- Reading udev properties (`ID_INPUT_KEYBOARD`, `ID_INTEGRATION`, bus) through libudev or sysfs
  needs no special permission. Only opening the event node does.

## Implications for Tessera (facts a decision needs)

- The current TECHNICAL_DESIGN line "judged by the tablet-mode switch where the device has one and
  by evdev keyboard presence otherwise" matches only Stevia/Phoc, and on a laptop without a switch
  it means "never auto-show". No laptop-oriented project does this.
- Where a switch exists, it is the consensus primary signal. Even then, **do not** count the
  internal keyboard as present in tablet mode: libinput keeps reporting it as enabled.
- For no-switch devices, prior art offers three fallbacks:
  (a) last-input-device (KWin: a key press counts against the OSK; GNOME: key presses are ignored
  and only touch versus pointer counts);
  (b) "touchscreen and no pointer" (Mutter, KWin tablet mode);
  (c) external-keyboard presence, using udev `ID_INTEGRATION`, the bus heuristic
  (ps2 = internal, bluetooth = external) or libinput-style quirks. No project uses (c) for OSK
  policy, and USB and I2C internal keyboards make it unreliable without quirks.
- On Plasma, Tessera could read KWin's `org.kde.KWin.TabletModeManager` D-Bus property instead of
  evdev, but KWin already gates the input panel itself with its last-input rule. On Phosh, Phoc's
  `zphoc_device_state_v1` gives both the switch and the (laptop-blind) keyboard capability. On
  Hyprland, nothing is exposed, so direct evdev access needs the `input` group.
- Always provide a user override. Every project has one: `ignore-hw-keyboards`,
  `screen-keyboard-enabled`, `VirtualKeyboardMode` and `TabletMode`.
