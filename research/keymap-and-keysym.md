# Virtual-keyboard keymap generation and IM v1 keysym semantics

Research for issue #5. Checked against source on Nov 5, 2026.

Sources and revisions read:

| Project | Revision |
| --- | --- |
| Squeekboard | `main` @ `27c359c4` (2026-03-31), version 1.44.0-alpha0 |
| wvkbd | `master` @ `f7e9ea3a` (2026-10-27) |
| wlroots | `master` @ `1dedee12` (2026-11-02), 0.21.0-dev |
| sway | `master` @ `1652c54b` (2026-09-21) |
| KWin | `master` @ `cbf247d0` (2026-11-05) |
| KWin MR !10253 | head `9b64cdee`, state `opened` (created 2026-10-27, last update 2026-10-29) |
| virtual-keyboard-unstable-v1.xml | copy in `wlroots/protocol/` |
| input-method-unstable-v1.xml | `wayland-protocols/unstable/input-method/` |

## Short answer

- **Squeekboard** builds its own minimal keymaps from the layout. Each distinct symbol gets one keycode with a single level. It reuses the same 47 "printable" evdev keycodes across several keymaps, so a layout needs `2 + ceil(n/47)` keymaps. Squeekboard swaps keymaps (`zwp_virtual_keyboard_v1.keymap`) right before a key press that needs a different one. Text goes through IM v2 `commit_string` whenever possible. Keycodes are only the fallback.
- **wvkbd** uploads one full, xkbcomp-style keymap per script (about 64 KB for Latin). For a character outside the keymap (a "Copy" key), it rewrites the `<COMP>` key (keycode 135) to `U%08X` and re-uploads the whole keymap before every such press.
- **Limits**: keycodes have to stay in 8..255 (both projects declare `minimum = 8; maximum = 255`; Squeekboard says Xwayland cannot take more). Squeekboard avoids keycode 8 (evdev 0), because compositors drop it, and avoids non-printing evdev codes, because some clients read keycodes as evdev. wlroots imposes no size limit, but it holds at most 32 pressed keys per virtual keyboard (`WLR_KEYBOARD_KEYS_CAP`).
- **Modifiers on virtual-keyboard-v1**: in wlroots, `key` does **not** update the xkb state (`update_state = false`). Only the `modifiers` request changes the modifier state. Both OSKs send real-modifier bits (Shift=0x1, Lock=0x2, Control=0x4, Mod1=0x8, Mod4=0x40, Mod5=0x80). wvkbd also sends key events for `KEY_LEFTSHIFT` and the other modifier keys, for clients that look at raw keys.
- **Repeat on virtual-keyboard-v1** comes from the compositor or the client: hold the key down (press, no release) and the normal wl_keyboard repeat applies. Neither OSK implements repeat itself. wlroots defaults to 25 Hz / 600 ms; sway applies its configured repeat rate to virtual keyboards too.
- **KWin `keysym` today (master)**: it acts on **press only** and sends press and release together right away, so holding a key does nothing and there is no repeat. It ignores the `modifiers` argument, except when the client uses text-input-v2. It looks the keysym up in the active keymap and sends **only** the modifiers that keysym's level needs (latched and locked set to 0), then restores the real state. An unmapped keysym goes through a temporary keymap (`pc+us` plus `<CSTM>` = evdev 194 / KEY_F24 → the keysym), which KWin sends to the focused client, presses, releases and then replaces with the real keymap. The keysym bypasses KWin's input filters, so global shortcuts and Alt+Tab never fire.
- **KWin MR !10253** (open, not merged) sends keysyms through the normal keyboard input path. Shortcuts then fire, presses and releases are tracked separately so modifiers can be held, a repeated press becomes `Repeated`, `modifiers` is decoded through `modifiers_map`, and held keys are released when the context goes away. KWin's own key-repeat timer still skips input-method keys, so with compositor-repeat clients the IM has to resend presses.

## 1. Squeekboard

### Keycode allocation

`src/keyboard.rs`, `generate_keycodes()` (line 105):

