# Building a Yazi-Style TUI with Bubble Tea

> **Translating Yazi's actor-based, event-driven architecture into Go using Charm's Bubble Tea framework. We'll build "Gitz" — a Git TUI — as the running example.**

---

## Table of Contents

1. [Why Bubble Tea?](#why-bubble-tea)
2. [Yazi vs. Bubble Tea: Mental Model Mapping](#yazi-vs-bubble-tea-mental-model-mapping)
3. [The Core Philosophy](#the-core-philosophy)
4. [Project Setup](#project-setup)
5. [The Centralized State (Model)](#the-centralized-state-model)
6. [Messages: The Event Bus](#messages-the-event-bus)
7. [Update: The Dispatcher + Executor Combined](#update-the-dispatcher--executor-combined)
8. [The Layer System (Screens as a Stack)](#the-layer-system-screens-as-a-stack)
9. [Key Input Routing](#key-input-routing)
10. [Background Work with tea.Cmd](#background-work-with-teacmd)
11. [Rendering with lipgloss](#rendering-with-lipgloss)
12. [Modals and Popups](#modals-and-popups)
13. [Sub-Models as "Actors"](#sub-models-as-actors)
14. [A Complete Working Example](#a-complete-working-example)
15. [Advanced Patterns](#advanced-patterns)
16. [When Bubble Tea Differs from Yazi](#when-bubble-tea-differs-from-yazi)

---

## Why Bubble Tea?

**Bubble Tea** is a Go framework by Charm for building terminal user interfaces. It's built on The Elm Architecture (TEA) — the same conceptual foundation as Yazi's design.

```
┌─────────────────────────────────────────────────────────────────┐
│                    BUBBLE TEA ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐     │
│  │   Init()    │─────►│   Update()  │◄─────│    Msg      │     │
│  │  (startup)  │      │  (dispatch) │      │  (events)   │     │
│  └─────────────┘      └──────┬──────┘      └─────────────┘     │
│                              │                                   │
│                              │ returns (Model, Cmd)              │
│                              ▼                                   │
│                       ┌─────────────┐                            │
│                       │   Model     │                            │
│                       │  (state)    │                            │
│                       └──────┬──────┘                            │
│                              │                                   │
│                              ▼                                   │
│                       ┌─────────────┐                            │
│                       │   View()    │                            │
│                       │  (render)   │                            │
│                       └──────┬──────┘                            │
│                              │                                   │
│                              ▼                                   │
│                       ┌─────────────┐                            │
│                       │  Terminal   │                            │
│                       └─────────────┘                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight:** Bubble Tea **is** your event loop. You don't write `loop { rx.recv() }`. You implement `Update(msg tea.Msg) (tea.Model, tea.Cmd)` and Bubble Tea calls it for every message.

---

## Yazi vs. Bubble Tea: Mental Model Mapping

| Yazi (Rust) | Bubble Tea (Go) | What It Is |
|-------------|----------------|------------|
| `Core` | `Model` (struct) | Centralized application state |
| `Event` enum | `tea.Msg` (interface{}) | All messages/events |
| `Event::Key` | `tea.KeyMsg` | Key press events |
| `Dispatcher::dispatch()` | `Update(msg tea.Msg)` | Message routing handler |
| `Executor::execute()` | Type switch in `Update()` | Action dispatch by layer |
| `Actor::act()` | Sub-model's `Update()` method | Individual command handler |
| `Scheduler` + workers | `tea.Cmd` returning `tea.Msg` | Background effects |
| `emit!(Call(...))` | `return m, cmd` | Sending a command for later |
| `Root::render()` | `View() string` | Render function |
| `ratatui::Buffer` | `string` + `lipgloss` styles | Rendering output |
| `tokio::spawn` | `tea.Cmd` (framework-managed) | Concurrent execution |
| `mpsc::unbounded_channel` | Built into Bubble Tea runtime | Event bus |

**The biggest difference:** In Yazi, you **write** the event loop. In Bubble Tea, the framework **is** the event loop. You just provide the `Update` function.

---

## The Core Philosophy

When building a Yazi-style app in Bubble Tea, follow these rules:

1. **One `Model` struct owns everything** — Just like Yazi's `Core`
2. **All state changes happen through `Update()`** — Just like Yazi's event bus
3. **Background work uses `tea.Cmd`** — Just like Yazi's scheduler
4. **Screens are a stack of visibility flags** — Just like Yazi's `Core::layer()`
5. **Key routing checks the active layer first** — Just like Yazi's `Router`

The difference is that Go doesn't have `&mut` or the borrow checker. Instead, Bubble Tea uses **functional updates**: `Update` returns a **new** model. But because Go is garbage-collected, this is cheap.

---

## Project Setup

```bash
mkdir gitz && cd gitz
go mod init gitz
```

```go
// go.mod
module gitz

go 1.21

require (
	github.com/charmbracelet/bubbletea v1.1.0
	github.com/charmbracelet/lipgloss v0.13.0
)
```

```bash
go mod tidy
```

---

## The Centralized State (Model)

Just like Yazi's `Core`, define **one** struct that holds all application state:

```go
// model.go
package main

import tea "github.com/charmbracelet/bubbletea"

// Model is the single source of truth — equivalent to Yazi's Core
type Model struct {
	// Screen visibility flags (like Yazi's visible/active flags)
	dashboardActive    bool
	repoDetailActive   bool
	commitDialogOpen   bool
	branchPickerOpen   bool
	helpVisible        bool

	// Dashboard data
	repos          []Repo
	selectedRepo   int

	// Repo detail data
	statusItems    []StatusItem
	selectedStatus int
	currentBranch  string
	commits        []Commit

	// Commit dialog
	commitMessage  string
	commitCursor   int

	// Branch picker
	branches       []string
	selectedBranch int

	// Dimensions (updated on WindowSizeMsg)
	width          int
	height         int

	// Notifications
	notification   string
	notifTimer     int // frames until auto-clear
}

// Screen identifies which layer is active
type Screen int

const (
	ScreenDashboard Screen = iota
	ScreenRepoDetail
	ScreenCommitDialog
	ScreenBranchPicker
	ScreenHelp
)

// activeScreen returns the topmost visible screen (like Yazi's Core.layer())
func (m Model) activeScreen() Screen {
	if m.commitDialogOpen {
		return ScreenCommitDialog
	}
	if m.branchPickerOpen {
		return ScreenBranchPicker
	}
	if m.helpVisible {
		return ScreenHelp
	}
	if m.repoDetailActive {
		return ScreenRepoDetail
	}
	return ScreenDashboard
}

// hasModalOpen returns true if any modal is blocking the base screen
func (m Model) hasModalOpen() bool {
	return m.commitDialogOpen || m.branchPickerOpen || m.helpVisible
}
```

**Key differences from Yazi:**
- Go uses **value semantics** by default — `Update` receives and returns a `Model` value
- No `&mut` needed — Bubble Tea handles the "exclusive access" by calling `Update` serially
- The framework guarantees `Update` is never called concurrently

---

## Messages: The Event Bus

In Bubble Tea, **everything is a `tea.Msg`**. It's an empty interface, so you define your own message types.

```go
// messages.go
package main

// gitStatusMsg is sent when a background git command completes
type gitStatusMsg struct {
	repoIdx int
	output  string
}

// gitBranchesMsg is sent when branch list is fetched
type gitBranchesMsg struct {
	branches []string
}

// gitCommitDoneMsg is sent when commit completes
type gitCommitDoneMsg struct {
	err error
}

// notificationMsg clears the notification after a delay
type notificationMsg struct{}

// errorMsg displays an error
type errorMsg struct {
	err string
}
```

**This is equivalent to Yazi's `Event` enum**, but using Go's structural typing (any type can be a `tea.Msg`).

---

## Update: The Dispatcher + Executor Combined

In Bubble Tea, `Update` is **both** the Dispatcher and the Executor. It receives a message and returns a new model + optional command.

```go
// update.go
package main

import (
	"fmt"
	tea "github.com/charmbracelet/bubbletea"
)

// Update is called by Bubble Tea for EVERY message.
// It's equivalent to Yazi's Dispatcher + Executor combined.
func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {

	// ─── System Messages ──────────────────────────────

	case tea.KeyMsg:
		// Route to the appropriate handler based on active screen
		return m.handleKey(msg)

	case tea.WindowSizeMsg:
		// Terminal resized
		m.width = msg.Width
		m.height = msg.Height
		return m, nil

	// ─── Git Operation Results ────────────────────────

	case gitStatusMsg:
		m.parseGitStatus(msg.repoIdx, msg.output)
		return m, nil

	case gitBranchesMsg:
		m.branches = msg.branches
		return m, nil

	case gitCommitDoneMsg:
		m.commitDialogOpen = false
		if msg.err != nil {
			m.notification = fmt.Sprintf("Error: %v", msg.err)
			m.notifTimer = 30 // 3 seconds at 100ms tick
		} else {
			m.notification = "Committed successfully!"
			m.notifTimer = 30
		}
		// Refresh status after commit
		return m, m.refreshCurrentRepo()

	// ─── Timer / Animation Messages ───────────────────

	case notificationMsg:
		if m.notifTimer > 0 {
			m.notifTimer--
			return m, tea.Tick(100*time.Millisecond, func(t time.Time) tea.Msg {
				return notificationMsg{}
			})
		}
		m.notification = ""
		return m, nil

	// ─── Default ──────────────────────────────────────

	default:
		return m, nil
	}
}
```

**How this maps to Yazi:**

| Yazi | Bubble Tea |
|------|-----------|
| `Dispatcher::dispatch(Event::Key)` | `Update(tea.KeyMsg)` → `handleKey()` |
| `Executor::execute(action)` | Type switch on `msg` in `Update()` |
| `act!(mgr:cd, cx, action)` | `m.parseGitStatus(...)` + return updated model |
| `emit!(Call(relay!(app:update_progress)))` | `return m, cmd` where `cmd` returns a message later |
| `render!()` | Automatic — `View()` is called after every `Update()` |

---

## Key Input Routing

This is where the layer system comes to life. Route keys to the appropriate handler.

```go
// keys.go
package main

import tea "github.com/charmbracelet/bubbletea"

// handleKey is equivalent to Yazi's Router::route()
func (m Model) handleKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch m.activeScreen() {
	case ScreenCommitDialog:
		return m.handleCommitKey(msg)
	case ScreenBranchPicker:
		return m.handleBranchPickerKey(msg)
	case ScreenHelp:
		return m.handleHelpKey(msg)
	case ScreenRepoDetail:
		return m.handleRepoDetailKey(msg)
	default:
		return m.handleDashboardKey(msg)
	}
}

// handleDashboardKey handles keys when on the main dashboard
func (m Model) handleDashboardKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.String() {
	case "q", "ctrl+c":
		return m, tea.Quit
	case "j", "down":
		if m.selectedRepo < len(m.repos)-1 {
			m.selectedRepo++
		}
	case "k", "up":
		if m.selectedRepo > 0 {
			m.selectedRepo--
		}
	case "enter":
		if len(m.repos) > 0 {
			m.repoDetailActive = true
			return m, m.refreshCurrentRepo()
		}
	case "?", "h":
		m.helpVisible = true
	}
	return m, nil
}

// handleRepoDetailKey handles keys when viewing a repo
func (m Model) handleRepoDetailKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.String() {
	case "q", "esc":
		m.repoDetailActive = false
	case "j", "down":
		if m.selectedStatus < len(m.statusItems)-1 {
			m.selectedStatus++
		}
	case "k", "up":
		if m.selectedStatus > 0 {
			m.selectedStatus--
		}
	case "s":
		return m, m.stageSelected()
	case "u":
		return m, m.unstageSelected()
	case "c":
		m.commitDialogOpen = true
		m.commitMessage = ""
		m.commitCursor = 0
	case "b":
		m.branchPickerOpen = true
		return m, m.fetchBranches()
	case "?", "h":
		m.helpVisible = true
	}
	return m, nil
}

// handleCommitKey handles keys inside the commit dialog
func (m Model) handleCommitKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.String() {
	case "esc":
		m.commitDialogOpen = false
	case "enter":
		return m, m.doCommit(m.commitMessage)
	case "backspace":
		if m.commitCursor > 0 {
			m.commitMessage = m.commitMessage[:m.commitCursor-1] + m.commitMessage[m.commitCursor:]
			m.commitCursor--
		}
	case "left":
		if m.commitCursor > 0 {
			m.commitCursor--
		}
	case "right":
		if m.commitCursor < len(m.commitMessage) {
			m.commitCursor++
		}
	default:
		// Insert character
		if len(msg.String()) == 1 {
			m.commitMessage = m.commitMessage[:m.commitCursor] + msg.String() + m.commitMessage[m.commitCursor:]
			m.commitCursor++
		}
	}
	return m, nil
}

// handleBranchPickerKey handles keys in the branch picker
func (m Model) handleBranchPickerKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.String() {
	case "esc":
		m.branchPickerOpen = false
	case "enter":
		if m.selectedBranch < len(m.branches) {
			branch := m.branches[m.selectedBranch]
			return m, m.checkoutBranch(branch)
		}
	case "j", "down":
		if m.selectedBranch < len(m.branches)-1 {
			m.selectedBranch++
		}
	case "k", "up":
		if m.selectedBranch > 0 {
			m.selectedBranch--
		}
	}
	return m, nil
}

// handleHelpKey handles keys when help is visible
func (m Model) handleHelpKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.String() {
	case "esc", "q", "?", "h":
		m.helpVisible = false
	}
	return m, nil
}
```

**Key insight:** Each handler receives a **copy** of the model, modifies it, and returns it. Bubble Tea replaces the old model with the new one. There's no `&mut` — Go passes structs by value.

---

## Background Work with tea.Cmd

This is where Bubble Tea shines. `tea.Cmd` is a function that returns a `tea.Msg`. Bubble Tea runs it concurrently and feeds the result back into `Update`.

```go
// commands.go
package main

import (
	"fmt"
	"os/exec"
	"strings"
	"time"
	tea "github.com/charmbracelet/bubbletea"
)

// refreshCurrentRepo fetches git status for the selected repository.
// This is equivalent to Yazi's scheduler submitting a background task.
func (m Model) refreshCurrentRepo() tea.Cmd {
	if m.selectedRepo >= len(m.repos) {
		return nil
	}
	repo := m.repos[m.selectedRepo]
	idx := m.selectedRepo

	return func() tea.Msg {
		cmd := exec.Command("git", "status", "--porcelain", "-b")
		cmd.Dir = repo.path
		out, err := cmd.Output()
		if err != nil {
			return errorMsg{err: fmt.Sprintf("git status failed: %v", err)}
		}
		return gitStatusMsg{repoIdx: idx, output: string(out)}
	}
}

// fetchBranches fetches the list of branches.
func (m Model) fetchBranches() tea.Cmd {
	if m.selectedRepo >= len(m.repos) {
		return nil
	}
	repo := m.repos[m.selectedRepo]

	return func() tea.Msg {
		cmd := exec.Command("git", "branch", "-a", "--format=%(refname:short)")
		cmd.Dir = repo.path
		out, err := cmd.Output()
		if err != nil {
			return errorMsg{err: fmt.Sprintf("git branch failed: %v", err)}
		}
		branches := strings.Split(strings.TrimSpace(string(out)), "\n")
		return gitBranchesMsg{branches: branches}
	}
}

// stageSelected stages the currently selected file.
func (m Model) stageSelected() tea.Cmd {
	if m.selectedStatus >= len(m.statusItems) {
		return nil
	}
	item := m.statusItems[m.selectedStatus]
	repo := m.repos[m.selectedRepo]

	return func() tea.Msg {
		cmd := exec.Command("git", "add", item.path)
		cmd.Dir = repo.path
		_, err := cmd.Output()
		if err != nil {
			return errorMsg{err: fmt.Sprintf("git add failed: %v", err)}
		}
		// Return a message that triggers a refresh
		return gitStatusMsg{repoIdx: m.selectedRepo, output: ""} // empty = will be refilled
	}
}

// doCommit creates a commit with the given message.
func (m Model) doCommit(message string) tea.Cmd {
	if m.selectedRepo >= len(m.repos) {
		return nil
	}
	repo := m.repos[m.selectedRepo]

	return func() tea.Msg {
		cmd := exec.Command("git", "commit", "-m", message)
		cmd.Dir = repo.path
		_, err := cmd.Output()
		return gitCommitDoneMsg{err: err}
	}
}

// checkoutBranch checks out a branch.
func (m Model) checkoutBranch(branch string) tea.Cmd {
	if m.selectedRepo >= len(m.repos) {
		return nil
	}
	repo := m.repos[m.selectedRepo]

	return func() tea.Msg {
		cmd := exec.Command("git", "checkout", branch)
		cmd.Dir = repo.path
		_, err := cmd.Output()
		if err != nil {
			return errorMsg{err: fmt.Sprintf("git checkout failed: %v", err)}
		}
		// Refresh after checkout
		return gitStatusMsg{repoIdx: m.selectedRepo, output: ""}
	}
}
```

**How this maps to Yazi:**

| Yazi | Bubble Tea |
|------|-----------|
| `Scheduler::file(FileIn::Copy)` | `return m, m.stageSelected()` |
| `tokio::spawn(async { ... })` | `tea.Cmd` function |
| `AppProxy::update_progress(summary)` | `return gitStatusMsg{...}` |
| `emit!(Call(app:update_progress))` | Bubble Tea runtime delivers the message to `Update()` |

**The beautiful part:** You don't manage channels, threads, or locks. You just return a `tea.Cmd`, and Bubble Tea handles the concurrency.

---

## Rendering with lipgloss

Bubble Tea uses `lipgloss` for styling. The `View()` function returns a string that represents the entire screen.

```go
// view.go
package main

import (
	"fmt"
	"strings"
	"github.com/charmbracelet/lipgloss"
)

// View renders the entire UI. Bubble Tea calls this after every Update().
// It's equivalent to Yazi's Root::render().
func (m Model) View() string {
	if m.width == 0 || m.height == 0 {
		return "Loading..."
	}

	var sections []string

	// Always render the base screen
	if m.repoDetailActive {
		sections = append(sections, m.viewRepoDetail())
	} else {
		sections = append(sections, m.viewDashboard())
	}

	// Render modals on top (back to front, like Yazi's painter's algorithm)
	if m.branchPickerOpen {
		sections = append(sections, m.viewBranchPickerOverlay())
	}
	if m.commitDialogOpen {
		sections = append(sections, m.viewCommitDialogOverlay())
	}
	if m.helpVisible {
		sections = append(sections, m.viewHelpOverlay())
	}
	if m.notification != "" {
		sections = append(sections, m.viewNotification())
	}

	// Join everything
	return lipgloss.JoinVertical(lipgloss.Left, sections...)
}
```

**Wait — how do overlays work if we're just returning a string?**

In Bubble Tea, overlays are typically handled differently than in Yazi. You either:
1. **Replace the entire view** when a modal is open
2. **Use a layout that conditionally shows panels**

Here's the more common Bubble Tea approach:

```go
func (m Model) View() string {
	// Main view depends on base screen
	var mainView string
	if m.repoDetailActive {
		mainView = m.viewRepoDetail()
	} else {
		mainView = m.viewDashboard()
	}

	// If a modal is open, show it instead of or on top of the main view
	switch m.activeScreen() {
	case ScreenCommitDialog:
		return m.viewCommitDialog(mainView)
	case ScreenBranchPicker:
		return m.viewBranchPicker(mainView)
	case ScreenHelp:
		return m.viewHelp(mainView)
	default:
		return mainView
	}
}
```

### Dashboard View

```go
func (m Model) viewDashboard() string {
	// Title bar
	titleStyle := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#00FFFF")).
		Bold(true)
	title := titleStyle.Render(" Gitz — Git TUI ")

	// Repo list
	var repoLines []string
	for i, repo := range m.repos {
		style := lipgloss.NewStyle()
		if i == m.selectedRepo {
			style = style.Background(lipgloss.Color("#0000FF")).Foreground(lipgloss.Color("#FFFFFF"))
		} else if repo.dirty {
			style = style.Foreground(lipgloss.Color("#FFFF00"))
		}
		line := fmt.Sprintf(" %s [%s ↑%d↓%d]", repo.name, repo.branch, repo.ahead, repo.behind)
		repoLines = append(repoLines, style.Render(line))
	}
	repoList := lipgloss.JoinVertical(lipgloss.Left, repoLines...)
	repoBox := lipgloss.NewStyle().
		Border(lipgloss.RoundedBorder()).
		BorderForeground(lipgloss.Color("#555555")).
		Width(m.width - 2).
		Height(m.height - 6).
		Render(repoList)

	// Status bar
	statusBar := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#666666")).
		Render(fmt.Sprintf(" %d repos | j/k: navigate | Enter: open | ?: help | q: quit ", len(m.repos)))

	return lipgloss.JoinVertical(lipgloss.Left, title, repoBox, statusBar)
}
```

### Repo Detail View

```go
func (m Model) viewRepoDetail() string {
	repo := m.repos[m.selectedRepo]

	// Header
	headerStyle := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#00FF00")).
		Bold(true)
	header := headerStyle.Render(fmt.Sprintf(" %s — %s ", repo.name, repo.branch))

	// Status table
	var rows []string
	for i, item := range m.statusItems {
		statusColor := "#FFFFFF"
		switch item.status {
		case 'M':
			statusColor = "#FFFF00"
		case 'A':
			statusColor = "#00FF00"
		case 'D':
			statusColor = "#FF0000"
		case '?':
			statusColor = "#0088FF"
		}
		statusStyle := lipgloss.NewStyle().Foreground(lipgloss.Color(statusColor))
		rowStyle := lipgloss.NewStyle()
		if i == m.selectedStatus {
			rowStyle = rowStyle.Background(lipgloss.Color("#0000FF")).Foreground(lipgloss.Color("#FFFFFF"))
		}
		line := fmt.Sprintf(" %s  %s", statusStyle.Render(string(item.status)), item.path)
		rows = append(rows, rowStyle.Render(line))
	}
	tableContent := lipgloss.JoinVertical(lipgloss.Left, rows...)
	statusBox := lipgloss.NewStyle().
		Border(lipgloss.RoundedBorder()).
		BorderForeground(lipgloss.Color("#555555")).
		Width(m.width - 2).
		Height(m.height - 8).
		Render(tableContent)

	// Key hints
	hints := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#666666")).
		Render(" s: stage | u: unstage | c: commit | b: branches | ?: help | Esc: back ")

	return lipgloss.JoinVertical(lipgloss.Left, header, statusBox, hints)
}
```

---

## Modals and Popups

In Bubble Tea, modals are typically rendered **instead of** the main view, or as **separate screens**. Here's the commit dialog:

```go
func (m Model) viewCommitDialog(background string) string {
	// The background is dimmed by not rendering it — we just show the dialog
	dialogWidth := int(float64(m.width) * 0.6)
	dialogHeight := 7

	// Dialog border
	borderStyle := lipgloss.NewStyle().
		Border(lipgloss.RoundedBorder()).
		BorderForeground(lipgloss.Color("#00FF00")).
		Width(dialogWidth).
		Height(dialogHeight)

	// Title
	title := lipgloss.NewStyle().
		Bold(true).
		Foreground(lipgloss.Color("#00FF00")).
		Render(" Commit ")

	// Input area
	inputBox := lipgloss.NewStyle().
		Border(lipgloss.NormalBorder()).
		Width(dialogWidth - 4).
		Render(m.commitMessage)

	// Hint
	hint := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#666666")).
		Render("Enter: confirm | Esc: cancel")

	content := lipgloss.JoinVertical(lipgloss.Left,
		"Enter commit message:",
		inputBox,
		"",
		hint,
	)

	dialog := borderStyle.Render(content)

	// Center the dialog on screen
	return m.placeCenter(dialog, background)
}

func (m Model) placeCenter(dialog, background string) string {
	// Simple centering: pad the dialog vertically to center it
	bgLines := strings.Split(background, "\n")
	dialogLines := strings.Split(dialog, "\n")

	paddingTop := (len(bgLines) - len(dialogLines)) / 2
	if paddingTop < 0 {
		paddingTop = 0
	}

	var result []string
	for i := 0; i < paddingTop; i++ {
		result = append(result, "")
	}
	result = append(result, dialogLines...)

	return strings.Join(result, "\n")
}
```

**Alternative approach:** Some Bubble Tea apps use **separate Bubble Tea models** for modals and compose them together. This is closer to Yazi's actor pattern:

```go
// Separate model for commit dialog
type CommitModel struct {
	text   string
	cursor int
	width  int
	height int
}

func (c CommitModel) Init() tea.Cmd { return nil }

func (c CommitModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyMsg:
		switch msg.String() {
		case "esc":
			return c, func() tea.Msg { return closeCommitMsg{} }
		case "enter":
			return c, func() tea.Msg { return doCommitMsg{text: c.text} }
		case "backspace":
			if c.cursor > 0 {
				c.text = c.text[:c.cursor-1] + c.text[c.cursor:]
				c.cursor--
			}
		default:
			if len(msg.String()) == 1 {
				c.text = c.text[:c.cursor] + msg.String() + c.text[c.cursor:]
				c.cursor++
			}
		}
	}
	return c, nil
}

func (c CommitModel) View() string {
	// Render the commit dialog
}
```

---

## Sub-Models as "Actors"

If you want to go full Yazi-style, you can decompose your app into **sub-models** that act like actors. This is Bubble Tea's idiomatic way to structure large apps.

```go
// actor.go
package main

import tea "github.com/charmbracelet/bubbletea"

// Actor is the interface for sub-models (like Yazi's Actor trait)
type Actor interface {
	tea.Model
	Name() string
	CanHandle(msg tea.Msg) bool
}

// DashboardActor handles the dashboard screen
type DashboardActor struct {
	repos        []Repo
	selectedRepo int
	width        int
	height       int
}

func (d DashboardActor) Name() string { return "dashboard" }

func (d DashboardActor) CanHandle(msg tea.Msg) bool {
	_, ok := msg.(dashboardMsg)
	return ok
}

func (d DashboardActor) Init() tea.Cmd { return nil }

func (d DashboardActor) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyMsg:
		switch msg.String() {
		case "j", "down":
			if d.selectedRepo < len(d.repos)-1 {
				d.selectedRepo++
			}
		case "k", "up":
			if d.selectedRepo > 0 {
				d.selectedRepo--
			}
		}
	}
	return d, nil
}

func (d DashboardActor) View() string {
	// Render dashboard
	return ""
}
```

**However**, in practice most Bubble Tea apps don't use this "actor" pattern. Instead, they use a **flat model with helper functions** — which is what we've shown above. The actor pattern is more useful when you have truly independent subsystems.

---

## A Complete Working Example

Here's a minimal but complete Bubble Tea app that demonstrates all the concepts:

```go
// main.go
package main

import (
	"fmt"
	"os"
	"time"

	tea "github.com/charmbracelet/bubbletea"
	"github.com/charmbracelet/lipgloss"
)

// ─── Messages ─────────────────────────────────────────────

type tickMsg time.Time

// ─── Model ────────────────────────────────────────────────

type model struct {
	items      []string
	selected   int
	showModal  bool
	modalInput string
	showHelp   bool
	width      int
	height     int
}

func initialModel() model {
	return model{
		items: []string{
			"Item 1 — Press 'c' for modal",
			"Item 2 — Press '?' for help",
			"Item 3 — Press 'q' to quit",
		},
	}
}

func (m model) Init() tea.Cmd {
	return tea.Tick(100*time.Millisecond, func(t time.Time) tea.Msg {
		return tickMsg(t)
	})
}

// ─── Update (Dispatcher + Executor) ───────────────────────

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {

	case tea.KeyMsg:
		return m.handleKey(msg)

	case tea.WindowSizeMsg:
		m.width = msg.Width
		m.height = msg.Height
		return m, nil

	case tickMsg:
		return m, tea.Tick(100*time.Millisecond, func(t time.Time) tea.Msg {
			return tickMsg(t)
		})

	default:
		return m, nil
	}
}

func (m model) handleKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	// Route by active layer
	if m.showModal {
		return m.handleModalKey(msg)
	}
	if m.showHelp {
		return m.handleHelpKey(msg)
	}
	return m.handleMainKey(msg)
}

func (m model) handleMainKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.String() {
	case "q", "ctrl+c":
		return m, tea.Quit
	case "j", "down":
		if m.selected < len(m.items)-1 {
			m.selected++
		}
	case "k", "up":
		if m.selected > 0 {
			m.selected--
		}
	case "c":
		m.showModal = true
		m.modalInput = ""
	case "?":
		m.showHelp = true
	}
	return m, nil
}

func (m model) handleModalKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.String() {
	case "esc":
		m.showModal = false
	case "enter":
		m.items = append(m.items, fmt.Sprintf("Added: %s", m.modalInput))
		m.showModal = false
	case "backspace":
		if len(m.modalInput) > 0 {
			m.modalInput = m.modalInput[:len(m.modalInput)-1]
		}
	default:
		if len(msg.String()) == 1 {
			m.modalInput += msg.String()
		}
	}
	return m, nil
}

func (m model) handleHelpKey(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.String() {
	case "esc", "q", "?", "h":
		m.showHelp = false
	}
	return m, nil
}

// ─── View (Renderer) ──────────────────────────────────────

func (m model) View() string {
	if m.showModal {
		return m.viewModal()
	}
	if m.showHelp {
		return m.viewHelp()
	}
	return m.viewMain()
}

func (m model) viewMain() string {
	title := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#00FFFF")).
		Bold(true).
		Render(" Gitz Demo ")

	var items []string
	for i, item := range m.items {
		style := lipgloss.NewStyle()
		if i == m.selected {
			style = style.Background(lipgloss.Color("#0000FF")).Foreground(lipgloss.Color("#FFFFFF"))
		}
		items = append(items, style.Render(" "+item))
	}
	list := lipgloss.JoinVertical(lipgloss.Left, items...)
	listBox := lipgloss.NewStyle().
		Border(lipgloss.RoundedBorder()).
		Width(m.width - 2).
		Height(m.height - 4).
		Render(list)

	status := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#666666")).
		Render(" j/k: navigate | c: modal | ?: help | q: quit ")

	return lipgloss.JoinVertical(lipgloss.Left, title, listBox, status)
}

func (m model) viewModal() string {
	// Show main view dimmed + modal on top
	mainView := m.viewMain()

	dialog := lipgloss.NewStyle().
		Border(lipgloss.RoundedBorder()).
		BorderForeground(lipgloss.Color("#00FF00")).
		Width(40).
		Height(7).
		Render(fmt.Sprintf(
			" Modal \n\n %s\n\n Enter: confirm | Esc: cancel ",
			m.modalInput,
		))

	// Simple overlay: just return the dialog
	// For a true overlay, you'd need to composite the strings
	_ = mainView
	return dialog
}

func (m model) viewHelp() string {
	mainView := m.viewMain()

	helpText := lipgloss.JoinVertical(lipgloss.Left,
		"j / k / ↑ / ↓ — Navigate",
		"c — Open modal",
		"? / h — Toggle help",
		"q — Quit",
		"Esc — Close modal/help",
	)

	helpBox := lipgloss.NewStyle().
		Border(lipgloss.RoundedBorder()).
		BorderForeground(lipgloss.Color("#00FFFF")).
		Width(50).
		Height(10).
		Render(helpText)

	_ = mainView
	return helpBox
}

// ─── Main ─────────────────────────────────────────────────

func main() {
	p := tea.NewProgram(initialModel(), tea.WithAltScreen())
	if _, err := p.Run(); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}
```

Run it:
```bash
go run main.go
```

Press `j/k` to navigate, `c` to open modal, type something, press `Enter` to add, `?` for help, `q` to quit.

---

## Advanced Patterns

### Pattern 1: Sub-Model Composition

For complex apps, compose multiple Bubble Tea models:

```go
type Model struct {
	dashboard tea.Model
	detail    tea.Model
	commit    tea.Model
	active    tea.Model  // Points to whichever is active
}

func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	var cmd tea.Cmd
	m.active, cmd = m.active.Update(msg)
	return m, cmd
}
```

### Pattern 2: tea.Batch for Multiple Commands

Run multiple background tasks simultaneously:

```go
func (m Model) refreshAll() tea.Cmd {
	var cmds []tea.Cmd
	for i := range m.repos {
		cmds = append(cmds, m.refreshRepo(i))
	}
	return tea.Batch(cmds...)
}
```

### Pattern 3: tea.Sequence for Ordered Commands

Run commands one after another:

```go
func (m Model) stageAndCommit(file, message string) tea.Cmd {
	return tea.Sequence(
		m.stageFile(file),
		m.doCommit(message),
	)
}
```

### Pattern 4: Custom tea.Cmd for Timers

Auto-clear notifications:

```go
func clearNotificationAfter(delay time.Duration) tea.Cmd {
	return tea.Tick(delay, func(t time.Time) tea.Msg {
		return clearNotificationMsg{}
	})
}
```

### Pattern 5: tea.Exec for External Programs

Launch `$EDITOR` or `git diff`:

```go
func (m Model) openEditor() tea.Cmd {
	editor := os.Getenv("EDITOR")
	if editor == "" {
		editor = "vim"
	}
	c := exec.Command(editor, "some-file.txt")
	return tea.ExecProcess(c, func(err error) tea.Msg {
		return editorClosedMsg{err: err}
	})
}
```

---

## When Bubble Tea Differs from Yazi

| Aspect | Yazi | Bubble Tea |
|--------|------|-----------|
| **Event loop** | You write it (`loop { rx.recv() }`) | Framework provides it |
| **State mutation** | `&mut Core` (imperative) | Return new `Model` (functional) |
| **Concurrency** | `tokio::spawn` + `mpsc` | `tea.Cmd` (framework-managed) |
| **Background I/O** | Direct `tokio::process::Command` | `tea.Cmd` wrapping `exec.Command` |
| **Rendering** | Direct buffer writes (`ratatui::Buffer`) | String return (`View() string`) |
| **Styling** | `ratatui::style::Style` | `lipgloss.Style` |
| **Modals** | `Clear` widget + overlay | Usually full-screen replacement |
| **Testing** | Unit test actors individually | Unit test `Update` + `View` |

**The philosophical difference:**

Yazi says: *"You control everything. The borrow checker guarantees safety."*

Bubble Tea says: *"The framework controls the loop. You provide pure-ish functions. The GC handles memory."*

Both achieve the same outcome: **centralized state, message-driven updates, sequential execution, concurrent background work.**

---

## Summary

| Yazi Pattern | Bubble Tea Equivalent | Code |
|--------------|----------------------|------|
| `Core` | `Model` struct | `type Model struct { ... }` |
| `Core::layer()` | `activeScreen()` | `if m.modalOpen { return ScreenModal }` |
| `Event` enum | `tea.Msg` interface | `type gitStatusMsg struct { ... }` |
| `Dispatcher` | `Update()` method | `func (m Model) Update(msg tea.Msg)` |
| `Executor` | Type switch in `Update()` | `switch msg := msg.(type)` |
| `Actor::act()` | Sub-model `Update()` | `(subModel, cmd) := m.sub.Update(msg)` |
| `Scheduler` | `tea.Cmd` functions | `return m, m.fetchGitStatus()` |
| `emit!()` | Return a `tea.Cmd` | `return m, func() tea.Msg { ... }` |
| `Root::render()` | `View()` method | `func (m Model) View() string` |
| `tokio::spawn` | Bubble Tea runtime | Automatic — just return `tea.Cmd` |
| `mpsc` channel | Built into framework | You don't write it |

Bubble Tea makes the architecture **simpler** than Yazi because:
1. You don't write the event loop
2. You don't manage channels
3. You don't handle `&mut` / borrow checker
4. The framework guarantees serial `Update` calls

But it also gives you **less control**. You can't:
- Batch events like Yazi's `recv_many(&mut events, 50)`
- Throttle renders to 100fps
- Implement custom backpressure
- Share state between goroutines (you must send messages)

**For most TUI applications, Bubble Tea's trade-offs are excellent.** You get 90% of Yazi's architecture with 50% of the complexity.
