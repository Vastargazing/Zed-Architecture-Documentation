# 9. Terminal & Tasks: Execution Environment

The ninth architectural atom — how Zed provides an integrated terminal and task execution system. This covers **terminal emulation, PTY management, task templates, variable substitution, and the run/debug flow**.

---

## 1) Terminal Architecture

### Built on Alacritty

Zed's terminal is built on top of the `alacritty_terminal` crate — the same terminal emulator that powers the Alacritty terminal. This gives Zed:

- Full VT100/VT220 terminal emulation
- 256-color and truecolor support
- Mouse tracking
- Alternate screen buffer (for vim, less, etc.)

### Core Structure

```
Terminal (Entity<Terminal>)
├── alacritty_terminal::Term  ← actual terminal state machine
├── PTY process               ← shell subprocess (bash, zsh, pwsh, etc.)
├── Event stream              ← output from PTY → Term → UI
└── Input handling            ← keyboard/mouse → PTY
```

---

## 2) PTY (Pseudo Terminal)

A PTY is an OS-level construct that provides a virtual terminal device:

```
┌──────────┐    stdin     ┌──────────────┐
│  Zed     │ ──────────> │  Shell       │
│  (master)│             │  (bash/zsh)  │
│          │ <────────── │              │
│          │    stdout    │              │
└──────────┘             └──────────────┘
```

### Platform Differences

| Platform | PTY Implementation |
|----------|-------------------|
| macOS/Linux | POSIX `openpty()` / `forkpty()` |
| Windows | Windows ConPTY API |

---

## 3) Terminal Events

```rust
pub enum Event {
    TitleChanged,                          // Shell set window title
    BreadcrumbsChanged,                    // CWD changed
    CloseTerminal,                         // Process exited
    Bell,                                  // BEL character received
    Wakeup,                                // New output available
    BlinkChanged(bool),                    // Cursor blink state
    SelectionsChanged,                     // User selected text
    NewNavigationTarget(MaybeNavigationTarget),  // Hyperlink detected
    Open(MaybeNavigationTarget),           // User clicked link
}

pub enum MaybeNavigationTarget {
    Url(String),                           // HTTP/HTTPS URLs
    PathLike(PathLikeTarget),              // "file.rs:1:23" pattern
}
```

### PathLike Detection

The terminal detects file references in output:

```
$ cargo build
error[E0308]: mismatched types
 --> src/main.rs:10:5
```

`src/main.rs:10:5` is detected as a `PathLikeTarget`. Ctrl+Click opens the file at that position.

---

## 4) Terminal Environment

When spawning a shell, Zed sets these environment variables:

```
ZED_TERM=true              # Detect we're inside Zed's terminal
TERM_PROGRAM=zed           # Standard terminal identification
TERM=xterm-256color        # Terminal capabilities
COLORTERM=truecolor        # 24-bit color support
```

Plus the inherited environment from `ProjectEnvironment` (which may include direnv, shell-specific vars, etc.)

---

## 5) Terminal Actions

```rust
actions!(terminal, [
    Copy, Paste, Clear,
    ScrollUp, ScrollDown, ScrollPageUp, ScrollPageDown, 
    ScrollHalfPageUp, ScrollHalfPageDown, ScrollToTop, ScrollToBottom,
    ToggleViMode,    // Vim-like navigation in terminal
    SelectAll,
    SearchTest,      // Find in terminal output
]);
```

---

## 6) Task System Overview

Tasks in Zed are **templated shell commands** that can be saved, shared, and triggered via keybindings or the command palette.

### Where Tasks Come From

```
1. .zed/tasks.json (project-level)
2. Global tasks from settings
3. Language runnables (detected from code via Tree-sitter queries)
4. Extension-provided tasks
```

---

## 7) Task Template Structure

```rust
pub struct SpawnInTerminal {
    pub id: TaskId,                    // Terminal tab affinity
    pub command: Option<String>,       // The executable
    pub args: Vec<String>,             // Command arguments
    pub cwd: Option<PathBuf>,          // Working directory
    pub env: HashMap<String, String>,  // Environment variables
    pub use_new_terminal: bool,        // Reuse existing or new tab
    pub allow_concurrent_runs: bool,   // Multiple instances allowed?
    pub reveal: RevealStrategy,        // When to show the terminal
    pub hide: HideStrategy,            // When to hide it
    pub shell: Shell,                  // Which shell to use
    pub save: SaveStrategy,            // Save buffers before running?
}
```

### Example: tasks.json

```json
[
  {
    "label": "Run tests",
    "command": "cargo",
    "args": ["test", "--package", "$ZED_PACKAGE"],
    "cwd": "$WORKTREE_ROOT",
    "env": { "RUST_LOG": "debug" },
    "use_new_terminal": false,
    "reveal": "always"
  },
  {
    "label": "Run current file",
    "command": "python",
    "args": ["$FILE"],
    "cwd": "$WORKTREE_ROOT"
  }
]
```

---

## 8) Variable Substitution

Tasks support **13+ predefined variables** that are resolved at execution time:

