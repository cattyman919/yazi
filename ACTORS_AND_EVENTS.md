# Actor-Based Architecture & Event-Driven Systems

> **A practical guide using Yazi as the case study**  
> If you've ever wondered *"when should I use actors?"* or *"what's the difference between actors and events?"*, this is for you.

---

## Table of Contents

1. [The Big Picture](#the-big-picture)
2. [Part I: The Actor Model](#part-i-the-actor-model)
   - [What Is an Actor?](#what-is-an-actor)
   - [The Three Rules](#the-three-rules)
   - [Actors in Yazi](#actors-in-yazi)
   - [Message Passing vs. Direct Calls](#message-passing-vs-direct-calls)
   - [Why Actors Scale](#why-actors-scale)
3. [Part II: Event-Driven Architecture](#part-ii-event-driven-architecture)
   - [What Is Event-Driven?](#what-is-event-driven)
   - [The Event Bus Pattern](#the-event-bus-pattern)
   - [Events in Yazi](#events-in-yazi)
   - [Publish/Subscribe vs. Request/Response](#publishsubscribe-vs-requestresponse)
4. [Part III: How They Work Together](#part-iii-how-they-work-together)
   - [The Composition Pattern](#the-composition-pattern)
   - [From Keypress to File Open: The Full Journey](#from-keypress-to-file-open-the-full-journey)
5. [Part IV: Design Decisions & Trade-offs](#part-iv-design-decisions--trade-offs)
   - [Actor Model: Pros & Cons](#actor-model-pros--cons)
   - [Event-Driven: Pros & Cons](#event-driven-pros--cons)
   - [When to Use What](#when-to-use-what)
   - [Common Anti-Patterns](#common-anti-patterns)
6. [Part V: Building Your Own](#part-v-building-your-own)
   - [A Minimal Actor System in Rust](#a-minimal-actor-system-in-rust)
   - [A Minimal Event Bus in Rust](#a-minimal-event-bus-in-rust)
   - [Putting Them Together](#putting-them-together)

---

## The Big Picture

Most software starts simple: function A calls function B, which modifies some state. This is **direct call architecture**. It works beautifully until:

- You need to do work in the background without freezing the UI
- Multiple parts of the system need to react to the same change
- You want plugins to intercept or extend core behavior
- You need to distribute work across threads or processes

**Actor-based architecture** and **event-driven architecture** are two answers to these problems. They are often used together (as in Yazi) because they solve *different* but *complementary* problems.

| Concern | Actor Model Solves... | Event-Driven Solves... |
|---------|----------------------|----------------------|
| **State** | Encapsulates state behind message handlers | Decouples state changes from reactions |
| **Concurrency** | One actor = one thread of logic (no locks needed) | Multiple consumers process the same event |
| **Extensibility** | New actors can be added without changing old ones | New subscribers react without changing publishers |
| **Backpressure** | Mailbox limits prevent overwhelming slow actors | Queue depth controls event flow |
| **Testing** | Actors are isolated units | Events are replayable logs |

Yazi uses **both**: actors handle *commands* ("do this now"), while events handle *notifications* ("this happened").

---

## Part I: The Actor Model

### What Is an Actor?

The Actor Model, invented by Carl Hewitt in 1973, is a mathematical model of concurrent computation. In its pure form, an actor is the **fundamental unit of computation** with three capabilities:

1. **Receive messages** into a mailbox
2. **Process messages** one at a time (sequentially)
3. **Send messages** to other actors and **create** new actors

That's it. There are no shared variables, no mutexes, no `Arc<Mutex<T>>`. Just actors talking to each other by sending messages.

### The Three Rules

#### Rule 1: Actors Don't Share State

```rust
// WRONG: Shared mutable state (the enemy of actors)
static COUNTER: AtomicUsize = AtomicUsize::new(0);

// RIGHT: State lives inside the actor, accessed only by its messages
struct CounterActor {
    value: usize,
}
```

#### Rule 2: Messages Are Async and Unordered (Between Actors)

When Actor A sends to Actor B, the message goes into B's mailbox. A does not wait. B processes it when B is ready. If A sends two messages, B might process them in either order — or concurrently with messages from Actor C.

```rust
// A fires and forgets
actor_b.send(Message::Increment).await?;  // A doesn't block
actor_b.send(Message::GetValue).await?;   // Might arrive before Increment!
```

#### Rule 3: One Actor Processes One Message at a Time

This is the golden rule that makes actors safe. Even if an actor runs in a multi-threaded pool, **its mailbox guarantees sequential processing**. You never need locks inside an actor's message handler.

```rust
// Inside Actor B's message loop — always single-threaded per actor
while let Some(msg) = mailbox.recv().await {
    match msg {
        Message::Increment => self.value += 1,  // No lock needed!
        Message::GetValue => reply.send(self.value),
    }
}
```

### Actors in Yazi

Yazi doesn't implement a *pure* actor system (there's no `ActorRef` or supervision tree like in Akka). Instead, it uses a **practical actor pattern**: every user action is an actor, but state is centralized in `Core` rather than distributed across many actor instances.

#### The Yazi Actor Trait

```rust
// yazi-actor/src/actor.rs
pub trait Actor {
    type Form;                          // The "message" this actor accepts
    const NAME: &str;

    fn act(cx: &mut Ctx, form: Self::Form) -> Result<Data>;
    fn hook(_cx: &Ctx, _form: &Self::Form) -> Option<SparkKind> { None }
}
```

This is a **synchronous actor**. When you press a key, the system:
1. Creates the message (`Action` → `Form`)
2. Calls `Actor::act(&mut Ctx, form)`
3. The actor mutates state through `&mut Ctx` (which derefs to `&mut Core`)
4. Returns `Data` (or `Nil`)

Here's a concrete example — the `Quit` actor:

```rust
// yazi-actor/src/app/quit.rs
pub struct Quit;

impl Actor for Quit {
    type Form = QuitForm;
    const NAME: &str = "quit";

    fn act(cx: &mut Ctx, opt: Self::Form) -> Result<Data> {
        cx.mgr.quit(&opt)?;           // Mutate manager state
        cx.tasks.shutdown(&opt);      // Signal scheduler to stop
        succ!();                       // Return Ok(Nil)
    }
}
```

And the `Cd` (change directory) actor:

```rust
// yazi-actor/src/mgr/cd.rs
pub struct Cd;

impl Actor for Cd {
    type Form = CdForm;
    const NAME: &str = "cd";

    fn act(cx: &mut Ctx, opt: Self::Form) -> Result<Data> {
        let tab = cx.active_mut();
        if !tab.cd(&opt.target)? {
            return Ok(Data::Nil);     // Already there, nothing to do
        }

        // Trigger side effects
        if opt.interactive {
            cx.mgr.remove_history(&opt.target);  // Clear history for this path
        }
        render!();                        // Flag UI for re-render
        succ!();
    }
}
```

Notice:
- The actor has **no idea** who called it. It receives a `Ctx` and a `Form`.
- It mutates state **directly** (because Yazi uses a centralized state model, not distributed actor state).
- It can trigger side effects (render, clear history) but doesn't call other functions directly — it uses the event system for that.

#### The `act!` Macro: Actor Dispatch

The `act!` macro (`yazi-macro/src/actor.rs`) is what makes the actor pattern ergonomic. It handles the entire lifecycle:

```rust
macro_rules! act {
    // Full form: parse action, run preflight, execute actor
    ($layer:ident : $name:ident, $cx:ident, $action:expr) => {
        <act!($layer:$name) as yazi_actor::Actor>::Form::try_from($action)
            .map_err(anyhow::Error::from)
            .and_then(|opt| act!(@impl $layer:$name, $cx, opt))
    };

    // Pre-flight: let plugins intercept/transform the message
    (@pre $layer:ident : $name:ident, $cx:ident, $opt:ident) => {
        if let Some(hook) = <act!($layer:$name) as yazi_actor::Actor>::hook($cx, &$opt) {
            <act!(core:preflight)>::act($cx, (hook, spark!($layer:$name, $opt)))
                .map(|spark| spark.try_into().unwrap())
        } else {
            Ok($opt)
        }
    };

    // Execution: increment nesting, call act(), decrement nesting
    (@impl $layer:ident : $name:ident, $cx:ident, $opt:ident) => {{
        $cx.level += 1;  // Track call depth (for debugging)
        let result = match act!(@pre $layer:$name, $cx, $opt) {
            Ok(opt) => <act!($layer:$name) as yazi_actor::Actor>::act($cx, opt),
            Err(e) => Err(e),
        };
        $cx.level -= 1;
        result
    }};
}
```

Usage in the executor:

```rust
// yazi-fm/src/executor.rs
fn mgr(&mut self, action: ActionCow) -> Result<Data> {
    let cx = &mut Ctx::new(&action, &mut self.app.core, &mut self.app.term)?;

    if action.name == "cd" {
        return act!(mgr:cd, cx, action);   // Parse → Preflight → Execute
    }
    if action.name == "open" {
        return act!(mgr:open, cx, action);
    }
    // ... dozens more
}
```

### Message Passing vs. Direct Calls

Let's compare three ways to handle a "copy file" operation:

#### Approach 1: Direct Call (Traditional)

```rust
fn on_key_enter(app: &mut App) {
    let file = app.current_file();
    copy_file(file, &app.destination);  // Blocks the UI!
    app.refresh();                       // Only happens after copy finishes
}
```

**Problem:** If copying a 10GB file, your UI is frozen for minutes.

#### Approach 2: Callbacks (Callback Hell)

```rust
fn on_key_enter(app: &mut App) {
    let file = app.current_file();
    app.scheduler.copy(file, || {
        app.refresh();  // Captures &mut App — borrow checker says NO
    });
}
```

**Problem:** Rust's borrow checker makes callbacks across threads nearly impossible. Even in GC languages, this leads to spaghetti code.

#### Approach 3: Actor + Event (Yazi's Way)

```rust
// 1. Key press becomes an Action
emit!(Call(relay!(mgr:copy).with("src", file.url)));

// 2. Executor routes to the Copy actor
act!(mgr:copy, cx, action)?;  // Synchronous: updates Core state

// 3. Actor submits work to scheduler (async background)
cx.scheduler.file(FileIn::Copy { src, dest });

// 4. Worker completes, sends event back
AppProxy::update_progress(summary);  // emit!(Call(app:update_progress))

// 5. UI updates
render!();
```

**Why this works:**
- The actor runs synchronously in the UI thread (fast: just queues work)
- The scheduler runs asynchronously in worker threads (slow: actual file I/O)
- They communicate through messages, not shared mutable references

### Why Actors Scale

Actors scale in three dimensions:

1. **Horizontal (more actors):** Add new actions without touching old ones. Yazi has 80+ actors across 10 modules. Adding "bulk rename" didn't require changing "open" or "cd".

2. **Vertical (nesting):** Actors can call actors. `mgr:paste` calls the scheduler, which eventually calls `app:update_progress`. The `act!` macro tracks nesting depth for debugging.

3. **Temporal (async):** Background workers are also actors in spirit — they receive messages (file operations) and process them sequentially per worker thread.

---

## Part II: Event-Driven Architecture

### What Is Event-Driven?

Event-driven architecture (EDA) is a pattern where **the flow of the program is determined by events** — discrete occurrences that something happened, rather than by a predetermined sequence of steps.

In EDA, you have:
- **Producers** — emit events ("I did a thing")
- **Consumers** — react to events ("When that thing happens, I'll do this")
- **Event Bus** — transports events from producers to consumers

The key insight: **producers don't know about consumers**. When a file is copied, the copier just says "copy done." The UI, the notification system, and the DDS sync system all decide independently whether they care.

### The Event Bus Pattern

An event bus is the postal service of your application:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Producer   │────►│  Event Bus  │◄────│  Consumer   │
│  (anywhere) │     │  (channel)  │     │  (anywhere) │
└─────────────┘     └──────┬──────┘     └─────────────┘
                           │
                     ┌─────┴─────┐
                     │ Consumer  │
                     │ (another) │
                     └───────────┘
```

The bus doesn't care who produces or who consumes. It just moves messages.

### Events in Yazi

Yazi's event system lives in `yazi-shared/src/event/event.rs`:

```rust
pub enum Event {
    Call(ActionCow),          // "Execute this actor action"
    Seq(Vec<ActionCow>),      // "Execute these actions in order"
    Render(bool),             // "Redraw the UI (partial or full)"
    Key(KeyEvent),            // "User pressed a key"
    Mouse(MouseEvent),        // "User moved/clicked mouse"
    Resize,                   // "Terminal window changed size"
    Focus,                    // "Terminal gained focus"
    Paste(String),            // "User pasted text"
}
```

Notice that **everything is an event**. Key presses, actor invocations, render requests — they all travel through the same channel. This is called **unified event handling**.

#### The Channel

```rust
// yazi-shared/src/event/event.rs
static TX: RoCell<mpsc::UnboundedSender<Event>> = RoCell::new();
static RX: RoCell<mpsc::UnboundedReceiver<Event>> = RoCell::new();

impl Event {
    pub fn init() {
        let (tx, rx) = mpsc::unbounded_channel();
        TX.init(tx);
        RX.init(rx);
    }

    pub fn emit(self) {
        TX.send(self).ok();  // Fire and forget
    }
}
```

`RoCell` is Yazi's runtime-initialized static cell. After `Event::init()` at startup, any code anywhere can call `Event::emit()` to send an event. No need to thread a channel through 20 layers of function arguments.

#### The Render Event

Here's a beautiful example of event-driven decoupling. Any actor can request a render:

```rust
// yazi-macro/src/render.rs
macro_rules! render {
    () => {
        yazi_shared::event::NEED_RENDER
            .store(1, std::sync::atomic::Ordering::Relaxed);
    };
}
```

But notice: the actor doesn't call `app.render()` directly. It sets an atomic flag. The main loop checks this flag and decides *when* to render:

```rust
// yazi-fm/src/app/app.rs
loop {
    select! {
        // Wait up to 10ms for more events (batching)
        _ = sleep(timeout) => {
            if need_render > 0 {
                app.render(need_render == 2)?;  // Actually render
                last_render = Instant::now();
            }
        }
        // Or process events immediately
        n = rx.recv_many(&mut events, 50) => {
            for event in events.drain(..) {
                Dispatcher::new(&mut app).dispatch(event);
            }
        }
    }
}
```

**Decoupling benefits:**
- 50 rapid "copy" operations might all set `NEED_RENDER`, but only one actual terminal redraw happens
- Actors don't know about rendering logic, frame timing, or terminal state
- The render logic doesn't know which actors triggered it

#### Emitting Events from Anywhere

The `emit!` macro makes event production ergonomic:

```rust
// Emit a simple event
emit!(Render(false));  // Request full render

// Emit an actor call (the most common case)
emit!(Call(relay!(mgr:cd).with("target", "/home/user")));

// Emit a sequence of actions
emit!(Seq(vec![
    relay!(mgr:yank),
    relay!(mgr:paste),
]));
```

The `relay!` macro constructs an `Action` targeting a specific actor:

```rust
macro_rules! relay {
    ($layer:ident : $name:ident) => {
        yazi_shared::event::Action::new_relay(
            concat!(stringify!($layer), ":", stringify!($name))
        )
    };
}
```

This creates an action with name `"mgr:cd"`, layer `Layer::Mgr`, and empty arguments.

### Publish/Subscribe vs. Request/Response

EDA has two communication styles:

#### Pub/Sub (One-to-Many)

One event, multiple consumers. In Yazi, this is used for DDS (inter-process pub/sub):

```rust
// yazi-dds/src/pubsub.rs
pub static LOCAL: RoCell<RwLock<HashMap<String, HashMap<String, Function>>>>;

// Plugin A subscribes to "cd" events
Pubsub::sub("my-plugin", "cd", lua_callback);

// When ANY actor changes directory, all "cd" subscribers get called
Pubsub::pub_after_cd(&url);
```

#### Request/Response (One-to-One)

Most UI actions in Yazi are request/response:

```rust
// User presses Enter → Router emits → Executor calls actor → result returned
Dispatcher::dispatch_call(action) {
    let tx = action.replier().cloned();   // Optional reply channel
    let result = Executor::new(app).execute(action);
    if let Some(tx) = tx {
        tx.send(result).ok();              // Send result back to caller
    }
}
```

The `replier` is a `tokio::sync::mpsc::UnboundedSender` attached to the action. This lets callers (like plugins) await the result asynchronously.

---

## Part III: How They Work Together

### The Composition Pattern

Yazi's architecture is a **layered composition** of actors and events:

```
┌─────────────────────────────────────────────────────────────────┐
│                         INPUT LAYER                              │
│  Terminal → Crossterm → KeyEvent/MouseEvent → Event::Key         │
├─────────────────────────────────────────────────────────────────┤
│                      DISPATCH LAYER (Event-Driven)               │
│  Event Bus → Dispatcher → Router (keys) / Executor (calls)       │
├─────────────────────────────────────────────────────────────────┤
│                       ACTOR LAYER (Actor Model)                  │
│  Executor → act!(layer:name, cx, action) → Actor::act()          │
│                                              ↓                   │
│                                         mutate Core state        │
├─────────────────────────────────────────────────────────────────┤
│                      PROXY LAYER (Event-Driven)                  │
│  Actor → AppProxy::update_progress() → emit!(Call(...))          │
├─────────────────────────────────────────────────────────────────┤
│                     SCHEDULER LAYER (Actor-like)                 │
│  Worker queues → tokio::spawn(async { process messages })        │
└─────────────────────────────────────────────────────────────────┘
```

### From Keypress to File Open: The Full Journey

Let's trace what happens when you press `Enter` on a file in Yazi:

```
STEP 1: INPUT CAPTURE (Event-Driven)
═══════════════════════════════════════
[Terminal] detects Enter keypress
    │
    ▼
[Crossterm] produces KeyEvent { code: Enter, modifiers: None }
    │
    ▼
Event::Key(key).emit() → TX.send(Event::Key) → Event Bus


STEP 2: DISPATCH (Event-Driven)
═══════════════════════════════════════
[App::serve] rx.recv_many(&mut events, 50)
    │
    ▼
Dispatcher::dispatch(Event::Key(key))
    │
    ▼
Router::route(Key::Enter)
    │
    ▼
KEYMAP[Layer::Mgr] lookup → "open" action bound to Enter
    │
    ▼
emit!(Seq([Action("mgr:open")]))  // Back into the event bus!


STEP 3: EXECUTION (Actor Model)
═══════════════════════════════════════
Dispatcher::dispatch(Event::Call(Action("mgr:open")))
    │
    ▼
Executor::execute(action) → Layer::Mgr → mgr(action)
    │
    ▼
act!(mgr:open, cx, action)
    │
    ├─► Parse action into OpenForm { targets, interactive }
    │
    ├─► Preflight: check if any plugin wants to intercept "open"
    │       Plugin returns Nil → action cancelled!
    │       Plugin returns modified form → use that instead
    │
    ▼
Open::act(cx, OpenForm { ... })
    │
    ├─► Determine opener from MIME type (image? video? text?)
    │
    ├─► If interactive (blocking): scheduler.process.block(...)
    │
    └─► If non-blocking: scheduler.process.bg(...)


STEP 4: BACKGROUND WORK (Actor-like Workers)
═══════════════════════════════════════
Scheduler queues ProcessIn::Block { cmd: "nvim file.txt" }
    │
    ▼
Process worker tokio::spawn(async { ... })
    │
    ├─► Spawn child process: std::process::Command
    │
    ├─► Wait for process to exit
    │
    └─► Process completes


STEP 5: COMPLETION NOTIFICATION (Event-Driven)
═══════════════════════════════════════
Worker sends TaskOp { id, out: TaskOut::ProcessDone }
    │
    ▼
Op worker receives TaskOp
    │
    ├─► Update progress
    │
    ├─► Run hook if configured
    │
    └─► Task fully complete


STEP 6: UI UPDATE (Event-Driven)
═══════════════════════════════════════
(App may receive DDS events from other instances,
 or Watcher events from filesystem changes,
 or Scheduler progress updates)
    │
    ▼
Any of these → render!() → NEED_RENDER.store(1)
    │
    ▼
App loop detects render flag → app.render(false)
    │
    ▼
[Terminal] redraws with updated state
```

This journey touches **every architectural layer**:
- **Event-driven** for input, dispatch, and notification
- **Actor model** for command execution and state mutation
- **Actor-like workers** for background processing
- **Event-driven proxies** for cross-layer communication

---

## Part IV: Design Decisions & Trade-offs

### Actor Model: Pros & Cons

#### ✅ Pros

| Benefit | How Yazi Gets It |
|---------|-----------------|
| **Isolation** | Each actor only knows its `Form` and `Ctx`. `Cd` can't accidentally mess up `Tasks` state. |
| **Testability** | `Cd::act(&mut fake_ctx, CdForm { target: "/tmp".into() })` is trivial to unit test. |
| **Extensibility** | New actions are just new `Actor` impls. The executor auto-routes them by name. |
| **Plugin hooks** | The preflight system lets plugins wrap any actor without modifying the actor. |
| **Clear boundaries** | The `act!` macro enforces a consistent lifecycle for every action. |

#### ❌ Cons

| Cost | How Yazi Pays It |
|------|-----------------|
| **Boilerplate** | Every action needs: trait impl, `Form` type, parser support, executor match arm. Yazi has ~150 files in `yazi-actor` alone. |
| **Centralized state** | Yazi cheats: actors share `Core` through `Ctx`, so it's not *pure* actor model. Pure actors would have `MgrActor`, `TasksActor`, etc. with message passing between them. |
| **Synchronous only** | Yazi's actors run in the UI thread. Heavy work must be delegated to the scheduler. In a pure actor system, actors can be distributed across threads naturally. |
| **Debugging** | `act!(mgr:cd)` looks simple but hides preflight, parsing, and nested calls. The `#[cfg(debug_assertions)]` backtrace helps, but call stacks are macro-generated. |

### Event-Driven: Pros & Cons

#### ✅ Pros

| Benefit | How Yazi Gets It |
|---------|-----------------|
| **Decoupling** | The scheduler doesn't know about the UI. It calls `AppProxy::update_progress()`, which emits an event. The UI doesn't know about the scheduler. |
| **Composability** | Multiple systems react to the same event. A file change triggers: UI refresh, DDS broadcast, Watcher update, and plugin callbacks. |
| **Async friendly** | Events work naturally with `tokio::mpsc`. No callback hell, no `Arc<Mutex<State>>`. |
| **Replayability** | In theory, you could log all events and replay them to reproduce a bug. Yazi doesn't do this, but the architecture supports it. |

#### ❌ Cons

| Cost | How Yazi Pays It |
|------|-----------------|
| **Indirection** | `emit!(Call(...))` → channel → recv → dispatch → execute. That's 4 hops instead of 1 function call. For hot paths, this adds latency. |
| **No guaranteed ordering** | Events from different producers can arrive in unexpected order. Yazi uses `Seq` for atomic action sequences and `recv_many` batching for throughput. |
| **Memory pressure** | Unbounded channels can grow forever if consumers are slow. Yazi mitigates this with batch rendering and worker backpressure via `Ongoing` limits. |
| **Harder to trace** | "Why did this render?" requires grep-ing for all `render!()` calls. In direct-call code, you'd just follow the call stack. |

### When to Use What

| Scenario | Use Actor Model | Use Event-Driven | Use Both |
|----------|----------------|------------------|----------|
| UI command handling | ✅ | | |
| Background job queue | ✅ (worker as actor) | ✅ (job completion events) | ✅ |
| File system watching | | ✅ | |
| Cross-module notification | | ✅ | |
| Plugin extensibility | ✅ (preflight hooks) | ✅ (pub/sub) | ✅ |
| Real-time collaboration | | ✅ (broadcast) | |
| State machine transitions | ✅ | | |
| Logging/telemetry | | ✅ | |

### Common Anti-Patterns

#### Anti-Pattern 1: "Actors" That Are Just Functions

```rust
// BAD: This is just a function with extra steps
struct DoThing;
impl Actor for DoThing {
    fn act(cx: &mut Ctx, form: Self::Form) -> Result<Data> {
        // 200 lines of unrelated logic
        // No clear input/output
        // Side effects everywhere
    }
}
```

**Fix:** An actor should do one thing. If it's getting long, split it into multiple actors or delegate to the scheduler.

#### Anti-Pattern 2: Events as Function Calls

```rust
// BAD: Synchronous event handling that blocks the bus
emit!(Call(action));
// ... immediately wait for result
```

**Fix:** Events should be fire-and-forget. If you need a result, use a `Replier` channel or make it an actor call (which is synchronous in Yazi's model).

#### Anti-Pattern 3: Shared Mutable State Behind Actors

```rust
// BAD: Actor holds Arc<Mutex<State>>
struct BadActor {
    state: Arc<Mutex<HashMap<String, String>>>,
}
```

**Fix:** State should be in the actor itself (single-threaded access) or in a dedicated state actor that receives update messages. Yazi's centralized `Core` is a pragmatic exception because the UI loop guarantees exclusive access.

#### Anti-Pattern 4: Event Spaghetti

```rust
// BAD: Every module emits events that every other module listens to
// You end up with invisible dependencies
```

**Fix:** Yazi solves this with **layered events**. `Event::Call` always targets a specific `Layer` and actor name. It's not a free-for-all pub/sub within the core — that's reserved for DDS/plugins.

---

## Part V: Building Your Own

### A Minimal Actor System in Rust

Here's a minimal actor system using `tokio`:

```rust
use tokio::sync::{mpsc, oneshot};
use std::fmt::Debug;

// ── Messages ─────────────────────────────────────────────

#[derive(Debug)]
enum CounterMsg {
    Increment,
    Get(oneshot::Sender<usize>),
}

// ── Actor ────────────────────────────────────────────────

struct Counter {
    value: usize,
    rx: mpsc::UnboundedReceiver<CounterMsg>,
}

impl Counter {
    fn new(rx: mpsc::UnboundedReceiver<CounterMsg>) -> Self {
        Self { value: 0, rx }
    }

    async fn run(mut self) {
        while let Some(msg) = self.rx.recv().await {
            match msg {
                CounterMsg::Increment => {
                    self.value += 1;
                    println!("Counter: {}", self.value);
                }
                CounterMsg::Get(tx) => {
                    let _ = tx.send(self.value);
                }
            }
        }
    }
}

// ── Handle (external API) ────────────────────────────────

#[derive(Clone)]
struct CounterHandle {
    tx: mpsc::UnboundedSender<CounterMsg>,
}

impl CounterHandle {
    fn new() -> (Self, Counter) {
        let (tx, rx) = mpsc::unbounded_channel();
        (Self { tx }, Counter::new(rx))
    }

    fn increment(&self) {
        let _ = self.tx.send(CounterMsg::Increment);
    }

    async fn get(&self) -> usize {
        let (tx, rx) = oneshot::channel();
        let _ = self.tx.send(CounterMsg::Get(tx));
        rx.await.expect("actor died")
    }
}

// ── Usage ────────────────────────────────────────────────

#[tokio::main]
async fn main() {
    let (handle, actor) = CounterHandle::new();

    // Spawn the actor
    tokio::spawn(actor.run());

    // Use the handle from anywhere
    handle.increment();
    handle.increment();
    println!("Value: {}", handle.get().await); // Value: 2
}
```

**Key design:**
- `Counter` owns its state and mailbox. Nobody else can touch `value`.
- `CounterHandle` is `Clone + Send`, so you can pass it to any thread.
- Messages are an enum. Each variant can carry data and reply channels.

### A Minimal Event Bus in Rust

```rust
use tokio::sync::broadcast;
use std::sync::Arc;

// ── Event Types ──────────────────────────────────────────

#[derive(Clone, Debug)]
enum AppEvent {
    FileOpened(String),
    DirectoryChanged(String),
    Progress(u8),
}

// ── Event Bus ────────────────────────────────────────────

struct EventBus {
    tx: broadcast::Sender<AppEvent>,
}

impl EventBus {
    fn new() -> Self {
        let (tx, _) = broadcast::channel(256); // Capacity: 256 events
        Self { tx }
    }

    fn emit(&self, event: AppEvent) {
        let _ = self.tx.send(event); // Fire and forget
    }

    fn subscribe(&self) -> broadcast::Receiver<AppEvent> {
        self.tx.subscribe()
    }
}

// ── Consumers ────────────────────────────────────────────

async fn ui_updater(mut rx: broadcast::Receiver<AppEvent>) {
    while let Ok(event) = rx.recv().await {
        match event {
            AppEvent::Progress(p) => println!("[UI] Progress: {}%", p),
            AppEvent::FileOpened(f) => println!("[UI] Opened: {}", f),
            _ => {}
        }
    }
}

async fn logger(mut rx: broadcast::Receiver<AppEvent>) {
    while let Ok(event) = rx.recv().await {
        println!("[LOG] {:?}", event);
    }
}

// ── Usage ────────────────────────────────────────────────

#[tokio::main]
async fn main() {
    let bus = Arc::new(EventBus::new());

    // Spawn two consumers
    tokio::spawn(ui_updater(bus.subscribe()));
    tokio::spawn(logger(bus.subscribe()));

    // Producer
    bus.emit(AppEvent::FileOpened("README.md".into()));
    bus.emit(AppEvent::Progress(50));
    bus.emit(AppEvent::Progress(100));

    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
}
```

**Key design:**
- `broadcast::channel` gives one-to-many semantics. Every subscriber gets every event.
- Producers don't know about consumers. The bus is the only shared thing.
- Backpressure: if a subscriber is slow, old events are dropped (broadcast behavior).

### Putting Them Together

Here's how you'd compose actors and events in a real application:

```rust
use tokio::sync::{broadcast, mpsc, oneshot};

// ── Shared Event Bus ─────────────────────────────────────

#[derive(Clone, Debug)]
enum SystemEvent {
    WorkStarted { id: u64 },
    WorkCompleted { id: u64, result: String },
    Render,
}

// ── Background Worker Actor ──────────────────────────────

struct Worker {
    rx: mpsc::UnboundedReceiver<WorkMsg>,
    events: broadcast::Sender<SystemEvent>,
}

enum WorkMsg {
    DoWork { id: u64, payload: String },
}

impl Worker {
    async fn run(mut self) {
        while let Some(msg) = self.rx.recv().await {
            match msg {
                WorkMsg::DoWork { id, payload } => {
                    self.events.send(SystemEvent::WorkStarted { id }).ok();

                    // Simulate work
                    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
                    let result = format!("Processed: {}", payload);

                    self.events.send(SystemEvent::WorkCompleted { id, result }).ok();
                    self.events.send(SystemEvent::Render).ok();
                }
            }
        }
    }
}

// ── UI Actor ─────────────────────────────────────────────

struct Ui {
    rx: mpsc::UnboundedReceiver<UiMsg>,
    events: broadcast::Sender<SystemEvent>,
    work_tx: mpsc::UnboundedSender<WorkMsg>,
    pending: Vec<u64>,
}

enum UiMsg {
    ButtonPressed(String),
}

impl Ui {
    async fn run(mut self) {
        // Also listen to system events
        let mut event_rx = self.events.subscribe();

        loop {
            tokio::select! {
                Some(msg) = self.rx.recv() => match msg {
                    UiMsg::ButtonPressed(payload) => {
                        let id = rand::random::<u64>();
                        self.pending.push(id);
                        self.work_tx.send(WorkMsg::DoWork { id, payload }).ok();
                    }
                },
                Ok(event) = event_rx.recv() => match event {
                    SystemEvent::WorkStarted { id } => {
                        println!("[UI] Job {} started...", id);
                    }
                    SystemEvent::WorkCompleted { id, result } => {
                        self.pending.retain(|&x| x != id);
                        println!("[UI] Job {} done: {}", id, result);
                    }
                    SystemEvent::Render => {
                        println!("[UI] Rendering... ({} pending)", self.pending.len());
                    }
                    _ => {}
                },
            }
        }
    }
}

// ── Main ─────────────────────────────────────────────────

#[tokio::main]
async fn main() {
    let (event_tx, _) = broadcast::channel(128);
    let (work_tx, work_rx) = mpsc::unbounded_channel();
    let (ui_tx, ui_rx) = mpsc::unbounded_channel();

    let worker = Worker {
        rx: work_rx,
        events: event_tx.clone(),
    };

    let ui = Ui {
        rx: ui_rx,
        events: event_tx.clone(),
        work_tx,
        pending: vec![],
    };

    tokio::spawn(worker.run());
    tokio::spawn(ui.run());

    // Simulate user pressing buttons
    ui_tx.send(UiMsg::ButtonPressed("file1.txt".into())).unwrap();
    ui_tx.send(UiMsg::ButtonPressed("file2.txt".into())).unwrap();

    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
}
```

**What this demonstrates:**
1. **UI Actor** receives user input, delegates work to Worker
2. **Worker Actor** processes work asynchronously
3. **Event Bus** notifies everyone of progress (UI, potential loggers, metrics, etc.)
4. **No shared mutable state** — only message passing and event broadcasting

This is exactly the pattern Yazi uses, just scaled up to 28 crates and hundreds of message types.

---

## Summary

| Concept | Actor Model | Event-Driven | Yazi Uses... |
|---------|-------------|--------------|--------------|
| **Core idea** | Encapsulated state + message handlers | Producers emit, consumers react | Both |
| **Communication** | Addressed messages (actor A → actor B) | Anonymous broadcasts (anyone → bus) | Both |
| **Coupling** | Loosely coupled (don't know internals) | Fully decoupled (don't know existence) | Both |
| **State access** | Sequential, single-threaded per actor | Read-only or owned copies | Sequential via `&mut Ctx` |
| **Concurrency** | Natural (each actor is independent) | Natural (multiple consumers) | Workers + event bus |
| **Best for** | Commands, state machines, workers | Notifications, side effects, telemetry | Commands + notifications |

Yazi's architecture is a masterclass in **pragmatic composition**:
- It uses **actors** where strong typing and ordered execution matter (UI commands)
- It uses **events** where decoupling and broadcast matter (rendering, DDS, progress)
- It keeps the core state **single-threaded** for simplicity, while pushing slow work to **actor-like workers**
- It uses **macros** to keep the patterns ergonomic without runtime overhead

The result is a codebase that scales in features without scaling in complexity — each new actor is just a new file, and each new event consumer is just a new subscriber.
