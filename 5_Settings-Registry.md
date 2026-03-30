# 5. Settings & Registry: Configuration System

The fifth architectural atom — how Zed resolves configuration at every level. This covers **hierarchical settings, keymap binding, settings traits, and the registry pattern**.

---

## 1) The Settings Problem

A code editor needs settings at multiple levels:
- **Defaults** — shipped with the app
- **User** — `~/.config/zed/settings.json`
- **Server** — pushed from Zed's cloud
- **Project** — `.zed/settings.json` in the workspace
- **Language-specific** — different tab width for Python vs Rust
- **File-specific** — read-only for generated files

These must be merged with clear precedence, and changes must propagate instantly.

---

## 2) Settings Precedence Chain

```
Default (lowest priority)
    ↓ overridden by
Global (user settings.json)
    ↓ overridden by
Server (pushed from cloud)
    ↓ overridden by
Project (.zed/settings.json)    ← highest for general settings
    ↓
Language-specific overrides
    ↓
File-specific overrides (path matchers)
```

**Rule:** Higher-priority sources override lower-priority sources field-by-field (not replace entirely). If project settings specify `tab_size: 4` but don't mention `font_size`, the user's `font_size` still applies.

---

## 3) SettingsStore: The Global Manager

```rust
pub struct SettingsStore {
    setting_values: HashMap<TypeId, Box<dyn AnySettingValue>>,
    // ... raw sources, local settings, keymap
}
```

The store holds **type-erased** setting values indexed by `TypeId`. Each registered setting type gets one slot.

### Setting Registration

Settings are registered via the `inventory` crate (compile-time collection):

```rust
// In the crate that defines the setting:
#[derive(Deserialize, Default)]
pub struct EditorSettings {
    pub tab_size: Option<u32>,
    pub soft_wrap: Option<SoftWrap>,
    pub show_line_numbers: Option<bool>,
    // ... all fields are Option<T> for partial override
}

impl Settings for EditorSettings {
    const KEY: Option<&'static str> = Some("editor");
    // ...
}

// Registration happens automatically via inventory::collect!
```

### Why All Fields Are `Option<T>`

Each settings source might only override some fields:

```json
// User settings
{ "editor": { "tab_size": 4 } }

// Project settings
{ "editor": { "soft_wrap": "editor_width" } }

// Merged result:
// tab_size: 4 (from user), soft_wrap: "editor_width" (from project)
```

If fields were required, every source would need to specify everything.

---

## 4) The Settings Trait

```rust
pub trait Settings: 'static + Send + Sync {
    /// JSON key in the settings file (e.g., "editor", "terminal")
    const KEY: Option<&'static str>;

    /// The deserialization type (usually Self or a FileContent wrapper)
    type FileContent: for<'de> Deserialize<'de> + Default;

    /// Load from deserialized file content
    fn load(sources: SettingsSources<Self::FileContent>, cx: &App) -> Result<Self>
    where
        Self: Sized;
}
```

### SettingsSources

```rust
pub struct SettingsSources<T> {
    pub default: &'static T,
    pub user: Option<&'static T>,
    pub server: Option<&'static T>,
    pub project: &'static [&'static T],  // Multiple project files possible
}
```

The `load()` method receives all sources and merges them. The typical implementation uses a cascading merge:

```rust
fn load(sources: SettingsSources<Self::FileContent>, _cx: &App) -> Result<Self> {
    sources.json_merge()
}
```

---

## 5) Accessing Settings

### Global Access

```rust
// Read a setting from anywhere with App context:
let tab_size = EditorSettings::get_global(cx).tab_size;

// Or with a specific scope (file path + language):
let settings = EditorSettings::get(Some(&language_settings_location), cx);
```

### Language-Specific Overrides

Settings can vary by language:

```json
{
  "languages": {
    "Python": {
      "tab_size": 4,
      "soft_wrap": "editor_width"
    },
    "Rust": {
      "tab_size": 4,
      "formatter": "language_server"
    }
  }
}
```

### File-Specific Overrides

The `.zed/settings.json` supports path matchers:

```json
{
  "file_types": {
    "*.generated.rs": {
      "read_only": true
    }
  }
}
```

---

## 6) Settings Change Propagation

When a settings file changes:

```
User edits settings.json
    ↓
File watcher detects change
    ↓
SettingsStore::load_user_settings(json_content)
    ↓
Deserialize into FileContent
    ↓
Re-run load() for each registered Setting type
    ↓
cx.notify() → all observers of settings refresh
    ↓
UI re-renders with new settings
```

**Key property:** Settings changes are synchronous within a frame. After `load_user_settings()` returns, all subsequent reads see the new values.

---

## 7) Keymap System

### Keymap File Structure

