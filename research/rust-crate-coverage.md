# Rust crate coverage for our Wayland protocols

Answers issue #2. Checked on 2026-11-05 against the published `.crate` sources from static.crates.io (the files a `cargo build` actually compiles), version metadata from the crates.io API, and KWin and plasma-wayland-protocols `master` on invent.kde.org.

## Short answer

Every protocol Stages 0 and 1 need has published, generated client bindings. All of them come from the Smithay `wayland-rs` family and are MIT-licensed. **We don't need to generate any bindings from XML ourselves for Stages 0 and 1.** There is one optional exception: `org_kde_kwin_fake_input` v6 (`keyboard_keysym`), which no crate release ships yet (see Gaps).

smithay-client-toolkit (SCTK) covers layer-shell, seat/pointer/touch/keyboard handling, wl_shm buffer pools and calloop integration. It does **not** cover input-method-v1, virtual-keyboard-v1 or fake_input. Its input-method-v2 helper is partial. For those protocols we write the `Dispatch` impls ourselves against the raw generated bindings.

## Crate versions

| Crate | Latest | Released | Licence | MSRV |
|---|---|---|---|---|
| `wayland-client` | 0.31.15 | 2026-07-22 | MIT | — |
| `wayland-protocols` | 0.32.13 | 2026-06-19 | MIT | 1.86 |
| `wayland-protocols-misc` | 0.3.12 | 2026-03-31 | MIT | 1.71 |
| `wayland-protocols-wlr` | 0.3.12 | 2026-03-31 | MIT | 1.71 |
| `wayland-protocols-plasma` | 0.3.12 | 2026-03-31 | MIT | 1.71 |
| `wayland-protocols-experimental` | 20251230.0.3 | 2026-03-31 | MIT | 1.71 |
| `smithay-client-toolkit` | 0.21.1 | 2026-07-23 | MIT | 1.86 |
| `wayland-scanner` (only if we generate) | 0.31.11 | 2026-07-22 | MIT | — |

Source: `https://crates.io/api/v1/crates/<name>`; MSRV from each crate's `Cargo.toml` `rust-version`. Repositories: github.com/smithay/wayland-rs (all `wayland-*`) and github.com/smithay/client-toolkit (SCTK).

## Protocol by protocol

| Protocol (interfaces) | Crate and feature flags | Module path | Interface version in bundled XML | SCTK helper |
|---|---|---|---|---|
| input-method-v1 (`zwp_input_method_v1`, `zwp_input_method_context_v1`, `zwp_input_panel_v1`, `zwp_input_panel_surface_v1`) | `wayland-protocols`, features `client` + `unstable` | `wayland_protocols::wp::input_method::zv1::client` | 1 (all four) | none |
| input-method-v2 (`zwp_input_method_manager_v2`, `zwp_input_method_v2`, `zwp_input_popup_surface_v2`, `zwp_input_method_keyboard_grab_v2`) | `wayland-protocols-misc`, feature `client` | `wayland_protocols_misc::zwp_input_method_v2::client` | 1 | partial: `sctk::seat::input_method` |
| virtual-keyboard-v1 (`zwp_virtual_keyboard_manager_v1`, `zwp_virtual_keyboard_v1`) | `wayland-protocols-misc`, feature `client` | `wayland_protocols_misc::zwp_virtual_keyboard_v1::client` | 1 | none |
| text-input-v3 (`zwp_text_input_manager_v3`, `zwp_text_input_v3`) | `wayland-protocols`, features `client` + `unstable` | `wayland_protocols::wp::text_input::zv3::client` | 2 | none (IM does not bind it) |
| `org_kde_kwin_fake_input` | `wayland-protocols-plasma`, feature `client` | `wayland_protocols_plasma::fake_input::client` | 5 | none |
| wlr-layer-shell (`zwlr_layer_shell_v1`, `zwlr_layer_surface_v1`) | `wayland-protocols-wlr`, feature `client` | `wayland_protocols_wlr::layer_shell::v1::client` | 5 | yes: `sctk::shell::wlr_layer` |

Evidence, by file in the published crates:

