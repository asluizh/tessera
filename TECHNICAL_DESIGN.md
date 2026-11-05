# Universal Wayland On-Screen Keyboard — Technical Design

Nov 5, 2026 · @L

## Overview

The project (working name **Tessera**) is one on-screen keyboard that runs on every serious Wayland compositor. It hides the protocol split behind swappable backends and ships its text engine as a reusable library.

**Problem.** Today each desktop has its own keyboard. KWin has Plasma Keyboard and Maliit, Phosh has Squeekboard and Stevia, and wlroots users get wvkbd or a shell plugin. Each one is tied to one compositor's protocol and toolkit. Features like prediction and glide get rebuilt, or not built, in each.

**Users.** Touchscreen laptops and tablets on Hyprland and Plasma come first. Phones on Phosh are supported but secondary. When priorities conflict, the keyboard comes first and the engine is extracted from what the keyboard proves it needs.

**Languages.** The design is language-agnostic: any left-to-right alphabetic language is a layout plus a data pack, with no code changes. Stages 0 and 1 target English (en-US) only. Two script families need more than that and are deferred to Stage 6: CJK, through existing input engines, and right-to-left or complex-shaping scripts such as Farsi, Arabic and Hebrew.

**Goals**

- One binary that works on KWin, the wlroots family, Hyprland, niri, COSMIC and phoc.
- Mobile-grade typing: accents, special keys, prediction, glide, and an adaptive layout.
- A text engine, `libtessera-engine`, with a C ABI so other keyboards can adopt it. This is how the project reduces fragmentation instead of adding to it.

**Non-goals**

- GNOME/Mutter. Its keyboard is internal to the shell.
- X11. A uinput fallback covers odd cases, but X11 is not a target.
- Full CJK input in-house. That is delegated to existing engines later (Stage 6).

**License: GPL-3.0-or-later.** It is the only choice that lets us copy from Squeekboard, Stevia and wvkbd (all GPLv3) while also taking Apache-2.0 code from FlorisBoard. The engine library is GPL-3.0-or-later too: the keyboards expected to adopt it (Squeekboard, Stevia, Plasma Keyboard) are GPL-compatible, and Maliit is being phased out. See [ADR-0001](docs/adr/0001-engine-is-gpl.md).

## Protocol landscape

The split is between two input-method protocols: KWin and Weston speak **input-method-v1**, while everything built on wlroots or Smithay speaks **input-method-v2**. Two backends cover every target compositor. Virtual-keyboard-v1 and uinput are fallbacks.