```json
[
  {
    "context": "Editor && !menu",
    "bindings": {
      "ctrl-z": "editor::Undo",
      "ctrl-shift-z": "editor::Redo",
      "ctrl-c": "editor::Copy"
    }
  },
  {
    "context": "Workspace",
    "bindings": {
      "ctrl-shift-p": "command_palette::Toggle"
    }
  }
]
```

### Context Matching

Each binding block has a `context` predicate that matches against the focused element's `KeyContext`. The syntax is boolean expressions:

```
"Editor"                        → true if Editor is in focus path
"Editor && vim_mode == normal"  → true if Editor + vim normal mode
"Editor && !menu"               → true if Editor and no menu open
"Terminal"                      → true if Terminal is focused
```

### Keymap Resolution Flow

```
Keystroke: Ctrl+Z
    ↓
Walk DispatchTree from focused node → root
    ↓
At each node, get KeyContext
    ↓
Check all keymap entries where context matches
    ↓
Return first matching binding → Action
    ↓
Dispatch Action through focus tree
```

### Keymap Layering

```
Default keybindings (shipped with Zed)
    ↓ overridden by
User keybindings (~/.config/zed/keymap.json)
    ↓ overridden by
Extension keybindings
```

Later bindings override earlier ones for the same keystroke + context.

---

## 8) Local Settings: Per-Directory Configuration

`.zed/settings.json` in any directory applies only to files in that directory:

```
project/
├── .zed/
│   └── settings.json          ← applies to all files in project/
├── frontend/
│   └── .zed/
│       └── settings.json      ← applies only to frontend/ files
└── backend/
    └── .zed/
        └── settings.json      ← applies only to backend/ files
```

### LocalSettingsPath

```rust
// Distinguishes between in-worktree vs. external settings
enum LocalSettingsPath {
    InWorktree { worktree_id, relative_path },
    OutsideWorktree { abs_path },
}
```

---

## 9) Theme and Font Settings

Themes and fonts are special settings that require extra processing:

```json
{
  "theme": "One Dark",
  "buffer_font_family": "JetBrains Mono",
  "buffer_font_size": 14,
  "ui_font_family": "Inter",
  "ui_font_size": 13
}
```

Theme changes trigger a full UI re-render. Font changes may trigger text layout reflow.

---

## 10) Feature Flags

```rust
pub struct FeatureFlags {
    // Controlled by server, not user settings
    pub flags: HashSet<String>,
}
```

Feature flags come from the Zed server — they're not user-configurable. Used for gradual rollouts of new features.

---

## 11) Settings Validation

Settings are validated during load. Invalid values are:
- Logged as warnings
- Replaced with defaults

The system never crashes on bad settings — it degrades gracefully.

---

## 12) Anti-Patterns

### Reading Settings Without Scope

```rust
// BAD: ignores language/project overrides
let tab_size = EditorSettings::get_global(cx).tab_size;
// Always returns the global value, ignoring Python's tab_size: 4

// GOOD: pass the language/file context
let location = SettingsLocation { worktree_id, path };
let tab_size = EditorSettings::get(Some(&location), cx).tab_size;
```

### Caching Settings

```rust
// BAD: cached value becomes stale when settings change
struct MyFeature {
    tab_size: u32,  // Cached on creation, never updated
}

// GOOD: read from SettingsStore on every use
fn tab_size(&self, cx: &App) -> u32 {
    EditorSettings::get_global(cx).tab_size
}
```

### Mutating Settings Directly

```rust
// BAD: bypasses the merge chain
settings_store.set_field_directly("tab_size", 4);

// GOOD: write to the appropriate settings file
// The SettingsStore will re-merge all sources
```

---

## 13) Mental Model

```
SettingsStore (global singleton)
│
├─── Registered Setting Types
│    ├─ EditorSettings (KEY = "editor")
│    ├─ TerminalSettings (KEY = "terminal")
│    ├─ ThemeSettings (KEY = "theme")
│    ├─ VimSettings (KEY = "vim")
│    └─ ... (each crate registers its own)
│
├─── Setting Sources (precedence chain)
│    ├─ Default (compiled-in)
│    ├─ User (~/.config/zed/settings.json)
│    ├─ Server (pushed from cloud)
│    └─ Project (.zed/settings.json per directory)
│
├─── Keymap
│    ├─ Default keybindings
│    ├─ User overrides (keymap.json)
│    └─ Context-based matching (Editor, Terminal, etc.)
│
├─── Resolution
│    ├─ Global: merge all sources
│    ├─ Per-language: merge + language overrides
│    └─ Per-file: merge + path matcher overrides
│
└─── Change Propagation
     ├─ File watcher detects change
     ├─ Re-merge all sources
     ├─ cx.notify() to all observers
     └─ UI re-renders with new values
```

**Core insight:** Settings are hierarchical, type-safe, and auto-merged. Every setting type is registered via `inventory` at compile time. Sources override field-by-field (not wholesale). Changes propagate synchronously through the observer system.