- `wayland-protocols-0.32.13/src/wp.rs`: `pub mod input_method { pub mod zv1 { wayland_protocol!("./protocols/unstable/input-method/input-method-unstable-v1.xml") } }` and `pub mod text_input { pub mod zv1 …; pub mod zv3 … }`, both behind `#[cfg(feature = "unstable")]`. Features in its `Cargo.toml`: `client`, `server`, `staging`, `unstable`. None are on by default.
- `wayland-protocols-misc-0.3.12/src/lib.rs` declares `zwp_input_method_v2` (built from `input-method-unstable-v2.xml` and depending on `wayland_protocols::wp::text_input::zv3`), `zwp_virtual_keyboard_v1`, `gtk_primary_selection` and `server_decoration`. Its `Cargo.toml` already turns on `wayland-protocols/unstable`, so text-input-v3 types come along with it.
- `wayland-protocols-plasma-0.3.12/src/lib.rs`: `pub mod fake_input { wayland_protocol!("./plasma-wayland-protocols/src/protocols/fake-input.xml") }`. The bundled XML has `<interface name="org_kde_kwin_fake_input" version="5">`, with `keyboard_key` since 4 and `destroy` since 5.
- `wayland-protocols-wlr-0.3.12/src/lib.rs`: `pub mod layer_shell { pub mod v1 { … wlr-layer-shell-unstable-v1.xml, [wayland_protocols::xdg::shell] } }`. The XML is version 5: `on_demand` keyboard interactivity since 4, `set_exclusive_edge` since 5.

### text-input-v3 from the IM side

An input method never binds `zwp_text_input_v3`; the application does. What the IM sees is the IM v2 events. Their arguments reuse text-input-v3 enums: `text_change_cause` uses `zwp_text_input_v3.change_cause`, and `content_type` uses `zwp_text_input_v3.content_hint` / `content_purpose` (`input-method-unstable-v2.xml` lines 166, 180-181). This is why `wayland-protocols-misc` depends on `wayland_protocols::wp::text_input::zv3`. The Rust types we need are `wayland_protocols::wp::text_input::zv3::client::zwp_text_input_v3::{ChangeCause, ContentHint, ContentPurpose}`.

The text-input-v3 XML bundled in `wayland-protocols` 0.32.13 is **version 2**. Version 2 adds `show_input_panel`/`hide_input_panel`, `set_available_actions`/`action`, `language`, `preedit_hint`, and the content hints `on_screen_input_provided`, `no_emoji` and `preedit_shown`. Input-method-v2 is still version 1 and has no events that carry any of these to the IM. So an IM v2 keyboard cannot learn about explicit show/hide or actions through IM v2. This is a protocol limit, not a crate gap. Only `xx-input-method` (below) carries them.

The IM v1 `content_type` event arrives as raw `uint`s. We map them ourselves.

## What SCTK 0.21.1 gives us

SCTK depends on `wayland-protocols` (`client`, `staging`, `unstable`), `wayland-protocols-misc` (`client`), `wayland-protocols-wlr` (`client`) and `wayland-protocols-experimental` (`client`). It re-exports them as `sctk::reexports::{protocols, protocols_misc, protocols_wlr, protocols_experimental, client, calloop, calloop_wayland_source}`. Default features: `calloop`, `xkbcommon`. Other feature: `system` (libwayland backend).

Covered, usable as-is:

- **Layer shell**: `sctk::shell::wlr_layer::{LayerShell, LayerSurface, Layer, Anchor, KeyboardInteractivity, LayerShellHandler}`. Provides `set_size`, `set_anchor`, `set_exclusive_zone`, `set_margin`, `set_keyboard_interactivity`, `set_layer` and `get_popup`. `LayerShell::bind` binds versions `1..=4` (`src/shell/wlr_layer/mod.rs:41`), so `set_exclusive_edge` (v5) is unavailable through SCTK. We don't need it for a bottom-docked keyboard.
- **Input on our own surface**: `sctk::seat::{SeatState, SeatHandler, Capability}`, `seat::pointer` (with cursor themes), `seat::touch`, and `seat::keyboard` (xkbcommon, key repeat). This is what component 6 (gesture) consumes.
- **Buffers**: `sctk::shm::{Shm, slot::SlotPool, multi, raw}` covers wl_shm pools for the tiny-skia renderer.
- **Event loop**: the `calloop` feature (`calloop-wayland-source`) matches the daemon's calloop design.

Partial:

