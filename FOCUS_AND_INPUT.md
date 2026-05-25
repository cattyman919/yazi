# Yazi's Focus Screens & Key Input Routing

> **How Yazi decides which screen is "active" and where each keystroke goes**

---

## Table of Contents

1. [The Big Idea: A Layered Stack](#the-big-idea-a-layered-stack)
2. [The `Layer` Enum](#the-layer-enum)
3. [How Yazi Decides the Active Layer](#how-yazi-decides-the-active-layer)
4. [Rendering: The Painter's Algorithm](#rendering-the-painters-algorithm)
5. [Key Input Routing](#key-input-routing)
6. [The Keymap System](#the-keymap-system)
7. [Component-Specific Input Handling](#component-specific-input-handling)
8. [Chord (Multi-Key) Handling](#chord-multi-key-handling)
9. [Special Cases & Edge Cases](#special-cases--edge-cases)
10. [From Keystroke to Action: A Complete Trace](#from-keystroke-to-action-a-complete-trace)

---

## The Big Idea: A Layered Stack

Yazi's UI is not a single screen with widgets. It is a **stack of modal layers**, where each layer can be shown or hidden independently. Only the **topmost visible layer** receives key input.

Think of it like a stack of transparencies:

```
┌─────────────────────────────────────┐  ← Which (key chord hints)
│         "g → ?  g → g  g → G"       │    (active flag)
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │  ← Cmp (completion popup)
│ │  completions...                 │ │    (visible flag)
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │  ← Help (keymap help)
│ │  Key bindings for Mgr...        │ │    (visible flag)
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │  ← Confirm (yes/no dialog)
│ │  Delete 3 files?  [Y]es [n]o    │ │    (visible flag)
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │  ← Input (text prompt)
│ │  Rename: [file.txt________]     │ │    (visible flag)
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │  ← Pick (select from list)
│ │  > item 1                       │ │    (visible flag)
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│                                     │  ← Spot (file inspector)
│   file preview / metadata           │    (lock.is_some())
│                                     │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │  ← Tasks (background jobs)
│ │  Copying... 45%                 │ │    (visible flag)
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│                                     │  ← Mgr (file manager)
│   ..  Documents  Downloads  file.txt│    (always the bottom layer)
│                                     │
└─────────────────────────────────────┘
```

Each layer is **independent** — it has its own:
- Visibility state (`visible: bool` or `active: bool`)
- Keymap bindings (`keymap.toml` section)
- Render logic (`Widget` impl)
- Internal state (cursor position, scroll offset, filter text, etc.)

---

## The `Layer` Enum

Yazi defines all possible layers in `yazi-shared/src/layer.rs`:

```rust
#[derive(Clone, Copy, Debug, Default, Eq, PartialEq, Hash)]
#[repr(u8)]
pub enum Layer {
    #[default]
    Null,       // No layer / uninitialized
    App,        // Application-level (not a screen)
    Mgr,        // File manager (the base layer)
    Tasks,      // Background tasks overlay
    Spot,       // File spotter/inspector
    Pick,       // Picker dialog
    Input,      // Text input modal
    Confirm,    // Confirmation dialog
    Help,       // Keymap help overlay
    Cmp,        // Completion popup
    Which,      // Multi-key chord hints
    Notify,     // Notification toasts
}
```

Every action and key binding is tagged with a `Layer`. This is how Yazi knows **which screen an action belongs to**.

---

## How Yazi Decides the Active Layer

The function `Core::layer()` (`yazi-core/src/core.rs`) is the single source of truth for "what screen is the user interacting with right now?"

```rust
impl Core {
    pub fn layer(&self) -> Layer {
        if self.which.active {
            Layer::Which
        } else if self.cmp.visible {
            Layer::Cmp
        } else if self.help.visible {
            Layer::Help
        } else if self.confirm.visible {
            Layer::Confirm
        } else if self.input.visible {
            Layer::Input
        } else if self.pick.visible {
            Layer::Pick
        } else if self.active().spot.visible() {
            Layer::Spot
        } else if self.tasks.visible {
            Layer::Tasks
        } else {
            Layer::Mgr
        }
    }
}
```

This is a **hardcoded priority stack** (highest to lowest):

| Priority | Layer | Trigger Flag | What It Is |
|----------|-------|-------------|------------|
| 1 (top) | `Which` | `which.active` | Multi-key chord disambiguation popup |
| 2 | `Cmp` | `cmp.visible` | Tab-completion popup |
| 3 | `Help` | `help.visible` | Key bindings reference |
| 4 | `Confirm` | `confirm.visible` | Yes/No/Cancel dialog |
| 5 | `Input` | `input.visible` | Text input prompt |
| 6 | `Pick` | `pick.visible` | Select-from-list dialog |
| 7 | `Spot` | `spot.lock.is_some()` | File inspector/spotter |
| 8 | `Tasks` | `tasks.visible` | Background job list |
| 9 (bottom) | `Mgr` | (always) | Main file manager |

**Key insight:** This is checked on **every keypress**. When you close a dialog (e.g., press `Esc` in `Input`), `input.visible` becomes `false`, and the very next keypress falls through to the next visible layer (usually `Mgr`).

---

## Rendering: The Painter's Algorithm

Yazi renders layers using the **painter's algorithm** — draw from back to front, so overlapping content correctly obscures what's underneath.

In `Root::render()` (`yazi-fm/src/root.rs`):

```rust
impl Widget for Root<'_> {
    fn render(self, area: Rect, buf: &mut Buffer) {
        // 1. Base layer: Lua-driven main UI (file list, status bar, preview)
        let root = LUA.globals().raw_get::<Table>("Root")?;
        render_once(root.call_method("redraw", ())?, buf, ...);

        // 2. Preview panel (always visible)
        mgr::Preview::new(self.core).render(area, buf);
        mgr::Modal::new(self.core).render(area, buf);

        // 3. Overlays, bottom to top:
        if self.core.tasks.visible {
            tasks::Tasks::new(self.core).render(area, buf);
        }
        if self.core.active().spot.visible() {
            spot::Spot::new(self.core).render(area, buf);
        }
        if self.core.pick.visible {
            pick::Pick::new(self.core).render(area, buf);
        }
        if self.core.input.visible {
            input::Input::new(self.core).render(area, buf);
        }
        if self.core.confirm.visible {
            confirm::Confirm::new(self.core).render(area, buf);
        }
        if self.core.help.visible {
            help::Help::new(self.core).render(area, buf);
        }
        if self.core.cmp.visible {
            cmp::Cmp::new(self.core).render(area, buf);
        }
        if self.core.which.active {
            which::Which::new(self.core).render(area, buf);
        }
    }
}
```

**Why this order matters:**
- `Mgr` is drawn first — it's the background
- `Tasks`, `Spot`, `Pick` are drawn on top
- `Confirm` obscures everything below it
- `Help` obscures `Confirm`
- `Cmp` (completion) draws over everything, including `Help`
- `Which` draws on top of all — it must be visible even when you're in the middle of a chord

Each overlay uses `yazi_widgets::Clear` to blank out its rectangular area before drawing, ensuring clean borders and no bleed-through.

---

## Key Input Routing

When a key is pressed, it goes through this pipeline:

```
[Terminal] → Crossterm → Event::Key(key) → Event Bus
                                              │
                                              ▼
                                    Dispatcher::dispatch_key(key)
                                              │
                                              ▼
                                    Router::route(Key::from(key))
                                              │
                                              ▼
                                    ┌─────────────────────────┐
                                    │  1. Which layer active? │
                                    │     → Check help.type() │
                                    │     → Check input.type()│
                                    │     → Match keymap      │
                                    └─────────────────────────┘
```

### The Router (`yazi-fm/src/router.rs`)

```rust
impl Router<'_> {
    pub fn route(&mut self, key: Key) -> Result<bool> {
        let core = &mut self.app.core;
        let layer = core.layer();  // ← "Which screen is on top?"

        // STEP 1: Let Help intercept keys if it's in filter mode
        if core.help.visible && core.help.r#type(&key)? {
            return Ok(true);  // Key was consumed by Help's filter input
        }

        // STEP 2: Let Input intercept keys if it's visible
        if core.input.visible && core.input.r#type(&key)? {
            return Ok(true);  // Key was consumed by Input (e.g., typing text)
        }

        // STEP 3: Route to the appropriate layer's keymap
        use Layer as L;
        Ok(match layer {
            L::Null | L::App | L::Notify => unreachable!(),

            // Most layers just look up their own keymap
            L::Mgr | L::Tasks | L::Spot | L::Pick
            | L::Input | L::Confirm | L::Help => {
                self.matches(layer, key)
            }

            // Completion popup: check BOTH cmp and input keymaps
            L::Cmp => self.matches(L::Cmp, key) || self.matches(L::Input, key),

            // Which handles its own chord logic
            L::Which => core.which.r#type(key),
        })
    }
}
```

**Three-tier routing system:**

1. **Component-internal handlers** (`help.r#type()`, `input.r#type()`)
2. **Layer keymap lookup** (`KEYMAP.get(layer)`)
3. **Fallback / passthrough** (`Cmp` → `Input`, `Which` self-managed)

### The `matches()` Method

```rust
fn matches(&mut self, layer: Layer, key: Key) -> bool {
    // Iterate all key bindings for this layer
    for chord @ Chord { on, .. } in KEYMAP.get(layer) {
        // Does this binding start with the key we just pressed?
        if on.is_empty() || on[0] != key {
            continue;
        }

        if on.len() > 1 {
            // Multi-key chord (e.g., "g g" for "go to top")
            // Activate the "which-key" popup to show options
            let cx = &mut Ctx::active(&mut self.app.core, &mut self.app.term);
            act!(which:activate, cx, (layer, key)).ok();
        } else {
            // Single-key binding — execute immediately
            emit!(Seq(ChordCow::from(chord).into_seq()));
        }
        return true;  // Key was handled
    }
    false  // Key was not handled by this layer
}
```

---

## The Keymap System

### `Chord` — A Key Binding

A `Chord` (`yazi-config/src/keymap/chord.rs`) represents one key binding:

```rust
pub struct Chord<const L: u8 = { Layer::App as u8 }> {
    pub on:   Vec<Key>,       // Keys to press (e.g., [g, g] or [Enter])
    pub run:  Vec<Action>,    // Actions to execute
    pub desc: String,         // Human-readable description
    pub r#for: Platform,      // Platform filter (unix, windows, macos)
}
```

The `const L: u8` generic parameter ensures the chord is **typed to its layer** at compile time. When deserialized from `keymap.toml`, the layer is automatically injected.

### `Keymap` — All Bindings Per Layer

```rust
pub struct Keymap {
    pub mgr:     KeymapRules<{ Layer::Mgr as u8 }>,
    pub tasks:   KeymapRules<{ Layer::Tasks as u8 }>,
    pub spot:    KeymapRules<{ Layer::Spot as u8 }>,
    pub pick:    KeymapRules<{ Layer::Pick as u8 }>,
    pub input:   KeymapRules<{ Layer::Input as u8 }>,
    pub confirm: KeymapRules<{ Layer::Confirm as u8 }>,
    pub help:    KeymapRules<{ Layer::Help as u8 }>,
    pub cmp:     KeymapRules<{ Layer::Cmp as u8 }>,
}
```

`Keymap::get(layer)` returns the slice of chords for that layer. This is how `Router::matches()` knows which bindings to check.

### Example `keymap.toml`

```toml
[[mgr.prepend_keymap]]
on = [ "g", "g" ]
run = "arrow -99999999"  # Go to top
desc = "Go to the top"

[[mgr.prepend_keymap]]
on = [ "<Enter>" ]
run = "open"
desc = "Open selected file"

[[input.prepend_keymap]]
on = [ "<C-c>" ]
run = "close"
desc = "Cancel input"

[[help.prepend_keymap]]
on = [ "/" ]
run = "filter"
desc = "Filter key bindings"
```

Each section (`mgr`, `input`, `help`, etc.) is completely isolated. The same key (`Esc`) can do different things in different layers.

---

## Component-Specific Input Handling

Some components intercept keys **before** the keymap is consulted. This is crucial for text input.

### `Input` — Typing Text

```rust
// yazi-core/src/input/input.rs (simplified)
pub struct Input {
    pub(super) inner: yazi_widgets::input::Input,
    pub visible:  bool,
    pub title:    String,
    pub position: Position,
}
```

When `input.visible == true`, `Router::route()` calls `core.input.r#type(&key)` first:

```rust
// Router checks this BEFORE keymap lookup
if core.input.visible && core.input.r#type(&key)? {
    return Ok(true);  // Input consumed the key (typing a character)
}
```

The `Input` widget's `r#type()` method handles:
- **Printable characters** → insert into buffer
- **Arrow keys** → move cursor
- **Backspace** → delete character
- **Ctrl+keys** → shortcuts (if not consumed by keymap first)

But wait — how do keys like `Esc` or `Enter` work in Input? They're handled by the **keymap**, not `r#type()`. The `r#type()` returns `false` for keys it doesn't recognize (like `Esc`), so they fall through to the keymap lookup.

### `Help` — Filtering Bindings

The Help overlay has a special filter mode:

```rust
pub struct Help {
    pub visible: bool,
    pub in_filter: Option<yazi_widgets::input::Input>,
    // ...
}

impl Help {
    pub fn r#type(&mut self, key: &Key) -> Result<bool> {
        let Some(input) = &mut self.in_filter else {
            return Ok(false);  // Not in filter mode, let keymap handle it
        };

        match key {
            // Esc → exit filter mode
            Key { code: KeyCode::Esc, .. } => { self.in_filter = None; render!(); }
            // Enter → apply filter
            Key { code: KeyCode::Enter, .. } => { self.in_filter = None; }
            // Backspace → delete in filter input
            Key { code: KeyCode::Backspace, .. } => { act!(backspace, input)?; }
            // Any other key → type into filter
            _ => { input.r#type(key)?; }
        }

        self.filter_apply();  // Re-filter the bindings list
        Ok(true)  // Consumed
    }
}
```

When you press `/` in Help (bound to `filter`), `in_filter` becomes `Some(...)`. From then on, **all keys** go to the filter input until you press `Esc` or `Enter`.

### `Which` — Chord Disambiguation

The `Which` component is unique — it's not a modal overlay but a **state machine** for multi-key chords:

```rust
pub struct Which {
    pub active: bool,           // Is which-key popup showing?
    pub cands: Vec<ChordCow>,   // Candidate chords matching so far
    pub times: usize,           // How many keys of the chord have been pressed
}

impl Which {
    pub fn r#type(&mut self, key: Key) -> bool {
        // Narrow down candidates: keep only chords whose next key matches
        self.cands.retain(|c| c.on.len() > self.times && c.on[self.times] == key);
        self.times += 1;

        if self.cands.is_empty() {
            // No match — dismiss
            self.dismiss(None);
        } else if self.cands.len() == 1 {
            // Single match — execute it!
            let chord = self.cands.remove(0);
            self.dismiss(Some(chord));
        } else if let Some(i) = self.cands.iter().position(|c| c.on.len() == self.times) {
            // Exact match found among multiple candidates — execute it
            let chord = self.cands.remove(i);
            self.dismiss(Some(chord));
        }

        render_and!(true)
    }

    pub fn dismiss(&mut self, chord: Option<ChordCow>) {
        self.active = false;
        self.cands.clear();
        self.times = 0;

        if let Some(chord) = chord {
            emit!(Seq(chord.into_seq()));  // Execute the matched chord's actions
        }
    }
}
```

When you press the first key of a chord (e.g., `g`), `Router::matches()` detects `on.len() > 1` and activates `Which`. From then on, `Which::r#type()` takes over all key handling until the chord is completed or cancelled.

---

## Chord (Multi-Key) Handling

Chords are sequences of keys that together trigger an action. Examples from Yazi's default keymap:

| Chord | Action | Description |
|-------|--------|-------------|
| `g` `g` | `arrow -99999999` | Go to top |
| `g` `G` | `arrow 99999999` | Go to bottom |
| `g` `~` | `cd` | Go to home directory |
| `<Space>` | `toggle` | Select/deselect file |
| `y` `y` | `yank` | Yank file |
| `d` `d` | `remove` | Delete file |

### How Chords Work

```
User presses 'g'
    │
    ▼
Router::matches(Layer::Mgr, Key('g'))
    │
    ├─► Chord "gg" matches first key → on.len() == 2 > 1
    │   act!(which:activate, cx, (Mgr, Key('g')))
    │       which.active = true
    │       which.cands = all chords starting with 'g'
    │       which.times = 1
    │
    └─► Chord "g~" also matches → also in candidates
    │
    ▼
User presses 'g' again
    │
    ▼
Router sees layer == Which → core.which.r#type(Key('g'))
    │
    ▼
Which::r#type()
    │
    ├─► cands.retain(|c| c.on[1] == Key('g'))
    │       "g~" is removed (its 2nd key is '~')
    │       Only "gg" remains
    │
    ├─► cands.len() == 1 → exact match!
    │
    ▼
which.dismiss(Some("gg"))
    │
    ▼
emit!(Seq([Action("arrow -99999999")]))
    │
    ▼
Dispatcher::dispatch_seq() → Executor::execute() → act!(mgr:arrow, cx, ...)
```

**Key insight:** The `Which` popup is not just decorative — it's the **state machine** that tracks partial chord progress. Without it, you'd have to implement timeout-based chord detection, which is fragile.

---

## Special Cases & Edge Cases

### 1. `Cmp` Falls Through to `Input`

The completion popup (`Cmp`) appears while typing in an `Input` field. When `Cmp` is visible, the Router checks **both** keymaps:

```rust
L::Cmp => self.matches(L::Cmp, key) || self.matches(L::Input, key),
```

This means:
- `<Tab>` can be bound in `cmp` keymap to cycle completions
- `<Enter>` can be bound in `input` keymap to submit
- Typing characters still goes to the input field

### 2. Paste Handling

Clipboard paste (`Event::Paste`) bypasses the Router entirely:

```rust
fn dispatch_paste(&mut self, str: String) -> Result<()> {
    if self.app.core.input.visible {
        let input = &mut self.app.core.input;
        match input.mode() {
            InputMode::Insert => input.type_str(&str)?,
            InputMode::Replace => input.replace_str(&str)?,
        }
    }
    Ok(())
}
```

Paste **only** goes to the Input component. If Input is not visible, paste is silently ignored.

### 3. Mouse Events

Mouse events (`Event::Mouse`) go directly to the `app:mouse` actor:

```rust
fn dispatch_mouse(&mut self, mouse: MouseEvent) -> Result<()> {
    let cx = &mut Ctx::active(&mut self.app.core, &mut self.app.term);
    act!(app:mouse, cx, mouse).map(|_| ())
}
```

The actor decides what to do based on mouse position and active layer — not the Router.

### 4. Resize Events

Terminal resize (`Event::Resize`) triggers layout recalculation:

```rust
fn dispatch_resize(&mut self) -> Result<()> {
    let cx = &mut Ctx::active(&mut self.app.core, &mut self.app.term);
    act!(app:resize, cx, crate::Root::reflow as fn(_) -> _)
}
```

This affects **all** layers since they all need to reposition themselves.

### 5. Help Shows Bindings for the Layer *Underneath*

When you press `~` (default Help key) in `Mgr`, Help shows `Mgr` bindings. When you press it in `Input`, Help shows `Input` bindings. This is because `help.layer` is set to `core.layer()` **before** `help.visible` becomes true:

```rust
// help:toggle actor
fn act(cx: &mut Ctx, layer: Layer) -> Result<Data> {
    if cx.help.visible {
        cx.help.visible = false;
    } else {
        cx.help.layer = layer;   // Remember which layer's bindings to show
        cx.help.visible = true;
        cx.help.bindings = KEYMAP.get(layer).iter().collect();
    }
    render!();
    succ!();
}
```

### 6. `Notify` Layer

`Layer::Notify` exists in the enum but has **no keymap** and is never returned by `Core::layer()`. Notifications are passive toasts that don't intercept input.

---

## From Keystroke to Action: A Complete Trace

Let's trace what happens when you're in the file manager and press `r` to rename a file:

```
STEP 1: KEY CAPTURE
═══════════════════════════════════════
User presses 'r'
    │
    ▼
[Crossterm] KeyEvent { code: Char('r'), modifiers: NONE }
    │
    ▼
Event::Key(key).emit() → mpsc channel


STEP 2: EVENT DISPATCH
═══════════════════════════════════════
App loop: rx.recv_many() → events = [Event::Key('r')]
    │
    ▼
Dispatcher::dispatch(Event::Key('r'))
    │
    ▼
dispatch_key(Key('r')) → Router::route(Key('r'))


STEP 3: LAYER DETERMINATION
═══════════════════════════════════════
core.layer() checks flags:
    which.active?  false
    cmp.visible?   false
    help.visible?  false
    confirm.visible? false
    input.visible? false
    pick.visible?  false
    spot.visible?  false
    tasks.visible? false
    → Layer::Mgr


STEP 4: COMPONENT INTERCEPT CHECK
═══════════════════════════════════════
help.visible? false → skip
input.visible? false → skip


STEP 5: KEYMAP LOOKUP
═══════════════════════════════════════
Router::matches(Layer::Mgr, Key('r'))
    │
    ▼
for chord in KEYMAP.get(Layer::Mgr) {
    chord.on[0] == Key('r')?
        Yes! chord "r" → on.len() == 1 (single key)
    }
    │
    ▼
emit!(Seq([Action("rename")]))


STEP 6: ACTION EXECUTION
═══════════════════════════════════════
Dispatcher::dispatch(Event::Seq([Action("rename")]))
    │
    ▼
dispatch_seq() → dispatch_call(Action("rename"))
    │
    ▼
Executor::execute(action) → action.layer == Layer::Mgr
    │
    ▼
fn mgr(&mut self, action: ActionCow) {
    // ... dozens of if action.name == ...
    "rename" → act!(mgr:rename, cx, action)
}


STEP 7: ACTOR EXECUTION
═══════════════════════════════════════
Rename::act(cx, RenameForm { ... })
    │
    ├─► Determine hovered file
    │
    ├─► Show input modal:
    │       cx.input.visible = true
    │       cx.input.title = "Rename:"
    │       cx.input.value = current_filename
    │
    └─► render!()  → NEED_RENDER.store(1)


STEP 8: RENDER
═══════════════════════════════════════
App loop sees NEED_RENDER == 1
    │
    ▼
Root::render()
    │
    ├─► Draw Mgr (file list)
    ├─► Draw Input modal on top
    │       ┌──────────────┐
    │       │ Rename: file │
    │       └──────────────┘
    │
    └─► Terminal flushed


STEP 9: NEXT KEYPRESS (Now in Input layer)
═══════════════════════════════════════
User types "new_name"
    │
    ▼
Router::route() → core.layer() == Layer::Input
    │
    ▼
core.input.r#type(Key('n'))? → true (consumed)
core.input.r#type(Key('e'))? → true (consumed)
...


STEP 10: SUBMIT
═══════════════════════════════════════
User presses Enter
    │
    ▼
Router::route() → Layer::Input
    │
    ▼
input.r#type(Key(Enter))? → false (not a typing key)
    │
    ▼
matches(Layer::Input, Key(Enter))
    │
    ▼
KEYMAP.input: "<Enter>" → run = "close"
    │
    ▼
emit!(Seq([Action("close")]))
    │
    ▼
InputClose::act(cx, ...)
    │
    ├─► Get value from input: "new_name"
    ├─► input.visible = false
    ├─► Actually rename the file
    └─► render!()


STEP 11: RETURN TO MGR
═══════════════════════════════════════
core.layer() now returns Layer::Mgr again
Next keypress goes to Mgr keymap
```

---

## Design Takeaways

### 1. Priority Stack for Focus

Yazi uses a **hardcoded priority list** instead of a dynamic focus tree. This is simpler and sufficient because:
- Only one modal can be "active" at a time
- The order never changes (Which is always on top)
- New layers can be inserted easily

### 2. Two-Phase Input Handling

1. **Component intercept** — `input.r#type()`, `help.r#type()` get first dibs
2. **Keymap lookup** — Generic bindings from `keymap.toml`

This lets text input work naturally while still allowing `Esc` and `Enter` to have configurable actions.

### 3. `visible` Flags Over a Scene Graph

Yazi doesn't use a scene graph or focus tree. Each component has a `visible` bool. The `Root` renderer checks each flag independently. This is:
- **Simple** — no parent/child relationships to manage
- **Fast** — just a chain of `if` checks
- **Explicit** — you can see exactly what renders when

### 4. Keymap Is Data, Not Code

All key bindings live in `keymap.toml`, not in source code. The `Chord` struct with its `const L` generic ensures type-safe layer association at compile time, while still allowing runtime configuration.

### 5. Events Unify Everything

Whether it's a keypress, a mouse click, or a programmatic action, everything becomes an `Event`. The Dispatcher normalizes all inputs into this uniform shape, making the system extensible:
- Macros? Emit `Event::Seq`
- Plugin-triggered actions? `emit!(Call(action))`
- Timer-based updates? `Event::Render`
