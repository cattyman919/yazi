# Building a Yazi-Style TUI with Ratatui

> **A practical guide to implementing Yazi's architecture patterns — layered screens, key routing, and state management — using the Ratatui library in Rust.**

---

## Table of Contents

1. [What is Ratatui?](#what-is-ratatui)
2. [How Yazi Uses Ratatui](#how-yazi-uses-ratatui)
3. [The Architecture Overview](#the-architecture-overview)
4. [Project Setup](#project-setup)
5. [The Event Loop with Crossterm + Ratatui](#the-event-loop-with-crossterm--ratatui)
6. [Centralized State Management](#centralized-state-management)
7. [The Layer System (Screens as a Stack)](#the-layer-system-screens-as-a-stack)
8. [Key Input Routing](#key-input-routing)
9. [Building Screens as Ratatui Widgets](#building-screens-as-ratatui-widgets)
10. [Modals and Popups](#modals-and-popups)
11. [The Painter's Algorithm in Practice](#the-painters-algorithm-in-practice)
12. [Background Workers + State Updates](#background-workers--state-updates)
13. [A Complete Working Example](#a-complete-working-example)
14. [Advanced Patterns](#advanced-patterns)
15. [Common Pitfalls](#common-pitfalls)

---

## What is Ratatui?

**Ratatui** is a Rust library for building rich terminal user interfaces. It provides:

- A **Buffer** — a 2D grid of cells (characters + styles)
- **Widgets** — reusable UI components (lists, tables, paragraphs, blocks, etc.)
- **Layout engine** — constraint-based layout (like CSS flexbox)
- **Frame** — a rendering context that manages the buffer

```
┌─────────────────────────────────────────┐
│  Ratatui Frame                          │
│  ┌─────────────────────────────────────┐│
│  │  Buffer (80x25 cells)              ││
│  │  ┌───┬───┬───┬───┬───┐             ││
│  │  │ H │ e │ l │ l │ o │  ← Cells   ││
│  │  └───┴───┴───┴───┴───┘             ││
│  │  Each cell: char, fg, bg, mods     ││
│  └─────────────────────────────────────┘│
└─────────────────────────────────────────┘
```

Unlike a web browser's DOM, Ratatui has **no retained widget tree**. You redraw everything every frame. This is actually perfect for a TUI because:

- Terminal screens are small (typically 80x25 = 2,000 cells)
- Full redraws are fast enough
- No need for complex diffing algorithms
- Simpler mental model: "draw what the state looks like right now"

---

## How Yazi Uses Ratatui

Yazi uses Ratatui as its **rendering backend**, but wraps it in its own architecture:

| Yazi Component | Ratatui Equivalent | Purpose |
|----------------|-------------------|---------|
| `yazi_term::Term` | `ratatui::DefaultTerminal` | Terminal initialization, raw mode, alternate screen |
| `yazi-fm/src/root.rs` | ` ratatui::Frame` | Top-level render function |
| `yazi-fm/src/mgr/` | Custom `Widget` impls | File list, preview panel |
| `yazi-fm/src/input/` | Custom `Widget` impls | Text input modal |
| `yazi-fm/src/help/` | Custom `Widget` impls | Help overlay |
| `yazi-fm/src/cmp/` | Custom `Widget` impls | Completion popup |
| `yazi-fm/src/which/` | Custom `Widget` impls | Key chord hint overlay |

Yazi's `Root::render()` is essentially a big function that calls various Ratatui widgets based on what's visible in `Core`.

---

## The Architecture Overview

Here's the full picture of how everything fits together:

```
┌─────────────────────────────────────────────────────────────────┐
│                         YOUR APPLICATION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐     │
│  │   Crossterm │─────►│  Event Bus  │─────►│   App Loop  │     │
│  │  (Keyboard) │      │  (mpsc)     │      │  (&mut App) │     │
│  └─────────────┘      └─────────────┘      └──────┬──────┘     │
│                                                    │             │
│                           ┌────────────────────────┘             │
│                           │                                      │
│                           ▼                                      │
│                    ┌─────────────┐                               │
│                    │   State     │                               │
│                    │  (&mut self)│                               │
│                    └──────┬──────┘                               │
│                           │                                      │
│                           ▼                                      │
│                    ┌─────────────┐                               │
│                    │  Router     │                               │
│                    │(which key?) │                               │
│                    └──────┬──────┘                               │
│                           │                                      │
│              ┌────────────┼────────────┐                        │
│              ▼            ▼            ▼                        │
│         ┌────────┐  ┌────────┐  ┌────────┐                     │
│         │Command │  │Command │  │Command │                     │
│         │  (j)   │  │  (k)   │  │  (c)   │                     │
│         └───┬────┘  └───┬────┘  └───┬────┘                     │
│             │           │           │                           │
│             └───────────┴───────────┘                           │
│                         │                                       │
│                         ▼                                       │
│                  ┌─────────────┐                                │
│                  │  State      │                                │
│                  │  Updated    │                                │
│                  └──────┬──────┘                                │
│                         │                                       │
│                         ▼                                       │
│                  ┌─────────────┐                                │
│                  │  Ratatui    │                                │
│                  │  Render     │                                │
│                  │  (Frame)    │                                │
│                  └──────┬──────┘                                │
│                         │                                       │
│                         ▼                                       │
│                  ┌─────────────┐                                │
│                  │  Terminal   │                                │
│                  │  (stdout)   │                                │
│                  └─────────────┘                                │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                      BACKGROUND WORKERS                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │ Git Cmd  │  │ Git Cmd  │  │ Git Cmd  │                       │
│  │ Worker 1 │  │ Worker 2 │  │ Worker 3 │                       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
│       │             │             │                              │
│       └─────────────┴─────────────┘                              │
│                     │                                            │
│                     ▼                                            │
│              ┌─────────────┐                                     │
│              │  Event Bus  │  (send results back)                │
│              │  (mpsc tx)  │                                     │
│              └─────────────┘                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Project Setup

```toml
# Cargo.toml
[package]
name = "gitz"
version = "0.1.0"
edition = "2021"

[dependencies]
# Terminal UI framework
ratatui = "0.29"

# Cross-platform terminal events (keys, mouse, resize, focus)
crossterm = { version = "0.28", features = ["event-stream"] }

# Async runtime
tokio = { version = "1.42", features = ["full"] }

# Error handling
anyhow = "1.0"

# For global static initialization
once_cell = "1.20"
```

---

## The Event Loop with Crossterm + Ratatui

The event loop is the heart of your application. It initializes the terminal, reads events, and renders frames.

```rust
// src/main.rs
use std::time::Duration;
use anyhow::Result;
use crossterm::event::{self, Event as CrosstermEvent, KeyEvent};
use ratatui::DefaultTerminal;
use tokio::sync::mpsc;

#[tokio::main]
async fn main() -> Result<()> {
    // 1. Initialize terminal (raw mode, alternate screen, clear)
    let terminal = ratatui::init();
    
    // 2. Create event channel
    let (tx, mut rx) = mpsc::unbounded_channel::<AppEvent>();
    
    // 3. Spawn input reader thread
    std::thread::spawn(move || {
        loop {
            // Block until crossterm has an event (keys, resize, mouse)
            if let Ok(CrosstermEvent::Key(key)) = event::read() {
                let _ = tx.send(AppEvent::Key(key));
            }
        }
    });
    
    // 4. Run app
    let mut app = App::new(terminal);
    app.run(&mut rx).await?;
    
    // 5. Restore terminal (disable raw mode, show cursor, etc.)
    ratatui::restore();
    Ok(())
}

#[derive(Debug, Clone)]
enum AppEvent {
    Key(KeyEvent),
    Tick,           // For animations, blinking cursors
    GitStatus(String),
    Quit,
}
```

**Important:** Crossterm's `event::read()` is **blocking**. We put it in a dedicated thread that sends events into our async channel. The main async loop stays responsive.

### The Async App Loop

```rust
// src/app.rs
pub struct App {
    terminal: DefaultTerminal,
    state: AppState,
    should_quit: bool,
}

impl App {
    pub fn new(terminal: DefaultTerminal) -> Self {
        Self {
            terminal,
            state: AppState::default(),
            should_quit: false,
        }
    }

    pub async fn run(&mut self, rx: &mut mpsc::UnboundedReceiver<AppEvent>) -> Result<()> {
        // Initial render
        self.draw()?;
        
        loop {
            tokio::select! {
                // Process terminal events
                Some(event) = rx.recv() => {
                    self.handle_event(event);
                    if self.should_quit {
                        break;
                    }
                    self.draw()?;
                }
                
                // Periodic tick for animations
                _ = tokio::time::sleep(Duration::from_millis(100)) => {
                    self.handle_event(AppEvent::Tick);
                    self.draw()?;
                }
            }
        }
        
        Ok(())
    }
    
    fn draw(&mut self) -> Result<()> {
        self.terminal.draw(|frame| {
            // frame.area() gives us the full terminal rect
            let area = frame.area();
            
            // Route to the appropriate renderer based on active screen
            match self.state.active_screen() {
                Screen::Dashboard => render_dashboard(frame, area, &self.state),
                Screen::RepoDetail => render_repo_detail(frame, area, &self.state),
                Screen::CommitDialog => render_commit_dialog(frame, area, &mut self.state),
                Screen::Help => render_help(frame, area, &self.state),
            }
        })?;
        Ok(())
    }
}
```

---

## Centralized State Management

Just like Yazi's `Core`, keep all state in one struct:

```rust
// src/state.rs
use ratatui::widgets::{ListState, TableState};

#[derive(Default)]
pub struct AppState {
    // Screen visibility flags
    pub dashboard_active: bool,
    pub repo_detail_active: bool,
    pub commit_dialog_visible: bool,
    pub help_visible: bool,
    pub branch_picker_visible: bool,
    
    // Dashboard data
    pub repos: Vec<Repo>,
    pub repo_list_state: ListState,
    pub selected_repo: usize,
    
    // Repo detail data
    pub status_items: Vec<StatusItem>,
    pub status_table_state: TableState,
    pub selected_status: usize,
    pub current_branch: String,
    pub commits: Vec<Commit>,
    
    // Commit dialog data
    pub commit_message: String,
    pub commit_cursor: usize,
    
    // Branch picker
    pub branches: Vec<String>,
    pub branch_filter: String,
    pub selected_branch: usize,
    
    // Notifications
    pub notification: Option<String>,
    pub notification_time: Option<std::time::Instant>,
}

#[derive(Clone)]
pub struct Repo {
    pub name: String,
    pub path: std::path::PathBuf,
    pub branch: String,
    pub ahead: usize,
    pub behind: usize,
    pub dirty: bool,
}

#[derive(Clone)]
pub struct StatusItem {
    pub staged: bool,
    pub status: char,
    pub path: String,
}

#[derive(Clone)]
pub struct Commit {
    pub hash: String,
    pub message: String,
    pub author: String,
    pub time: String,
}

impl AppState {
    /// Determine which screen should receive input/rendering
    pub fn active_screen(&self) -> Screen {
        if self.commit_dialog_visible {
            Screen::CommitDialog
        } else if self.branch_picker_visible {
            Screen::BranchPicker
        } else if self.help_visible {
            Screen::Help
        } else if self.repo_detail_active {
            Screen::RepoDetail
        } else {
            Screen::Dashboard
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Screen {
    Dashboard,
    RepoDetail,
    CommitDialog,
    BranchPicker,
    Help,
}
```

---

## The Layer System (Screens as a Stack)

Yazi's `Core::layer()` returns the topmost visible layer. We implement the same pattern:

```rust
impl AppState {
    /// Returns the topmost active screen (like Yazi's `core.layer()`)
    pub fn active_screen(&self) -> Screen {
        // Check in priority order (topmost first)
        if self.commit_dialog_visible {
            Screen::CommitDialog
        } else if self.branch_picker_visible {
            Screen::BranchPicker
        } else if self.help_visible {
            Screen::Help
        } else if self.repo_detail_active {
            Screen::RepoDetail
        } else {
            Screen::Dashboard
        }
    }
    
    /// Check if a modal is currently open (blocks main screen input)
    pub fn has_modal_open(&self) -> bool {
        self.commit_dialog_visible 
            || self.branch_picker_visible 
            || self.help_visible
    }
}
```

**Why this works:** When you press `c` to open the commit dialog, `commit_dialog_visible` becomes `true`. The very next key press is routed to `Screen::CommitDialog` instead of `Screen::RepoDetail`.

---

## Key Input Routing

This is where the magic happens. Route keys based on the active screen:

```rust
// src/app.rs
impl App {
    fn handle_event(&mut self, event: AppEvent) {
        match event {
            AppEvent::Key(key) => self.handle_key(key),
            AppEvent::Tick => self.handle_tick(),
            AppEvent::GitStatus(output) => self.handle_git_status(output),
            AppEvent::Quit => self.should_quit = true,
        }
    }
    
    fn handle_key(&mut self, key: KeyEvent) {
        // Route to the active screen's handler
        match self.state.active_screen() {
            Screen::Dashboard => self.handle_dashboard_key(key),
            Screen::RepoDetail => self.handle_repo_detail_key(key),
            Screen::CommitDialog => self.handle_commit_key(key),
            Screen::BranchPicker => self.handle_branch_picker_key(key),
            Screen::Help => self.handle_help_key(key),
        }
    }
    
    fn handle_dashboard_key(&mut self, key: KeyEvent) {
        use crossterm::event::KeyCode;
        
        match key.code {
            KeyCode::Char('q') => self.should_quit = true,
            KeyCode::Char('j') | KeyCode::Down => self.next_repo(),
            KeyCode::Char('k') | KeyCode::Up => self.prev_repo(),
            KeyCode::Enter => {
                if !self.state.repos.is_empty() {
                    self.state.repo_detail_active = true;
                    self.load_repo_status();
                }
            }
            KeyCode::Char('?') | KeyCode::Char('h') => {
                self.state.help_visible = true;
            }
            _ => {}
        }
    }
    
    fn handle_repo_detail_key(&mut self, key: KeyEvent) {
        use crossterm::event::KeyCode;
        
        match key.code {
            KeyCode::Esc | KeyCode::Char('q') => {
                self.state.repo_detail_active = false;
            }
            KeyCode::Char('j') | KeyCode::Down => self.next_status(),
            KeyCode::Char('k') | KeyCode::Up => self.prev_status(),
            KeyCode::Char('s') => self.stage_selected(),
            KeyCode::Char('u') => self.unstage_selected(),
            KeyCode::Char('c') => {
                self.state.commit_dialog_visible = true;
                self.state.commit_message.clear();
                self.state.commit_cursor = 0;
            }
            KeyCode::Char('b') => {
                self.state.branch_picker_visible = true;
                self.load_branches();
            }
            KeyCode::Char('?') | KeyCode::Char('h') => {
                self.state.help_visible = true;
            }
            _ => {}
        }
    }
    
    fn handle_commit_key(&mut self, key: KeyEvent) {
        use crossterm::event::KeyCode;
        
        match key.code {
            KeyCode::Esc => {
                self.state.commit_dialog_visible = false;
            }
            KeyCode::Enter => {
                let msg = self.state.commit_message.clone();
                self.do_commit(msg);
                self.state.commit_dialog_visible = false;
            }
            KeyCode::Char(c) => {
                let cursor = self.state.commit_cursor;
                self.state.commit_message.insert(cursor, c);
                self.state.commit_cursor += 1;
            }
            KeyCode::Backspace => {
                let cursor = self.state.commit_cursor;
                if cursor > 0 {
                    self.state.commit_message.remove(cursor - 1);
                    self.state.commit_cursor -= 1;
                }
            }
            KeyCode::Left => {
                if self.state.commit_cursor > 0 {
                    self.state.commit_cursor -= 1;
                }
            }
            KeyCode::Right => {
                if self.state.commit_cursor < self.state.commit_message.len() {
                    self.state.commit_cursor += 1;
                }
            }
            _ => {}
        }
    }
    
    fn handle_branch_picker_key(&mut self, key: KeyEvent) {
        use crossterm::event::KeyCode;
        
        match key.code {
            KeyCode::Esc => {
                self.state.branch_picker_visible = false;
            }
            KeyCode::Enter => {
                if let Some(branch) = self.state.branches.get(self.state.selected_branch) {
                    let branch = branch.clone();
                    self.checkout_branch(branch);
                }
                self.state.branch_picker_visible = false;
            }
            KeyCode::Char('j') | KeyCode::Down => {
                let max = self.state.branches.len().saturating_sub(1);
                if self.state.selected_branch < max {
                    self.state.selected_branch += 1;
                }
            }
            KeyCode::Char('k') | KeyCode::Up => {
                if self.state.selected_branch > 0 {
                    self.state.selected_branch -= 1;
                }
            }
            _ => {}
        }
    }
    
    fn handle_help_key(&mut self, key: KeyEvent) {
        use crossterm::event::KeyCode;
        match key.code {
            KeyCode::Esc | KeyCode::Char('q') | KeyCode::Char('?') | KeyCode::Char('h') => {
                self.state.help_visible = false;
            }
            _ => {}
        }
    }
}
```

**Key insight:** Each screen has its own `handle_*_key` function. When a modal is visible, keys go to the modal handler, not the main screen. This is identical to Yazi's Router pattern.

---

## Building Screens as Ratatui Widgets

Each screen is just a function that draws to a `Frame`. Ratatui uses a declarative API — you describe what the UI should look like, and Ratatui draws it.

### The Dashboard Screen

```rust
// src/screens/dashboard.rs
use ratatui::{
    layout::{Constraint, Direction, Layout, Margin, Rect},
    style::{Color, Modifier, Style, Stylize},
    text::{Line, Span},
    widgets::{Block, Borders, Cell, List, ListItem, Paragraph, Row, Table},
    Frame,
};
use crate::state::{AppState, Repo};

pub fn render_dashboard(frame: &mut Frame, area: Rect, state: &AppState) {
    // Split the screen into vertical chunks
    let chunks = Layout::default()
        .direction(Direction::Vertical)
        .constraints([
            Constraint::Length(3),   // Title bar
            Constraint::Min(5),      // Main content
            Constraint::Length(1),   // Status line
        ])
        .split(area);
    
    // Title bar
    let title = Paragraph::new(" Gitz — Git TUI ")
        .style(Style::default().fg(Color::Cyan).add_modifier(Modifier::BOLD))
        .block(Block::default().borders(Borders::ALL));
    frame.render_widget(title, chunks[0]);
    
    // Main content: repo list
    render_repo_list(frame, chunks[1], state);
    
    // Status line
    let status = format!(
        " {} repos | j/k: navigate | Enter: open | ?: help | q: quit ",
        state.repos.len()
    );
    let status_bar = Paragraph::new(status)
        .style(Style::default().fg(Color::DarkGray));
    frame.render_widget(status_bar, chunks[2]);
}

fn render_repo_list(frame: &mut Frame, area: Rect, state: &AppState) {
    let items: Vec<ListItem> = state.repos.iter().enumerate()
        .map(|(idx, repo)| {
            let style = if idx == state.selected_repo {
                Style::default().bg(Color::Blue).fg(Color::White)
            } else if repo.dirty {
                Style::default().fg(Color::Yellow)
            } else {
                Style::default()
            };
            
            let branch_info = format!("[{} ↑{}↓{}]", repo.branch, repo.ahead, repo.behind);
            let text = Line::from(vec![
                Span::styled(&repo.name, style.add_modifier(Modifier::BOLD)),
                Span::raw(" "),
                Span::styled(branch_info, Style::default().fg(Color::DarkGray)),
            ]);
            
            ListItem::new(text)
        })
        .collect();
    
    let list = List::new(items)
        .block(Block::default().title(" Repositories ").borders(Borders::ALL))
        .highlight_style(Style::default().bg(Color::Blue).fg(Color::White));
    
    frame.render_stateful_widget(
        list,
        area,
        &mut state.repo_list_state.clone(),
    );
}
```

### The Repo Detail Screen

```rust
// src/screens/repo_detail.rs
use ratatui::{
    layout::{Constraint, Direction, Layout, Rect},
    style::{Color, Style, Stylize},
    text::Line,
    widgets::{Block, Borders, Cell, Row, Table, Paragraph},
    Frame,
};
use crate::state::{AppState, StatusItem};

pub fn render_repo_detail(frame: &mut Frame, area: Rect, state: &AppState) {
    let chunks = Layout::default()
        .direction(Direction::Vertical)
        .constraints([
            Constraint::Length(3),   // Header with branch info
            Constraint::Min(10),     // Status table
            Constraint::Length(3),   // Commit log preview
            Constraint::Length(1),   // Key hints
        ])
        .split(area);
    
    // Header
    let repo = state.repos.get(state.selected_repo);
    let header_text = if let Some(repo) = repo {
        format!(" {} — {} ", repo.name, repo.branch)
    } else {
        " No repo selected ".to_string()
    };
    let header = Paragraph::new(header_text)
        .style(Style::default().fg(Color::Green).bold())
        .block(Block::default().borders(Borders::ALL));
    frame.render_widget(header, chunks[0]);
    
    // Status table
    render_status_table(frame, chunks[1], state);
    
    // Commit log
    render_commit_preview(frame, chunks[2], state);
    
    // Key hints
    let hints = " s: stage | u: unstage | c: commit | b: branches | ?: help | Esc: back ";
    let hints_bar = Paragraph::new(hints)
        .style(Style::default().fg(Color::DarkGray));
    frame.render_widget(hints_bar, chunks[3]);
}

fn render_status_table(frame: &mut Frame, area: Rect, state: &AppState) {
    let header = Row::new(vec!["Status", "File"])
        .style(Style::default().fg(Color::Yellow).bold())
        .height(1);
    
    let rows: Vec<Row> = state.status_items.iter().enumerate()
        .map(|(idx, item)| {
            let status_style = match item.status {
                'M' => Style::default().fg(Color::Yellow),
                'A' => Style::default().fg(Color::Green),
                'D' => Style::default().fg(Color::Red),
                '?' => Style::default().fg(Color::Blue),
                _ => Style::default(),
            };
            
            let row_style = if idx == state.selected_status {
                Style::default().bg(Color::DarkGray)
            } else {
                Style::default()
            };
            
            Row::new(vec![
                Cell::from(item.status.to_string()).style(status_style),
                Cell::from(item.path.clone()),
            ])
            .style(row_style)
        })
        .collect();
    
    let table = Table::new(rows, [
        Constraint::Length(8),
        Constraint::Percentage(100),
    ])
    .header(header)
    .block(Block::default().title(" Git Status ").borders(Borders::ALL))
    .row_highlight_style(Style::default().bg(Color::Blue).fg(Color::White));
    
    frame.render_stateful_widget(
        table,
        area,
        &mut state.status_table_state.clone(),
    );
}

fn render_commit_preview(frame: &mut Frame, area: Rect, state: &AppState) {
    let text = if state.commits.is_empty() {
        " No commits yet ".to_string()
    } else {
        state.commits.iter()
            .take(3)
            .map(|c| format!("{} {}", &c.hash[..7], c.message))
            .collect::<Vec<_>>()
            .join("\n")
    };
    
    let preview = Paragraph::new(text)
        .block(Block::default().title(" Recent Commits ").borders(Borders::ALL));
    frame.render_widget(preview, area);
}
```

---

## Modals and Popups

Modals are rendered **on top** of the current screen. They use `Clear` to blank the area behind them.

### The Commit Dialog

```rust
// src/screens/commit_dialog.rs
use ratatui::{
    layout::{Alignment, Constraint, Direction, Layout, Margin, Rect},
    style::{Color, Style, Stylize},
    text::Text,
    widgets::{Block, Borders, Clear, Paragraph},
    Frame,
};
use crate::state::AppState;

pub fn render_commit_dialog(frame: &mut Frame, _area: Rect, state: &mut AppState) {
    // Calculate centered popup area (60% width, 30% height)
    let popup_area = centered_rect(60, 30, frame.area());
    
    // Clear the area behind the popup (draws blank spaces)
    frame.render_widget(Clear, popup_area);
    
    // Draw the dialog box
    let block = Block::default()
        .title(" Commit ")
        .title_alignment(Alignment::Center)
        .borders(Borders::ALL)
        .border_style(Style::default().fg(Color::Green));
    
    // Split popup into title and input areas
    let chunks = Layout::default()
        .direction(Direction::Vertical)
        .margin(1)
        .constraints([
            Constraint::Length(1),   // Label
            Constraint::Min(1),      // Input area
            Constraint::Length(1),   // Hint
        ])
        .split(popup_area);
    
    // Label
    let label = Paragraph::new("Enter commit message:");
    frame.render_widget(label, chunks[0]);
    
    // Input area
    let input = Paragraph::new(state.commit_message.as_str())
        .block(Block::default().borders(Borders::ALL));
    frame.render_widget(input, chunks[1]);
    
    // Hint
    let hint = Paragraph::new("Enter: confirm | Esc: cancel")
        .style(Style::default().fg(Color::DarkGray));
    frame.render_widget(hint, chunks[2]);
    
    // Set the cursor position inside the input box
    let input_area = chunks[1].inner(Margin::new(1, 1));
    let cursor_x = input_area.x + state.commit_cursor as u16;
    let cursor_y = input_area.y;
    frame.set_cursor_position((cursor_x, cursor_y));
}

/// Helper: create a centered rectangle
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

**The `Clear` widget is crucial.** Without it, the popup would draw over existing content, creating a mess of overlapping text. `Clear` fills the popup area with blank cells first.

### The Help Overlay

```rust
// src/screens/help.rs
use ratatui::{
    layout::{Alignment, Constraint, Direction, Layout, Rect},
    style::{Color, Style},
    text::Text,
    widgets::{Block, Borders, Clear, Paragraph},
    Frame,
};
use crate::state::AppState;

pub fn render_help(frame: &mut Frame, _area: Rect, state: &AppState) {
    let popup_area = centered_rect(70, 70, frame.area());
    frame.render_widget(Clear, popup_area);
    
    let help_text = if state.repo_detail_active {
        Text::from(vec![
            "j / ↓     — Next file".into(),
            "k / ↑     — Previous file".into(),
            "s         — Stage selected file".into(),
            "u         — Unstage selected file".into(),
            "c         — Open commit dialog".into(),
            "b         — Open branch picker".into(),
            "? / h     — Toggle help".into(),
            "Esc / q   — Back to dashboard".into(),
        ])
    } else {
        Text::from(vec![
            "j / ↓     — Next repository".into(),
            "k / ↑     — Previous repository".into(),
            "Enter     — Open repository".into(),
            "? / h     — Toggle help".into(),
            "q         — Quit".into(),
        ])
    };
    
    let help = Paragraph::new(help_text)
        .block(Block::default()
            .title(" Help ")
            .title_alignment(Alignment::Center)
            .borders(Borders::ALL)
            .border_style(Style::default().fg(Color::Cyan)))
        .alignment(Alignment::Left);
    
    frame.render_widget(help, popup_area);
}
```

---

## The Painter's Algorithm in Practice

Yazi renders screens from back to front. Here's how to implement that with Ratatui:

```rust
fn draw(&mut self) -> Result<()> {
    self.terminal.draw(|frame| {
        let area = frame.area();
        
        // ALWAYS draw the base screen first
        match self.state.active_base_screen() {
            BaseScreen::Dashboard => render_dashboard(frame, area, &self.state),
            BaseScreen::RepoDetail => render_repo_detail(frame, area, &self.state),
        }
        
        // THEN draw overlays on top (back to front)
        // 1. Branch picker (bottom overlay)
        if self.state.branch_picker_visible {
            render_branch_picker(frame, area, &self.state);
        }
        
        // 2. Commit dialog
        if self.state.commit_dialog_visible {
            render_commit_dialog(frame, area, &mut self.state);
        }
        
        // 3. Help (topmost overlay)
        if self.state.help_visible {
            render_help(frame, area, &self.state);
        }
        
        // 4. Notification toast (drawn last, always visible)
        if let Some(ref notif) = self.state.notification {
            render_notification(frame, area, notif);
        }
    })?;
    Ok(())
}
```

**Note:** Even if `help_visible` is `true`, the base screen (Dashboard or RepoDetail) is still drawn first. The help overlay then draws on top. This ensures the user can still see the dimmed content behind the help panel.

---

## Background Workers + State Updates

Here's the complete pattern for async work:

```rust
// src/git_worker.rs
use std::path::PathBuf;
use tokio::process::Command;

/// Fetch git status for a repository
pub fn fetch_status(repo_path: PathBuf, repo_idx: usize) {
    tokio::spawn(async move {
        let output = Command::new("git")
            .args(&["status", "--porcelain", "-b"])
            .current_dir(&repo_path)
            .output()
            .await;
        
        match output {
            Ok(o) => {
                let text = String::from_utf8_lossy(&o.stdout).to_string();
                crate::event::emit(crate::AppEvent::GitStatus { repo_idx, output: text });
            }
            Err(e) => {
                crate::event::emit(crate::AppEvent::Error(format!(
                    "Failed to get status: {}", e
                )));
            }
        }
    });
}

/// Stage a file
pub fn stage_file(repo_path: PathBuf, file_path: String) {
    tokio::spawn(async move {
        let _ = Command::new("git")
            .args(&["add", &file_path])
            .current_dir(&repo_path)
            .output()
            .await;
        crate::event::emit(crate::AppEvent::Refresh);
    });
}

/// Commit with message
pub fn commit(repo_path: PathBuf, message: String) {
    tokio::spawn(async move {
        let result = Command::new("git")
            .args(&["commit", "-m", &message])
            .current_dir(&repo_path)
            .output()
            .await;
        
        match result {
            Ok(o) if o.status.success() => {
                crate::event::emit(crate::AppEvent::Notification("Committed!".into()));
            }
            Ok(o) => {
                let err = String::from_utf8_lossy(&o.stderr);
                crate::event::emit(crate::AppEvent::Error(err.to_string()));
            }
            Err(e) => {
                crate::event::emit(crate::AppEvent::Error(e.to_string()));
            }
        }
        crate::event::emit(crate::AppEvent::Refresh);
    });
}
```

**The event module for global access:**

```rust
// src/event.rs
use once_cell::sync::OnceCell;
use tokio::sync::mpsc::UnboundedSender;

static SENDER: OnceCell<UnboundedSender<AppEvent>> = OnceCell::new();

pub fn init(tx: UnboundedSender<AppEvent>) {
    let _ = SENDER.set(tx);
}

pub fn emit(event: AppEvent) {
    if let Some(tx) = SENDER.get() {
        let _ = tx.send(event);
    }
}

#[derive(Debug, Clone)]
pub enum AppEvent {
    Key(crossterm::event::KeyEvent),
    Tick,
    GitStatus { repo_idx: usize, output: String },
    Notification(String),
    Error(String),
    Refresh,
    Quit,
}
```

---

## A Complete Working Example

Here's a minimal but complete application you can compile and run:

```rust
// main.rs — complete working example
use anyhow::Result;
use crossterm::event::{self, Event as CrosstermEvent, KeyCode, KeyEvent};
use once_cell::sync::OnceCell;
use ratatui::{
    layout::{Alignment, Constraint, Direction, Layout, Rect},
    style::{Color, Modifier, Style, Stylize},
    text::{Line, Span, Text},
    widgets::{Block, Borders, Clear, List, ListItem, Paragraph},
    DefaultTerminal,
};
use std::time::Duration;
use tokio::sync::mpsc;

// ─── Global Event Sender ──────────────────────────────────

static EVENT_TX: OnceCell<mpsc::UnboundedSender<AppEvent>> = OnceCell::new();

fn emit(event: AppEvent) {
    if let Some(tx) = EVENT_TX.get() {
        let _ = tx.send(event);
    }
}

// ─── Event Enum ───────────────────────────────────────────

#[derive(Debug, Clone)]
enum AppEvent {
    Key(KeyEvent),
    Tick,
    Quit,
}

// ─── State ────────────────────────────────────────────────

#[derive(Default)]
struct AppState {
    items: Vec<String>,
    selected: usize,
    show_help: bool,
    show_modal: bool,
    modal_input: String,
}

impl AppState {
    fn active_screen(&self) -> Screen {
        if self.show_modal { Screen::Modal }
        else if self.show_help { Screen::Help }
        else { Screen::Main }
    }
}

#[derive(Clone, Copy)]
enum Screen {
    Main,
    Modal,
    Help,
}

// ─── App ──────────────────────────────────────────────────

struct App {
    terminal: DefaultTerminal,
    state: AppState,
    should_quit: bool,
}

impl App {
    fn new(terminal: DefaultTerminal) -> Self {
        Self {
            terminal,
            state: AppState {
                items: vec![
                    "Item 1 — Press 'c' for modal".into(),
                    "Item 2 — Press '?' for help".into(),
                    "Item 3 — Press 'q' to quit".into(),
                ],
                selected: 0,
                show_help: false,
                show_modal: false,
                modal_input: String::new(),
            },
            should_quit: false,
        }
    }

    async fn run(&mut self, rx: &mut mpsc::UnboundedReceiver<AppEvent>) -> Result<()> {
        self.draw()?;
        
        loop {
            tokio::select! {
                Some(event) = rx.recv() => {
                    self.handle(event);
                    if self.should_quit { break; }
                    self.draw()?;
                }
                _ = tokio::time::sleep(Duration::from_millis(100)) => {
                    self.draw()?;
                }
            }
        }
        Ok(())
    }

    fn handle(&mut self, event: AppEvent) {
        match event {
            AppEvent::Key(key) => self.handle_key(key),
            AppEvent::Tick => {}
            AppEvent::Quit => self.should_quit = true,
        }
    }

    fn handle_key(&mut self, key: KeyEvent) {
        match self.state.active_screen() {
            Screen::Modal => self.handle_modal_key(key),
            Screen::Help => self.handle_help_key(key),
            Screen::Main => self.handle_main_key(key),
        }
    }

    fn handle_main_key(&mut self, key: KeyEvent) {
        match key.code {
            KeyCode::Char('q') => emit(AppEvent::Quit),
            KeyCode::Char('j') | KeyCode::Down => {
                if self.state.selected < self.state.items.len().saturating_sub(1) {
                    self.state.selected += 1;
                }
            }
            KeyCode::Char('k') | KeyCode::Up => {
                if self.state.selected > 0 {
                    self.state.selected -= 1;
                }
            }
            KeyCode::Char('c') => {
                self.state.show_modal = true;
                self.state.modal_input.clear();
            }
            KeyCode::Char('?') | KeyCode::Char('h') => {
                self.state.show_help = true;
            }
            _ => {}
        }
    }

    fn handle_modal_key(&mut self, key: KeyEvent) {
        match key.code {
            KeyCode::Esc => self.state.show_modal = false,
            KeyCode::Enter => {
                let msg = format!("You typed: {}", self.state.modal_input);
                self.state.items.push(msg);
                self.state.show_modal = false;
            }
            KeyCode::Char(c) => self.state.modal_input.push(c),
            KeyCode::Backspace => { self.state.modal_input.pop(); }
            _ => {}
        }
    }

    fn handle_help_key(&mut self, key: KeyEvent) {
        match key.code {
            KeyCode::Esc | KeyCode::Char('q') | KeyCode::Char('?') | KeyCode::Char('h') => {
                self.state.show_help = false;
            }
            _ => {}
        }
    }

    fn draw(&mut self) -> Result<()> {
        self.terminal.draw(|frame| {
            let area = frame.area();
            
            // 1. Draw base screen
            self.render_main(frame, area);
            
            // 2. Draw overlays back-to-front
            if self.state.show_modal {
                self.render_modal(frame, area);
            }
            if self.state.show_help {
                self.render_help(frame, area);
            }
        })?;
        Ok(())
    }

    fn render_main(&self, frame: &mut ratatui::Frame, area: Rect) {
        let chunks = Layout::default()
            .direction(Direction::Vertical)
            .constraints([Constraint::Length(3), Constraint::Min(5), Constraint::Length(1)])
            .split(area);
        
        // Title
        let title = Paragraph::new(" Gitz Demo ")
            .style(Style::default().fg(Color::Cyan).bold())
            .block(Block::default().borders(Borders::ALL));
        frame.render_widget(title, chunks[0]);
        
        // List
        let items: Vec<ListItem> = self.state.items.iter().enumerate()
            .map(|(i, text)| {
                let style = if i == self.state.selected {
                    Style::default().bg(Color::Blue).fg(Color::White)
                } else {
                    Style::default()
                };
                ListItem::new(text.as_str()).style(style)
            })
            .collect();
        let list = List::new(items)
            .block(Block::default().title(" Items ").borders(Borders::ALL));
        frame.render_widget(list, chunks[1]);
        
        // Status
        let status = " j/k: navigate | c: modal | ?: help | q: quit ";
        frame.render_widget(
            Paragraph::new(status).style(Style::default().fg(Color::DarkGray)),
            chunks[2]
        );
    }

    fn render_modal(&self, frame: &mut ratatui::Frame, _area: Rect) {
        let popup = centered_rect(50, 20, frame.area());
        frame.render_widget(Clear, popup);
        
        let block = Block::default()
            .title(" Modal ")
            .title_alignment(Alignment::Center)
            .borders(Borders::ALL)
            .border_style(Style::default().fg(Color::Green));
        
        let content = Layout::default()
            .direction(Direction::Vertical)
            .margin(1)
            .constraints([Constraint::Length(1), Constraint::Min(1)])
            .split(popup);
        
        frame.render_widget(Paragraph::new("Type something:"), content[0]);
        frame.render_widget(
            Paragraph::new(self.state.modal_input.as_str())
                .block(Block::default().borders(Borders::ALL)),
            content[1]
        );
    }

    fn render_help(&self, frame: &mut ratatui::Frame, _area: Rect) {
        let popup = centered_rect(60, 50, frame.area());
        frame.render_widget(Clear, popup);
        
        let text = Text::from(vec![
            "j / k / ↑ / ↓ — Navigate".into(),
            "c — Open modal".into(),
            "? / h — Toggle help".into(),
            "q — Quit".into(),
            "Esc — Close modal/help".into(),
        ]);
        
        let help = Paragraph::new(text)
            .block(Block::default()
                .title(" Help ")
                .title_alignment(Alignment::Center)
                .borders(Borders::ALL)
                .border_style(Style::default().fg(Color::Cyan)));
        frame.render_widget(help, popup);
    }
}

fn centered_rect(percent_x: u16, percent_y: u16, r: Rect) -> Rect {
    let v = Layout::default()
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
        .split(v[1])[1]
}

// ─── Main ─────────────────────────────────────────────────

#[tokio::main]
async fn main() -> Result<()> {
    let terminal = ratatui::init();
    let (tx, mut rx) = mpsc::unbounded_channel();
    EVENT_TX.set(tx).unwrap();
    
    // Spawn input reader
    std::thread::spawn(|| {
        loop {
            if let Ok(CrosstermEvent::Key(key)) = event::read() {
                emit(AppEvent::Key(key));
            }
        }
    });
    
    let mut app = App::new(terminal);
    app.run(&mut rx).await?;
    
    ratatui::restore();
    Ok(())
}
```

---

## Advanced Patterns

### Pattern 1: Stateful Widgets with Ratatui

Ratatui widgets can be **stateful** (like `ListState`, `TableState`). They track scroll position, selected item, etc.:

```rust
use ratatui::widgets::{ListState, TableState};

#[derive(Default)]
struct AppState {
    pub list_state: ListState,
    pub table_state: TableState,
    pub scroll_offset: usize,
}

// In render:
let mut list_state = ListState::default();
list_state.select(Some(self.state.selected));
frame.render_stateful_widget(list, area, &mut list_state);
```

### Pattern 2: Conditional Styling

Change styles based on application state:

```rust
fn get_status_style(status: char) -> Style {
    match status {
        'M' => Style::default().fg(Color::Yellow),   // Modified
        'A' => Style::default().fg(Color::Green),    // Added
        'D' => Style::default().fg(Color::Red),      // Deleted
        'R' => Style::default().fg(Color::Magenta),  // Renamed
        'C' => Style::default().fg(Color::Cyan),     // Copied
        'U' => Style::default().fg(Color::Blue),     // Updated
        '?' => Style::default().fg(Color::White),    // Untracked
        _ => Style::default(),
    }
}
```

### Pattern 3: Scrollable Content

For long text (like git diff output):

```rust
use ratatui::widgets::{Paragraph, Wrap};

fn render_diff(frame: &mut Frame, area: Rect, diff_text: &str, scroll: u16) {
    let diff = Paragraph::new(diff_text)
        .wrap(Wrap { trim: false })
        .scroll((scroll, 0));
    frame.render_widget(diff, area);
}
```

### Pattern 4: Tabs

```rust
use ratatui::widgets::Tabs;

fn render_tabs(frame: &mut Frame, area: Rect, state: &AppState) {
    let titles = vec!["Status", "Log", "Branches", "Stash"];
    let tabs = Tabs::new(titles)
        .select(state.active_tab)
        .block(Block::default().borders(Borders::ALL))
        .highlight_style(Style::default().fg(Color::Green).bold());
    frame.render_widget(tabs, area);
}
```

### Pattern 5: Split Panes (like Yazi's file manager + preview)

```rust
fn render_split_view(frame: &mut Frame, area: Rect, state: &AppState) {
    let chunks = Layout::default()
        .direction(Direction::Horizontal)
        .constraints([Constraint::Percentage(40), Constraint::Percentage(60)])
        .split(area);
    
    render_file_list(frame, chunks[0], state);
    render_file_preview(frame, chunks[1], state);
}
```

---

## Common Pitfalls

### ❌ Forgetting `Clear` Before Modals

Without `Clear`, your modal draws over existing content:
```rust
// BAD — text bleeds through
frame.render_widget(modal, popup_area);

// GOOD — blank the area first
frame.render_widget(Clear, popup_area);
frame.render_widget(modal, popup_area);
```

### ❌ Blocking the Event Loop

```rust
// BAD — UI freezes
fn handle_key(&mut self, key: KeyEvent) {
    if key.code == KeyCode::Char('r') {
        let output = std::process::Command::new("git").arg("status").output().unwrap();
        // ↑ BLOCKS for 50ms+
    }
}

// GOOD — spawn async task
fn handle_key(&mut self, key: KeyEvent) {
    if key.code == KeyCode::Char('r') {
        let path = self.repo_path.clone();
        tokio::spawn(async move {
            let output = tokio::process::Command::new("git").arg("status").current_dir(path).output().await;
            emit(AppEvent::GitStatus(output));
        });
    }
}
```

### ❌ Multiple Sources of State

```rust
// BAD — which is the real state?
struct Sidebar { items: Vec<String> }
struct MainView { items: Vec<String> }

// GOOD — one state, multiple views
struct AppState {
    items: Vec<String>,
}
```

### ❌ Forgetting to Handle Resize

```rust
// Add this to your event handler:
CrosstermEvent::Resize(w, h) => {
    // Ratatui automatically handles terminal resize
    // Just redraw on next frame
    self.needs_render = true;
}
```

### ❌ Not Setting the Cursor Position

For text input, you must manually set the cursor:
```rust
// After rendering the input box:
let cursor_x = input_area.x + state.cursor as u16;
let cursor_y = input_area.y;
frame.set_cursor_position((cursor_x, cursor_y));
```

---

## Summary

| Yazi Pattern | Ratatui Implementation |
|--------------|----------------------|
| `Core` state | `AppState` struct with `visible` flags |
| `Core::layer()` | `active_screen()` priority check |
| Router | `match active_screen() { ... }` |
| `Root::render()` | `terminal.draw(|frame| { ... })` |
| Modals | `Clear` widget + centered rect |
| `emit!()` | `tokio::sync::mpsc` + static sender |
| Background workers | `tokio::spawn` + emit events |
| Painter's algorithm | Draw base first, overlays back-to-front |

The key insight: **Ratatui is just the renderer.** The architecture (state management, routing, modals, async workers) is your code. Yazi proves that with a clean architecture, Ratatui can power complex, responsive terminal applications.