| Variable | Description | Example |
|----------|-------------|---------|
| `$FILE` | Full path to current file | `/home/user/project/src/main.rs` |
| `$RELATIVE_FILE` | Relative path from worktree root | `src/main.rs` |
| `$FILENAME` | Just the filename | `main.rs` |
| `$STEM` | Filename without extension | `main` |
| `$ROW` | Cursor row (1-based) | `42` |
| `$COLUMN` | Cursor column (1-based) | `10` |
| `$SELECTED_TEXT` | Currently selected text | `println!("hello")` |
| `$WORKTREE_ROOT` | Root directory of current worktree | `/home/user/project` |
| `$SYMBOL` | Symbol under cursor (from Tree-sitter) | `my_function` |
| `$LANGUAGE` | Language of current buffer | `Rust` |
| `$RUNNABLE_SYMBOL` | Symbol detected by runnables.scm | `test_my_function` |
| `$PICK_PROCESS_ID` | Interactive process picker (for debug) | `12345` |
| `$CUSTOM_*` | User-defined variables | `$CUSTOM_MY_VAR` |

### Resolution Flow

```
TaskTemplate (with $VARIABLES)
    ↓
Resolve variables from:
├── Current editor state (file, cursor, selection)
├── Project state (worktree root, language)
├── Tree-sitter queries (symbol, runnable)
└── User input (for $CUSTOM_* or $PICK_*)
    ↓
ResolvedTask (concrete command string)
    ↓
SpawnInTerminal (execute in terminal)
```

---

## 9) Language Runnables

Tree-sitter queries can detect "runnable" patterns in code:

```scheme
;; runnables.scm for Rust
(
  (attribute_item (attribute (identifier) @run_tag (#eq? @run_tag "test")))
  .
  (function_item name: (identifier) @run)
)
```

This detects `#[test]` functions and makes them runnable:

```rust
#[test]                    ← detected by runnables.scm
fn test_addition() {       ← $RUNNABLE_SYMBOL = "test_addition"
    assert_eq!(2 + 2, 4);
}
```

The user sees a "Run" gutter icon next to test functions.

---

## 10) Task Reveal and Hide Strategies

### RevealStrategy

```rust
pub enum RevealStrategy {
    Always,    // Always show terminal pane when task starts
    Never,     // Don't show (run silently in background)
}
```

### HideStrategy

```rust
pub enum HideStrategy {
    Never,              // Terminal stays visible after task ends
    Always,             // Hide terminal after task ends
    OnSuccess,          // Hide only if exit code = 0
}
```

### SaveStrategy

```rust
pub enum SaveStrategy {
    None,               // Don't save anything
    CurrentFile,        // Save current buffer only
    AllModifiedFiles,   // Save all dirty buffers
}
```

---

## 11) Terminal Reuse

**`use_new_terminal: false`** → Zed reuses a terminal tab with the same `TaskId`:

```
First run: "cargo test" → creates Terminal tab [cargo test]
Second run: "cargo test" → reuses same tab, kills old process
Third run: "cargo build" (different task) → creates new tab
```

**`use_new_terminal: true`** → always creates a fresh terminal tab.

**`allow_concurrent_runs: true`** → multiple instances of the same task can run simultaneously.

---

## 12) Shell Configuration

```rust
pub enum Shell {
    System,                             // Default system shell
    Program(String),                    // Specific shell ("/bin/bash")
    WithArguments { program, args },    // Shell with custom args
}
```

Users can configure the shell per-task or globally in settings:

```json
{
  "terminal": {
    "shell": {
      "program": "/usr/bin/fish"
    }
  }
}
```

---

## 13) Anti-Patterns

### Forgetting to Set CWD

```json
// BAD: runs in Zed's process directory (undefined)
{ "command": "cargo", "args": ["build"] }

// GOOD: explicit working directory
{ "command": "cargo", "args": ["build"], "cwd": "$WORKTREE_ROOT" }
```

### Blocking the UI with Terminal Output

```rust
// BAD: processing all terminal output synchronously
while let Some(byte) = pty.read() {
    terminal.process_byte(byte); // blocks UI thread!
}

// GOOD: Zed processes terminal output in batches
// Wakeup event triggers UI refresh
// Alacritty's Term handles buffering
```

---

## 14) Mental Model

```
Terminal & Task System
│
├─── Terminal (Entity<Terminal>)
│    ├─ Alacritty terminal emulator (Term)
│    ├─ PTY subprocess (shell process)
│    ├─ Event stream (output, title, links)
│    ├─ Vi mode (optional vim-like navigation)
│    └─ Environment (ZED_TERM, TERM_PROGRAM, etc.)
│
├─── Task System
│    ├─ Task Templates (from .zed/tasks.json or settings)
│    ├─ Variable Substitution ($FILE, $ROW, $SYMBOL, etc.)
│    ├─ Resolution: Template → ResolvedTask → SpawnInTerminal
│    └─ Reveal/Hide/Save strategies
│
├─── Language Runnables
│    ├─ Tree-sitter queries (runnables.scm)
│    ├─ Detected patterns → Run button in gutter
│    └─ $RUNNABLE_SYMBOL variable
│
└─── Integration
     ├─ Terminal panel (bottom dock)
     ├─ Individual terminal tabs (in Pane)
     ├─ Task reuse by TaskId
     └─ PathLike detection (click to open file)
```

**Core insight:** Terminal is built on Alacritty for full terminal emulation. Tasks are templated commands with variable substitution, making them portable and context-aware. Language runnables use Tree-sitter to detect runnable patterns (tests, main functions) directly in code.
