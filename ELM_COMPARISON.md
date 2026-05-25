# Yazi vs. The Elm Architecture (TEA)

> **Yes — Yazi's architecture is deeply influenced by The Elm Architecture. But the differences are where the real engineering insights live.**

---

## Table of Contents

1. [What Is The Elm Architecture?](#what-is-the-elm-architecture)
2. [The Side-by-Side Mapping](#the-side-by-side-mapping)
3. [Similarities: Where They Agree](#similarities-where-they-agree)
4. [Difference 1: Mutability vs. Immutability](#difference-1-mutability-vs-immutability)
5. [Difference 2: One Update Function vs. Many Actors](#difference-2-one-update-function-vs-many-actors)
6. [Difference 3: Pure vs. Impure](#difference-3-pure-vs-impure)
7. [Difference 4: Virtual DOM vs. Direct Rendering](#difference-4-virtual-dom-vs-direct-rendering)
8. [Difference 5: `Cmd` vs. Message Passing + Direct I/O](#difference-5-cmd-vs-message-passing--direct-io)
9. [Difference 6: Single-Threaded vs. Multi-Threaded](#difference-6-single-threaded-vs-multi-threaded)
10. [Why These Differences Exist](#why-these-differences-exist)
11. [What Yazi Could Learn From Elm](#what-yazi-could-learn-from-elm)
12. [What Elm Could Learn From Yazi](#what-elm-could-learn-from-yazi)
13. [The Verdict](#the-verdict)

---

## What Is The Elm Architecture?

The Elm Architecture (TEA), popularized by the Elm programming language, is a pattern for building interactive programs with three core concepts:

```
┌─────────────────────────────────────────────────────────────┐
│                    THE ELM ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐   │
│  │   Model     │────►│    View     │────►│    HTML     │   │
│  │  (State)    │     │  (Render)   │     │  (Output)   │   │
│  └──────▲──────┘     └─────────────┘     └──────┬──────┘   │
│         │                                         │          │
│         │                                         │          │
│         │    ┌─────────────┐     ┌─────────────┐  │          │
│         └─── │   Update    │◄────│    Msg      │◄─┘          │
│              │  (Logic)    │     │  (Events)   │             │
│              └──────┬──────┘     └─────────────┘             │
│                     │                                        │
│                     │     ┌─────────────┐                    │
│                     └──►  │    Cmd      │                    │
│                           │  (Effects)  │                    │
│                           └──────┬──────┘                    │
│                                  │                           │
│                                  ▼                           │
│                           ┌─────────────┐                    │
│                           │  (The World) │                    │
│                           │ HTTP, Time,  │                    │
│                           │  Keyboard    │                    │
│                           └─────────────┘                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**The three pillars:**

1. **`Model`** — Immutable application state
2. **`update : Msg -> Model -> (Model, Cmd Msg)`** — Pure function that takes a message and the current model, returns a new model and commands (effects)
3. **`view : Model -> Html Msg`** — Pure function that takes the model and returns HTML

**Key properties:**
- **Unidirectional data flow** — Messages flow in one direction: events → update → view
- **Immutability** — `Model` is never mutated; `update` returns a new one
- **Purity** — `update` and `view` are pure functions (no side effects)
- **Centralized state** — All state lives in one `Model`
- **Message passing** — All state transitions happen through `Msg`

---

## The Side-by-Side Mapping

| Elm Concept | Yazi Equivalent | File |
|-------------|-----------------|------|
| `Model` | `Core` | `yazi-core/src/core.rs` |
| `Msg` | `Event` + `Action` | `yazi-shared/src/event/event.rs` |
| `update` | `Actor::act()` | `yazi-actor/src/actor.rs` |
| `view` | `Root::render()` | `yazi-fm/src/root.rs` |
| `Cmd` | `Scheduler` + `AppProxy` | `yazi-scheduler/`, `yazi-core/src/proxy.rs` |
| `subscriptions` | `Event::take()` + `rx.recv()` | `yazi-fm/src/app/app.rs` |
| `Html` | `ratatui::Buffer` | `yazi-fm/src/*/mod.rs` |

**Yazi is TEA adapted for Rust, terminals, and multi-threaded async I/O.**

---

## Similarities: Where They Agree

### 1. Centralized State

Both architectures reject distributed state in favor of a **single source of truth**.

**Elm:**
```elm
type alias Model =
    { files : List File
    , cursor : Int
    , yanked : List File
    , modal : Maybe Modal
    }
```

**Yazi:**
```rust
pub struct Core {
    pub mgr: Mgr,
    pub tasks: Tasks,
    pub input: Input,
    pub confirm: Confirm,
    pub help: Help,
    pub cmp: Cmp,
    pub which: Which,
}
```

### 2. Message Passing for State Transitions

Both architectures insist that state changes happen through discrete messages, not direct mutation by random functions.

**Elm:**
```elm
type Msg
    = Arrow Int
    | Yank
    | Paste
    | Open
    | InputChanged String
```

**Yazi:**
```rust
pub enum Event {
    Call(ActionCow),      // "Execute this actor"
    Seq(Vec<ActionCow>),  // "Execute these actors in order"
    Key(KeyEvent),        // "User pressed a key"
    Mouse(MouseEvent),    // "User moved the mouse"
}
```

### 3. Unidirectional Data Flow

Both architectures follow the same cycle:

```
User Input → Message → State Update → Render → Display
```

Neither allows components to directly mutate each other's state. Changes flow through a central mechanism.

### 4. Separation of State, Logic, and Rendering

| Concern | Elm | Yazi |
|---------|-----|------|
| State | `Model` (immutable) | `Core` (mutable) |
| Logic | `update` (pure) | `Actor::act` (imperative) |
| Rendering | `view` → Virtual DOM | `Root::render` → Terminal buffer |
| Effects | `Cmd` (declarative) | `Scheduler` + proxies (event-driven) |

### 5. The "Impossible State" Problem Is Solved Similarly

In Elm, you use custom types to make illegal states unrepresentable:

```elm
type Modal
    = InputModal InputState
    | ConfirmModal ConfirmState
    | HelpModal HelpState
```

Yazi uses the `Layer` enum + `visible` flags:

```rust
// These are mutually exclusive — enforced by Core::layer() priority
if self.input.visible { Layer::Input }
else if self.confirm.visible { Layer::Confirm }
else { Layer::Mgr }
```

Both avoid representing states like "input is visible AND confirm is visible" as meaningful.

---

## Difference 1: Mutability vs. Immutability

### Elm: Immutable State

```elm
update : Msg -> Model -> (Model, Cmd Msg)
update msg model =
    case msg of
        Arrow n ->
            ( { model | cursor = model.cursor + n }
            , Cmd.none
            )

        Yank ->
            ( { model | yanked = model.selected }
            , Cmd.none
            )
```

Elm's `update` returns a **new** model. The old model is discarded. This is possible because:
- Elm compiles to JavaScript (GC handles memory)
- Persistent data structures (trees with structural sharing) make copying cheap
- The compiler optimizes record updates

### Yazi: Mutable State

```rust
impl Actor for Yank {
    type Form = YankForm;

    fn act(cx: &mut Ctx, opt: Self::Form) -> Result<Data> {
        cx.yanked = Yanked::new(true, opt.urls);  // ← Direct mutation!
        render!();
        succ!();
    }
}
```

Yazi mutates `Core` in place via `&mut Ctx`. This is necessary because:
- Rust has no GC — persistent data structures would require `Rc<RefCell<T>>` everywhere
- `Core` contains ~50 fields — copying it per message would be a memory and CPU disaster
- Terminal rendering is not DOM diffing — it needs actual state, not snapshots

### The Trade-off

| Aspect | Elm (Immutable) | Yazi (Mutable) |
|--------|----------------|----------------|
| **Time-travel debugging** | Free — just keep a list of past models | Hard — would need to clone `Core` every frame |
| **Undo/redo** | Trivial — swap models | Needs explicit snapshot logic |
| **Testing** | Perfect — `update` is pure | Good — actors are isolated, but `Ctx` is mutable |
| **Performance** | GC + persistent structures | Zero-copy, cache-hot |
| **Reasoning** | "What if I had a different state?" | "What state do I have right now?" |

---

## Difference 2: One Update Function vs. Many Actors

### Elm: Single `update` Function

Elm has **one** `update` function that handles **all** messages:

```elm
update : Msg -> Model -> (Model, Cmd Msg)
update msg model =
    case msg of
        Arrow n ->
            let newModel = { model | cursor = model.cursor + n } in
            (newModel, Cmd.none)

        Yank ->
            let newModel = { model | yanked = model.selected } in
            (newModel, Cmd.none)

        Paste ->
            let newModel = pasteFiles model in
            (newModel, copyFiles model.selected model.target)

        -- ... hundreds more cases
```

This is elegant but becomes a **giant switch statement** as the app grows. Elm mitigates this with:
- **Component modules** — each has its own `update` and `Msg` type
- **Top-level delegation** — the main `update` calls sub-updates

### Yazi: Many Actors

Yazi splits the monolithic `update` into **80+ independent actors**:

```rust
// Each actor is a separate struct + trait impl + file
pub struct Cd;
pub struct Yank;
pub struct Open;
pub struct Rename;
pub struct Sort;
// ... 80 more

pub trait Actor {
    type Form;
    const NAME: &str;
    fn act(cx: &mut Ctx, form: Self::Form) -> Result<Data>;
}
```

The `Executor` routes actions to actors, but the logic itself is modular:

```rust
fn mgr(&mut self, action: ActionCow) -> Result<Data> {
    let cx = &mut Ctx::new(&action, &mut self.app.core, ...)?;

    if action.name == "cd"     { return act!(mgr:cd, cx, action); }
    if action.name == "yank"   { return act!(mgr:yank, cx, action); }
    if action.name == "open"   { return act!(mgr:open, cx, action); }
    // ... dozens more
}
```

### The Trade-off

| Aspect | Elm (Single Update) | Yazi (Many Actors) |
|--------|---------------------|-------------------|
| **Discoverability** | All logic in one place | Scattered across files |
| **Modularity** | Components with delegation | Each actor is a module |
| **Type safety** | `Msg` enum guarantees exhaustiveness | `match` on string names (runtime dispatch) |
| **Compile-time errors** | Missing `Msg` case is a compiler error | Wrong action name is a runtime error |
| **Hot-swapping logic** | Hard — all logic is one function | Easy — just add a new actor file |
| **Plugin hooks** | Requires explicit extension points | `Actor::hook()` intercepts any actor |

Yazi's `act!` macro + `Executor` pattern is essentially Elm's `update` function **spread across a module hierarchy**. The routing table (`Executor::mgr()`) is the moral equivalent of Elm's `case msg of`.

---

## Difference 3: Pure vs. Impure

### Elm: Pure Functions Everywhere

```elm
update : Msg -> Model -> (Model, Cmd Msg)
--      ↑ Msg is data
--              ↑ Model is data
--                      ↑ Returns data (new Model)
--                              ↑ Cmd is a data structure representing an effect
```

`update` is a **pure function**. It does not:
- Mutate any state
- Make HTTP requests
- Read from localStorage
- Generate random numbers

Effects are represented as **data** (`Cmd`), which the Elm runtime interprets:

```elm
update msg model =
    case msg of
        FetchData ->
            ( { model | loading = True }
            , Http.get { url = "/api/data", expect = DataReceived }
            )
```

`Http.get` returns a `Cmd` — a data structure saying "please make this HTTP request and send me back a `DataReceived` message."

### Yazi: Imperative Actors

```rust
impl Actor for Cd {
    fn act(cx: &mut Ctx, opt: CdForm) -> Result<Data> {
        cx.mgr.tabs.active_mut().cd(&opt.target)?;
        // ↑ Directly mutates state!

        if opt.interactive {
            cx.mgr.remove_history(&opt.target);
        }
        // ↑ More direct mutation!

        render!();
        // ↑ Side effect (sets atomic flag)!

        succ!();
    }
}
```

Yazi actors are **imperative**. They:
- Mutate `Core` directly via `&mut Ctx`
- Call `render!()` which sets `NEED_RENDER` atomically
- Submit work to the scheduler via `Arc<Scheduler>`
- Emit events via `emit!()` which calls `mpsc::UnboundedSender::send()`

### Why Yazi Can't Be Pure

Elm purity works because:
- **JavaScript is single-threaded** — no race conditions to worry about
- **Elm runtime handles effects** — the runtime, not user code, does I/O
- **GC makes copying cheap** — returning a new model is free-ish

Yazi is imperative because:
- **Rust is a systems language** — zero-cost abstractions demand direct mutation
- **Terminal I/O is imperative** — you can't represent "draw this" as pure data efficiently
- **Multi-threading requires explicit control** — async I/O needs direct spawn calls
- **File operations are effects** — reading directory contents is inherently impure

### The `Cmd` Equivalent in Yazi

Yazi does have a limited form of `Cmd` — the `emit!` macro + proxy pattern:

```rust
// Actor: "I want to do something async"
AppProxy::plugin(opt);
// → emit!(Call(relay!(app:plugin).with_any("opt", opt)))
// → Sends Event::Call into the channel
```

But this is **not** a pure data structure. It's an immediate side effect. The "runtime" (the event loop) receives it and processes it, but the actor already caused the side effect.

---

## Difference 4: Virtual DOM vs. Direct Rendering

### Elm: Virtual DOM Diffing

```elm
view : Model -> Html Msg
view model =
    div []
        [ h1 [] [ text "File Manager" ]
        , ul [] (List.map viewFile model.files)
        , viewModal model.modal
        ]
```

Elm's `view` returns a **Virtual DOM** tree. The Elm runtime:
1. Calls `view` to get the new VDOM
2. Diff's it against the previous VDOM
3. Applies only the **minimal DOM mutations** needed

This means:
- `view` can be called on every frame
- The runtime optimizes actual DOM changes
- Components can re-render independently

### Yazi: Direct Terminal Buffer Writes

```rust
impl Widget for Root<'_> {
    fn render(self, area: Rect, buf: &mut Buffer) {
        // 1. Draw base layer (Lua-driven file list)
        render_once(root.call_method("redraw", ())?, buf, ...);

        // 2. Draw each overlay, bottom to top
        if self.core.tasks.visible {
            tasks::Tasks::new(self.core).render(area, buf);
        }
        if self.core.input.visible {
            input::Input::new(self.core).render(area, buf);
        }
        // ... more overlays
    }
}
```

Yazi writes directly to a `ratatui::Buffer`, which is a 2D grid of cells. There is:
- **No virtual DOM** — each render pass redraws everything
- **No diffing** — ratatui optimizes by tracking dirty regions internally
- **No component isolation** — `Root::render()` orchestrates everything

### Why No Virtual DOM?

| Aspect | Browser (Elm) | Terminal (Yazi) |
|--------|--------------|-----------------|
| **DOM size** | Thousands of nodes | ~2000 cells (80x25 terminal) |
| **Update cost** | DOM manipulation is expensive | Writing to a buffer is cheap |
| **Partial updates** | Critical for performance | Full redraw is fast enough |
| **Styling** | Complex CSS cascade | Simple foreground/background colors |
| **Event handling** | Event delegation on DOM | Direct key event routing |

A terminal is ~2000 cells. A browser page is ~10,000+ DOM nodes. Full terminal redraws are so fast that diffing is unnecessary overhead.

---

## Difference 5: `Cmd` vs. Message Passing + Direct I/O

### Elm: Declarative Effects

```elm
update msg model =
    case msg of
        Paste ->
            ( { model | loading = True }
            , Http.post
                { url = "/api/paste"
                , body = jsonBody model.yanked
                , expect = PasteCompleted
                }
            )

        PasteCompleted (Ok result) ->
            ( { model | loading = False, files = result }
            , Cmd.none
            )
```

Elm's `Cmd` is **declarative**: "Here is what I want to happen." The runtime handles the actual HTTP request. When it completes, the runtime sends a `PasteCompleted` message back to `update`.

**Key properties:**
- `update` never calls `fetch()` directly
- Effects are **composable** — `Cmd.batch [ cmd1, cmd2 ]`
- Effects are **testable** — just compare the `Cmd` data structure
- No callbacks, no promises, no async/await

### Yazi: Direct Scheduler Submission + Event Bus

```rust
impl Actor for Paste {
    fn act(cx: &mut Ctx, opt: PasteForm) -> Result<Data> {
        for u in opt.sources {
            cx.tasks.scheduler.file(FileIn::Copy { src: u, dest: opt.target });
            // ↑ Directly submits work to a background worker!
        }
        succ!();
    }
}

// Meanwhile, in a background worker...
tokio::spawn(async move {
    // ... copy files ...
    AppProxy::update_progress(new_summary);
    // ↑ Emits an event that the UI loop will receive
});
```

Yazi splits effects across two mechanisms:

1. **Direct submission** — Actors call `scheduler.file()` directly (imperative)
2. **Event bus** — Workers emit events that the UI loop receives (message passing)

There is no "runtime" managing effects. The scheduler is a library, not a framework.

### The `Cmd` Gap

Yazi doesn't have an exact `Cmd` equivalent. The closest is:

```rust
// This is what a "Cmd" looks like in Yazi:
pub enum FileIn {
    Copy { id: Id, src: UrlBuf, dest: UrlBuf },
    Cut  { id: Id, src: UrlBuf, dest: UrlBuf },
    Delete { id: Id, target: UrlBuf },
    // ...
}
```

`FileIn` is a **data structure representing a file operation**. The scheduler receives it and executes it. But:
- **No composition** — you can't `Cmd.batch` file operations
- **No cancellation** at the "Cmd" level — cancellation is done via `CompletionToken`
- **No testing** — you can't easily assert "this actor should produce a `FileIn::Copy`"

---

## Difference 6: Single-Threaded vs. Multi-Threaded

### Elm: Single-Threaded (Cooperative)

Elm runs in the browser's single JavaScript thread. Everything is cooperative:

```elm
-- This blocks the entire UI until it completes
heavyComputation : Model -> Model
heavyComputation model =
    List.foldl process model (List.range 1 1000000)
```

To avoid blocking, Elm uses:
- **`Process.sleep`** — yields control to the event loop
- **`Task` chains** — breaks work into small steps
- **Web Workers** — for true parallelism (rarely used in Elm)

### Yazi: Single-Threaded UI + Multi-Threaded Workers

Yazi runs on `tokio` with a **multi-threaded runtime**. But the UI loop is pinned to a single task:

```rust
// UI thread: single task, single &mut Core
async fn serve() {
    let mut app = App::new()?;
    loop {
        let event = rx.recv().await;
        app.dispatch(event)?;  // ← Synchronous actor execution
    }
}

// Background: many tokio tasks
fn file_worker(rx: Receiver<FileIn>) -> JoinHandle<()> {
    tokio::spawn(async move {
        while let Ok((job, _)) = rx.recv().await {
            do_file_work(job).await;
            AppProxy::work_done(result);
        }
    })
}
```

**This is TEA extended for multi-core systems.**

The UI is single-threaded (like Elm), but background work is multi-threaded (unlike Elm). Workers communicate with the UI through the event bus — the same message-passing pattern, but across thread boundaries.

### The Concurrency Model

| Concern | Elm | Yazi |
|---------|-----|------|
| **UI updates** | Single-threaded | Single-threaded (`&mut Core`) |
| **HTTP requests** | Runtime-managed async | `tokio::spawn` + channels |
| **File I/O** | Not applicable (browser) | `tokio::fs` + worker pool |
| **Parallel work** | Web Workers (explicit) | `tokio::spawn` (implicit) |
| **Communication** | `Port` (JS interop) | `tokio::sync::mpsc` |

---

## Why These Differences Exist

The differences between Yazi and Elm are not accidental — they are **domain-driven**:

| Domain Constraint | Elm (Browser) | Yazi (Terminal) |
|-------------------|---------------|-----------------|
| **Runtime** | JavaScript (GC, single-threaded) | Rust (no GC, multi-threaded) |
| **Rendering target** | HTML DOM (complex, dynamic) | Terminal cells (simple, static) |
| **I/O model** | Event-driven, async callbacks | `tokio`, multi-threaded workers |
| **State size** | Small (UI state only) | Large (files, metadata, previews) |
| **State access pattern** | Read-heavy (VDOM diffing) | Read-write mix (direct mutations) |
| **Effect complexity** | HTTP requests, WebSockets | File I/O, subprocesses, plugins |

**Elm optimizes for correctness and simplicity** in a browser environment. **Yazi optimizes for performance and control** in a systems environment.

---

## What Yazi Could Learn From Elm

### 1. Effect Representation as Data

Yazi's `AppProxy` / `MgrProxy` pattern could be formalized into a proper `Effect` enum:

```rust
// Hypothetical Yazi "Cmd"
pub enum Effect {
    FileCopy { src: UrlBuf, dest: UrlBuf },
    FileDelete { target: UrlBuf },
    PluginRun { name: String, args: Data },
    HttpRequest { url: String },
    Render,
}

// Actor returns effects instead of mutating directly
fn act(cx: &mut Ctx, opt: PasteForm) -> Result<(Data, Vec<Effect>)> {
    let effects = opt.sources.into_iter()
        .map(|src| Effect::FileCopy { src, dest: opt.target })
        .collect();
    Ok((Data::Nil, effects))
}
```

This would make actors:
- **More testable** — assert on effects, not state mutations
- **More composable** — batch effects together
- **More analyzable** — static analysis could track all possible effects

### 2. Model Update Purity

If Yazi separated "compute new state" from "apply new state":

```rust
// Instead of:
fn act(cx: &mut Ctx, opt: CdForm) -> Result<Data> {
    cx.mgr.tabs.active_mut().cd(&opt.target)?;  // ← mutates
    render!();
    succ!();
}

// Imagine:
fn update(core: &Core, opt: CdForm) -> Result<Core> {
    let mut new_core = core.clone();  // ← expensive, but pure
    new_core.mgr.tabs.active_mut().cd(&opt.target)?;
    Ok(new_core)
}
```

This would enable:
- **Time-travel debugging** — keep a Vec<Core> history
- **Optimistic UI** — show state before effects complete
- **Undo** — just swap back to a previous Core

But cloning `Core` (which contains ~50 fields, Vecs, HashMaps) is prohibitively expensive in Rust without a GC.

### 3. Exhaustive Message Handling

Elm's `case msg of` is exhaustive — the compiler warns if you forget a message case. Yazi's string-based dispatch (`if action.name == "cd"`) is not:

```rust
// This compiles fine even though "cd" is misspelled:
if action.name == "cd_typo" {  // ← silent bug
    return act!(mgr:cd, cx, action);
}
// "cd" action falls through to succ!() — does nothing
```

Yazi could use a macro-generated enum for action names:

```rust
enum MgrAction {
    Cd,
    Yank,
    Open,
    // ...
}

// Compiler guarantees all cases are handled
match action {
    MgrAction::Cd => act!(mgr:cd, cx, opt),
    MgrAction::Yank => act!(mgr:yank, cx, opt),
    // Missing a case? Compiler error!
}
```

### 4. Decoupled Rendering

Elm's `view` is a pure function of state. Yazi's rendering is mixed with state:

```rust
// Yazi: render flag is a side effect
fn act(cx: &mut Ctx, opt: CdForm) -> Result<Data> {
    cx.mgr.tabs.active_mut().cd(&opt.target)?;
    render!();  // ← side effect
    succ!();
}
```

If Yazi tracked "render needed" in the return type:

```rust
fn act(cx: &mut Ctx, opt: CdForm) -> Result<(Data, bool)> {
    let changed = cx.mgr.tabs.active_mut().cd(&opt.target)?;
    Ok((Data::Nil, changed))  // ← pure: "I changed state, you decide to render"
}
```

The event loop would call `render()` based on the return value, not an atomic flag.

---

## What Elm Could Learn From Yazi

### 1. Layered Modal System

Elm handles modals via conditional rendering in `view`:

```elm
view model =
    div []
        [ viewMain model
        , case model.modal of
            Just m -> viewModal m
            Nothing -> text ""
        ]
```

Yazi's `Core::layer()` + priority stack is more sophisticated:
- **Multiple overlapping modals** (Which on top of Cmp on top of Input)
- **Per-layer keymaps** — each modal has its own key bindings
- **Layer-specific input interception** — Help filter mode captures all keys

Elm apps often struggle with "modal z-index" and keyboard shortcuts in nested modals. Yazi's explicit priority stack is a clean solution.

### 2. Actor Decomposition

Elm's single `update` function becomes unwieldy at scale. Yazi's actor pattern shows how to split it:

```elm
-- Elm: one giant update
update msg model =
    case msg of
        MgrMsg m -> updateMgr m model
        InputMsg m -> updateInput m model
        -- ...

-- Inspired by Yazi: each component is an "actor"
type alias Actor model msg =
    { act : msg -> model -> (model, Cmd msg)
    , name : String
    }
```

### 3. Multi-Threading via Message Passing

Elm is strictly single-threaded. Yazi shows how to extend TEA to multi-core:

```elm
-- Hypothetical Elm with workers
type Msg
    = StartCopy Files
    | CopyProgressed Progress
    | CopyCompleted Result

update msg model =
    case msg of
        StartCopy files ->
            ( model
            , Worker.send FileWorker (CopyCmd files)
            )

        CopyProgressed p ->
            ( { model | progress = p }, Cmd.none )
```

Elm's `Port` system is the closest thing, but it's JavaScript interop, not first-class concurrency.

### 4. Proxy Pattern for Decoupling

Yazi's `AppProxy` / `MgrProxy` allow any code to trigger UI updates without holding a reference to `App`:

```rust
// Deep inside a background worker:
AppProxy::update_progress(summary);
// → Works from anywhere, no &mut App needed
```

Elm's equivalent is `Cmd`, but it's less flexible — you can only return `Cmd` from `update`. Yazi's proxies work from any async context.

---

## The Verdict

### Is Yazi "The Elm Architecture in Rust"?

**Yes, at the conceptual level.** Both architectures share:
- Centralized state
- Message-passing state transitions
- Unidirectional data flow
- Separation of state, logic, and rendering

**No, at the implementation level.** The differences are deep:

| Dimension | Elm | Yazi |
|-----------|-----|------|
| **State** | Immutable | Mutable (`&mut Core`) |
| **Update** | One pure function | Many imperative actors |
| **Effects** | Declarative `Cmd` | Imperative + event bus |
| **Rendering** | Virtual DOM | Direct buffer writes |
| **Concurrency** | Single-threaded | Multi-threaded workers |
| **Language** | Functional (Haskell-like) | Imperative (systems) |

### The Right Mental Model

Think of Yazi as **"TEA adapted for Rust's borrow checker, terminal constraints, and multi-threaded async I/O."**

| TEA Concept | Yazi Adaptation | Reason |
|-------------|----------------|--------|
| `Model` | `Core` (owned, mutable) | Rust has no GC; cloning large structs is expensive |
| `update` | `Actor::act()` (many, imperative) | Rust's module system favors small, testable units |
| `Msg` | `Event` + `Action` (string-based) | Dynamic plugin system needs runtime dispatch |
| `Cmd` | `Scheduler` + proxies | Direct I/O is idiomatic in systems programming |
| `view` | `Root::render()` | Terminals are small enough for full redraws |
| Subscriptions | `Event::take()` + `rx.recv()` | `tokio::sync::mpsc` is the Rust idiom |

### The Core Insight

Both architectures answer the same question: **"How do you manage state in a complex interactive application?"**

Elm answers: **"With purity, immutability, and a runtime that manages effects."**

Yazi answers: **"With exclusive ownership, message passing, and direct mutation within a single-threaded loop."**

Both are correct for their domains. Elm is beautiful for web apps. Yazi is fast for terminal file management. The shared DNA — centralized state, message passing, unidirectional flow — is what makes both architectures robust.

The lesson: **architecture patterns transcend languages and domains, but their implementations must adapt to the constraints of the runtime.**
