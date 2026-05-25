# Building Your Own Yazi-Style TUI Application

> **A practical, step-by-step guide to building a terminal application using Yazi's architecture patterns. We'll build "Gitz" — a Git TUI — as the running example.**

---

## Table of Contents

1. [What We're Building](#what-were-building)
2. [The Core Philosophy](#the-core-philosophy)
3. [Prerequisites](#prerequisites)
4. [Step 0: Project Setup](#step-0-project-setup)
5. [Step 1: The Single Source of Truth](#step-1-the-single-source-of-truth)
6. [Step 2: The Event Loop](#step-2-the-event-loop)
7. [Step 3: The Event Bus](#step-3-the-event-bus)
8. [Step 4: Layered Screens](#step-4-layered-screens)
9. [Step 5: Command Dispatch](#step-5-command-dispatch)
10. [Step 6: Background Workers](#step-6-background-workers)
11. [Step 7: Rendering](#step-7-rendering)
12. [Step 8: Input Routing](#step-8-input-routing)
13. [Step 9: Putting It All Together](#step-9-putting-it-all-together)
14. [Common Patterns & Recipes](#common-patterns--recipes)
15. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
16. [Full Working Example](#full-working-example)

---

## What We're Building

**Gitz** — a terminal UI for Git operations. Think of it as a simpler version of lazygit, built with Yazi's architecture:

```
┌──────────────────────────────────────┐
│ Gitz — Git TUI                        │
├──────────────────────────────────────┤
│                                       │
│  ▶ my-project/         [main ↑2↓1]   │
│    another-repo/       [feature]      │
│    dotfiles/           [master]       │
│                                       │
├──────────────────────────────────────┤
│ M  src/main.rs      |  +45  -12      │
│ M  Cargo.toml       |   +3   -1      │
│ ?? README.md        |  new file       │
│                                       │
├──────────────────────────────────────┤
│ j/k: navigate  |  s: stage  |  c: commit│
└──────────────────────────────────────┘
```

Press `c` → commit dialog opens:
```
┌──────────────────────────────────────┐
│ Commit                                │
│ ┌──────────────────────────────────┐ │
│ │ fix: resolve merge conflict      │ │
│ └──────────────────────────────────┘ │
│                                       │
│ <Enter> to confirm  <Esc> to cancel   │
└──────────────────────────────────────┘
```

---

## The Core Philosophy

Before writing code, understand these 5 principles:

### Principle 1: One Struct Owns Everything

All application state lives in **one struct**. Not `Arc<Mutex<State>>`. Not scattered globals. One owned value.

```rust
// ✅ Good
struct App {
    state: State,  // Everything is here
}

// ❌ Bad
static STATE: Lazy<Mutex<State>> = Lazy::new(|| Mutex::new(State::new()));
```

### Principle 2: One Loop Processes Everything

A single async loop receives events and mutates state. Nothing else touches state.

```rust
// ✅ Good
loop {
    let event = rx.recv().await;
    app.handle(event);  // &mut App — exclusive access
}

// ❌ Bad
tokio::spawn(async {
    // Worker thread directly mutating state
    state.files.push(new_file);  // Data race!
});
```

### Principle 3: Background Workers Only Send Mail

Workers do not touch state. They send messages through a channel.

```rust
// ✅ Good
tokio::spawn(async move {
    let result = do_work().await;
    tx.send(Event::WorkDone(result)).ok();
});

// ❌ Bad
tokio::spawn(async move {
    state.files = do_work().await;  // Data race!
});
```

### Principle 4: Everything is an Event

Keys, mouse clicks, timer ticks, worker completions, errors — everything becomes an `Event`.

```rust
enum Event {
    Key(KeyEvent),
    TimerTick,
    GitStatusFetched(String),
    Error(String),
}
```

### Principle 5: Screens are a Stack

Only the topmost visible screen handles input. Screens render back-to-front (painter's algorithm).

```
Which (chord hints)
  ↓
Help (keymap overlay)
  ↓
CommitDialog (text input)
  ↓
Main (file list)
```

---

## Prerequisites

Add these dependencies to `Cargo.toml`:

```toml
[dependencies]
crossterm = { version = "0.28", features = ["event-stream"] }
ratatui = "0.29"
tokio = { version = "1.42", features = ["full"] }
anyhow = "1.0"
once_cell = "1.20"
```

---

## Step 0: Project Setup

```bash
cargo new gitz
cd gitz
```

`Cargo.toml`:
```toml
[package]
name = "gitz"
version = "0.1.0"
edition = "2021"

[dependencies]
crossterm = { version = "0.28", features = ["event-stream"] }
ratatui = "0.29"
tokio = { version = "1.42", features = ["full"] }
anyhow = "1.0"
once_cell = "1.20"
```

`src/main.rs`:
```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let mut app = App::new()?;
    app.run().await
}
```

---

## Step 1: The Single Source of Truth

Define your entire application state in one place.

```rust
// src/state.rs
use ratatui::widgets::{ListState, TableState};

/// All application state lives here.
pub struct State {
    // --- Main screen state ---
    pub repos: Vec<Repo>,
    pub repo_list_state: ListState,
    pub selected_repo: usize,
    
    // --- Git status for selected repo ---
    pub status_items: Vec<StatusItem>,
    pub status_table_state: TableState,
    pub selected_status: usize,
    
    // --- Modals / Overlays ---
    pub commit_dialog: CommitDialog,
    pub branch_picker: BranchPicker,
    pub help_visible: bool,
    
    // --- Notifications ---
    pub notification: Option<String>,
    pub error_message: Option<String>,
}

#[derive(Clone)]
pub struct Repo {
    pub name: String,
    pub path: std::path::PathBuf,
    pub branch: String,
    pub ahead: usize,
    pub behind: usize,
}

#[derive(Clone)]
pub struct StatusItem {
    pub status: char,  // M, A, D, ?, etc.
    pub path: String,
    pub additions: i32,
    pub deletions: i32,
}

#[derive(Default)]
pub struct CommitDialog {
    pub visible: bool,
    pub text: String,
    pub cursor: usize,
}

#[derive(Default)]
pub struct BranchPicker {
    pub visible: bool,
    pub branches: Vec<String>,
    pub selected: usize,
    pub filter: String,
}

impl Default for State {
    fn default() -> Self {
        Self {
            repos: Vec::new(),
            repo_list_state: ListState::default(),
            selected_repo: 0,
            status_items: Vec::new(),
            status_table_state: TableState::default(),
            selected_status: 0,
            commit_dialog: CommitDialog::default(),
            branch_picker: BranchPicker::default(),
            help_visible: false,
            notification: None,
            error_message: None,
        }
    }
}
```

**Why one struct?**
- Easy to pass around: `&mut State`
- Easy to serialize for debugging
- Easy to reset: `self.state = State::default()`
- The borrow checker can track all mutations

---

## Step 2: The Event Loop

This is the heart of your application. It owns `State` and never lets go.

```rust
// src/app.rs
use std::time::Duration;
use tokio::sync::mpsc;

pub struct App {
    state: State,
    terminal: ratatui::DefaultTerminal,
    needs_render: bool,
}

impl App {
    pub fn new() -> anyhow::Result<Self> {
        let terminal = ratatui::init();
        Ok(Self {
            state: State::default(),
            terminal,
            needs_render: true,
        })
    }

    pub async fn run(&mut self) -> anyhow::Result<()> {
        // Create the event channel
        let (tx, mut rx) = mpsc::unbounded_channel::<Event>();
        
        // Store sender globally so ANYONE can emit events
        EVENT_TX.set(tx).map_err(|_| anyhow::anyhow!("Event tx already set"))?;
        
        // Start background event reader (keys, resize, etc.)
        spawn_input_reader();
        
        // Initial render
        self.render()?;
        
        // THE MAIN LOOP
        loop {
            tokio::select! {
                // Process incoming events
                Some(event) = rx.recv() => {
                    if !self.handle(event)? {
                        break;  // App wants to quit
                    }
                }
                
                // Periodic render (for animations, blinking cursor, etc.)
                _ = tokio::time::sleep(Duration::from_millis(50)) => {
                    if self.needs_render {
                        self.render()?;
                    }
                }
            }
        }
        
        ratatui::restore();
        Ok(())
    }
}
```

**Key insight:** `self` is `&mut App`. The entire loop holds exclusive mutable access. Nothing else can touch `self.state` while this function runs.

---

## Step 3: The Event Bus

Define every possible thing that can happen as an `Event`.

```rust
// src/event.rs
use crossterm::event::KeyEvent;

/// Every possible thing that can happen in the app.
#[derive(Debug, Clone)]
pub enum Event {
    // User input
    Key(KeyEvent),
    Mouse(crossterm::event::MouseEvent),
    Resize(u16, u16),
    Paste(String),
    
    // Git operations completed
    GitStatusFetched { repo_idx: usize, output: String },
    GitBranchesFetched(Vec<String>),
    GitCommitDone(Result<(), String>),
    
    // UI actions
    ShowCommitDialog,
    HideCommitDialog,
    ShowBranchPicker,
    HideBranchPicker,
    ShowHelp,
    HideHelp,
    SetNotification(String),
    ClearNotification,
    
    // Navigation
    NextRepo,
    PrevRepo,
    NextStatus,
    PrevStatus,
    StageSelected,
    UnstageSelected,
    
    // App lifecycle
    Refresh,
    Quit,
}
```

**Global sender** so any code can emit events:

```rust
// src/event.rs
use once_cell::sync::OnceCell;
use tokio::sync::mpsc::UnboundedSender;

static EVENT_TX: OnceCell<UnboundedSender<Event>> = OnceCell::new();

pub fn emit(event: Event) {
    if let Some(tx) = EVENT_TX.get() {
        let _ = tx.send(event);
    }
}

pub fn emit_key(key: KeyEvent) {
    emit(Event::Key(key));
}
```

**Input reader** running in a background task:

```rust
// src/input.rs
use crossterm::event::{self, Event as CrosstermEvent};
use std::time::Duration;

pub fn spawn_input_reader() {
    tokio::spawn(async move {
        loop {
            // Check for events with a timeout so we don't block forever
            if event::poll(Duration::from_millis(100)).unwrap_or(false) {
                if let Ok(CrosstermEvent::Key(key)) = event::read() {
                    crate::event::emit_key(key);
                }
            }
        }
    });
}
```

**Why a global sender?**
- Background workers can send events without holding `&mut App`
- Plugins can emit events
- Any async task can notify the UI

---

## Step 4: Layered Screens

Determine which screen is active based on visible flags.

```rust
// src/layer.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Layer {
    Main,
    Commit,
    BranchPicker,
    Help,
}

impl App {
    fn active_layer(&self) -> Layer {
        if self.state.commit_dialog.visible {
            Layer::Commit
        } else if self.state.branch_picker.visible {
            Layer::BranchPicker
        } else if self.state.help_visible {
            Layer::Help
        } else {
            Layer::Main
        }
    }
}
```

**Priority matters:** The first `true` wins. `Commit` is checked before `Help` because a commit dialog should block help.

---

## Step 5: Command Dispatch

Handle events by delegating to the right handler based on the active layer.

```rust
// src/app.rs
impl App {
    /// Returns true to continue running, false to quit.
    fn handle(&mut self, event: Event) -> anyhow::Result<bool> {
        match event {
            // App lifecycle
            Event::Quit => return Ok(false),
            Event::Refresh => self.refresh_all_repos(),
            
            // Let the active layer handle it first
            Event::Key(key) => self.handle_key(key),
            Event::Mouse(mouse) => self.handle_mouse(mouse),
            
            // Git operation results
            Event::GitStatusFetched { repo_idx, output } => {
                self.parse_git_status(repo_idx, &output);
            }
            Event::GitBranchesFetched(branches) => {
                self.state.branch_picker.branches = branches;
            }
            Event::GitCommitDone(result) => {
                self.state.commit_dialog.visible = false;
                match result {
                    Ok(()) => self.set_notification("Committed!"),
                    Err(e) => self.state.error_message = Some(e),
                }
            }
            
            // UI actions
            Event::ShowCommitDialog => self.show_commit_dialog(),
            Event::HideCommitDialog => self.state.commit_dialog.visible = false,
            Event::ShowBranchPicker => self.show_branch_picker(),
            Event::HideBranchPicker => self.state.branch_picker.visible = false,
            Event::ShowHelp => self.state.help_visible = true,
            Event::HideHelp => self.state.help_visible = false,
            Event::SetNotification(msg) => self.set_notification(msg),
            Event::ClearNotification => self.state.notification = None,
            
            // Navigation
            Event::NextRepo => self.next_repo(),
            Event::PrevRepo => self.prev_repo(),
            Event::NextStatus => self.next_status(),
            Event::PrevStatus => self.prev_status(),
            Event::StageSelected => self.stage_selected(),
            Event::UnstageSelected => self.unstage_selected(),
            
            _ => {}
        }
        
        self.needs_render = true;
        Ok(true)
    }
}
```

---

## Step 6: Background Workers

Never block the UI loop. Spawn tasks for anything slow.

```rust
// src/git_worker.rs
use tokio::process::Command;
use std::path::PathBuf;

/// Fetch git status for a repo.
pub fn fetch_git_status(repo_idx: usize, repo_path: PathBuf) {
    tokio::spawn(async move {
        let output = Command::new("git")
            .args(&["status", "--porcelain", "-b", "--show-stash"])
            .current_dir(&repo_path)
            .output()
            .await;
            
        match output {
            Ok(o) => {
                let text = String::from_utf8_lossy(&o.stdout).to_string();
                crate::event::emit(Event::GitStatusFetched {
                    repo_idx,
                    output: text,
                });
            }
            Err(e) => {
                crate::event::emit(Event::Error(format!(
                    "git status failed for {:?}: {}", repo_path, e
                )));
            }
        }
    });
}

/// Fetch branch list.
pub fn fetch_branches(repo_path: PathBuf) {
    tokio::spawn(async move {
        let output = Command::new("git")
            .args(&["branch", "-a", "--format=%(refname:short)"])
            .current_dir(&repo_path)
            .output()
            .await;
            
        match output {
            Ok(o) => {
                let branches: Vec<String> = String::from_utf8_lossy(&o.stdout)
                    .lines()
                    .map(String::from)
                    .collect();
                crate::event::emit(Event::GitBranchesFetched(branches));
            }
            Err(e) => {
                crate::event::emit(Event::Error(format!("git branch failed: {}", e)));
            }
        }
    });
}

/// Commit with a message.
pub fn commit(repo_path: PathBuf, message: String) {
    tokio::spawn(async move {
        let result = Command::new("git")
            .args(&["commit", "-m", &message])
            .current_dir(&repo_path)
            .output()
            .await;
            
        match result {
            Ok(o) if o.status.success() => {
                crate::event::emit(Event::GitCommitDone(Ok(())));
            }
            Ok(o) => {
                let stderr = String::from_utf8_lossy(&o.stderr);
                crate::event::emit(Event::GitCommitDone(Err(stderr.to_string())));
            }
            Err(e) => {
                crate::event::emit(Event::GitCommitDone(Err(e.to_string())));
            }
        }
    });
}
```

**Critical rule:** These functions never touch `State`. They only emit events.

---

## Step 7: Rendering

Draw everything, back to front.

```rust
// src/render.rs
use ratatui::{
    layout::{Constraint, Direction, Layout, Rect},
    style::{Color, Style},
    widgets::{Block, Borders, Clear, List, ListItem, Paragraph, Row, Table, Wrap},
    Frame,
};

impl App {
    fn render(&mut self) -> anyhow::Result<()> {
        self.terminal.draw(|frame| {
            let area = frame.area();
            
            // 1. Draw main layout
            let chunks = Layout::default()
                .direction(Direction::Vertical)
                .constraints([
                    Constraint::Length(3),   // Header
                    Constraint::Min(10),     // Main content
                    Constraint::Length(3),   // Status bar
                ])
                .split(area);
            
            self.draw_header(frame, chunks[0]);
            self.draw_main_content(frame, chunks[1]);
            self.draw_status_bar(frame, chunks[2]);
            
            // 2. Draw overlays, back to front
            if self.state.branch_picker.visible {
                self.draw_branch_picker(frame, area);
            }
            if self.state.commit_dialog.visible {
                self.draw_commit_dialog(frame, area);
            }
            if self.state.help_visible {
                self.draw_help(frame, area);
            }
            if let Some(ref notif) = self.state.notification {
                self.draw_notification(frame, area, notif);
            }
        })?;
        
        self.needs_render = false;
        Ok(())
    }
    
    fn draw_header(&self, frame: &mut Frame, area: Rect) {
        let title = format!(" Gitz — {} repos ", self.state.repos.len());
        let header = Paragraph::new(title)
            .block(Block::default().borders(Borders::ALL));
        frame.render_widget(header, area);
    }
    
    fn draw_main_content(&mut self, frame: &mut Frame, area: Rect) {
        let chunks = Layout::default()
            .direction(Direction::Horizontal)
            .constraints([Constraint::Percentage(40), Constraint::Percentage(60)])
            .split(area);
        
        // Left: repo list
        let items: Vec<ListItem> = self.state.repos.iter()
            .map(|r| {
                let line = format!("{} [{} ↑{}↓{}]", r.name, r.branch, r.ahead, r.behind);
                ListItem::new(line)
            })
            .collect();
        let list = List::new(items)
            .block(Block::default().title("Repos").borders(Borders::ALL))
            .highlight_style(Style::default().bg(Color::Blue));
        frame.render_stateful_widget(list, chunks[0], &mut self.state.repo_list_state);
        
        // Right: status table
        let rows: Vec<Row> = self.state.status_items.iter()
            .map(|item| {
                Row::new(vec![
                    item.status.to_string(),
                    item.path.clone(),
                    format!("+{}/-{}", item.additions, item.deletions),
                ])
            })
            .collect();
        let table = Table::new(rows, [
            Constraint::Length(3),
            Constraint::Percentage(70),
            Constraint::Length(12),
        ])
        .header(Row::new(vec!["S", "File", "Δ"]).style(Style::default().fg(Color::Yellow)))
        .block(Block::default().title("Status").borders(Borders::ALL))
        .highlight_style(Style::default().bg(Color::Blue));
        frame.render_stateful_widget(table, chunks[1], &mut self.state.status_table_state);
    }
    
    fn draw_commit_dialog(&self, frame: &mut Frame, area: Rect) {
        // Center the dialog
        let popup_area = centered_rect(60, 20, area);
        
        // Clear background
        frame.render_widget(Clear, popup_area);
        
        let block = Block::default()
            .title("Commit")
            .borders(Borders::ALL);
        
        let text = format!("{}", self.state.commit_dialog.text);
        let input = Paragraph::new(text)
            .block(block)
            .wrap(Wrap { trim: true });
        frame.render_widget(input, popup_area);
        
        // Set cursor position
        let x = popup_area.x + 1 + self.state.commit_dialog.cursor as u16;
        let y = popup_area.y + 1;
        frame.set_cursor_position((x, y));
    }
    
    fn draw_branch_picker(&self, frame: &mut Frame, area: Rect) {
        let popup_area = centered_rect(50, 60, area);
        frame.render_widget(Clear, popup_area);
        
        let items: Vec<ListItem> = self.state.branch_picker.branches.iter()
            .map(|b| ListItem::new(b.as_str()))
            .collect();
        let list = List::new(items)
            .block(Block::default().title("Branches").borders(Borders::ALL));
        frame.render_widget(list, popup_area);
    }
    
    fn draw_help(&self, frame: &mut Frame, area: Rect) {
        let popup_area = centered_rect(70, 80, area);
        frame.render_widget(Clear, popup_area);
        
        let text = "j/k: navigate | s: stage | u: unstage | c: commit | b: branches | q: quit";
        let help = Paragraph::new(text)
            .block(Block::default().title("Help").borders(Borders::ALL))
            .wrap(Wrap { trim: true });
        frame.render_widget(help, popup_area);
    }
    
    fn draw_notification(&self, frame: &mut Frame, area: Rect, msg: &str) {
        let notif_area = Rect {
            x: area.x + area.width.saturating_sub(30),
            y: area.y + area.height.saturating_sub(3),
            width: 30,
            height: 3,
        };
        let notif = Paragraph::new(msg)
            .block(Block::default().borders(Borders::ALL))
            .style(Style::default().fg(Color::Green));
        frame.render_widget(notif, notif_area);
    }
}

/// Helper: create a centered rect with given percentage of screen
fn centered_rect(percent_x: u16, percent_y: u16, r: Rect) -> Rect {
    let popup_layout = Layout::default()
        .direction(Direction::Vertical)
        .constraints([
            Constraint::Percentage((100 - percent_y) / 2),
            Constraint::Percentage(percent_y),
            Constraint::Percentage((100 - percent_y) / 2),
        ])
        .split(r);
    
    Layout::default()
        .direction(Direction::Horizontal)
        .constraints([
            Constraint::Percentage((100 - percent_x) / 2),
            Constraint::Percentage(percent_x),
            Constraint::Percentage((100 - percent_x) / 2),
        ])
        .split(popup_layout[1])[1]
}
```

---

## Step 8: Input Routing

Route keys to the right handler based on active layer.

```rust
// src/app.rs
impl App {
    fn handle_key(&mut self, key: crossterm::event::KeyEvent) {
        // Layer-specific input
        match self.active_layer() {
            Layer::Commit => {
                if self.handle_commit_key(key) {
                    return;  // Commit dialog consumed the key
                }
            }
            Layer::BranchPicker => {
                if self.handle_branch_key(key) {
                    return;
                }
            }
            Layer::Help => {
                if key.code == crossterm::event::KeyCode::Esc {
                    self.state.help_visible = false;
                }
                return;
            }
            Layer::Main => {}  // Fall through to main keymap
        }
        
        // Main keymap
        match key.code {
            crossterm::event::KeyCode::Char('q') => {
                emit(Event::Quit);
            }
            crossterm::event::KeyCode::Char('j') | crossterm::event::KeyCode::Down => {
                emit(Event::NextRepo);
            }
            crossterm::event::KeyCode::Char('k') | crossterm::event::KeyCode::Up => {
                emit(Event::PrevRepo);
            }
            crossterm::event::KeyCode::Char('s') => {
                emit(Event::StageSelected);
            }
            crossterm::event::KeyCode::Char('u') => {
                emit(Event::UnstageSelected);
            }
            crossterm::event::KeyCode::Char('c') => {
                emit(Event::ShowCommitDialog);
            }
            crossterm::event::KeyCode::Char('b') => {
                emit(Event::ShowBranchPicker);
            }
            crossterm::event::KeyCode::Char('?') | crossterm::event::KeyCode::Char('h') => {
                emit(Event::ShowHelp);
            }
            crossterm::event::KeyCode::Char('r') => {
                emit(Event::Refresh);
            }
            _ => {}
        }
    }
    
    fn handle_commit_key(&mut self, key: crossterm::event::KeyEvent) -> bool {
        match key.code {
            crossterm::event::KeyCode::Esc => {
                emit(Event::HideCommitDialog);
                true
            }
            crossterm::event::KeyCode::Enter => {
                let msg = self.state.commit_dialog.text.clone();
                let path = self.current_repo_path();
                crate::git_worker::commit(path, msg);
                self.state.commit_dialog.text.clear();
                self.state.commit_dialog.cursor = 0;
                true
            }
            crossterm::event::KeyCode::Char(c) => {
                let cursor = self.state.commit_dialog.cursor;
                self.state.commit_dialog.text.insert(cursor, c);
                self.state.commit_dialog.cursor += 1;
                true
            }
            crossterm::event::KeyCode::Backspace => {
                let cursor = self.state.commit_dialog.cursor;
                if cursor > 0 {
                    self.state.commit_dialog.text.remove(cursor - 1);
                    self.state.commit_dialog.cursor -= 1;
                }
                true
            }
            crossterm::event::KeyCode::Left => {
                if self.state.commit_dialog.cursor > 0 {
                    self.state.commit_dialog.cursor -= 1;
                }
                true
            }
            crossterm::event::KeyCode::Right => {
                if self.state.commit_dialog.cursor < self.state.commit_dialog.text.len() {
                    self.state.commit_dialog.cursor += 1;
                }
                true
            }
            _ => false,  // Let it fall through
        }
    }
    
    fn handle_branch_key(&mut self, key: crossterm::event::KeyEvent) -> bool {
        match key.code {
            crossterm::event::KeyCode::Esc => {
                emit(Event::HideBranchPicker);
                true
            }
            crossterm::event::KeyCode::Enter => {
                // Checkout selected branch
                if let Some(branch) = self.state.branch_picker.branches.get(
                    self.state.branch_picker.selected
                ) {
                    let path = self.current_repo_path();
                    let branch = branch.clone();
                    tokio::spawn(async move {
                        let _ = tokio::process::Command::new("git")
                            .args(&["checkout", &branch])
                            .current_dir(&path)
                            .output()
                            .await;
                        emit(Event::HideBranchPicker);
                        emit(Event::Refresh);
                    });
                }
                true
            }
            crossterm::event::KeyCode::Char('j') | crossterm::event::KeyCode::Down => {
                let max = self.state.branch_picker.branches.len().saturating_sub(1);
                if self.state.branch_picker.selected < max {
                    self.state.branch_picker.selected += 1;
                }
                true
            }
            crossterm::event::KeyCode::Char('k') | crossterm::event::KeyCode::Up => {
                if self.state.branch_picker.selected > 0 {
                    self.state.branch_picker.selected -= 1;
                }
                true
            }
            _ => false,
        }
    }
}
```

---

## Step 9: Putting It All Together

Here's the complete `App` implementation with all the pieces connected:

```rust
// src/app.rs (complete)
use crate::{event::*, git_worker, layer::Layer, render::*, state::*};
use anyhow::Result;
use crossterm::event::KeyEvent;
use ratatui::DefaultTerminal;
use std::time::Duration;
use tokio::sync::mpsc;

pub struct App {
    state: State,
    terminal: DefaultTerminal,
    needs_render: bool,
}

impl App {
    pub fn new() -> Result<Self> {
        let mut state = State::default();
        
        // Load some initial data
        state.repos = vec![
            Repo {
                name: "my-project".into(),
                path: std::path::PathBuf::from("/home/user/my-project"),
                branch: "main".into(),
                ahead: 2,
                behind: 1,
            },
        ];
        
        if !state.repos.is_empty() {
            state.repo_list_state.select(Some(0));
        }
        
        Ok(Self {
            state,
            terminal: ratatui::init(),
            needs_render: true,
        })
    }

    pub async fn run(&mut self) -> Result<()> {
        let (tx, mut rx) = mpsc::unbounded_channel::<Event>();
        EVENT_TX.set(tx).map_err(|_| anyhow::anyhow!("Event tx already set"))?;
        
        crate::input::spawn_input_reader();
        
        // Fetch initial status
        if let Some(repo) = self.state.repos.first() {
            git_worker::fetch_git_status(0, repo.path.clone());
        }
        
        self.render()?;
        
        loop {
            tokio::select! {
                Some(event) = rx.recv() => {
                    if !self.handle(event)? {
                        break;
                    }
                }
                _ = tokio::time::sleep(Duration::from_millis(50)) => {
                    if self.needs_render {
                        self.render()?;
                    }
                }
            }
        }
        
        ratatui::restore();
        Ok(())
    }

    fn handle(&mut self, event: Event) -> Result<bool> {
        match event {
            Event::Quit => return Ok(false),
            Event::Refresh => self.refresh_all_repos(),
            Event::Key(key) => self.handle_key(key),
            Event::GitStatusFetched { repo_idx, output } => {
                self.parse_git_status(repo_idx, &output);
            }
            Event::GitBranchesFetched(branches) => {
                self.state.branch_picker.branches = branches;
            }
            Event::GitCommitDone(result) => {
                self.state.commit_dialog.visible = false;
                match result {
                    Ok(()) => self.set_notification("Committed successfully!"),
                    Err(e) => self.state.error_message = Some(e),
                }
                self.refresh_current_repo();
            }
            Event::ShowCommitDialog => self.show_commit_dialog(),
            Event::HideCommitDialog => self.state.commit_dialog.visible = false,
            Event::ShowBranchPicker => self.show_branch_picker(),
            Event::HideBranchPicker => self.state.branch_picker.visible = false,
            Event::ShowHelp => self.state.help_visible = true,
            Event::HideHelp => self.state.help_visible = false,
            Event::SetNotification(msg) => self.set_notification(msg),
            Event::ClearNotification => self.state.notification = None,
            Event::NextRepo => self.next_repo(),
            Event::PrevRepo => self.prev_repo(),
            Event::NextStatus => self.next_status(),
            Event::PrevStatus => self.prev_status(),
            Event::StageSelected => self.stage_selected(),
            Event::UnstageSelected => self.unstage_selected(),
            Event::Error(e) => self.state.error_message = Some(e),
            _ => {}
        }
        
        self.needs_render = true;
        Ok(true)
    }

    // --- Navigation ---
    
    fn next_repo(&mut self) {
        let max = self.state.repos.len().saturating_sub(1);
        if self.state.selected_repo < max {
            self.state.selected_repo += 1;
            self.state.repo_list_state.select(Some(self.state.selected_repo));
            self.refresh_current_repo();
        }
    }
    
    fn prev_repo(&mut self) {
        if self.state.selected_repo > 0 {
            self.state.selected_repo -= 1;
            self.state.repo_list_state.select(Some(self.state.selected_repo));
            self.refresh_current_repo();
        }
    }
    
    fn next_status(&mut self) {
        let max = self.state.status_items.len().saturating_sub(1);
        if self.state.selected_status < max {
            self.state.selected_status += 1;
            self.state.status_table_state.select(Some(self.state.selected_status));
        }
    }
    
    fn prev_status(&mut self) {
        if self.state.selected_status > 0 {
            self.state.selected_status -= 1;
            self.state.status_table_state.select(Some(self.state.selected_status));
        }
    }
    
    // --- Git operations ---
    
    fn stage_selected(&mut self) {
        if let Some(item) = self.state.status_items.get(self.state.selected_status) {
            let path = self.current_repo_path();
            let file = item.path.clone();
            tokio::spawn(async move {
                let _ = tokio::process::Command::new("git")
                    .args(&["add", &file])
                    .current_dir(&path)
                    .output()
                    .await;
                emit(Event::Refresh);
            });
        }
    }
    
    fn unstage_selected(&mut self) {
        if let Some(item) = self.state.status_items.get(self.state.selected_status) {
            let path = self.current_repo_path();
            let file = item.path.clone();
            tokio::spawn(async move {
                let _ = tokio::process::Command::new("git")
                    .args(&["restore", "--staged", &file])
                    .current_dir(&path)
                    .output()
                    .await;
                emit(Event::Refresh);
            });
        }
    }
    
    fn refresh_current_repo(&mut self) {
        if let Some(repo) = self.state.repos.get(self.state.selected_repo) {
            git_worker::fetch_git_status(self.state.selected_repo, repo.path.clone());
        }
    }
    
    fn refresh_all_repos(&mut self) {
        for (idx, repo) in self.state.repos.iter().enumerate() {
            git_worker::fetch_git_status(idx, repo.path.clone());
        }
    }
    
    fn parse_git_status(&mut self, repo_idx: usize, output: &str) {
        // Parse git status --porcelain output
        let mut items = Vec::new();
        for line in output.lines() {
            if line.len() < 3 { continue; }
            let status = line.chars().next().unwrap_or(' ');
            let path = line[3..].to_string();
            items.push(StatusItem {
                status,
                path,
                additions: 0,  // Would parse from diff --stat
                deletions: 0,
            });
        }
        self.state.status_items = items;
        if repo_idx == self.state.selected_repo {
            self.state.status_table_state.select(Some(0));
            self.state.selected_status = 0;
        }
    }
    
    // --- Modals ---
    
    fn show_commit_dialog(&mut self) {
        self.state.commit_dialog.visible = true;
        self.state.commit_dialog.text.clear();
        self.state.commit_dialog.cursor = 0;
    }
    
    fn show_branch_picker(&mut self) {
        self.state.branch_picker.visible = true;
        self.state.branch_picker.selected = 0;
        if let Some(repo) = self.state.repos.get(self.state.selected_repo) {
            git_worker::fetch_branches(repo.path.clone());
        }
    }
    
    // --- Helpers ---
    
    fn current_repo_path(&self) -> std::path::PathBuf {
        self.state.repos.get(self.state.selected_repo)
            .map(|r| r.path.clone())
            .unwrap_or_default()
    }
    
    fn set_notification(&mut self, msg: String) {
        self.state.notification = Some(msg);
        // Auto-clear after 3 seconds
        let tx = EVENT_TX.get().unwrap().clone();
        tokio::spawn(async move {
            tokio::time::sleep(Duration::from_secs(3)).await;
            let _ = tx.send(Event::ClearNotification);
        });
    }
    
    fn active_layer(&self) -> Layer {
        if self.state.commit_dialog.visible { Layer::Commit }
        else if self.state.branch_picker.visible { Layer::BranchPicker }
        else if self.state.help_visible { Layer::Help }
        else { Layer::Main }
    }
}
```

---

## Common Patterns & Recipes

### Pattern 1: Debounced Refresh

Don't refresh on every keystroke. Batch refreshes:

```rust
use std::time::{Duration, Instant};

pub struct App {
    last_refresh: Option<Instant>,
    pending_refresh: bool,
}

fn request_refresh(&mut self) {
    self.pending_refresh = true;
}

fn maybe_refresh(&mut self) {
    if self.pending_refresh {
        if self.last_refresh.map_or(true, |t| t.elapsed() > Duration::from_millis(100)) {
            self.do_refresh();
            self.last_refresh = Some(Instant::now());
            self.pending_refresh = false;
        }
    }
}
```

### Pattern 2: Loading State

Show a spinner while work is happening:

```rust
#[derive(Default)]
pub struct LoadingState {
    pub active: bool,
    pub message: String,
    pub spinner_frame: usize,
}

impl App {
    fn draw_loading(&self, frame: &mut Frame, area: Rect) {
        if !self.state.loading.active { return; }
        let spinner = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"];
        let frame = spinner[self.state.loading.spinner_frame % spinner.len()];
        let text = format!("{} {}...", frame, self.state.loading.message);
        // Draw...
    }
}
```

### Pattern 3: Confirmation Dialog

Reusable yes/no dialog:

```rust
#[derive(Default)]
pub struct ConfirmDialog {
    pub visible: bool,
    pub message: String,
    pub on_yes: Option<Event>,  // What to emit when user confirms
}

// Usage:
fn delete_file(&mut self) {
    self.state.confirm = ConfirmDialog {
        visible: true,
        message: "Delete this file?".into(),
        on_yes: Some(Event::DeleteConfirmed),
    };
}
```

### Pattern 4: Search/Filter

Filter a list as the user types:

```rust
pub struct FilterableList<T> {
    pub all_items: Vec<T>,
    pub filtered: Vec<T>,
    pub filter_text: String,
    pub selected: usize,
}

impl<T: Clone> FilterableList<T> {
    pub fn update_filter(&mut self) {
        self.filtered = if self.filter_text.is_empty() {
            self.all_items.clone()
        } else {
            self.all_items.iter()
                .filter(|item| item.matches(&self.filter_text))
                .cloned()
                .collect()
        };
        self.selected = 0;
    }
}
```

### Pattern 5: Async Command Queue

Queue multiple git operations to run sequentially:

```rust
use tokio::sync::mpsc;

pub struct CommandQueue {
    tx: mpsc::UnboundedSender<GitCommand>,
}

pub enum GitCommand {
    Add(String),
    Commit(String),
    Push,
}

// Background task processes queue:
tokio::spawn(async move {
    while let Some(cmd) = rx.recv().await {
        match cmd {
            GitCommand::Add(file) => { /* git add */ }
            GitCommand::Commit(msg) => { /* git commit */ }
            GitCommand::Push => { /* git push */ }
        }
        // Emit event after each command
        emit(Event::CommandDone);
    }
});
```

---

## Anti-Patterns to Avoid

### ❌ Don't Use `Arc<Mutex<State>>`

```rust
// BAD: Shared mutable state across threads
static STATE: Lazy<Arc<Mutex<State>>> = Lazy::new(|| Arc::new(Mutex::new(State::new())));

// WHY: You'll deadlock. You'll poison the lock. You'll have race conditions.
// The borrow checker can't help you anymore.
```

### ❌ Don't Block the Event Loop

```rust
// BAD: Blocking the UI thread
fn handle(&mut self, event: Event) {
    match event {
        Event::Refresh => {
            let output = std::process::Command::new("git")
                .arg("status")
                .output()
                .unwrap();  // ← BLOCKS UI FOR 50ms+!
        }
    }
}

// GOOD: Spawn a task
fn handle(&mut self, event: Event) {
    match event {
        Event::Refresh => {
            let path = self.current_repo_path();
            tokio::spawn(async move {
                let output = tokio::process::Command::new("git")
                    .arg("status")
                    .current_dir(path)
                    .output()
                    .await;
                // Send result back via event
            });
        }
    }
}
```

### ❌ Don't Mutate State from Background Tasks

```rust
// BAD: Worker thread directly touching state
tokio::spawn(async move {
    let result = do_work().await;
    STATE.lock().unwrap().data = result;  // ← DATA RACE!
});

// GOOD: Send event to UI loop
tokio::spawn(async move {
    let result = do_work().await;
    emit(Event::WorkDone(result));
});
```

### ❌ Don't Call `render()` from Everywhere

```rust
// BAD: Scattered render calls
fn on_data_loaded(&mut self) {
    self.state.items = new_items;
    self.render().unwrap();  // ← Who else is rendering? Race condition risk.
}

// GOOD: Set a flag, let the loop handle it
fn on_data_loaded(&mut self) {
    self.state.items = new_items;
    self.needs_render = true;  // ← Loop will render on next tick
}
```

### ❌ Don't Use Multiple Sources of Truth

```rust
// BAD: State scattered everywhere
struct Sidebar { items: Vec<String> }
struct MainPanel { items: Vec<String> }
// Which one is correct? They can get out of sync.

// GOOD: One state, multiple views
struct State {
    items: Vec<String>,  // Single source of truth
}
// Sidebar and MainPanel both read from State.items
```

---

## Full Working Example

Here's a minimal but complete `main.rs` you can compile and run:

```rust
// src/main.rs
use crossterm::event::{self, Event as CrosstermEvent, KeyCode, KeyEvent};
use ratatui::{
    layout::{Constraint, Direction, Layout},
    style::{Color, Style},
    widgets::{Block, Borders, List, ListItem, Paragraph},
    DefaultTerminal,
};
use std::time::Duration;
use tokio::sync::mpsc;

static EVENT_TX: once_cell::sync::OnceCell<mpsc::UnboundedSender<Event>> = 
    once_cell::sync::OnceCell::new();

#[derive(Debug, Clone)]
enum Event {
    Key(KeyEvent),
    Tick,
    Quit,
}

fn emit(event: Event) {
    if let Some(tx) = EVENT_TX.get() {
        let _ = tx.send(event);
    }
}

struct App {
    items: Vec<String>,
    selected: usize,
    terminal: DefaultTerminal,
    needs_render: bool,
}

impl App {
    fn new() -> Self {
        Self {
            items: vec![
                "Item 1".into(),
                "Item 2".into(),
                "Item 3".into(),
            ],
            selected: 0,
            terminal: ratatui::init(),
            needs_render: true,
        }
    }

    async fn run(&mut self) -> anyhow::Result<()> {
        let (tx, mut rx) = mpsc::unbounded_channel();
        EVENT_TX.set(tx).unwrap();

        // Spawn input reader
        tokio::spawn(async move {
            loop {
                if event::poll(Duration::from_millis(100)).unwrap_or(false) {
                    if let Ok(CrosstermEvent::Key(key)) = event::read() {
                        emit(Event::Key(key));
                    }
                }
            }
        });

        self.render()?;

        loop {
            tokio::select! {
                Some(event) = rx.recv() => {
                    match event {
                        Event::Quit => break,
                        Event::Key(key) => self.handle_key(key),
                        Event::Tick => {}
                    }
                    if self.needs_render {
                        self.render()?;
                    }
                }
                _ = tokio::time::sleep(Duration::from_millis(50)) => {
                    if self.needs_render {
                        self.render()?;
                    }
                }
            }
        }

        ratatui::restore();
        Ok(())
    }

    fn handle_key(&mut self, key: KeyEvent) {
        match key.code {
            KeyCode::Char('q') => emit(Event::Quit),
            KeyCode::Char('j') | KeyCode::Down => {
                if self.selected < self.items.len().saturating_sub(1) {
                    self.selected += 1;
                    self.needs_render = true;
                }
            }
            KeyCode::Char('k') | KeyCode::Up => {
                if self.selected > 0 {
                    self.selected -= 1;
                    self.needs_render = true;
                }
            }
            _ => {}
        }
    }

    fn render(&mut self) -> anyhow::Result<()> {
        self.terminal.draw(|frame| {
            let area = frame.area();
            let items: Vec<ListItem> = self.items.iter()
                .map(|i| ListItem::new(i.as_str()))
                .collect();
            let list = List::new(items)
                .block(Block::default().title("Gitz Demo").borders(Borders::ALL))
                .highlight_style(Style::default().bg(Color::Blue));
            frame.render_stateful_widget(
                list, 
                area, 
                &mut ratatui::widgets::ListState::default().with_selected(Some(self.selected))
            );
        })?;
        self.needs_render = false;
        Ok(())
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let mut app = App::new();
    app.run().await
}
```

---

## Summary

| Principle | Implementation |
|-----------|---------------|
| **One struct owns everything** | `struct App { state: State, ... }` |
| **One loop processes everything** | `loop { select! { event => handle(event) } }` |
| **Workers only send mail** | `tokio::spawn` + `mpsc::unbounded_channel` |
| **Everything is an event** | `enum Event { Key, WorkDone, ... }` |
| **Screens are a stack** | `active_layer()` checks visible flags top-down |
| **Render back-to-front** | Draw overlays after base content |
| **Never block the loop** | Spawn tasks for I/O, emit events on completion |

Start with the minimal example above. Add one feature at a time:
1. ✅ Show a list
2. ✅ Navigate with j/k
3. ✅ Add a modal dialog
4. ✅ Spawn a background task
5. ✅ Parse command output
6. ✅ Add keybindings
7. ✅ Add multiple screens

You've now built a Yazi-style TUI. The architecture scales from a 100-line demo to a 10,000-line application because every piece follows the same pattern: **events flow in, state updates, screen redraws.**
