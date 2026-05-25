# Yazi's Centralized State Model: No Mutex Required

> **How Yazi manages all application state with exclusive `&mut` references, eliminating synchronization primitives entirely from the UI code path.**

---

## Table of Contents

1. [The Problem Yazi Solved](#the-problem-yazi-solved)
2. [The Single-Threaded UI Loop](#the-single-threaded-ui-loop)
3. [Core: The Single Source of Truth](#core-the-single-source-of-truth)
4. [Ctx<'a>: The Borrowed Gateway](#ctxa-the-borrowed-gateway)
5. [How Background Workers Talk to Core (Without Touching It)](#how-background-workers-talk-to-core-without-touching-it)
6. [Where ARE Locks Used? (And Why They Don't Matter for UI)](#where-are-locks-used-and-why-they-dont-matter-for-ui)
7. [RoCell & SyncCell: Lock-Free Global Statics](#rocell--synccell-lock-free-global-statics)
8. [The Data Flow: Exclusive Access in Practice](#the-data-flow-exclusive-access-in-practice)
9. [Why This Matters: Performance & Correctness](#why-this-matters-performance--correctness)
10. [Trade-offs & Limitations](#trade-offs--limitations)
11. [Lessons for Your Own Code](#lessons-for-your-own-code)

---

## The Problem Yazi Solved

Most async terminal applications face a fundamental tension:

```
┌─────────────────────────────────────────────────────────────┐
│  UI Thread              │  Background Workers               │
│  (main thread)          │  (file I/O, previews, plugins)    │
│                         │                                   │
│  wants &mut State       │  wants &mut State                 │
│         ↑               │         ↑                         │
│         └───────────────┴─────────┘                         │
│                  ↑ CANNOT COEXIST ↑                         │
└─────────────────────────────────────────────────────────────┘
```

The standard Rust solution is `Arc<Mutex<State>>` or `Arc<RwLock<State>>`. Every thread gets a clone of the `Arc`, and `Mutex` serializes access. This works, but it comes with costs:

- **Lock contention** — threads spin or sleep waiting for the lock
- **Cache coherency traffic** — `Mutex` uses atomic operations that flush CPU caches
- **Deadlock risk** — forgetting lock ordering rules can freeze the app
- **Poisoning** — panics while holding a lock corrupt the guard
- **Complex lifetimes** — `MutexGuard` ties your code to specific scopes

**Yazi solved this by not sharing mutable state at all.**

---

## The Single-Threaded UI Loop

Yazi's entire UI runs in a **single async task** on a single thread. The `App` struct owns everything:

```rust
// yazi-fm/src/app/app.rs
pub(crate) struct App {
    pub(crate) core: Core,              // ← ALL application state
    pub(crate) term: Option<Raterm>,    // ← Terminal handle

    need_render: u8,
    last_render: Instant,
    next_render: Option<Duration>,
}
```

The main loop holds `&mut self` for the entire lifetime of the application:

```rust
impl App {
    pub(crate) async fn serve() -> Result<()> {
        let term = Raterm::start()?;
        Signals::start()?;

        let mut app = Self::make(term)?;
        app.bootstrap()?;

        let mut rx = Event::take();
        loop {
            if let Some(t) = app.next_render.take() {
                select! {
                    _ = sleep(t) => { app.render(app.need_render == 2)?; }
                    r = app.drain(&mut rx) => if !r? { break; }
                }
            } else if !app.drain(&mut rx).await? {
                break;
            }
        }
        Ok(())
    }

    async fn drain(&mut self, rx: &mut mpsc::UnboundedReceiver<Event>) -> Result<bool> {
        let Some(event) = rx.recv().await else { return Ok(false); };
        self.dispatch(event)?;
        while let Ok(e) = rx.try_recv() {
            self.dispatch(e)?;
        }
        Ok(true)
    }

    fn dispatch(&mut self, event: Event) -> Result<()> {
        Dispatcher::new(self).dispatch(event);
        // ... render logic
    }
}
```

**Key insight:** `app` is a **local variable** on the stack. `&mut app` means **exclusive mutable access** to the entire application state. The Rust borrow checker guarantees that nothing else can touch `app.core` while this function is running.

---

## Core: The Single Source of Truth

All mutable UI state lives in one struct:

```rust
// yazi-core/src/core.rs
pub struct Core {
    pub mgr:     Mgr,        // File manager (tabs, folders, selection, yanked)
    pub tasks:   Tasks,      // Background task queue UI state
    pub pick:    Pick,       // Picker modal state
    pub input:   Input,      // Text input modal state
    pub confirm: Confirm,    // Confirmation dialog state
    pub help:    Help,       // Help overlay state
    pub cmp:     Cmp,        // Completion popup state
    pub which:   Which,      // Key chord hint state
    pub notify:  Notify,     // Notification toasts
}
```

Look at these fields carefully. **None of them contain a `Mutex`, `RwLock`, or `Atomic`.** They are plain structs with plain fields:

```rust
// yazi-core/src/mgr/mgr.rs
pub struct Mgr {
    pub tabs:   Tabs,        // Vec<Tab> + cursor index
    pub yanked: Yanked,      // IndexSet of URLs
    pub batcher: Batcher,    // ← The ONE exception (explained below)
    pub watcher: Watcher,    // File system watcher handle
    pub mimetype: Mimetype,  // MIME type cache
}

// yazi-core/src/tab/folder.rs
pub struct Folder {
    pub url:   UrlBuf,
    pub cha:   Cha,
    pub files: Files,        // Vec<File> — plain Vec
    pub stage: FolderStage,
    pub offset: usize,       // Scroll offset
    pub cursor: usize,       // Selected item
    pub page:  usize,
    pub trace: Option<PathBufDyn>,
}
```

Everything is **stack-friendly** and **cache-coherent**. When an actor modifies `folder.cursor`, it's just a CPU register write. No atomic operations. No cache flushes. No lock acquisition.

---

## Ctx<'a>: The Borrowed Gateway

Actors don't receive `&mut Core` directly. They receive `Ctx<'a>`, which **borrows** `Core` through a mutable reference:

```rust
// yazi-actor/src/context.rs
pub struct Ctx<'a> {
    pub core:   &'a mut Core,       // ← Exclusive borrow of ALL state
    pub term:   &'a mut Option<Raterm>,
    pub tab:    usize,              // Active tab index
    pub level:  usize,              // Call nesting depth (debugging)
    pub source: Source,             // Who triggered this action
}

// Ctx derefs to Core, so actors can use it like &mut Core
impl Deref for Ctx<'_> {
    type Target = Core;
    fn deref(&self) -> &Self::Target { self.core }
}

impl DerefMut for Ctx<'_> {
    fn deref_mut(&mut self) -> &mut Self::Target { self.core }
}
```

When the Executor creates a `Ctx`:

```rust
// yazi-fm/src/executor.rs
fn mgr(&mut self, action: ActionCow) -> Result<Data> {
    let cx = &mut Ctx::new(&action, &mut self.app.core, &mut self.app.term)?;
    // ... actor gets &mut Ctx, which means &mut Core
}
```

The `Ctx::new()` takes `&mut Core` and `&mut Option<Term>` from `&mut App`. Because these are **exclusive borrows**, the Rust compiler enforces:

1. **Only one `Ctx` exists at a time** — because `Ctx` holds `&mut Core`
2. **No other code can touch `Core` while an actor runs** — the borrow is exclusive
3. **Actors can mutate anything freely** — no locks needed because the borrow checker already proved safety

### Ctx Convenience Methods

```rust
impl<'a> Ctx<'a> {
    #[inline] pub fn tabs(&self) -> &Tabs { &self.mgr.tabs }
    #[inline] pub fn tabs_mut(&mut self) -> &mut Tabs { &mut self.mgr.tabs }
    #[inline] pub fn tab(&self) -> &Tab { &self.tabs()[self.tab] }
    #[inline] pub fn tab_mut(&mut self) -> &mut Tab { &mut self.core.mgr.tabs[self.tab] }
    #[inline] pub fn cwd(&self) -> &UrlBuf { self.tab().cwd() }
    #[inline] pub fn current(&self) -> &Folder { &self.tab().current }
    #[inline] pub fn current_mut(&mut self) -> &mut Folder { &mut self.tab_mut().current }
    #[inline] pub fn hovered(&self) -> Option<&File> { self.tab().hovered() }
}
```

These are **zero-cost abstractions**. They compile down to simple pointer offsets. No virtual calls, no locks, no runtime overhead.

---

## How Background Workers Talk to Core (Without Touching It)

This is the critical design decision. Background workers **never** touch `Core`. Instead, they emit events that the UI loop processes:

### The Event Bus

```rust
// yazi-shared/src/event/event.rs
static TX: RoCell<mpsc::UnboundedSender<Event>> = RoCell::new();
static RX: RoCell<mpsc::UnboundedReceiver<Event>> = RoCell::new();

pub enum Event {
    Call(ActionCow),      // "Execute this actor"
    Seq(Vec<ActionCow>),  // "Execute these actors"
    Render(bool),         // "Redraw the UI"
    Key(KeyEvent),        // "User pressed a key"
    // ...
}
```

### Proxy Pattern

Background workers use **proxies** to send events into the bus:

```rust
// yazi-core/src/proxy.rs
pub struct AppProxy;

impl AppProxy {
    pub fn update_progress(summary: TaskSummary) {
        // Background worker calls this...
        emit!(Call(relay!(app:update_progress).with_any("summary", summary)));
        // ...which sends an Event::Call into the mpsc channel
    }
}

pub struct MgrProxy;

impl MgrProxy {
    pub fn update_peeked(lock: PreviewLock) {
        emit!(Call(relay!(mgr:update_peeked).with_any("lock", lock)));
    }
}
```

The worker thread does **three things**:
1. Does its work (file copy, preview render, etc.)
2. Packages the result into a message (not a mutation!)
3. Sends the message into the event bus

Then the **UI thread** receives the message and mutates `Core` with `&mut`:

```
Background Worker                    UI Thread (single &mut Core)
─────────────────────────────────────────────────────────────────
File copy completes
    │
    ▼
AppProxy::update_progress(summary)
    │
    ▼
emit!(Call(action)) → mpsc channel
                         │
                         ▼
                    rx.recv().await
                         │
                         ▼
                    Dispatcher::dispatch()
                         │
                         ▼
                    Executor::execute()
                         │
                         ▼
                    act!(app:update_progress, cx, action)
                         │
                         ▼
                    cx.tasks.summary = summary;  ← &mut Core!
```

**No locks. No shared state. Just message passing.**

### The Tasks Bridge

`Core` does contain one `Arc`:

```rust
// yazi-core/src/tasks/tasks.rs
pub struct Tasks {
    pub scheduler: Arc<Scheduler>,   // ← ONLY Arc in Core state
    handle:        JoinHandle<()>,

    pub visible: bool,
    pub cursor:  usize,
    pub snaps:   Vec<TaskSnap>,
    pub summary: TaskSummary,
}
```

`Arc<Scheduler>` is a **bridge to the background world**. It lets the UI submit work:

```rust
// Actor submits a file copy
fn act(cx: &mut Ctx, opt: PasteForm) -> Result<Data> {
    for u in opt.sources {
        cx.tasks.scheduler.file(FileIn::Copy { src: u, dest: opt.target });
    }
    succ!();
}
```

But notice: `Arc<Scheduler>` is **read-only** from the UI's perspective. The UI calls methods on it to *submit* work. It never reads scheduler-internal state directly. When the scheduler finishes, it emits an event back to the UI, which updates `Tasks.visible`, `Tasks.snaps`, etc. — all plain fields, no locks.

---

## Where ARE Locks Used? (And Why They Don't Matter for UI)

Yazi does use locks — but **only in places that never touch Core**.

### 1. Scheduler Worker Pools (`yazi-scheduler`)

```rust
// yazi-scheduler/src/worker.rs
pub struct Worker {
    pub(super) file:    Arc<File>,
    pub(super) plugin:  Arc<Plugin>,
    pub fetch:          Arc<Fetch>,
    pub preload:        Arc<Preload>,
    pub size:           Arc<Size>,
    pub(super) process: Arc<Process>,
    pub(super) hook:    Arc<Hook>,

    pub ops:     TaskOps,
    pub ongoing: Arc<Mutex<Ongoing>>,   // ← parking_lot::Mutex
}
```

`ongoing: Arc<Mutex<Ongoing>>` tracks in-flight tasks. This is **inside the scheduler**, not in `Core`. The UI sees the scheduler through `Arc<Scheduler>`, which exposes only safe methods. The mutex protects internal scheduler bookkeeping, not UI state.

### 2. Batcher (`yazi-core/src/mgr/batcher.rs`)

```rust
#[derive(Clone, Default)]
pub struct Batcher {
    pending: Arc<Mutex<HashMap<PathBuf, Option<bool>>>>,
}
```

`Batcher` lives inside `Mgr`, which lives inside `Core`. This is the **only** mutex in the entire `Core` hierarchy. It's used for a very specific purpose: when pasting multiple files to the same destination, the batcher coordinates whether to overwrite/skip/append. The mutex protects a tiny decision map — not the folder state.

### 3. DDS Peer Tracking (`yazi-dds/src/client.rs`)

```rust
pub static PEERS: RoCell<RwLock<HashMap<Id, Peer>>> = RoCell::new();
```

`RwLock` for tracking other Yazi instances. This is in DDS, completely separate from `Core`.

### 4. Watcher File Tracking (`yazi-watcher/src/lib.rs`)

```rust
pub static WATCHED: RoCell<RwLock<Watched>> = RoCell::new();
```

Filesystem watcher state. Again, not in `Core`.

### 5. The Event Bus Itself

```rust
static TX: RoCell<mpsc::UnboundedSender<Event>> = RoCell::new();
```

`tokio::sync::mpsc` uses locks internally. But this is a **channel**, not shared state. Producers send messages, consumers receive them. The lock is inside `tokio`'s implementation and is optimized for this specific pattern.

---

## RoCell & SyncCell: Lock-Free Global Statics

Yazi replaces `lazy_static!` and `once_cell` with custom primitives that have zero runtime cost after initialization.

### RoCell<T> — Write-Once, Read-Forever

```rust
// yazi-shim/src/cell/ro_cell.rs
pub struct RoCell<T> {
    inner:       UnsafeCell<MaybeUninit<T>>,
    #[cfg(debug_assertions)]
    initialized: UnsafeCell<bool>,
}

unsafe impl<T> Sync for RoCell<T> {}

impl<T> RoCell<T> {
    pub const fn new() -> Self { ... }

    pub fn init(&self, value: T) {
        unsafe {
            #[cfg(debug_assertions)]
            assert!(!self.initialized.get().replace(true));
            *self.inner.get() = MaybeUninit::new(value);
        }
    }
}

impl<T> Deref for RoCell<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        unsafe { (*self.inner.get()).assume_init_ref() }
    }
}
```

**Properties:**
- `Sync` — can be stored in a `static`
- `init()` — called exactly once at startup
- After `init()`, access is just a raw pointer dereference — **no atomics, no locks**
- Used for: `TX`/`RX` event channels, `ID`, `LAYOUT`, `THEME`, `BOOT`, `KEYMAP`

### SyncCell<T> — Copy-Only Interior Mutability

```rust
// yazi-shim/src/cell/sync_cell.rs
pub struct SyncCell<T: ?Sized>(Cell<T>);
unsafe impl<T: ?Sized + Sync> Sync for SyncCell<T> {}

impl<T> Deref for SyncCell<T> {
    type Target = Cell<T>;
    fn deref(&self) -> &Self::Target { &self.0 }
}
```

`SyncCell` is just `Cell` that implements `Sync`. Because `Cell` only works with `Copy` types, this is safe. Used for:
- `LAYOUT: SyncCell<Layout>` — window dimensions that change on resize
- Atomic-like counters that need `get()`/`set()` across threads

**Neither `RoCell` nor `SyncCell` are used for Core state.** They're for global configuration and channel handles.

---

## The Data Flow: Exclusive Access in Practice

Let's trace what happens when you copy a file, watching every reference:

```
1. KEY PRESS
══════════════
User presses 'y' (yank)
    │
    ▼
Crossterm → Event::Key(Key('y')) → mpsc channel


2. UI LOOP RECEIVES EVENT
══════════════
App::drain() rx.recv().await → Event::Key('y')
    │
    ▼
App::dispatch(event)
    │  holds &mut self
    │  which is &mut App { core: Core, term: Option<Raterm> }
    ▼
Dispatcher::new(self) → &mut App
    │
    ▼
Dispatcher::dispatch_key(key)
    │
    ▼
Router::route(Key('y'))
    │  borrows self.app.core (as &Core, immutable)
    │  reads core.layer() → Layer::Mgr
    ▼
Router::matches(Layer::Mgr, Key('y'))
    │  finds Chord "y" in KEYMAP.mgr
    │  on.len() == 1 → single key
    ▼
emit!(Seq([Action("yank")]))
    │  sends Event::Seq([Action]) into mpsc


3. SAME UI LOOP PROCESSES THE ACTION
══════════════
App::drain() rx.recv().await → Event::Seq([Action("yank")])
    │  still holding &mut app
    ▼
Dispatcher::dispatch_call(Action("yank"))
    │
    ▼
Executor::execute(action)
    │  action.layer == Layer::Mgr
    ▼
fn mgr(&mut self, action: ActionCow) {
    // ...
    "yank" → act!(mgr:yank, cx, action)
    │
    ▼
    Ctx::new(&action, &mut self.app.core, &mut self.app.term)
        │
        ▼
        Ctx { core: &mut Core, term: &mut Option<Raterm>, tab: 0, ... }
        │
        ▼
    Yank::act(cx, YankForm { ... })
        │  cx is &mut Ctx → via DerefMut, &mut Core
        ▼
        cx.yanked = Yanked::new(true, selected_urls);
        // ↑ Direct field mutation! No lock. No Arc. Just a write.
        render!();
        succ!();
}


4. RENDER
══════════════
App::dispatch() sees NEED_RENDER == 1
    │  still holding &mut app
    ▼
app.render(false)
    │  reads core.layer(), core.mgr, core.yanked, etc.
    │  all immutable reads through &App (or &Core)
    ▼
Root::render(&core, area, buf)
    │  draws file list with yank highlights
    ▼
Terminal.flush()


5. BACKGROUND WORK STARTS (Later)
══════════════
User presses 'p' (paste)
    │
    ▼
Paste::act(cx, PasteForm { target, sources })
    │
    ▼
    for u in sources {
        cx.tasks.scheduler.file(FileIn::Copy { src: u, dest: target });
        // ↑ submits work to background scheduler
        //   scheduler.ongoing.lock().add(task) — THIS is where the lock is
        //   but it's inside the scheduler, not in Core
    }


6. BACKGROUND WORKER PROCESSES FILE
══════════════
tokio::spawn(async {
    loop {
        rx.recv().await → FileIn::Copy { src, dest }
            │
            ▼
        file.copy(src, dest).await
            │
            ▼
        // Copy completes
        ops.out(id, TaskOut::Done)
            │
            ▼
        // Op worker processes the completion
        ongoing.lock().get_mut(id).prog = TaskProg::Done;
            │
            ▼
        // Hook runs
        hook.submit(hook, LOW)
            │
            ▼
        // Eventually, progress poller notices change
        AppProxy::update_progress(new_summary);
            │  emit!(Call(app:update_progress))
            ▼
        // Back to step 1 — UI loop receives the event
    }
})
```

At **no point** does the background worker touch `Core`. The entire interaction is:

```
Background Worker          Event Bus              UI Thread
      │                       │                      │
      │── File copy done ────►│                      │
      │                       │── Event::Call(...) ─►│
      │                       │                      │── act!(app:update_progress)
      │                       │                      │   &mut Core mutation
```

---

## Why This Matters: Performance & Correctness

### Performance

| Metric | Mutex-Based App | Yazi's &mut Model |
|--------|----------------|-------------------|
| **State mutation** | Atomic CAS + cache flush | Plain register write |
| **Read access** | Lock acquisition + refcount | Plain pointer deref |
| **Cache locality** | False sharing between cores | Single-core, cache-hot |
| **Memory layout** | Heap-allocated `Arc<Mutex<T>>` | Contiguous `Core` struct |
| **No-contention overhead** | ~50-100ns per lock | 0ns |

In practice, Yazi's UI operations are **instantaneous** because there's zero synchronization overhead. When you press a key, the response is bounded only by:
1. Crossterm read latency
2. The actor's own logic (which is usually trivial)
3. Terminal flush bandwidth

### Correctness

The Rust borrow checker **mechanically proves** these safety properties:

1. **No data races** — `&mut Core` is exclusive; no other reference can exist
2. **No use-after-free** — `Core` is owned by `App`, which lives for the entire loop
3. **No double-mutation** — You cannot call two actors simultaneously because `&mut` cannot be aliased
4. **No deadlocks** — There are no locks to deadlock on in the UI path
5. **Deterministic execution order** — Events are processed FIFO; actors run to completion before the next event

The last point is crucial. Because actors are **synchronous** and **run to completion**, you never have:
- Half-applied state changes
- Interleaved mutations from concurrent events
- Re-entrancy bugs

---

## Trade-offs & Limitations

### 1. Single-Threaded UI

Yazi cannot parallelize UI work. If an actor were CPU-intensive (e.g., computing SHA-256 of a 10GB file), it would freeze the UI. Yazi solves this by:
- **Never doing CPU-intensive work in actors** — actors only mutate state and submit background jobs
- **All heavy work goes to the scheduler** — file I/O, hashing, previews, plugin execution

### 2. Event Latency

Background workers communicate via the event bus, which has inherent latency:
```
Worker completes → send event → UI receives → process → render
    │                  │              │            │
    └──────────────────┴──────────────┴────────────┘
                   ~1-10ms typical
```

For progress updates, Yazi polls every 500ms (`Tasks::serve()`), so the UI shows slightly stale progress. This is fine for a file manager.

### 3. No Shared Read Access During Mutation

If you wanted to render the UI while a background thread reads `Core`, you can't — because `&mut Core` is exclusive. Yazi solves this by:
- **Rendering only happens between events** — after an actor completes, before the next event
- **Background workers don't read Core** — they send data to the UI, which stores it in `Core`

### 4. Actor Execution Must Be Fast

Because the event loop is blocked while an actor runs, actors must complete quickly:

```rust
// GOOD: Fast actor
fn act(cx: &mut Ctx, opt: CdForm) -> Result<Data> {
    cx.mgr.tabs.active_mut().cd(&opt.target)?;
    render!();
    succ!();
}

// BAD: Slow actor (would freeze UI)
fn act(cx: &mut Ctx, opt: HashForm) -> Result<Data> {
    let hash = sha256_file(&opt.path)?;  // ← DON'T DO THIS
    cx.files[0].hash = hash;
    succ!();
}
```

The slow version should be: actor submits a scheduler job, worker computes hash, worker emits event with result, actor in UI thread updates state.

---

## Lessons for Your Own Code

### 1. Design for Single Ownership

If your application has a clear "main thread" that owns state, **don't share it**. Pass `&mut` down the call stack.

```rust
// Instead of this:
struct App {
    state: Arc<Mutex<State>>,
}

// Do this:
struct App {
    state: State,  // owned
}

fn handle_event(app: &mut App, event: Event) {
    match event {
        Event::Click => click_handler(&mut app.state),
        Event::Key(k) => key_handler(&mut app.state, k),
    }
}
```

### 2. Use Message Passing for Cross-Thread Communication

When you need async workers, don't give them `Arc<Mutex<T>>`. Give them a channel.

```rust
// Worker thread
async fn worker(mut rx: Receiver<Work>, tx: Sender<Result>) {
    while let Some(job) = rx.recv().await {
        let result = do_work(job).await;
        tx.send(result).ok();  // Fire and forget
    }
}

// Main thread
fn on_result(app: &mut App, result: Result) {
    app.state.apply(result);  // &mut State
}
```

### 3. Use Proxy Objects to Decouple

Yazi's `AppProxy` / `MgrProxy` pattern is powerful. Any code anywhere can call `AppProxy::update_progress()` without knowing about `App` or `Core`. The proxy just emits an event.

```rust
// Anywhere in the codebase
pub struct MyProxy;
impl MyProxy {
    pub fn thing_happened(data: Data) {
        emit!(Call(relay!(app:handle_thing).with_any("data", data)));
    }
}
```

### 4. Split "UI State" from "Background State"

Yazi's `Core` contains UI-facing state. The scheduler contains background-facing state. They're connected by events, not shared memory.

```rust
// UI state — no locks
struct Core {
    files: Vec<File>,
    cursor: usize,
    progress: f32,
}

// Background state — can use locks internally
struct Scheduler {
    ongoing: Mutex<Vec<Task>>,
}

// Bridge
struct App {
    core: Core,
    scheduler: Arc<Scheduler>,
}
```

### 5. Leverage Rust's Borrow Checker

The borrow checker is not an obstacle — it's a **design tool**. If you find yourself fighting `Arc<Mutex<T>>`, ask: "Can I restructure so one component owns this?"

Yazi proves that even complex interactive applications (file manager with plugins, previews, background jobs, modal dialogs, multi-key chords) can be built with zero mutexes in the hot path.

---

## Summary

| Aspect | How Yazi Does It |
|--------|-----------------|
| **State ownership** | `App` owns `Core` as a plain field |
| **Exclusive access** | `&mut App` → `&mut Core` via `Ctx<'a>` |
| **Actor execution** | Synchronous, single-threaded, `&mut Ctx` |
| **Background workers** | `tokio::spawn` with message channels |
| **Worker → UI communication** | Events via `tokio::sync::mpsc` + proxy objects |
| **Global statics** | `RoCell` (write-once) / `SyncCell` (Copy-only) |
| **Locks in UI path** | **Zero** |
| **Locks in background** | `parking_lot::Mutex` for scheduler internals only |
| **Cache efficiency** | Maximum — contiguous Core struct, no atomics |
| **Correctness guarantee** | Rust borrow checker + FIFO event processing |

Yazi's architecture demonstrates a powerful principle: **the best synchronization is no synchronization.** By designing the system so that mutable state has a single owner and background workers communicate through messages, you eliminate entire categories of bugs while making your code faster and simpler.
