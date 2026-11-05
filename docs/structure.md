# Project structure

One Cargo workspace. Crates map to the components in `TECHNICAL_DESIGN.md` (numbers in brackets). A crate is created in the stage that first needs it, not before.

```
/
├── Cargo.toml                 workspace, shared lints and dependency versions
├── deny.toml                  cargo-deny: licence allow-list, bans AGPL and non-OSI crates
├── LICENSE                    GPL-3.0
├── crates/
│   ├── tessera/               [1, 10, 12] daemon binary (Stage 0)
│   │   └── src/
│   │       ├── main.rs        calloop event loop, wiring
│   │       ├── visibility.rs  shown / hidden
│   │       ├── control.rs     D-Bus API (+ sm.puri.OSK0 in Stage 5)
│   │       ├── session.rs     hardware-keyboard and tablet-mode detection (Stage 1)
│   │       ├── rules.rs       per-app rules (Stage 2)
│   │       └── feedback.rs    feedbackd haptics and sound (Stage 2)
│   ├── tessera-wl/            [2, 3, 7] Wayland side (Stage 0)
│   │   └── src/
│   │       ├── backend/       InputBackend trait + one module per implementation
│   │       │   ├── vk.rs          virtual-keyboard only (Stage 0)
│   │       │   ├── im_v2.rs       IM v2 + virtual-keyboard (Stage 1)
│   │       │   ├── im_v1.rs       IM v1, keysym + fake_input fallback (Stage 1)
│   │       │   └── uinput.rs      (Stage 6)
│   │       ├── surface/       surface host: layer-shell, IM v1 input panel; reports supported modes
│   │       └── keymap.rs      xkb keymap generation for key events
│   ├── tessera-layout/        [5] layout schema, loading, key geometry per mode (Stage 0)
│   ├── tessera-gesture/       [6] touch/pointer → tap, long-press, gestures, glide paths (Stage 0)
│   ├── tessera-ui/            [4] renderer: tiny-skia + wl_shm first, femtovg later (Stage 0)
│   ├── tesseractl/            [10] CLI over the D-Bus API (Stage 0)
│   ├── tessera-engine/        [8] text engine, C ABI via cbindgen (Stage 3)
│   │   └── include/           generated tessera_engine.h
│   ├── tessera-dictgen/       [9] builds data packs from source word lists (Stage 3)
│   └── tessera-settings/      [11] Slint settings app (Stage 6)
├── layouts/                   Tessera YAML layouts; en-US only until Stage 2
├── data/                      install files: KWin .desktop entry, D-Bus service, systemd user unit, default config.toml
├── kcm/                       optional Plasma QML settings module (Stage 6)
├── tests/                     cross-crate integration tests, headless Weston (Stage 1)
├── tools/import/              one-off Squeekboard/FlorisBoard layout converters (Stage 2, deleted after import)
└── docs/
    ├── adr/
    └── agents/
```

## Dependency rules

- `tessera-engine` depends on no other workspace crate, so it can move to its own repository later. It defines its own key geometry type; the daemon converts the layout's output into it.
- `tessera-layout` and `tessera-gesture` are pure logic: no Wayland, no rendering. They are where most unit tests live.
- `tessera-ui` draws into buffers it is handed and knows nothing about input-method protocols.
- Only `tessera-wl` speaks Wayland protocols. Features use the `InputBackend` trait, never a concrete backend.
- The daemon (`tessera`) is the only crate that wires the others together.

## Other repositories

- `tessera-data` (data packs, per-pack licences, CC-BY-SA kept out of this tree): created at the start of Stage 3, when prediction needs it.
- Glide test recordings (Stage 4) go in `tessera-data` too, since they are data with their own licence.