- Key names are XKB keysym names. In `src/data/parsing.rs`, `create_action()` (line 291) turns a button's `text` into one keysym per Unicode codepoint. It uses the character's own name if `xkb_keysym_from_name` accepts it, and otherwise `format!("U{:04X}", codepoint)` (lines 408-420). That is how arbitrary Unicode gets a keysym. A button with `keysym:` uses that name directly; an invalid name falls back to `space` with a warning.
- The names are sorted, which keeps the assignment deterministic, and zipped with `util::cycle_count((9..255).filter(|x| allowed.contains(&(x - 8))))`. `allowed` lists 47 evdev codes that produce characters on a normal keyboard (`KEY_1`..`KEY_0`, `KEY_Q`..`KEY_P`, brackets, `KEY_A`..`KEY_L`, `;'\``, `\`, `KEY_Z`..`KEY_M`, `,./`). The comment says why: "Some clients try to interpret keymaps as if they were input-sequences coming from evdev". When the 47 codes run out, `cycle_count` starts over with the next keymap index.
- It starts at 9, not 8: "8 results in keycode 0, which the compositor likes to discard".
- Keymaps 0 and 1 are kept for a hard-coded US-like placement: `a`-`z`, digits, unshifted punctuation, `BackSpace`, `Return`, arrows, `F1`-`F12` and so on go in keymap 0 at their real evdev code. `A`-`Z` and shifted punctuation go in keymap 1 at the same codes. Everything else starts at keymap 2. (TODO in the code: "Add the Shift-modifier for keycodes on keymap 1". Keymap 1 is still single-level.)
- Workaround: if the layout has no `Return`, an `Unknown` keycode is added so that `BackSpace` works in GTK3.

### Keymap text

`generate_keymaps()` (line 313) and `generate_keymap()` (line 337): one bin of 256 slots per keymap index. Each keymap has:

- `xkb_keycodes { minimum = 8; maximum = 255; <I{n}> = n; ... indicator 1 = "Caps Lock"; }`. The comment says Xwayland won't accept a keymap without the indicator. Another comment: "Xorg can only consume up to 255 keys, so this may not work in Xwayland."
- `xkb_symbols`: `key <I{n}> { [ name ] };`, a single level.
- `xkb_types` with `virtual_modifiers Squeekboard;` and stub `ONE_LEVEL`/`TWO_LEVEL`/`ALPHABETIC`/`KEYPAD`/`SHIFT+ALT` types, "needed for Xwayland".
- `xkb_compatibility { interpret Any+AnyOf(all) { action= SetMods(modifiers=modMapMods,clearLocks); }; }`.
- No modifier keys and no `modifier_map`.

Upload (`eek/eek-keyboard.c`, `squeek_key_map_from_str()`): Squeekboard compiles the text with libxkbcommon, re-serialises it with `xkb_keymap_get_as_string`, writes it into an unlinked `shm_open` file and sends size `strlen + 1` (including the NUL). `src/wayland.c` `eek_virtual_keyboard_update_keymap()` calls `zwp_virtual_keyboard_v1_keymap(..., WL_KEYBOARD_KEYMAP_FORMAT_XKB_V1, fd, len)`. The fds are made once per layout (`Submission::use_layout`, `src/submission.rs` line 312) and reused on every swap.

### When the keymap is swapped

`src/submission.rs`:

- `handle_press()` (line 108): if an IM v2 service is active **and no modifier is latched**, `SubmitData::Text` goes out as `commit_string` + `commit`, and no keycodes are sent. Erase never uses `delete_surrounding_text`; the comment calls it a protocol bug, because the request takes byte offsets. Erase always sends `BackSpace` as a keycode.
- Otherwise, for each keycode of the key, `select_keymap(keycode.keymap_idx)` runs first. If there is one keycode, press goes out now and release on release. If there are several (a multi-codepoint string), each gets press+release at press time and nothing is sent on release.
- `select_keymap()` (line 299) only acts when the index changes. It clears modifiers, releases every virtual key still held, sends the new keymap, then restores the modifiers. The comment explains that the keymap update "may (or may not)" reset state, so Squeekboard resets it explicitly.
- `use_layout()` resets to keymap 0 whenever the layout changes.

### Modifiers and repeat

- `Modifiers` bitflags (`src/keyboard.rs`) follow the X11 core order (Shift 0x1, Lock 0x2, Control 0x4, Mod1/Alt 0x8, Mod4 0x40, Mod5/AltGr 0x80). `update_modifiers()` (line 240) ORs the active ones and `eek_virtual_keyboard_set_modifiers()` sends `modifiers(depressed, 0, 0, 0)`. The keymap has no modifier keys, so this works only because xkb's eight real modifiers always have indices 0-7. Squeekboard never sends latched or locked mods, and never sends a group.
- Squeekboard has no repeat code. A held single-keycode key stays pressed, so the compositor's or client's wl_keyboard repeat handles it. Multi-codepoint keys never repeat, because they are released at once.

## 2. wvkbd

`keyboard.c`, `keymap.mobintl.h`, `keyboard.h`:

- `keymap.mobintl.h` contains `NUMKEYMAPS 7` complete keymaps as C strings (`latin`, `cyrillic`, `arabic`, `georgian`, `greek`, `persian`, `hebrew`). Each is a full xkbcomp dump (`minimum = 8; maximum = 255;`, real `modifier_map`s, `<COMP> = 135`); the Latin one is about 64 KB. Each layout names its keymap (`layout.mobintl.h`, e.g. `{keys_full, "latin", ...}`).
- `kbd_switch_layout()` (line 27) re-uploads the keymap only when `keymap_name` changes.
- `Code` keys send raw evdev codes (`k->code`, or `k->shift_code` when Shift is on). These are real codes in the real keymap.
- `Copy` keys handle characters outside the keymap: `kbd_press_key()` (line 418) calls `create_and_upload_keymap(kb, name, unichr)` (line 771). It `sprintf`s the template so that `key <COMP> { [ U%08X, U%08X ] }` holds the character, uploads it (`os_create_anonymous_file`, size `strlen` without the NUL), then sends a press of key 127 (`<COMP>` = 135 − 8). The release is sent on unpress. The keymap is **not** put back afterwards, so every accented letter costs one full keymap upload.
- Modifiers: the `key_modifier_type` values are the same X11 bits (Shift 1, CapsLock 2, Ctrl 4, Alt 8, Super 64, AltGr 128). `zwp_virtual_keyboard_v1_key_mods()` (line 281) also sends key events for `KEY_LEFTSHIFT`, `KEY_CAPSLOCK`, `KEY_LEFTCTRL`, `KEY_LEFTALT`, `KEY_LEFTMETA` and `KEY_RIGHTALT`, and then `zwp_virtual_keyboard_v1_modifiers(mods, 0, 0, 0)`. Modifiers latch for one key and unlatch in `kbd_unpress_key()` (line 298).
- Repeat: the README says "holding a key will repeat it like on a physical keyboard". That repeat comes from the compositor or client. `hide()` in `main.c` (around line 748) releases the active key "so they're not repeated infinitely".

## 3. virtual-keyboard-unstable-v1 protocol

`zwp_virtual_keyboard_v1`, version 1 (`wlroots/protocol/virtual-keyboard-unstable-v1.xml`):

- `keymap(format, fd, size)`: format is a `wl_keyboard.keymap_format`. The protocol sets no limits on size or keycodes.
- `key(time, key, state)` and `modifiers(depressed, latched, locked, group)`: both raise `no_keymap` if no keymap has been set. `key` takes evdev codes (xkb code − 8), as `wl_keyboard.key` does.
- There is no request for repeat info. Repeat stays with the compositor.

## 4. wlroots and sway

`types/wlr_virtual_keyboard_v1.c`:

- `virtual_keyboard_keymap()` (line 38): validates the format, `mmap`s `size` bytes and compiles them with `xkb_keymap_new_from_buffer`. A failed compile is reported as `wl_client_post_no_memory`, so a bad keymap kills the client. On success it calls `wlr_keyboard_set_keymap()`. `NO_KEYMAP` is accepted.
- `virtual_keyboard_key()` (line 88): `update_state = false`. `wlr_keyboard_notify_key()` (`types/wlr_keyboard.c` line 110) only calls `xkb_state_update_key` when `update_state` is set. A virtual-keyboard key press therefore **never changes the modifier state**; only `modifiers` does, through `wlr_keyboard_notify_modifiers` → `xkb_state_update_mask`.
- `wlr_keyboard_set_keymap()` (`types/wlr_keyboard.c` line 200) re-serialises the keymap into a new shm file pair. `types/seat/wlr_seat_keyboard.c` `handle_keyboard_keymap()` re-sends `wl_keyboard.keymap` to every client of the seat when the active keyboard's keymap changes, and `wlr_seat_set_keyboard()` sends it when the active keyboard changes. Each swap therefore means one compile in the compositor and one keymap event per client. The swap is ordered before the next `key` on the same connection.
- Pressed keys are capped at `WLR_KEYBOARD_KEYS_CAP` = 32 (`include/wlr/types/wlr_keyboard.h`).
- Repeat defaults are 25 Hz / 600 ms (`wlr_keyboard_init`). sway's `sway_keyboard_configure()` (`sway/input/keyboard.c`) calls `wlr_keyboard_set_repeat_info` with the configured values for virtual keyboards too, and skips only `sway_keyboard_set_layout` for them. Virtual keyboards are never grouped.
- sway's `keyboard_get_im_grab()` does not forward a virtual keyboard's events to the IM v2 keyboard grab when both belong to the same client. This avoids an OSK feeding its own keys back to itself.

## 5. KWin and the IM v1 `keysym` request

Protocol (`input-method-unstable-v1.xml`): `keysym(serial, time, sym, state, modifiers)`. Its description says "Sym is an XKB keysym, state is a wl_keyboard key_state." The `modifiers` mask is interpreted through `modifiers_map(array)`, a list of NUL-separated modifier names. The protocol says key events "should not be used for normal text input operations".

KWin master, `src/wayland/inputmethod_v1.cpp`: `zwp_input_method_context_v1_keysym` converts `state` with `waylandKeyStatetoKeyboardKeyState` and emits `keysym`. `modifiers_map` is forwarded as a raw `QByteArray`.

KWin master, `src/inputmethod.cpp`, `InputMethod::keysymReceived()` (line 491):

1. If the focused client uses **text-input-v2**, KWin forwards `keysymPressed`/`keysymReleased(sym, modifiers)` unchanged. This is the only path where the `modifiers` argument is used.
2. If a KWin effect holds a keyboard grab, the key goes to the effect.
3. Otherwise KWin acts **only when `state == Pressed`**: `forwardKeySym(sym)`, then `xkb->forwardModifiers()` to put the real state back. Releases are dropped.

`InputMethod::forwardKeySym()` (line 553):

- `Xkb::keycodeFromKeysym()` (`src/xkb.cpp` line 1236) scans the **active keymap**, effective layout only, every level, and returns the first keycode and level that produce the keysym, with the first modifier mask for that level (`xkb_keymap_key_get_mods_for_level`). The loop is `keycode < max`, so it never checks the highest keycode.
- **Mapped**: `notifyKeyboardModifiers(level_mods, 0, 0, layout)`, then press and release of the keycode. This replaces depressed modifiers and clears latched and locked ones. Ctrl+keysym through this path is not possible, and Caps Lock is briefly cleared.
- **Unmapped**: `Xkb::keymapContentsForKeysym(scratchKeycode, sym)` → `createKeymapForKeysym()` (line 1339) builds `xkb_symbols { include "pc+us"; key <CSTM> { [ <name> ] }; }`, with `<CSTM>` = `scratchKeycode` (`src/xkb.h` line 194: `194; // KEY_F24`) + 8. KWin sets it on the seat keyboard (`setKeymap`, which sends a keymap event to the focused client), presses and releases 194, then calls `setKeymap(real keymap)`. A keysym with no name (`xkb_keysym_get_name` fails) or a keymap that fails to compile is dropped with a warning.
- The event goes straight to `SeatInterface::notifyKeyboardKey`. It skips `KeyboardInputRedirection`, so global shortcuts, Alt+Tab and KWin filters never see it.

Repeat: since a press produces press+release at once and releases are ignored, the client never sees a held key. With IM v1, repeat is the IM's job: it has to send repeated `Pressed` keysyms. In `installKeyboardGrab()` (line 943), KWin sends the IM a repeat rate of 0 when the focused client's wl_keyboard uses compositor-side repetition (wl_keyboard v10), and the real rate otherwise. That only applies to the hardware-keyboard grab.

`commitString()` with no text-input on the client (line 521) also turns the text into keysyms with `Xkb::textToKeySyms` (`xkb_utf32_to_keysym`, one per codepoint) and runs each through `forwardKeySym`. So `commit_string` reaches clients that have no text-input, through the same keymap-lookup and scratch-keymap path.

`InputMethod::key()` (line 729) and `modifiers()` (line 746) are the IM v1 forwarding requests for the hardware grab (`grab_keyboard`). `key` sends a raw keycode to the seat. `modifiers` updates KWin's xkb state.

### MR !10253, "inputmethod: Allow keysyms to trigger shortcuts"

<https://invent.kde.org/plasma/kwin/-/merge_requests/10253>, by Jason A. Donenfeld, **open**. Notes from its description and diff (`src/inputmethod.cpp`, `src/keyboard_input.cpp`, `src/xkb.cpp`, `src/wayland/inputmethod_v1.cpp`, plus an integration test):

- `keysymReceived` calls the new `KeyboardInputRedirection::processInputMethodKey(keycode, sym, state, time, modifiers, context)`. The keysym then goes through spies and filters, so global shortcuts and Alt+Tab work, and Plasma Keyboard can use IM v1 for these keys.
- `m_generatedKeys` tracks held keysyms. A second `Pressed` for a keysym already held becomes `KeyboardKeyState::Repeated`. A `Released` without a matching press is dropped. Press and release are **sent separately**, so modifiers like `Control_L` (see `isModifierKeysym`) can be held across later keys.
- `modifiers` is decoded: `updateModifiersMap` splits `modifiers_map` on NUL, and each set bit `i` is mapped to `xkb_keymap_mod_get_index(name[i])`. Those mods, plus the level's mods, are added for the duration of the key and then removed. Physical modifiers and latched/locked state are kept, not overwritten.
- Press counts per key and per source (`m_deviceKeyCounts`, `m_inputMethodKeyCounts`), so a physical key and an emulated key that overlap release only when the last holder lets go.
- Unmapped keysyms (`nativeScanCode == 0`) still go through `forwardKeySym`, which uses the scratch keymap, on press. Under text-input-v2 the keysym is still forwarded with its raw modifiers.
- When the context or resource is destroyed, or focus or text-input-v2 enable changes, `releaseGeneratedKeys()` releases everything still held.
- `processKeyEvent` skips `m_keyRepeatSpy` for IM keys, so KWin's compositor-side repeat (used by wl_keyboard v10 clients) never repeats an IM key. A client that repeats on its own side would repeat a held keysym, because the release now comes later. For v10 clients, the IM still has to resend presses.

## Implications for Tessera

These follow from the facts above. They are not recommendations already decided.

- **`keymap.rs` (vk backend)**: the Squeekboard model is the proven one. Give each symbol a single-level key, restrict to the 47 printable evdev codes (or at least avoid 0 and keep below 256), use several keymaps, swap before a press that needs another, and release everything held before each swap. Prefer `commit_string` whenever IM v2 is active, and send keycodes only for keysym actions, modifier combinations and erase. wvkbd's swap-per-character is simpler, but it uploads about 64 KB per accented letter.
- **Modifiers**: send real-mod bits through `modifiers`, since wlroots ignores modifier key presses for state. Sending `KEY_LEFTSHIFT` and the like as well, as wvkbd does, is optional.
- **Repeat**: on virtual-keyboard-v1, hold the key (press without release) and let the compositor or client repeat it. On KWin IM v1 today, Tessera has to repeat itself by sending repeated `keysym` presses, and modifier+key combos and shortcuts do not work. MR !10253 would fix the combos and shortcuts if it lands, but self-repeat is still needed for clients that rely on compositor-side repeat. This is the case for keeping `org_kde_kwin_fake_input` as the KWin fallback for Ctrl/Alt combos until the MR is merged.
