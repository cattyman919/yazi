# Yazi Project Architecture Analysis

> **Project:** Yazi — A blazing fast terminal file manager  
> **Version:** 26.2.2  
> **Language:** Rust (Edition 2024, MSRV 1.95.0)  
> **Workspace:** 28 crates (`yazi-*`)

---

## Table of Contents

1. [Executive Overview](#executive-overview)
2. [Actor-Based Architecture](#actor-based-architecture)
3. [Event Flow & Dispatch System](#event-flow--dispatch-system)
4. [Core State Model](#core-state-model)
5. [Scheduler & Worker Pool Architecture](#scheduler--worker-pool-architecture)
6. [DDS — Data Distribution Service](#dds--data-distribution-service)
7. [Proxy Pattern for Decoupled Communication](#proxy-pattern-for-decoupled-communication)
8. [Crate Dependency Graph](#crate-dependency-graph)
9. [Crate-by-Crate Breakdown](#crate-by-crate-breakdown)
10. [Key Architectural Patterns](#key-architectural-patterns)

---

## Executive Overview

Yazi is a highly modular terminal file manager built as a Rust workspace with **28 internal crates**. Its architecture is built around three foundational pillars:

1. **Actor Model** — Every UI action, file operation, and system event is modeled as a message sent to an actor that processes it synchronously against the central `Core` state.
2. **Event-Driven Dispatch** — A global async event bus (`tokio::sync::mpsc`) decouples input handling, rendering, and background work.
3. **Multi-Worker Scheduler** — A priority-based worker pool handles file I/O, plugin execution, previews, and subprocesses concurrently, with cancellation tokens and progress tracking.

The result is a **single-threaded UI loop** (for state consistency) combined with **multi-threaded background workers** (for performance), communicating through message passing.

---

## Actor-Based Architecture

### The `Actor` Trait

At the heart of Yazi's design is the `Actor` trait defined in `yazi-actor/src/actor.rs`:

```rust
pub trait Actor {
    type Form;                          // Input data type for this action
    const NAME: &str;                   // Human-readable action name

    fn act(cx: &mut Ctx, form: Self::Form) -> Result<Data>;
    fn hook(_cx: &Ctx, _form: &Self::Form) -> Option<SparkKind> { None }
}
```

Every user-facing action is an implementation of `Actor`:

- `app:bootstrap`, `app:quit`, `app:resize`
- `mgr:cd`, `mgr:arrow`, `mgr:open`, `mgr:yank`, `mgr:paste`
- `tasks:spawn`, `tasks:cancel`
- `input:show`, `input:close`
- `help:toggle`, `cmp:trigger`, `which:activate`

### The `act!` Macro

The `yazi_macro::act!` macro (in `yazi-macro/src/actor.rs`) is the dispatcher glue:

```rust
act!(mgr:cd, cx, action);     // Parse action into CdForm, then call Cd::act(cx, form)
act!(app:bootstrap, cx);       // Call with default form
```

The macro performs three steps:

1. **Parse** the `Action` into the actor's `Form` type via `TryFrom`
2. **Preflight** — run plugin hooks (`SparkKind`) that can transform or cancel the action
3. **Execute** the actor's `act()` method with the (possibly transformed) form

### Context (`Ctx`)

Actors don't own state. They borrow it through `Ctx<'a>` (`yazi-actor/src/context.rs`):

```rust
pub struct Ctx<'a> {
    pub core:   &'a mut Core,       // All application state
    pub term:   &'a mut Option<Term>, // Terminal handle
    pub tab:    usize,              // Active tab index
    pub level:  usize,              // Nesting level (for debugging)
    pub source: Source,             // Who triggered this action
}
```

`Ctx` implements `Deref`/`DerefMut` to `Core`, so actors can access any state directly, but always through a single mutable borrow. This ensures **no state synchronization is needed** — the UI loop processes one actor at a time.

### Actor Modules

| Module                | Actions                                                            | Purpose                 |
| --------------------- | ------------------------------------------------------------------ | ----------------------- |
| `yazi-actor::app`     | bootstrap, quit, resize, focus, theme, lua, plugin                 | Application lifecycle   |
| `yazi-actor::mgr`     | cd, arrow, enter, open, yank, paste, remove, search, sort, tab\_\* | File manager operations |
| `yazi-actor::tasks`   | spawn, cancel, arrow, inspect                                      | Background task UI      |
| `yazi-actor::input`   | show, close, escape, complete                                      | Input modal             |
| `yazi-actor::help`    | toggle, arrow, filter, escape                                      | Help modal              |
| `yazi-actor::cmp`     | trigger, show, close, arrow                                        | Completion popup        |
| `yazi-actor::which`   | activate, dismiss                                                  | Key chord hint          |
| `yazi-actor::confirm` | show, close, arrow                                                 | Confirm dialog          |
| `yazi-actor::pick`    | show, close, arrow                                                 | Picker dialog           |
| `yazi-actor::spot`    | arrow, close, swipe, copy                                          | File spotter            |
| `yazi-actor::notify`  | push, tick                                                         | Notification toast      |

---

## Event Flow & Dispatch System

### Global Event Bus

Yazi uses a single global `tokio::sync::mpsc::unbounded_channel` for all events (`yazi-shared/src/event/event.rs`):

```rust
pub enum Event {
    Call(ActionCow),          // Execute an actor action
    Seq(Vec<ActionCow>),      // Execute a sequence of actions
    Render(bool),             // Request UI render (partial/full)
    Key(KeyEvent),            // Keyboard input
    Mouse(MouseEvent),        // Mouse input
    Resize,                   // Terminal resized
    Focus,                    // Terminal focused
    Paste(String),            // Clipboard paste
}
```

Two thread-safe static cells hold the channel ends:

- `TX: RoCell<mpsc::UnboundedSender<Event>>` — for emitting events from anywhere
- `RX: RoCell<mpsc::UnboundedReceiver<Event>>` — consumed by the main app loop

### The Main Loop (`yazi-fm/src/app/app.rs`)

```rust
pub async fn serve() -> Result<()> {
    let term = Term::start()?;
    let (mut rx, signals) = (Event::take(), Signals::start()?);
    let mut app = Self { core: Core::make(), term: Some(term), signals };
    app.bootstrap()?;

    loop {
        select! {
            _ = sleep(timeout) => { app.render(need_render == 2)?; }
            n = rx.recv_many(&mut events, 50) => {
                if n == 0 { break }
                for event in events.drain(..) {
                    Dispatcher::new(&mut app).dispatch(event);
                }
            }
        }
    }
}
```

Key characteristics:

- **Batches up to 50 events** per async receive for throughput
- **10ms render throttling** — renders are deferred slightly to batch rapid updates
- **Single-threaded state mutation** — `Dispatcher` gets `&mut App` exclusive access

### Dispatcher → Executor → Actor

```
Event::Call(action)
    │
    ▼
Dispatcher::dispatch_call(action)
    │
    ▼
Executor::execute(action)  // Routes by Layer
    │
    ▼
match action.layer {
    Layer::Mgr => self.mgr(action),   // Creates Ctx, calls act!(mgr:cd, cx, action)
    Layer::App => self.app(action),
    Layer::Tasks => self.tasks(action),
    ...
}
```

The `Executor` (`yazi-fm/src/executor.rs`) is a giant match statement routing actions to their layer-specific handlers. Each handler:

1. Creates a `Ctx` from the action
2. Matches the action name
3. Invokes `act!($layer:$name, cx, action)`

### Router (Key Input)

The `Router` (`yazi-fm/src/router.rs`) maps raw `KeyEvent`s to configured key chords:

```rust
fn matches(&mut self, layer: Layer, key: Key) -> bool {
    for chord in KEYMAP.get(layer) {
        if chord.on[0] != key { continue; }
        if chord.on.len() > 1 {
            act!(which:activate, cx, (layer, key));  // Multi-key chord
        } else {
            emit!(Seq(ChordCow::from(chord).into_seq()));  // Single-key action
        }
    }
}
```

---

## Core State Model

All mutable state lives in a single `Core` struct (`yazi-core/src/core.rs`):

```rust
pub struct Core {
    pub mgr:     Mgr,        // File manager (tabs, folders, selection)
    pub tasks:   Tasks,      // Background task queue UI state
    pub pick:    Pick,       // Picker modal state
    pub input:   Input,      // Input modal state
    pub confirm: Confirm,    // Confirm dialog state
    pub help:    Help,       // Help modal state
    pub cmp:     Cmp,        // Completion popup state
    pub which:   Which,      // Key chord hint state
    pub notify:  Notify,     // Notification state
}
```

### Layer System

`Core::layer()` returns the active UI layer, forming a **stack**:

```
Which   ← top (highest priority)
Cmp
Help
Confirm
Input
Pick
Spot
Tasks
Mgr     ← bottom (default)
```

This allows modal dialogs to intercept input while still knowing what's underneath.

---

## Scheduler & Worker Pool Architecture

The `yazi-scheduler` crate manages all background work through a **multi-queue, multi-worker** design.

### Worker Queues

Each work type has its own `async_priority_channel::unbounded` queue:

| Queue        | Workers           | Work Type                                                  |
| ------------ | ----------------- | ---------------------------------------------------------- |
| `file_rx`    | `file_workers`    | Copy, cut, delete, trash, link, hardlink, download, upload |
| `plugin_rx`  | `plugin_workers`  | Plugin entry execution                                     |
| `fetch_rx`   | `fetch_workers`   | MIME type fetching                                         |
| `preload_rx` | `preload_workers` | File preview preloading                                    |
| `size_rx`    | 3                 | Directory size calculation                                 |
| `process_rx` | `process_workers` | External process spawn (block, bg, orphan)                 |
| `hook_rx`    | 3                 | Post-operation hooks (copy, cut, trash, etc.)              |
| `op_rx`      | 1                 | Task completion & progress aggregation                     |

### Worker Spawn Pattern

Each worker is a `tokio::spawn` async task that loops forever:

```rust
fn file(&self, rx: Receiver<FileIn, u8>) -> JoinHandle<()> {
    let me = self.clone();
    tokio::spawn(async move {
        loop {
            if let Ok((r#in, _)) = rx.recv().await {
                let id = r#in.id();
                let Some(token) = me.ongoing.lock().get_token(id) else { continue; };

                let result = select! {
                    r = me.file_do(r#in) => r,
                    false = token.future() => Ok(())  // Cancelled!
                };

                if let Err(out) = result {
                    me.ops.out(id, out);
                }
            }
        }
    })
}
```

Key features:

- **Cancellation via `CompletionToken`** — every task has a token; setting it cancels the `select!`
- **Priority channels** — `async_priority_channel` lets high-priority tasks jump ahead
- **Op aggregation** — a dedicated `op` worker aggregates progress updates and triggers hooks

### Task Lifecycle

```
Scheduler::add(FileIn::Copy { ... })
    │
    ▼
Ongoing::add() → creates Task with CompletionToken
    │
    ▼
sender.send((FileIn, priority)) → worker queue
    │
    ▼
Worker receives → runs file.copy() → sends TaskOp on completion
    │
    ▼
Op worker → reduces progress → if done, runs Hook → fulfills task
```

---

## DDS — Data Distribution Service

`yazi-dds` enables **inter-process communication** between Yazi instances and plugin pub/sub.

### Components

| Component | Purpose                                                               |
| --------- | --------------------------------------------------------------------- |
| `Client`  | Sends DDS messages to other Yazi instances over Unix/TCP sockets      |
| `Server`  | Accepts incoming DDS connections                                      |
| `Pump`    | Processes DDS event queues (duplicate, move, trash, delete, download) |
| `Pubsub`  | Local and remote pub/sub for plugins                                  |
| `Ember`   | Typed message payloads (hey, hi, cd, hover, yank, rename, etc.)       |

### Pub/Sub Model

Plugins can subscribe to events locally or remotely:

```rust
pub static LOCAL:  RoCell<RwLock<HashMap<String, HashMap<String, Function>>>>;
pub static REMOTE: RoCell<RwLock<HashMap<String, HashMap<String, Function>>>>;
```

- `LOCAL` — callbacks run in the current Yazi instance
- `REMOTE` — callbacks run when messages arrive from peer instances

When an action occurs (e.g., `cd`), DDS:

1. Flushes to local subscribers immediately
2. Broadcasts to remote peers via `Client::push()`
3. Remote peers dispatch to their subscribers

### Environment Propagation

DDS propagates identity across nested Yazi instances:

- `YAZI_ID` — unique instance ID
- `YAZI_PID` — parent instance ID
- `YAZI_LEVEL` — nesting depth (for `yazi` launched from within `yazi`)

---

## Proxy Pattern for Decoupled Communication

Yazi uses a **Proxy** pattern to let background tasks and plugins emit events without holding `App` references.

### `AppProxy` / `MgrProxy` (`yazi-core/src/proxy.rs`)

```rust
pub struct AppProxy;
impl AppProxy {
    pub fn plugin(opt: PluginOpt) {
        emit!(Call(relay!(app:plugin).with_any("opt", opt)));
    }
    pub fn update_progress(summary: TaskSummary) {
        emit!(Call(relay!(app:update_progress).with_any("summary", summary)));
    }
}

pub struct MgrProxy;
impl MgrProxy {
    pub fn update_paged_by<U>(page: usize, only_if: U) {
        emit!(Call(relay!(mgr:update_paged, [page]).with("only-if", only_if.as_url())));
    }
}
```

Proxies construct `Action` objects and `emit!` them into the global event bus. This lets:

- The scheduler update task progress from worker threads
- The previewer send rendered content back to the UI
- Plugins trigger UI actions asynchronously

### `yazi-proxy` Crate

`yazi-proxy` provides a **type-safe** interface for external crates:

```rust
pub struct AppProxy;
impl AppProxy {
    pub fn quit(opt: QuitOpt) { emit!(Call(relay!(app:quit).with_any("opt", opt))); }
    pub fn plugin_do(opt: PluginOpt) { emit!(Call(relay!(app:plugin_do).with_any("opt", opt))); }
}

pub struct MgrProxy;
impl MgrProxy {
    pub fn hover(url: &Url) { emit!(Call(relay!(mgr:hover).with("url", url))); }
    pub fn peek(target: Url, force: bool) { ... }
}
```

This is the **public API** for plugins and background workers to affect UI state.

---

## Crate Dependency Graph

### Foundation Layer (No internal deps except macro/shim)

```
yazi-macro   → (none, pure macros)
yazi-shim    → yazi-macro
yazi-codegen → (none, proc macros)
yazi-ffi     → yazi-macro
```

### Utilities Layer

```
yazi-shared  → yazi-macro, yazi-shim
yazi-tty     → yazi-macro, yazi-shared, yazi-shim
yazi-emulator→ yazi-macro, yazi-shared, yazi-shim, yazi-tty
yazi-fs      → yazi-macro, yazi-shared, yazi-shim, yazi-ffi
```

### Configuration Layer

```
yazi-config  → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-tty, yazi-codegen
yazi-boot    → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, yazi-adapter, yazi-emulator, yazi-vfs
```

### UI Primitives Layer

```
yazi-widgets → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-adapter, yazi-tty
yazi-binding → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-fs, yazi-adapter, yazi-vfs, yazi-widgets, yazi-codegen
yazi-term    → yazi-macro, yazi-shim, yazi-config, yazi-emulator, yazi-tty
yazi-adapter → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-fs, yazi-emulator, yazi-tty
```

### Background Services Layer

```
yazi-vfs     → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, yazi-sftp
yazi-watcher → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-adapter, yazi-dds, yazi-vfs
yazi-dds     → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-boot, yazi-binding
yazi-runner  → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, yazi-boot, yazi-dds, yazi-binding
yazi-scheduler→ yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, yazi-dds, yazi-runner, yazi-binding, yazi-term, yazi-vfs
```

### Core & Actor Layer

```
yazi-core    → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-fs, yazi-adapter, yazi-binding, yazi-dds, yazi-emulator, yazi-runner, yazi-scheduler, yazi-vfs, yazi-watcher, yazi-widgets, yazi-prebuilt (external)
yazi-parser  → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, yazi-core, yazi-dds, yazi-binding, yazi-scheduler, yazi-vfs, yazi-widgets, yazi-boot
yazi-proxy   → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-core, yazi-scheduler, yazi-widgets
yazi-actor   → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-fs, yazi-core, yazi-dds, yazi-emulator, yazi-parser, yazi-plugin, yazi-proxy, yazi-runner, yazi-scheduler, yazi-term, yazi-tty, yazi-vfs, yazi-watcher, yazi-widgets, yazi-binding, yazi-boot
```

### Plugin & Application Layer

```
yazi-plugin  → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, yazi-core, yazi-dds, yazi-adapter, yazi-binding, yazi-emulator, yazi-proxy, yazi-runner, yazi-scheduler, yazi-term, yazi-vfs, yazi-widgets, yazi-boot
yazi-fm      → ALL CRATES (the main application binary)
yazi-cli     → yazi-boot, yazi-dds, yazi-fs, yazi-macro, yazi-shared, yazi-shim
```

### Visual Dependency Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION BINARIES                       │
│  yazi-fm (main TUI)          yazi-cli (command-line tool)     │
├─────────────────────────────────────────────────────────────┤
│                    ACTOR & PLUGIN LAYER                       │
│  yazi-actor    yazi-plugin    yazi-parser    yazi-proxy       │
├─────────────────────────────────────────────────────────────┤
│                    CORE STATE LAYER                           │
│  yazi-core                                                    │
├─────────────────────────────────────────────────────────────┤
│                    BACKGROUND SERVICES                        │
│  yazi-scheduler  yazi-runner  yazi-dds  yazi-watcher  yazi-vfs│
├─────────────────────────────────────────────────────────────┤
│                    UI PRIMITIVES                              │
│  yazi-widgets  yazi-binding  yazi-term  yazi-adapter          │
├─────────────────────────────────────────────────────────────┤
│                    CONFIGURATION                              │
│  yazi-config  yazi-boot                                       │
├─────────────────────────────────────────────────────────────┤
│                    UTILITIES                                  │
│  yazi-shared  yazi-fs  yazi-tty  yazi-emulator                │
├─────────────────────────────────────────────────────────────┤
│                    FOUNDATION                                 │
│  yazi-macro  yazi-shim  yazi-codegen  yazi-ffi                │
└─────────────────────────────────────────────────────────────┘
```

---

## Crate-by-Crate Breakdown

| Crate            | Description                                  | Key Types                                               |
| ---------------- | -------------------------------------------- | ------------------------------------------------------- |
| `yazi-actor`     | **Actor trait + all action implementations** | `Actor`, `Ctx`, `Bootstrap`, `Cd`, `Open`, `Yank`, etc. |
| `yazi-adapter`   | Terminal capability detection                | Image protocol adapters (kitty, sixel, iTerm2)          |
| `yazi-binding`   | **Lua ↔ Rust bindings**                      | `Renderable`, `elements`, `runtime_scope!`              |
| `yazi-boot`      | CLI argument parsing & boot sequence         | `BOOT`, `ARGS`                                          |
| `yazi-build`     | Build scripts                                | —                                                       |
| `yazi-cli`       | `ya` command-line utility binary             | —                                                       |
| `yazi-codegen`   | Proc macros for derive                       | —                                                       |
| `yazi-config`    | Configuration loading & validation           | `YAZI`, `KEYMAP`, `LAYOUT`, `THEME`                     |
| `yazi-core`      | **Central state machine**                    | `Core`, `Mgr`, `Tab`, `Folder`, `Tasks`, etc.           |
| `yazi-dds`       | **Inter-process pub/sub**                    | `Pubsub`, `Client`, `Server`, `Pump`, `Ember`           |
| `yazi-emulator`  | Terminal emulator detection                  | Detects kitty, alacritty, etc.                          |
| `yazi-ffi`       | Foreign function interface helpers           | —                                                       |
| `yazi-fm`        | **Main application binary**                  | `App`, `Dispatcher`, `Executor`, `Router`               |
| `yazi-fs`        | File system abstractions                     | `File`, `Cha`, `Url`, `UrlBuf`                          |
| `yazi-macro`     | **Declarative macros**                       | `act!`, `emit!`, `relay!`, `render!`, `succ!`           |
| `yazi-parser`    | Command/form parsing                         | `Spark`, `SparkKind`, `VoidForm`, `ResumeForm`          |
| `yazi-plugin`    | **Lua plugin system**                        | `LUA`, `standard_lua()`, `elements`, `utils`            |
| `yazi-proxy`     | **Type-safe event proxies**                  | `AppProxy`, `MgrProxy`, `TasksProxy`, etc.              |
| `yazi-runner`    | Preview/fetch/preload runners                | `Runner`, `PeekJob`, `Fetcher`, `Preloader`             |
| `yazi-scheduler` | **Background task scheduler**                | `Scheduler`, `Worker`, `Task`, `Ongoing`                |
| `yazi-shared`    | **Shared primitives**                        | `Event`, `Action`, `Layer`, `Id`, `Data`, `RoCell`      |
| `yazi-shim`      | Compatibility shims                          | `RoCell` (runtime-initialized static cell)              |
| `yazi-sftp`      | SFTP virtual file system                     | —                                                       |
| `yazi-term`      | Terminal backend wrapper                     | `Term`, `start()`, `stop()`                             |
| `yazi-tty`       | Low-level TTY I/O                            | —                                                       |
| `yazi-vfs`       | Virtual file system layer                    | VFS over local + SFTP                                   |
| `yazi-watcher`   | File system change watching                  | `Watcher` (inotify/fsevents/kqueue)                     |
| `yazi-widgets`   | Reusable TUI widgets                         | `Input`, `List`, `Paragraph`, etc.                      |

---

## Key Architectural Patterns

### 1. Single-Threaded UI State + Multi-Threaded Workers

The `Core` is never locked — it's exclusively borrowed by the main async loop. Background workers communicate with the UI **only through the event bus** (never by direct mutation). This eliminates data races and deadlocks entirely.

### 2. Message-Passing Everywhere

- UI events: `tokio::sync::mpsc::unbounded_channel`
- Scheduler queues: `async_priority_channel::unbounded` (priority-aware)
- DDS inter-process: Unix domain sockets / TCP
- Task progress: `mpsc::UnboundedSender<String>` for live logs

### 3. Macro-Driven Boilerplate Reduction

`yazi-macro` provides domain-specific macros that make the actor system ergonomic:

| Macro     | Purpose                                                 |
| --------- | ------------------------------------------------------- |
| `act!`    | Dispatch to an actor with automatic parsing + preflight |
| `emit!`   | Send an event into the global bus                       |
| `relay!`  | Construct an `Action` targeting a specific actor        |
| `render!` | Flag the UI for re-rendering                            |
| `succ!`   | Return `Ok(Data::Nil)` from an actor                    |

### 4. Preflight Hooks (Plugin Interception)

Every actor action can be intercepted by Lua plugins **before** execution:

```rust
fn act(cx: &mut Ctx, opt: Self::Form) -> Result<Data> {
    // 1. Plugin preflight hook can transform/cancel
    if let Some(hook) = <Actor>::hook(cx, &opt) {
        let spark = Preflight::act(cx, (hook, spark))?;
        // If plugin returns Nil, action is cancelled
    }
    // 2. Run the actual actor
    <Actor>::act(cx, opt)
}
```

This is how Yazi achieves **extensibility without modifying core code**.

### 5. Render Throttling

Rendering is controlled by an `AtomicU8`:

- `0` — no render needed
- `1` — full render needed
- `2` — partial render needed

After state changes, actors call `render!()` (or `render_partial!()`). The main loop batches renders to ~100fps max, preventing terminal flooding during rapid updates.

### 6. Cancellation Tokens

Every background task gets a `CompletionToken`. Workers use `select!` to race between:

- The actual work completing
- The token being cancelled

This makes operations like "cancel a file copy in progress" safe and responsive.

### 7. `RoCell` — Runtime-Initialized Statics

`yazi-shim::cell::RoCell<T>` is a `OnceCell`-like abstraction that allows:

- Module-level statics that are initialized at startup
- No `lazy_static!` or `once_cell` dependency churn
- Used for: `TX`/`RX` event channels, `QUEUE_TX`/`QUEUE_RX` in DDS, `LAYOUT`, `THEME`, etc.

---

## Data Flow: Opening a File

To illustrate how all pieces fit together, here's the flow when a user presses `Enter` on a file:

```
[Terminal] ─Key(Enter)──► [Event Bus TX]
                              │
                              ▼
[App::serve] ◄──recv_many()──┘
    │
    ▼
Dispatcher::dispatch(Event::Key)
    │
    ▼
Router::route(Key::Enter)
    │
    ▼
KEYMAP lookup → "open" chord → emit!(Seq([Action("mgr:open")]))
    │
    ▼
Dispatcher::dispatch(Event::Call(Action("mgr:open")))
    │
    ▼
Executor::execute(action) → Layer::Mgr → mgr(action)
    │
    ▼
act!(mgr:open, cx, action)
    │
    ▼
Open::act(cx, OpenForm { ... })
    │
    ├─► Preflight hook (plugins can intercept)
    │
    ▼
Determine opener from MIME type
    │
    ▼
Scheduler::process.block(ProcessIn::Block { ... })
    │
    ▼
Process worker spawns external program (e.g., `nvim file.txt`)
    │
    ▼
Terminal suspends, user edits file, quits editor
    │
    ▼
Process worker completes → TaskOp → Op worker
    │
    ▼
Ongoing::fulfill(task_id) → AppProxy::update_progress()
    │
    ▼
emit!(Call(app:update_progress)) → UI refreshes
```

---

## Conclusion

Yazi's architecture is a masterclass in **Rust async design** for interactive terminal applications:

- **Actor model** provides clean separation of concerns and testable action handlers
- **Single mutable Core state** eliminates sync complexity
- **Global event bus** enables decoupled, fire-and-forget communication
- **Priority worker pools** keep the UI responsive while doing heavy I/O
- **Plugin preflight hooks** offer deep extensibility without architectural compromise
- **28 focused crates** enforce module boundaries and enable independent testing

The result is a file manager that feels instantaneous despite doing substantial background work — copy thousands of files, preview images, stream directory listings, and run plugins, all without blocking the user interface.
