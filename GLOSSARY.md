# Tessera

An on-screen keyboard for Wayland compositors, plus a text engine other keyboards can reuse.

## Language

### Typing

**Glide**:
Typing a whole word by tracing one continuous path across the keys.
_Avoid_: Swipe, swipe typing, gesture typing

**Gesture**:
A touch movement on a single key or bar that triggers an action, such as the space-bar cursor drag or backspace swipe.
_Avoid_: Using it for glide

**Compose**:
Building one character from a sequence of keys, via dead keys or a compose key.
_Avoid_: Composing (as a keyboard state)

### Output

**Transcribe**:
Sending text to the focused text field through the input method.
_Avoid_: Commit, type, insert

**Key event**:
A synthesized press and release of a key, delivered as if from a physical keyboard.
_Avoid_: Raw key, fake key

### Keyboard shape

**Layout**:
A named arrangement of keys, such as pt-BR, en-US or Terminal.
_Avoid_: Variant

**Layer**:
One page of a layout, such as letters, shifted or symbols.
_Avoid_: Page, level

**Mode**:
How a layout is placed on screen: full, split, one-handed (left or right) or floating.
_Avoid_: Variant

**Key geometry**:
The keys of the current layout and mode, each with its output and its position and size on screen. What the engine knows about the keyboard's shape.
_Avoid_: Key map (keymap means only the xkb keymap)

### Language data

**Data pack**:
One language's dictionary and word-frequency data, licensed and shipped separately from the code.
_Avoid_: Dictionary (a data pack holds more than one)

**Pack priority**:
The user's ordering of enabled data packs, used to rank equally good matches from different languages. Each installed data pack can be enabled or disabled; new packs start enabled.
_Avoid_: Primary language, active languages

### Visibility

**Shown** / **Hidden**:
Whether the keyboard is on screen. Independent of text-field focus and preedit.
_Avoid_: Composing, active