| Compositor | Input method | Panel surface | virtual-keyboard-v1 | Notes |
| --- | --- | --- | --- | --- |
| KWin (Plasma 6) | v1 | `zwp_input_panel_v1` | no | Only the client named in the Virtual Keyboard KCM gets IM v1, via `X-KDE-Wayland-VirtualKeyboard=true` in its .desktop file. No IM v2 either. Raw keys go through the IM v1 `keysym` request or `org_kde_kwin_fake_input`. The panel docks at the bottom (toplevel) or follows the text cursor (overlay); it cannot float. |
| Hyprland | v2 | layer-shell | yes | Main Omarchy target. |
| Sway, river, labwc, Wayfire | v2 | layer-shell | yes | Standard wlroots stack. |
| niri | v2 | layer-shell | yes | Smithay-based. IM v2 and virtual-keyboard since v0.1.0-alpha.1 (2023). Has a known focus-jump bug when tapping an OSK on multi-monitor setups ([#3971](https://github.com/niri-wm/niri/issues/3971); fix pending in [#3984](https://github.com/niri-wm/niri/pull/3984)). |
| COSMIC | v2 | layer-shell | yes | Smithay-based. Both protocols since epoch 1.0, hidden from sandboxed clients, so Tessera cannot run as a Flatpak there. |
| phoc (Phosh) | v2 | layer-shell | yes | Also expects the `sm.puri.OSK0` D-Bus interface for show/hide. |
| Weston | v1 | `zwp_input_panel_v1` | yes | Reference compositor; useful for CI. |

On the client side, apps request the keyboard through **text-input-v3** (GTK, Qt 6, Firefox, Chromium with flags). Most Electron apps don't, so they never trigger auto-show. For those, the keyboard falls back to manual show and raw key events.

Sources: [virtual-keyboard-v1 support table](https://wayland.app/protocols/virtual-keyboard-unstable-v1), [KWin protocol list](https://invent.kde.org/plasma/kwin/-/raw/master/src/wayland/CMakeLists.txt), [KWin input panel placement](https://invent.kde.org/plasma/kwin/-/raw/master/src/inputpanelv1window.cpp), [KWin virtual-keyboard-v1 request](https://bugs.kde.org/show_bug.cgi?id=497774), [niri IME protocols commit](https://github.com/niri-wm/niri/commit/d39da3f461), [cosmic-comp state](https://github.com/pop-os/cosmic-comp/blob/epoch-1.11.0/src/state.rs). Checked against source on Nov 5, 2026.

## Architecture

Touch input flows down through one daemon, and text flows back up through whichever input-method backend the compositor speaks. The text engine sits outside the daemon so other keyboards can use it too.

&#91;embedded content: Tessera architecture · 3 layers, 2 backends, 1 shared engine\]

The highlighted engine is the piece meant to reduce fragmentation: Squeekboard, Stevia or Plasma Keyboard could link it through the C ABI. The settings UI, its TOML config and feedbackd haptics sit beside the daemon and are left out of the picture.

## Component details

Rust is the default for every core piece. The Smithay ecosystem already has client bindings for every protocol we need, and memory safety matters in a process that sees every keystroke, including passwords. The only C surface is the engine's ABI. QML appears only in an optional Plasma settings module. All crates live in one Cargo workspace in this repository; the engine crate depends on none of the daemon's crates, so it can be split out later.

| # | Component | Language and key crates | Responsibility |
| --- | --- | --- | --- |
| 1 | `tessera` daemon | Rust; `calloop`, `smithay-client-toolkit` | Event loop and visibility (hidden or shown). Text-field focus and preedit are tracked separately from visibility. Wires backends, UI and engine together. |
| 2 | Protocol backends (`tessera-wl`) | Rust; `wayland-client`, `wayland-protocols`, `wayland-protocols-misc` | One `InputBackend` trait with four implementations: IM v2 + virtual-keyboard, IM v1 (KWin, Weston), virtual-keyboard only, and uinput. The IM v1 backend sends raw keys through the `keysym` request and falls back to `fake_input` on KWin for combinations `keysym` can't deliver. |
| 3 | Surface host | Rust; SCTK layer-shell, `zwp_input_panel_v1` | Gets a `wl_surface` from layer-shell or the IM v1 panel. Everything above it is identical. Reports which modes its surface can support. |
| 4 | Renderer (`tessera-ui`) | Rust; `femtovg` on GLES, `tiny-skia` + `wl_shm` fallback, `cosmic-text` | Draws keys, popovers, the suggestion bar and the glide trail. Theming from TOML. |
| 5 | Layout engine | Rust; `serde` + YAML layouts | Loads declarative layouts in Tessera's own YAML schema (layers, keys and their outputs, long-press popovers, relative widths; no positions) and computes key geometry in millimetres from the output's physical size. Places any layout in one of the modes: full, split, one-handed or floating. Hands the engine the resulting key geometry. |
| 6 | Gesture recognizer | Rust | Turns `wl_touch`/`wl_pointer` into tap, long-press, popover slide, space-bar cursor drag, backspace swipe and glide paths. |
| 7 | Keymap and modifiers | Rust; `xkbcommon` | Generates the virtual-keyboard keymap so any Unicode character can be sent. Handles sticky Ctrl, Alt and Super. |
| 8 | Text engine (`libtessera-engine`) | Rust with a C ABI via `cbindgen`; `fst`, optional Hunspell FFI | Dictionary lookup, n-gram prediction, proximity-aware correction, user learning, glide decoding, dead-key composition. Key geometry comes from the caller on every layout or mode change; the engine never reads layout files. Predicts from every enabled data pack at once, independent of the layout, and breaks ties by the user's pack priority. Each installed pack has its own toggle, on by default. |
| 9 | Data packs and `tessera-dictgen` | Rust CLI | Builds compact FST dictionaries and frequency tables offline. Each pack carries its own data license. |
| 10 | Control and session glue | Rust; `zbus`, `evdev` | Own D-Bus API plus `sm.puri.OSK0` compatibility, a `tesseractl` CLI, tablet-mode and hardware-keyboard detection. |
| 11 | Settings UI | Rust + Slint; optional QML KCM for Plasma | Edits the TOML config, which the daemon hot-reloads. |
| 12 | Feedback | Rust; `zbus` | Haptics and sound through feedbackd when present. |

Design rules that cut across components:

- **Privacy.** When text-input reports a password or sensitive purpose, disable learning and the suggestion bar. The engine never touches the network.
- **Backend parity.** Features are written against the `InputBackend` trait only. A feature that needs one compositor's protocol is a bug. A surface host may report that its surface cannot support a mode (floating on KWin's input panel), and the UI hides that mode.
- **Two output paths.** Text is transcribed through `commit_string` and preedit when the app supports text-input. Shortcuts, arrows and terminal input go out as key events.
- **Latency budget.** About 16 ms from touch-up to transcription. Prediction runs off the input path and fills the bar asynchronously.

## Reuse map

The biggest borrow is glide: FlorisBoard's glide classifier is Apache-2.0 and has already been ported once, to C# in [ScreenKeyboard.Uno](https://www.nuget.org/packages/ScreenKeyboard.Uno). Everything else is either layout data we can copy or designs we re-implement.

The four actions:

- **Copy**: take files as they are, keeping attribution.
- **Port**: translate code to Rust, keeping its logic and license notice.
- **Clone**: re-implement the design from scratch, with no code carried over.
- **Reference**: read only.

| Source | What we take | Action | License | Feeds component |
| --- | --- | --- | --- | --- |
| [FlorisBoard](https://github.com/florisboard/florisboard) (Kotlin) | Statistical glide classifier and pruner | Port | Apache-2.0 | 8 Text engine |
| FlorisBoard | Layout definitions, key sizing, layout merging, composers, emoji data | Port + convert | Apache-2.0 | 5 Layouts, 8 Engine |
| FlorisBoard dictionaries | Word frequencies (built from wordfreq) | Copy as data packs | CC-BY-SA-4.0 (data) | 9 Data packs |
| [Squeekboard](https://gitlab.gnome.org/World/Phosh/squeekboard) (Rust + C) | Layout YAML files for many languages, converted to Tessera's schema by one-off scripts | Copy + convert | GPL-3.0+ | 5 Layouts |
| Squeekboard | Keymap generation for virtual-keyboard, IM v2 handling | Port | GPL-3.0+ | 2 Backends, 7 Keymap |
| [Stevia](https://gitlab.gnome.org/World/Phosh/stevia) (C, GTK) | Pluggable completers, terminal shortcut row, auto-portrait scaling, hardware-keyboard detection | Clone | GPL-3.0+ | 8 Engine, 5 Layouts, 10 Session |
| [wvkbd](https://github.com/jjsullivan5196/wvkbd) (C) | Minimal layer-shell + virtual-keyboard flow | Reference | GPL-3.0 | 2 Backends, 3 Surface |
| [Plasma Keyboard](https://invent.kde.org/plasma/plasma-keyboard) (C++, QML) | KWin IM v1 and input-panel integration, KCM registration | Reference | GPL-2.0+ | 2 Backends, 11 Settings |
| [Maliit Keyboard](https://github.com/maliit/keyboard) (C++) | IM v1 behaviour, Pinyin and Chewing plugins | Reference | LGPL (verify) | 2 Backends, Stage 6 |
| AOSP LatinIME / HeliBoard | Correction and suggestion design | Reference | Apache-2.0 / GPL-3.0 | 8 Engine |
| Hunspell dictionaries | Spell-check for heavily inflected languages | Use via FFI | Per dictionary (MPL/LGPL/GPL) | 8 Engine |
| Unicode CLDR | Emoji names, per-locale characters for accent popovers | Copy as data | Unicode License | 5 Layouts, 9 Data |

Do not take:

- **The AOSP gesture decoder.** It is a proprietary binary, not open source.
- **FUTO Keyboard code.** It is source-available, not OSI open source.
- **swipeGuess code.** It is AGPL-3.0. At most, run it as a separate program to benchmark glide accuracy.

Keep data packs in a separate repository with per-pack licence files. This keeps CC-BY-SA data out of the GPL code tree. Each language ships as its own package, `tessera-data-<lang>`. Without any pack the keyboard still works, minus prediction.

## Implementation stages

KWin and Hyprland both have to work by the end of Stage 1. Every later feature then lands on both at once. Stages are ordered by dependency, not by calendar.

1. **Stage 0: Skeleton, at wvkbd parity**
   - Features: one static en-US layout, tap input, Shift and symbol layers, key repeat, `tesseractl show|hide|toggle`.
   - Plumbing: virtual-keyboard-v1 + layer-shell backend, CPU renderer, generated xkb keymap.
   - Compositors: Hyprland, Sway.
   - Done when: you can type into a terminal on Hyprland.
2. **Stage 1: Input-method backends**
   - Features: auto-show and auto-hide on text-field focus, `commit_string`, content hints (numeric, email, URL). No auto-show while a hardware keyboard is in use, judged by the tablet-mode switch where the device has one and by evdev keyboard presence otherwise.
   - Plumbing: IM v2 backend; IM v1 backend with the input-panel surface, `keysym` raw keys with a `fake_input` fallback, and a .desktop entry so Plasma lists it; Weston in CI.
   - Language: English (en-US) only, as in Stage 0.
   - Compositors: adds KWin and Weston. niri and COSMIC are tested here too.
   - Done when: GTK4 and Qt 6 apps auto-show the keyboard on KWin and Hyprland, and stay quiet on a laptop with its keyboard attached.
3. **Stage 2: Typing basics**
   - Accents: long-press popovers, slide-to-select, dead keys and compose.
   - Special keys: sticky Ctrl, Alt and Super; Esc, Tab, arrows and F-keys; a terminal layout with a shortcut row.
   - Also: layout switching, an emoji panel, haptics.
   - Per-app rules by app-id (always show, never show) for apps without text-input-v3, such as Electron apps.
   - Packaging: distro packages, no Flatpak (sandboxed clients can't get the input protocols on COSMIC, and KWin needs a system .desktop entry). Data packs as `tessera-data-<lang>`, with the pack matching the system locale recommended. Before this stage, Tessera is built from source only.
   - Layouts: imported from Squeekboard and FlorisBoard for any left-to-right alphabetic language they cover. No language gets special treatment.
   - Done when: Ctrl+C works in a terminal on every Stage 1 compositor, and accented letters can be typed without leaving the letter layer.
4. **Stage 3: Prediction**
   - Features: suggestion bar, completion, proximity-aware correction, next-word prediction, auto-capitalisation, user learning, preedit where the app supports it.
   - Engine: FST dictionaries, unigram and bigram tables, Hunspell fallback, first C ABI release (0.1).
   - Privacy: suggestion bar and learning are off in password and sensitive fields. Learning is also off on the Terminal layout and in apps with a "never learn" per-app rule. Learned data stays in one local file under `$XDG_DATA_HOME/tessera/`, mode 0600, never synced, encrypted with XChaCha20-Poly1305. The key comes from the Secret Service when one is running, otherwise from a key file in `$XDG_CONFIG_HOME/tessera/`; the fallback is obfuscation, not protection, and the docs say so. `tesseractl forget` deletes both the data and the key. See [ADR-0002](docs/adr/0002-encrypted-learning-file.md).
   - Done when: under 5 ms per suggestion on the reference device (to be chosen), and correction undoes with one backspace.
5. **Stage 4: Glide**
   - Features: glide typing from the ported FlorisBoard classifier, re-ranked by the Stage 3 language model; glide trail; gliding through the space bar.
   - Done when: about 90% top-3 accuracy on a recorded test set of common words for each language with a data pack.
6. **Stage 5: Adaptive UI**
   - Sizing: keys in millimetres from the output's physical size.
   - Modes: automatic split in tablet landscape, one-handed left and right, floating (not on KWin).
   - Reacts to: rotation, output changes, the tablet-mode switch, hardware-keyboard attach and detach. On multi-monitor setups, the keyboard follows the touchscreen.
   - Phones: phoc support, including `sm.puri.OSK0` for show and hide.
   - Done when: rotating a tablet re-lays out the keyboard in under 100 ms on KWin and Hyprland.
7. **Stage 6: Ecosystem**
   - Engine 1.0 with a stable C ABI. The ABI may change freely before this; no outside adoption is pursued earlier. Offer integrations to Squeekboard, Stevia and Plasma Keyboard.
   - Settings UI in Slint and an optional Plasma KCM.
   - Lock-screen support where the compositor allows it.
   - CJK through engine plugins (librime, libpinyin).
   - Right-to-left and complex-shaping scripts (Farsi, Arabic, Hebrew and similar): mirrored layouts, shaping in the renderer, and engine support.
   - uinput backend for Electron apps and other edge cases.

## Compatibility matrix

Every listed compositor gets the full feature set once its backend lands. The only differences are when it first works and what that compositor itself can't do.

| Compositor | Backend | Manual show from | Auto-show from | Known gaps |
| --- | --- | --- | --- | --- |
| KWin (Plasma 6) | IM v1 + input panel | Stage 1 | Stage 1 | One IM client at a time, so it replaces Maliit or Plasma Keyboard rather than running beside them. Some raw-key combinations need `fake_input`. No floating mode. |
| Hyprland | IM v2 + virtual-keyboard | Stage 0 | Stage 1 | None expected. |
| Sway, river, labwc, Wayfire | IM v2 + virtual-keyboard | Stage 0 | Stage 1 | None expected. |
| phoc (Phosh) | IM v2 + `sm.puri.OSK0` | Stage 5 | Stage 5 | Competes with Stevia as the default keyboard. |
| niri | IM v2 + virtual-keyboard | Stage 0 | Stage 1 | Multi-monitor focus jump until upstream PR #3984 lands. |
| COSMIC | IM v2 + virtual-keyboard | Stage 0 | Stage 1 | Cannot run sandboxed. |
| Weston | IM v1 + input panel | Stage 1 | Stage 1 | CI only. |
| Anything else, or X11 | uinput | Stage 6 | Never | No text-input, so no prediction context or auto-show. |

On every compositor, apps without text-input-v3 (most Electron apps) only get manual show and raw key input.

## Risks and open questions

The largest risk is KWin: its IM v1 path is tied to KDE's own keyboard work and may change under us.

| Risk | Impact | Mitigation |
| --- | --- | --- |
| KWin changes its IM v1 path or moves to IM v2 | Stage 1 backend breaks on Plasma | Keep the backend thin, test against Plasma beta releases, follow kde-devel. |
| IM v2 never leaves wlroots' unstable namespace, or an upstream input-method protocol replaces it | Another backend to write | The `InputBackend` trait already isolates this. A new protocol is one more implementation. Watch `xx-input-method` and `xx-text-input` in wayland-protocols' experimental tier; build no backend until they reach staging. |
| Glide accuracy on non-English languages | Glide feels broken outside English | Build test sets per language in Stage 4 and tune the pruner per language. |
| Dictionary licensing (CC-BY-SA data next to GPL code) | Distro packaging rejects the code repository | Keep data packs in a separate repository with their own licences. |
| Electron apps lack text-input-v3 | No auto-show in VS Code, Slack, Discord | Manual show (Stage 0) plus per-app rules by app-id (Stage 2), as Stevia does. |

Open questions:

- [x] Does KWin expose virtual-keyboard-v1 to the IM client? No. Raw keys go through `keysym`, with `fake_input` as fallback. KWin MR !10253 would let `keysym` trigger shortcuts.
- [x] Do niri and COSMIC ship input-method-v2 today? Yes, both.
- [x] Can the IM v1 panel surface on KWin float? No. It docks at the bottom or follows the text cursor.
- [x] What licence is swipeGuess under? AGPL-3.0, so it is benchmark-only.
- [ ] Which device are the latency and speed targets measured on?
