# Yazi's Cargo Workspace & Crate Architecture

> **How Yazi splits 30,000+ lines of Rust into 28 focused crates, and why that matters.**

---

## Table of Contents

1. [The Big Picture](#the-big-picture)
2. [Why So Many Crates?](#why-so-many-crates)
3. [The Root Workspace](#the-root-workspace)
4. [Crate Categories](#crate-categories)
5. [The Dependency Graph](#the-dependency-graph)
6. [How Crates Talk to Each Other](#how-crates-talk-to-each-other)
7. [The `yazi-macro` Crate: Glue Code](#the-yazi-macro-crate-glue-code)
8. [Public API Boundaries](#public-api-boundaries)
9. [Version Synchronization](#version-synchronization)
10. [Build Configuration](#build-configuration)
11. [Adding a New Crate](#adding-a-new-crate)
12. [Lessons for Your Own Workspace](#lessons-for-your-own-workspace)

---

## The Big Picture

Yazi is a **Cargo workspace** with **28 internal crates**, all prefixed with `yazi-`:

```
yazi/
├── Cargo.toml          # Workspace root
├── Cargo.lock          # Unified lockfile
├── yazi-actor/         # Actor trait + all action implementations
├── yazi-adapter/       # Terminal image protocol adapters
├── yazi-binding/       # Lua ↔ Rust bindings
├── yazi-boot/          # CLI argument parsing & boot
├── yazi-build/         # Build scripts
├── yazi-cli/           # `ya` command-line binary
├── yazi-codegen/       # Proc macros
├── yazi-config/        # Configuration loading
├── yazi-core/          # Central state machine
├── yazi-dds/           # Inter-process pub/sub
├── yazi-emulator/      # Terminal emulator detection
├── yazi-ffi/           # Foreign function interface
├── yazi-fm/            # Main TUI binary (`yazi`)
├── yazi-fs/            # File system abstractions
├── yazi-macro/         # Declarative macros
├── yazi-packing/       # Build packaging
├── yazi-parser/        # Command/form parsing
├── yazi-plugin/        # Lua plugin system
├── yazi-proxy/         # Type-safe event proxies
├── yazi-runner/        # Preview/fetch/preload runners
├── yazi-scheduler/     # Background task scheduler
├── yazi-shared/        # Shared primitives
├── yazi-shim/          # Compatibility shims
├── yazi-sftp/          # SFTP virtual file system
├── yazi-term/          # Terminal backend wrapper
├── yazi-tty/           # Low-level TTY I/O
├── yazi-vfs/           # Virtual file system
├── yazi-watcher/       # File system watcher
└── yazi-widgets/       # Reusable TUI widgets
```

---

## Why So Many Crates?

### Reason 1: Compile-Time Parallelism

Rust compiles crates in parallel. Splitting into 28 crates means:
- 28 independent compilation units
- Changes to `yazi-actor` don't recompile `yazi-scheduler`
- Incremental builds are fast during development

```
# Change one file in yazi-actor:
cargo check  # Only recompiles yazi-actor + dependents (yazi-fm)

# Instead of recompiling everything:
cargo check  # Would recompile 30,000+ lines if it were one crate
```

### Reason 2: Enforced Boundaries

A crate boundary is a **compile-time firewall**:
- You can't accidentally use private types from another crate
- You must explicitly declare dependencies in `Cargo.toml`
- Circular dependencies are impossible (Cargo forbids them)

```rust
// In yazi-actor, you CANNOT do this unless yazi-core is in Cargo.toml:
use yazi_core::Core;  // ← Compile error if dependency missing
```

### Reason 3: Independent Testing

Each crate has its own test suite:

```bash
cargo test -p yazi-fs     # Test file system abstractions only
cargo test -p yazi-shared # Test shared primitives
cargo test -p yazi-macro  # Test macro expansions
```

This is faster than `cargo test` (which runs everything) and makes it easier to isolate failures.

### Reason 4: Reusability

Some crates are genuinely reusable:
- `yazi-widgets` could be used by another TUI project
- `yazi-macro` provides general-purpose utilities
- `yazi-tty` abstracts TTY operations for any terminal app

### Reason 5: Feature Flags Per Crate

Each crate can have its own feature flags:

```toml
# yazi-adapter/Cargo.toml
[features]
default = ["sixel", "iterm2", "kitty"]
sixel = []
iterm2 = []
kitty = []
```

This lets you compile Yazi without image preview support by disabling the adapter crate's features.

---

## The Root Workspace

The root `Cargo.toml` defines the workspace:

```toml
# Cargo.toml (workspace root)
[workspace]
resolver = "3"
members = ["yazi-*"]        # All yazi-* folders are crates
default-members = ["yazi-fm", "yazi-cli"]

[workspace.package]
edition      = "2024"
version      = "26.2.2"
license      = "MIT"
authors      = ["sxyazi <sxyazi@gmail.com>"]
homepage     = "https://yazi-rs.github.io"
repository   = "https://github.com/sxyazi/yazi"
rust-version = "1.95.0"

[workspace.dependencies]
# All dependencies are declared here once, referenced by crates
anyhow      = { version = "1.0.102" }
tokio       = { version = "1.52.2", features = ["full"] }
ratatui     = { version = "0.30.0", features = ["serde"] }
crossterm   = { version = "0.29.0", features = ["event-stream"] }
serde       = { version = "1.0.228", features = ["derive"] }
# ... 50+ more
```

### Key Workspace Features

| Feature | Purpose |
|---------|---------|
| `members = ["yazi-*"]` | Glob pattern includes all yazi-* folders |
| `default-members` | `cargo build` builds these by default (the binaries) |
| `workspace.dependencies` | Declare deps once, use `workspace = true` in crates |
| `workspace.package` | Shared metadata (version, license, edition) |

### Inheriting in Child Crates

Each crate's `Cargo.toml` is minimal:

```toml
# yazi-core/Cargo.toml
[package]
name = "yazi-core"
version.workspace = true      # ← Uses 26.2.2 from workspace
edition.workspace = true      # ← Uses 2024 from workspace
license.workspace = true      # ← Uses MIT from workspace
authors.workspace = true
# ... etc

[dependencies]
yazi-macro     = { path = "../yazi-macro", version = "26.2.2" }
yazi-shared    = { path = "../yazi-shared", version = "26.2.2" }
yazi-fs        = { path = "../yazi-fs", version = "26.2.2" }
yazi-config    = { path = "../yazi-config", version = "26.2.2" }
# ... etc

anyhow      = { workspace = true }  # ← Uses version from workspace
tokio       = { workspace = true }
ratatui     = { workspace = true }
```

**Benefits:**
- Change a dependency version in **one place** (workspace root)
- All crates automatically use the new version
- No version drift between crates

---

## Crate Categories

Yazi's 28 crates fall into 7 logical layers:

### Layer 1: Foundation (No Internal Dependencies)

```
yazi-macro   → proc macros + declarative macros
yazi-shim    → RoCell, SyncCell, compatibility shims
yazi-codegen → proc macro helpers for deserialization
yazi-ffi     → FFI utilities
```

These crates have **zero dependencies** on other yazi-* crates. Everything else depends on them.

### Layer 2: Utilities (Depend on Foundation)

```
yazi-shared  → Event, Action, Layer, Data, Url, Id
yazi-tty     → TTY I/O abstractions
yazi-emulator→ Terminal emulator detection
yazi-fs      → File, Cha, UrlBuf, Files
```

These provide **domain primitives** used by higher layers. They don't know about UI or business logic.

### Layer 3: Configuration (Depend on Utilities)

```
yazi-config  → KEYMAP, THEME, LAYOUT, YAZI config
yazi-boot    → CLI args, boot sequence, cwds
```

These handle **configuration and startup**. They know about the file system and shared types, but not about the UI.

### Layer 4: UI Primitives (Depend on Config + Utilities)

```
yazi-widgets → Input, List, Scrollable, Clear
yazi-binding → Renderable, Lua bindings, elements
yazi-term    → Term::start(), Term::stop()
yazi-adapter → Image protocols (kitty, sixel, iTerm2)
```

These are **reusable UI building blocks**. They know about rendering but not about the application's business logic.

### Layer 5: Background Services (Depend on Config + Utilities)

```
yazi-vfs     → Virtual file system (local + SFTP)
yazi-watcher → File system change watching
yazi-dds     → Inter-process pub/sub
yazi-runner  → PeekJob, Fetcher, Preloader
yazi-scheduler→ Scheduler, Worker, Task, Ongoing
```

These handle **background work**. They run concurrently and emit events back to the UI. They don't touch `Core` state directly.

### Layer 6: Core + Actors (Depend on Almost Everything)

```
yazi-core    → Core, Mgr, Tab, Folder, Tasks, Input, etc.
yazi-parser  → Spark, SparkKind, action parsing
yazi-proxy   → AppProxy, MgrProxy, type-safe proxies
yazi-actor   → Actor trait + 80+ action implementations
yazi-plugin  → Lua plugin system
```

These form the **application's brain**. They define state, actions, and behavior. They import from all lower layers but are not imported by them (except `yazi-fm` and `yazi-plugin`).

### Layer 7: Binaries (Depend on Everything)

```
yazi-fm  → The main TUI binary (links everything)
yazi-cli → The `ya` command-line tool (links subset)
```

These are the **entry points**. They initialize all crates and start the event loop.

### Visual Dependency Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        BINARIES                                  │
│  yazi-fm                yazi-cli                                 │
├─────────────────────────────────────────────────────────────────┤
│                    CORE & ACTORS                                 │
│  yazi-actor  yazi-plugin  yazi-parser  yazi-proxy               │
├─────────────────────────────────────────────────────────────────┤
│                        CORE STATE                                │
│  yazi-core                                                       │
├─────────────────────────────────────────────────────────────────┤
│                    BACKGROUND SERVICES                           │
│  yazi-scheduler  yazi-runner  yazi-dds  yazi-watcher  yazi-vfs  │
├─────────────────────────────────────────────────────────────────┤
│                      UI PRIMITIVES                               │
│  yazi-widgets  yazi-binding  yazi-term  yazi-adapter             │
├─────────────────────────────────────────────────────────────────┤
│                      CONFIGURATION                               │
│  yazi-config  yazi-boot                                          │
├─────────────────────────────────────────────────────────────────┤
│                        UTILITIES                                 │
│  yazi-shared  yazi-fs  yazi-tty  yazi-emulator                   │
├─────────────────────────────────────────────────────────────────┤
│                        FOUNDATION                                │
│  yazi-macro  yazi-shim  yazi-codegen  yazi-ffi                   │
└─────────────────────────────────────────────────────────────────┘
```

**Rule:** Arrows point upward. A crate can only depend on crates in its own layer or layers below it.

---

## The Dependency Graph

Here's the actual dependency graph (extracted from all `Cargo.toml` files):

### Foundation Crates

```
yazi-macro   → (no yazi deps)
yazi-shim    → yazi-macro
yazi-codegen → (no yazi deps)
yazi-ffi     → yazi-macro
```

### Utility Crates

```
yazi-shared  → yazi-macro, yazi-shim
yazi-tty     → yazi-macro, yazi-shared, yazi-shim
yazi-emulator→ yazi-macro, yazi-shared, yazi-shim, yazi-tty
yazi-fs      → yazi-macro, yazi-shared, yazi-shim, yazi-ffi
```

### Config Crates

```
yazi-config  → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-tty, yazi-codegen
yazi-boot    → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, 
               yazi-adapter, yazi-emulator, yazi-vfs
```

### UI Crates

```
yazi-widgets → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-adapter, yazi-tty
yazi-binding → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-fs, 
               yazi-adapter, yazi-vfs, yazi-widgets, yazi-codegen
yazi-term    → yazi-macro, yazi-shim, yazi-config, yazi-emulator, yazi-tty
yazi-adapter → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-fs, 
               yazi-emulator, yazi-tty
```

### Background Crates

```
yazi-vfs     → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, yazi-sftp
yazi-watcher → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-adapter, 
               yazi-dds, yazi-vfs
yazi-dds     → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-boot, yazi-binding
yazi-runner  → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, 
               yazi-boot, yazi-dds, yazi-binding
yazi-scheduler→ yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, 
                yazi-dds, yazi-runner, yazi-binding, yazi-term, yazi-vfs
```

### Core + Actor Crates

```
yazi-core    → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-fs, 
               yazi-adapter, yazi-binding, yazi-dds, yazi-emulator, yazi-runner, 
               yazi-scheduler, yazi-vfs, yazi-watcher, yazi-widgets

yazi-parser  → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, 
               yazi-core, yazi-dds, yazi-binding, yazi-scheduler, yazi-vfs, 
               yazi-widgets, yazi-boot

yazi-proxy   → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-core, 
               yazi-scheduler, yazi-widgets

yazi-actor   → yazi-macro, yazi-shared, yazi-shim, yazi-config, yazi-fs, 
               yazi-core, yazi-dds, yazi-emulator, yazi-parser, yazi-plugin, 
               yazi-proxy, yazi-runner, yazi-scheduler, yazi-term, yazi-tty, 
               yazi-vfs, yazi-watcher, yazi-widgets, yazi-binding, yazi-boot

yazi-plugin  → yazi-macro, yazi-shared, yazi-shim, yazi-fs, yazi-config, 
               yazi-core, yazi-dds, yazi-adapter, yazi-binding, yazi-emulator, 
               yazi-proxy, yazi-runner, yazi-scheduler, yazi-term, yazi-vfs, 
               yazi-widgets, yazi-boot
```

### Binary Crates

```
yazi-fm  → ALL CRATES (depends on everything)
yazi-cli → yazi-boot, yazi-dds, yazi-fs, yazi-macro, yazi-shared, yazi-shim
```

---

## How Crates Talk to Each Other

### Rule 1: Downward Dependencies Only

A crate can only depend on crates in its layer or below. `yazi-core` can import `yazi-fs`, but `yazi-fs` cannot import `yazi-core`. This prevents circular dependencies.

### Rule 2: Communication via Types, Not Functions

Crates communicate by sharing types, not by calling each other's internal functions:

```rust
// yazi-shared defines the types
pub enum Event { ... }
pub enum Layer { ... }
pub struct Action { ... }

// yazi-core uses them
use yazi_shared::{Event, Layer, Action};

// yazi-actor uses them
use yazi_shared::{Event, Layer, Action};

// yazi-scheduler uses them
use yazi_shared::event::Action;
```

### Rule 3: Proxies for Cross-Layer Communication

When a background crate needs to trigger a UI update, it uses a **proxy** rather than importing the UI crate:

```rust
// yazi-core/src/proxy.rs (lives in yazi-core, used by lower layers)
pub struct AppProxy;
impl AppProxy {
    pub fn update_progress(summary: TaskSummary) {
        // Emits an event — no direct dependency on yazi-fm
        emit!(Call(relay!(app:update_progress).with_any("summary", summary)));
    }
}

// yazi-scheduler uses AppProxy without importing yazi-fm
use yazi_core::AppProxy;
AppProxy::update_progress(new_summary);
```

This is **dependency inversion**: `yazi-scheduler` depends on `yazi-core` (which is fine, it's a lower layer), and `yazi-core` provides the proxy. The event bus handles the actual delivery.

### Rule 4: Macros Live in `yazi-macro`

All macros are centralized:

```rust
// yazi-macro/src/actor.rs
#[macro_export]
macro_rules! act {
    ($layer:ident : $name:ident, $cx:ident, $action:expr) => { ... };
}

// yazi-macro/src/event.rs
#[macro_export]
macro_rules! emit {
    (Call($action:expr)) => { ... };
}

// Used across all crates
use yazi_macro::{act, emit, relay, render, succ};
```

This avoids duplicating macro definitions and ensures consistency.

---

## The `yazi-macro` Crate: Glue Code

`yazi-macro` is one of the most important crates. It contains:

```
yazi-macro/src/
├── actor.rs      # act! macro
├── asset.rs      # Asset loading macros
├── context.rs    # Context helpers
├── event.rs      # emit!, relay!, succ! macros
├── fmt.rs        # Formatting macros
├── fs.rs         # File system macros
├── log.rs        # Logging macros
├── module.rs     # mod_pub!, mod_flat! macros
├── platform.rs   # Platform detection macros
├── render.rs     # render!, render_and!, render_partial! macros
└── stdio.rs      # Stdio macros
```

### `mod_pub!` and `mod_flat!`

These macros reduce boilerplate for module declarations:

```rust
// Instead of writing:
pub mod app;
pub mod cmp;
pub mod confirm;
// ... etc

// Yazi writes:
yazi_macro::mod_pub!(app cmp confirm help input lives mgr notify pick spot tasks which);

// Which expands to:
pub mod app;
pub mod cmp;
pub mod confirm;
// ... etc
```

```rust
// Instead of writing:
pub mod actor;
pub mod context;
pub use actor::*;
pub use context::*;

// Yazi writes:
yazi_macro::mod_flat!(actor context);

// Which expands to:
pub mod actor;
pub mod context;
pub use actor::*;
pub use context::*;
```

This keeps `lib.rs` files clean and consistent across all 28 crates.

---

## Public API Boundaries

Each crate exposes a **public API** through its `lib.rs`:

```rust
// yazi-actor/src/lib.rs
extern crate self as yazi_actor;

yazi_macro::mod_pub!(app cmp confirm core help input lives mgr notify pick spot tasks which);
yazi_macro::mod_flat!(actor context);
```

```rust
// yazi-core/src/lib.rs
extern crate self as yazi_core;

yazi_macro::mod_pub!(app cmp confirm help input mgr notify pick spot tab tasks which);
yazi_macro::mod_flat!(core highlighter proxy);
```

### Visibility Rules

| Visibility | Meaning | Example |
|-----------|---------|---------|
| `pub` | Public API of the crate | `pub struct Core` |
| `pub(crate)` | Internal to this crate only | `pub(crate) fn internal_helper()` |
| `pub(super)` | Visible to parent module | `pub(super) fn shared_helper()` |
| `pub(in path)` | Visible to specific module | Rarely used in Yazi |

Yazi uses `pub` for the API surface and keeps implementation details private.

---

## Version Synchronization

All crates share the **same version** via workspace inheritance:

```toml
# Cargo.toml (workspace root)
[workspace.package]
version = "26.2.2"
```

```toml
# Every crate's Cargo.toml
[package]
version.workspace = true
```

### Why Same Version?

1. **No version mismatches** — you never have `yazi-core = "26.2.1"` depending on `yazi-shared = "26.2.2"`
2. **Single version bump** — release time is one line change
3. **Clear compatibility** — all crates at version X are designed to work together

### Releasing a New Version

```bash
# 1. Edit one line in root Cargo.toml
version = "26.3.0"

# 2. All crates automatically pick it up
# No need to edit 28 individual Cargo.toml files
```

---

## Build Configuration

### Profiles

```toml
# Cargo.toml (workspace root)
[profile.dev]
debug = "line-tables-only"  # Faster builds, still debuggable

[profile.release]
codegen-units = 1    # Single codegen unit for max optimization
lto = true           # Link-time optimization
panic = "abort"      # Smaller binary, no unwinding
strip = true         # Strip symbols for smaller binary

[profile.release-windows]
inherits = "release"
panic = "unwind"     # Windows needs unwinding for some FFI

[profile.dev-opt]
inherits = "release"
codegen-units = 256  # Parallel codegen for faster release builds
incremental = true   # Incremental compilation
lto = false          # Disable LTO for faster linking
```

### Workspace Lints

```toml
[workspace.lints.clippy]
format_push_string = "warn"
use_self = "warn"
implicit_clone = "warn"
```

Shared clippy lints apply to all crates.

### Default Members

```toml
default-members = ["yazi-fm", "yazi-cli"]
```

Running `cargo build` only builds the two binaries by default. To build everything:

```bash
cargo build --workspace
```

---

## Adding a New Crate

Want to add `yazi-search` for fuzzy search functionality? Here's how:

### Step 1: Create the Folder

```bash
mkdir yazi-search
cd yazi-search
```

### Step 2: Create `Cargo.toml`

```toml
[package]
name = "yazi-search"
version.workspace = true
edition.workspace = true
license.workspace = true
authors.workspace = true
homepage.workspace = true
repository.workspace = true
rust-version.workspace = true

[lints]
workspace = true

[dependencies]
# Foundation
yazi-macro = { path = "../yazi-macro", version = "26.2.2" }
yazi-shim  = { path = "../yazi-shim", version = "26.2.2" }

# Utilities
yazi-shared = { path = "../yazi-shared", version = "26.2.2" }
yazi-fs     = { path = "../yazi-fs", version = "26.2.2" }

# External
anyhow  = { workspace = true }
tokio   = { workspace = true }
serde   = { workspace = true }
```

### Step 3: Create `src/lib.rs`

```rust
// yazi-search/src/lib.rs
extern crate self as yazi_search;

pub mod fuzzy;
pub mod matcher;

pub use fuzzy::FuzzySearcher;
pub use matcher::Matcher;
```

### Step 4: Add to Dependent Crates

```toml
# yazi-core/Cargo.toml
[dependencies]
yazi-search = { path = "../yazi-search", version = "26.2.2" }
```

### Step 5: Use It

```rust
// In yazi-core or yazi-actor
use yazi_search::FuzzySearcher;

fn search_files(query: &str, candidates: Vec<&str>) -> Vec<String> {
    FuzzySearcher::new(query).search(candidates)
}
```

---

## Lessons for Your Own Workspace

### Lesson 1: Start Small, Split Later

Don't create 28 crates on day one. Start with:
```
myapp/
├── Cargo.toml
└── src/
    └── main.rs
```

Split when you feel pain:
- "Tests are too slow" → split into library crate
- "This module could be reused" → extract into separate crate
- "I want different feature flags" → split into separate crate

### Lesson 2: Name Crates by Domain, Not Layer

Yazi names crates by **domain** (`yazi-fs`, `yazi-widgets`) not by layer (`yazi-layer1`, `yazi-layer2`). This makes dependencies self-documenting.

```toml
# Good: tells you what it does
yazi-fs = { path = "../yazi-fs" }

# Bad: tells you nothing
yazi-layer3 = { path = "../yazi-layer3" }
```

### Lesson 3: Keep Binary Crates Thin

`yazi-fm` is ~2,000 lines. The heavy logic lives in `yazi-actor`, `yazi-core`, etc. Binary crates should only:
- Initialize everything
- Start the event loop
- Handle shutdown

```rust
// yazi-fm/src/main.rs
fn main() {
    // 1. Initialize all crates
    yazi_shared::init();
    yazi_config::init().unwrap();
    yazi_plugin::init().unwrap();
    
    // 2. Create app
    let mut app = App::new();
    
    // 3. Run event loop
    app.run().unwrap();
}
```

### Lesson 4: Use `path` Dependencies for Internal Crates

```toml
yazi-core = { path = "../yazi-core", version = "26.2.2" }
```

The `path` tells Cargo to use the local directory. The `version` is required for publishing to crates.io (if you ever do).

### Lesson 5: One Lockfile to Rule Them All

A workspace has **one** `Cargo.lock` at the root. All crates share the same dependency versions. This prevents:
- `yazi-core` using `tokio 1.50`
- `yazi-scheduler` using `tokio 1.48`

### Lesson 6: Test at Crate Boundaries

```bash
# Fast: test one crate
cargo test -p yazi-fs

# Medium: test a crate and its dependents
cargo test -p yazi-core --workspace

# Slow: test everything
cargo test --workspace
```

---

## Summary

| Principle | Yazi's Approach |
|-----------|----------------|
| **Workspace size** | 28 crates, all in one repo |
| **Naming** | Domain-based (`yazi-fs`, not `yazi-layer3`) |
| **Dependencies** | Downward only (no circular deps) |
| **Versioning** | Single shared version via workspace inheritance |
| **Communication** | Shared types + event bus (not direct function calls) |
| **Macros** | Centralized in `yazi-macro` |
| **Binaries** | Thin shells (`yazi-fm`, `yazi-cli`) |
| **Testing** | Per-crate + workspace-wide |
| **Build** | Parallel compilation of independent crates |
| **Release** | One version bump, everything updates |

Yazi's workspace architecture proves that **monorepos with many small crates** can scale to complex applications while maintaining fast builds, clear boundaries, and independent testability.
