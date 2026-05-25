# Learning Resources: Actor Architecture, Event-Driven Systems & TUI Development

> **A curated collection of tutorials, books, videos, and courses for learning the patterns used in Yazi's architecture.**

---

## Table of Contents

1. [The Exact Patterns Yazi Uses](#the-exact-patterns-yazi-uses)
2. [Free Books & Tutorials](#free-books--tutorials)
3. [Video Courses & Talks](#video-courses--talks)
4. [Official Documentation](#official-documentation)
5. [Blog Posts & Articles](#blog-posts--articles)
6. [Example Projects to Study](#example-projects-to-study)
7. [Rust Architecture Patterns](#rust-architecture-patterns)
8. [Recommended Learning Path](#recommended-learning-path)

---

## The Exact Patterns Yazi Uses

Before diving into resources, here's a map of what you need to learn and where to find it:

| Pattern Yazi Uses | What It Is | Best Resource |
|-------------------|-----------|---------------|
| **Actor Model** | Message-passing concurrency | Alice Ryhl's "Actors with Tokio" blog post |
| **Event-Driven Architecture** | Events trigger state changes | "Event-Driven Architecture in Rust" by Atharva Pandey |
| **The Elm Architecture (TEA)** | Model-Update-View pattern | Iced Book (GUI, but same concepts) |
| **Centralized State + &mut** | Single owner, no locks | This is idiomatic Rust — learn ownership |
| **Layered Modal System** | Priority stack for screens | Ratatui tutorials + Yazi source code |
| **tokio::spawn workers** | Async background tasks | Tokio tutorial chapters 6-7 |
| **mpsc channels** | Message passing between tasks | Tokio channels tutorial |
| **Cargo workspaces** | Multi-crate projects | Rust Book chapter on workspaces |

---

## Free Books & Tutorials

### 1. "Developing Rust TUI Applications" — Free Online Book

**🔗 [github.com/cloudstreet-dev/Developing-Rust-TUI-Applications](https://github.com/cloudstreet-dev/Developing-Rust-TUI-Applications)**

**The closest thing to a course on building Yazi-style TUIs.** This is a complete book that takes you from zero to a fully-featured task manager with Ratatui.

| Chapter | What You'll Learn |
|---------|-----------------|
| 1. Introduction to TUI Dev | What TUIs are, why they matter |
| 2. Setting Up Your First TUI | Hello World with Ratatui |
| 3. **Understanding the Event Loop** | ⭐ The heartbeat pattern Yazi uses |
| 4. Widgets and Layouts | Constraint-based layouts (Flexbox for terminals) |
| 5. **State Management** | ⭐ Centralized state, the `Model` pattern |
| 6. Styling and Theming | Colors, borders, lipgloss/ratatui styles |
| 7. **Input Handling and Navigation** | ⭐ Key routing, modal handling, vim bindings |
| 8. Advanced Widgets | Tables, trees, tabs, scrolling |
| 9. Error Handling and Logging | Graceful failures in TUIs |
| 10. Testing TUI Applications | How to test visual apps |
| 11. **Final Project: Task Manager** | ⭐ Brings it all together — very Yazi-like |

**Why it's great:** Complete runnable code examples, progressive complexity, builds a real app.

---

### 2. Rust Design Patterns Book — Official Community Resource

**🔗 [rust-unofficial.github.io/patterns](https://rust-unofficial.github.io/patterns/)**

The official Rust design patterns book. Covers:
- **Idioms** — Rust community conventions
- **Design Patterns** — Solutions to common problems
- **Anti-patterns** — What NOT to do

**Relevant sections:**
- **Idioms:** Pass by value, iterator patterns, default traits
- **Patterns:** RAII, newtype, strategy, state machine
- **Anti-patterns:** `unwrap()` abuse, string typing, `Arc<Mutex<>>` when not needed

**Also available as PDF:** [rust-design-patterns.pdf](https://rust-unofficial.github.io/patterns/rust-design-patterns.pdf)

---

### 3. Tokio Tutorial — The Official Async Rust Guide

**🔗 [tokio.rs/tokio/tutorial](https://tokio.rs/tokio/tutorial)**

Yazi's entire async foundation is built on Tokio. This tutorial teaches:

| Chapter | Relevance to Yazi |
|---------|-----------------|
| 1. Getting Started | Setting up async runtime |
| 2. Spawning | `tokio::spawn` for background workers |
| 3. Shared State | Why `Arc<Mutex<>>` is sometimes needed (scheduler) |
| 4. **Channels** | ⭐ `mpsc` — Yazi's event bus |
| 5. I/O | Async file operations |
| 6. **Framing** | Message boundaries (relevant for DDS) |
| 7. **Graceful Shutdown** | How Yazi shuts down workers cleanly |
| 8. Async in Depth | Understanding `poll`, `Pin`, `Waker` |

**Exercise:** After chapter 4, try building a mini event bus with `tokio::sync::mpsc`.

---

### 4. "Actors with Tokio" by Alice Ryhl

**🔗 [ryhl.io/blog/actors-with-tokio](https://ryhl.io/blog/actors-with-tokio/)**

**The definitive article on building actor systems in Rust without a framework.** Alice Ryhl is a Tokio maintainer, and this post explains exactly how to build what Yazi uses.

**What it covers:**
- The actor pattern with Tokio directly (no Actix)
- Message passing with channels
- `tokio::spawn` for actor tasks
- The `Handle` pattern (like Yazi's `AppProxy`)
- Backpressure and bounded channels
- When NOT to use actors

**Code example from the post:**
```rust
use tokio::sync::{mpsc, oneshot};

struct Actor {
    receiver: mpsc::Receiver<ActorMessage>,
    next_id: u32,
}

enum ActorMessage {
    GetUniqueId { respond_to: oneshot::Sender<u32> },
}

impl Actor {
    fn handle_message(&mut self, msg: ActorMessage) {
        match msg {
            ActorMessage::GetUniqueId { respond_to } => {
                respond_to.send(self.next_id).unwrap();
                self.next_id += 1;
            }
        }
    }
}

async fn run_actor(mut actor: Actor) {
    while let Some(msg) = actor.receiver.recv().await {
        actor.handle_message(msg);
    }
}
```

**This is literally the pattern Yazi's scheduler uses.**

---

### 5. Iced Book — The Elm Architecture in Rust

**🔗 [book.iced.rs](https://book.iced.rs/)**

Iced is a cross-platform GUI framework for Rust built on The Elm Architecture. Even though it's for GUIs (not TUIs), it teaches the **exact same architectural pattern** as Yazi:

```
Model → Update → View
  ↑___________↓
    (messages)
```

**Relevant chapters:**
- **First Steps** — Building a counter app (the "hello world" of TEA)
- **Architecture** — How `Update`, `View`, and `Message` fit together
- **The Runtime** — How the event loop works under the hood

**Key insight:** Yazi's `Core` + `Actor` + `Event` is conceptually identical to Iced's `Model` + `Update` + `Message`. The difference is Yazi uses imperative mutation (`&mut Core`) while Iced uses functional updates (return new `Model`).

---

### 6. "Hecto" — Build Your Own Text Editor in Rust

**🔗 [https://www.flenker.blog/hecto/](https://www.flenker.blog/hecto/)**

A step-by-step tutorial for building a text editor in Rust. Not a TUI framework tutorial, but teaches:
- Raw terminal mode
- Keyboard input handling
- Screen buffers
- The event loop

**Great for understanding:** How Crossterm works under the hood, which Yazi uses.

---

### 7. Rust Cookbook — Actor Pattern

**🔗 [rust-lang-nursery.github.io/rust-cookbook/concurrency/actor.html](https://rust-lang-nursery.github.io/rust-cookbook/concurrency/actor.html)**

A concise, runnable example of the actor pattern in Rust. Good for a quick reference.

---

## Video Courses & Talks

### Ratatui Official Videos

**🔗 [youtube.com/@ratatui-rs](https://www.youtube.com/@ratatui-rs)**

The Ratatui YouTube channel has tutorials and talks:

| Video | What You'll Learn |
|-------|-----------------|
| **"Ratatui Tutorial by Jonkero"** | ⭐ Complete beginner-to-intermediate Ratatui guide |
| **Orhun Parmaksız on embedded + Ratatui** | Advanced use cases |
| **Bryan & Adam from Oxide Computer** | Systems programming with TUIs |

---

### Conference Talks

| Talk | Speaker | Topic |
|------|---------|-------|
| **"Building Terminal User Interfaces in Rust"** | Various (RustConf) | TUI patterns and performance |
| **"Async Rust: The State of the Ecosystem"** | Alice Ryhl | Tokio, channels, actor patterns |
| **"Zero-Cost Async IO"** | Without Boats | How async works under the hood |

Search YouTube for: `ratatui tutorial rust`, `tokio channels rust`, `rust tui event loop`

---

## Official Documentation

### Ratatui Documentation

**🔗 [ratatui.rs](https://ratatui.rs)** — The official Ratatui website with:
- **Concepts:** Rendering, widgets, layouts, backends
- **Recipes:** Common patterns (like a cookbook)
- **API Reference:** docs.rs

**🔗 [docs.rs/ratatui](https://docs.rs/ratatui)** — Auto-generated API docs

**🔗 [github.com/ratatui-org/ratatui/tree/main/examples](https://github.com/ratatui-org/ratatui/tree/main/examples)** — Official examples including:
- `demo` — Full showcase of all widgets
- `layout` — Layout constraints
- `popup` — Modal dialogs
- `user_input` — Text input handling

---

### Crossterm Documentation

**🔗 [docs.rs/crossterm](https://docs.rs/crossterm)**

Yazi uses Crossterm for cross-platform terminal events. Key modules:
- `event` — `read()`, `poll()`, `KeyEvent`, `MouseEvent`
- `terminal` — Raw mode, alternate screen, clearing
- `cursor` — Show/hide, position

---

### Tokio Documentation

**🔗 [tokio.rs](https://tokio.rs)** — The async Rust runtime

**🔗 [docs.rs/tokio](https://docs.rs/tokio)** — API docs

**Most relevant for Yazi's architecture:**
- `tokio::sync::mpsc` — Multi-producer, single-consumer channel
- `tokio::sync::oneshot` — Single-use channel (for request/response)
- `tokio::task::JoinHandle` — Background task handle
- `tokio::select!` — Wait for multiple async operations

---

## Blog Posts & Articles

### Event-Driven Architecture in Rust

**🔗 [atharvapandey.com/post/rust/rust-micro-event-driven](https://www.atharvapandey.com/post/rust/rust-micro-event-driven/)**

A practical guide to event-driven microservices in Rust. Covers:
- Event buses
- Decoupled systems
- Message passing

### Actor Model in Rust with Tokio

**🔗 [dev.to/dylan_dumont_266378d98367/actor-model-in-rust](https://dev.to/dylan_dumont_266378d98367/actor-model-in-rust-building-concurrent-systems-with-tokio-and-channels-5e9n)**

A beginner-friendly walkthrough of building actors with Tokio channels.

### Async Rust Design Patterns (O'Reilly Book)

**🔗 [oreilly.com/library/view/async-rust/9781098149086](https://www.oreilly.com/library/view/async-rust/9781098149086/)**

**Chapter 9: Design Patterns** covers:
- The actor pattern
- Request/response patterns
- Event-driven architectures
- Pipeline patterns

*Note: This is a paid book, but many companies have O'Reilly subscriptions.*

### "Designing Data-Intensive Applications" by Martin Kleppmann

**Not Rust-specific, but essential reading.**

Chapters relevant to Yazi's architecture:
- **Ch 4: Encoding and Evolution** — How messages evolve (Yazi's `Action` + `Form`)
- **Ch 5: Replication** — Leader/follower patterns
- **Ch 11: Stream Processing** — Event streams, message brokers

---

## Example Projects to Study

### 1. Yazi (Obviously)

**🔗 [github.com/sxyazi/yazi](https://github.com/sxyazi/yazi)**

Study these files in order:
1. `yazi-shared/src/event/event.rs` — The event bus
2. `yazi-core/src/core.rs` — Centralized state
3. `yazi-fm/src/app/app.rs` — The event loop
4. `yazi-fm/src/router.rs` — Key routing
5. `yazi-fm/src/executor.rs` — Action dispatch
6. `yazi-actor/src/actor.rs` — The Actor trait
7. `yazi-actor/src/context.rs` — `Ctx<'a>` borrowed state
8. `yazi-scheduler/src/worker.rs` — Background workers

### 2. Lazygit

**🔗 [github.com/jesseduffield/lazygit](https://github.com/jesseduffield/lazygit)**

A Git TUI in Go (not Rust), but architecturally similar:
- Centralized state
- Modal system
- Background git commands
- Key routing by context

### 3. Broot

**🔗 [github.com/Canop/broot](https://github.com/Canop/broot)**

A Rust file tree navigator. Simpler than Yazi but uses:
- Crossterm for events
- Custom rendering (not Ratatui)
- Modal dialogs

### 4. Oxker

**🔗 [github.com/mrjackwills/oxker](https://github.com/mrjackwills/oxker)**

A Docker container management TUI in Rust. Uses:
- Ratatui
- Tokio
- Async I/O
- Real-time updates

### 5. Glicol (Audio)

**🔗 [github.com/chaosprint/glicol](https://github.com/chaosprint/glicol)**

Not a TUI, but uses actor-like patterns with message passing for audio synthesis.

---

## Rust Architecture Patterns

### Microsoft Rust Patterns Book

**🔗 [microsoft.github.io/RustTraining/rust-patterns-book](https://microsoft.github.io/RustTraining/rust-patterns-book/)**

Microsoft's internal Rust training material. Covers:
- Error handling patterns
- Type state patterns
- Resource management
- Concurrency patterns

### Software Patterns Lexicon — Rust

**🔗 [softwarepatternslexicon.com/rust](https://softwarepatternslexicon.com/rust/)**

A comprehensive catalog of design patterns implemented in Rust:
- Creational, structural, behavioral patterns
- Concurrency and parallelism patterns
- The actor model in Rust

---

## Recommended Learning Path

### Phase 1: Rust Foundations (If New to Rust)
1. **The Rust Book** — [doc.rust-lang.org/book](https://doc.rust-lang.org/book/)
   - Ch 1-8: Basics
   - Ch 10: Generics, traits
   - Ch 13: Closures, iterators
   - Ch 15: Smart pointers (`Rc`, `RefCell`, `Arc`)
   - Ch 16: Concurrency (`thread`, `mpsc`, `Mutex`, `Arc`)

### Phase 2: Async Rust
2. **Tokio Tutorial** — [tokio.rs/tokio/tutorial](https://tokio.rs/tokio/tutorial)
   - Complete all 8 chapters
   - Focus on channels (ch 4) and graceful shutdown (ch 7)

### Phase 3: Actor Pattern
3. **"Actors with Tokio"** — [ryhl.io/blog/actors-with-tokio](https://ryhl.io/blog/actors-with-tokio/)
   - Read the blog post
   - Implement the counter actor example
   - Extend it with multiple message types

### Phase 4: TUI Basics
4. **Ratatui "Hello World" Tutorial** — [ratatui.rs/tutorials/hello-world](https://ratatui.rs/tutorials/hello-world/)
   - Set up a basic Ratatui app
   - Handle keyboard input
   - Render widgets

### Phase 5: TUI Architecture
5. **"Developing Rust TUI Applications"** — [github.com/cloudstreet-dev/Developing-Rust-TUI-Applications](https://github.com/cloudstreet-dev/Developing-Rust-TUI-Applications)
   - Ch 3: Event Loop
   - Ch 5: State Management
   - Ch 7: Input Handling
   - Ch 11: Final Project

### Phase 6: Elm Architecture Understanding
6. **Iced Book** — [book.iced.rs](https://book.iced.rs/)
   - Ch 1: First Steps (counter app)
   - Ch 2: Architecture (Model-Update-View)
   - Understand how this maps to Yazi's Core-Actor-Event

### Phase 7: Study Real Projects
7. **Read Yazi's source code** (use the guides we've written!)
   - Start with `yazi-fm/src/app/app.rs` (the loop)
   - Then `yazi-fm/src/router.rs` (key routing)
   - Then `yazi-actor/src/actor.rs` (the trait)
   - Then `yazi-scheduler/src/worker.rs` (background workers)

8. **Build your own mini-Yazi**
   - A file list that you can navigate
   - A modal that opens with 'c'
   - A background task that updates the list
   - Layered key routing

### Phase 8: Advanced Patterns
9. **Rust Design Patterns Book** — [rust-unofficial.github.io/patterns](https://rust-unofficial.github.io/patterns/)
   - Learn RAII, newtype, strategy, state machine patterns

10. **"Designing Data-Intensive Applications"** (Kleppmann)
    - Ch 4, 5, 11 — Messaging, replication, stream processing

---

## Quick Reference: Where to Learn Each Yazi Pattern

| Yazi Pattern | Resource | Section |
|-------------|----------|---------|
| `Core` (centralized state) | Developing Rust TUIs | Ch 5: State Management |
| `Event` bus | Tokio Tutorial | Ch 4: Channels |
| `Actor` trait | Alice Ryhl's Blog | "Actors with Tokio" |
| `Dispatcher` | Iced Book | Ch 2: Architecture |
| `Router` (key routing) | Developing Rust TUIs | Ch 7: Input Handling |
| `Scheduler` workers | Tokio Tutorial | Ch 3: Spawning |
| `mpsc` channels | Tokio Docs | `tokio::sync::mpsc` |
| `tokio::select!` | Tokio Tutorial | Ch 7: Graceful Shutdown |
| Ratatui widgets | Ratatui Docs | Concepts → Widgets |
| Ratatui layouts | Ratatui Docs | Concepts → Layouts |
| Crossterm events | Crossterm Docs | `event` module |
| Cargo workspaces | Rust Book | Ch 14: Cargo Workspaces |
| `&mut` state access | The Rust Book | Ch 4: Ownership |
| Layered modals | Developing Rust TUIs | Ch 7 + Yazi source |
| `RoCell` statics | Yazi source | `yazi-shim/src/cell/` |

---

## Summary

**Start here:**
1. Read [Alice Ryhl's "Actors with Tokio"](https://ryhl.io/blog/actors-with-tokio/) (30 min)
2. Do the [Tokio Tutorial](https://tokio.rs/tokio/tutorial) chapters 1-4 (2 hours)
3. Read ["Developing Rust TUI Applications"](https://github.com/cloudstreet-dev/Developing-Rust-TUI-Applications) chapters 3, 5, 7 (3 hours)
4. Build a [Ratatui Hello World](https://ratatui.rs/tutorials/hello-world/) (30 min)
5. Study [Yazi's source code](https://github.com/sxyazi/yazi) using our architecture guides (ongoing)

**Total time to go from zero to building Yazi-style TUIs:** ~8-12 hours of focused learning + practice.