- **Input-method-v2**: `sctk::seat::input_method::{InputMethodManager, InputMethod, InputMethodHandler, InputMethodEventState, Active, SurroundingText, CursorPosition}`. The module doc says: *"Currently only the input-method object is supported. No keyboard grab, no popup surface."* It handles the activate/deactivate/surrounding_text/content_type/done state machine and serials. It wraps `commit_string`, `set_preedit_string`, `delete_surrounding_text` and `commit`. That is enough for an OSK, which needs neither a keyboard grab nor a popup. We can use it, or copy its ~400-line state machine into `im_v2.rs`.
- **xx-input-method (experimental)**: `sctk::seat::input_method_v3` implements `xx_input_method_v1` / `xx_input_method_manager_v2` (bind range `2..=3`) from `xx-input-method-v2.xml`. The bindings come from `wayland_protocols_experimental::input_method::v1` and `text_input::v3` (`xx_text_input_v3`), plus `keyboard_filter::v3`. TECHNICAL_DESIGN.md says to build no backend until this reaches staging, but the bindings and an SCTK helper already exist for when it does.

Not covered (we write the `Dispatch` impls against raw bindings):

- input-method-v1, including `zwp_input_panel_v1` / `zwp_input_panel_surface_v1` (`set_toplevel`, `set_overlay_panel`).
- virtual-keyboard-v1 (`keymap` with an fd, `key`, `modifiers`). We generate the keymap with xkbcommon ourselves, which `keymap.rs` already plans to do.
- `org_kde_kwin_fake_input`.

`grep -rl 'virtual_keyboard\|fake_input\|input_panel' smithay-client-toolkit-0.21.1/src` returns nothing.

## Gaps

1. **fake_input v6 `keyboard_keysym`.** Upstream plasma-wayland-protocols added `keyboard_keysym` (since 6) in commit d50aab9be5 on 2025-09-17. KWin `master` advertises `s_version = 6` and implements `org_kde_kwin_fake_input_keyboard_keysym` (`src/backends/fakeinput/fakeinputbackend.cpp`). The `wayland-protocols-plasma` 0.3.12 release bundles the v5 XML. The wayland-rs `master` submodule points at plasma-wayland-protocols `3d8327f5`, so a future release may pick up v6, but none has yet. Two options:
   - v5 `keyboard_key` (evdev keycode, since 4) is enough for the planned fallback, for combinations the IM v1 `keysym` request can't deliver. No gap.
   - If we want keysym-based fake input, either vendor `fake-input.xml` v6 and generate with `wayland-scanner` 0.31 (`wayland_scanner::generate_client_code!`, about 20 lines in `tessera-wl`), or wait for a `wayland-protocols-plasma` bump.
2. **No SCTK helpers** for IM v1, virtual-keyboard or fake_input. This is hand-written dispatch code, not binding generation.
3. **text-input-v3 v2 features don't reach an IM v2 client.** This is a protocol limit (see above). No crate can fix it.

## KWin facts relevant to bindings

- fake_input is in KWin's `restrictedInterfaces` set. It is hidden only from **sandboxed** clients (`if (client->isSandboxed()) return !restrictedInterfaces.contains(...)`). Unsandboxed clients see it. `authenticate` currently sets the device authenticated unconditionally (`// TODO: make secure`). Source: `src/wayland_server.cpp` and `src/backends/fakeinput/fakeinputbackend.cpp` on KWin master.
- `zwp_input_panel_v1` and `zwp_input_method_v1` are advertised only to KWin's `inputMethodConnection()` (`inputmethodInterfaces` in `src/wayland_server.cpp`). Both use version 1 (`s_version = 1` in `src/wayland/inputmethod_v1.cpp`), which matches the binding.
- KWin's protocol list (`src/wayland/CMakeLists.txt`) includes `fake-input.xml`, `input-method-unstable-v1.xml`, `text-input-unstable-v1/v2/v3.xml` and `wlr-layer-shell-unstable-v1.xml`. It has no input-method-v2 and no virtual-keyboard-v1, consistent with TECHNICAL_DESIGN.md.

## Suggested `tessera-wl` dependencies

```toml
wayland-client = "0.31"
wayland-protocols = { version = "0.32", features = ["client", "unstable"] }
wayland-protocols-misc = { version = "0.3", features = ["client"] }
wayland-protocols-wlr = { version = "0.3", features = ["client"] }
wayland-protocols-plasma = { version = "0.3", features = ["client"] }
smithay-client-toolkit = "0.21"   # default features: calloop, xkbcommon
```

All are MIT, which is compatible with GPL-3.0 and the planned `deny.toml` allow-list. TECHNICAL_DESIGN.md's component 2 row should add `wayland-protocols-wlr`, `wayland-protocols-plasma` and `smithay-client-toolkit` to its key crates.
