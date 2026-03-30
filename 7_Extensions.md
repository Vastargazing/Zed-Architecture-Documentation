# 7. Extensions: Plugin Architecture

The seventh architectural atom — how Zed supports third-party extensions for languages, themes, slash commands, and more. This covers **WASM sandboxing, the Extension trait, the host proxy, and the extension index**.

---

## 1) Extension Philosophy

Zed extensions run in **WebAssembly (WASM) sandboxes**. This is fundamentally different from VS Code's approach (Node.js processes with full OS access).

| Property | Zed Extensions | VS Code Extensions |
|----------|----------------|-------------------|
| Runtime | WASM sandbox | Node.js process |
| OS access | None (sandboxed) | Full |
| File access | Via delegate only | Direct |
| Performance | Near-native | JS overhead |
| Security | Strong isolation | Trust-based |
| Language | Rust (compiled to WASM) | JavaScript/TypeScript |

---

## 2) Architecture

```
┌───────────────────────────────────┐
│  Zed Application                   │
│                                    │
│  ┌─────────────────────────────┐  │
│  │ ExtensionStore               │  │
│  │ (manages all extensions)     │  │
│  └────────────┬────────────────┘  │
│               │                    │
│  ┌────────────▼────────────────┐  │
│  │ ExtensionHostProxy           │  │
│  │ (marshaling layer)           │  │
│  └────────────┬────────────────┘  │
│               │                    │
│  ┌────────────▼────────────────┐  │
│  │ WasmHost                     │  │
│  │ (Wasmtime runtime)           │  │
│  └────────────┬────────────────┘  │
│               │                    │
│  ┌────────────▼────────────────┐  │
│  │ WasmExtension[0]  │ [1] │ [2]│  │
│  │ (per-extension)              │  │
│  └──────────────────────────────┘  │
│                                    │
│  Delegates (dependency injection)  │
│  ├─ WorktreeDelegate              │
│  ├─ ProjectDelegate               │
│  └─ KeyValueStoreDelegate         │
└───────────────────────────────────┘
```

---

## 3) The Extension Trait

The core public API that extension authors implement (from the `extension_api` crate):

```rust
pub trait Extension: Send + Sync {
    // Required: factory method called by the WASM host
    fn new() -> Self where Self: Sized;

    // === Language Server Customization ===

    /// Returns the command to start a language server.
    /// `Worktree` is a WIT-generated type (not a trait object).
    fn language_server_command(
        &mut self,
        language_server_id: &LanguageServerId,
        worktree: &Worktree,
    ) -> Result<Command>;

    /// Initialization options passed to the LSP on startup
    fn language_server_initialization_options(
        &mut self,
        language_server_id: &LanguageServerId,
        worktree: &Worktree,
    ) -> Result<Option<serde_json::Value>>;

    /// Per-workspace configuration options for the LSP
    fn language_server_workspace_configuration(
        &mut self,
        language_server_id: &LanguageServerId,
        worktree: &Worktree,
    ) -> Result<Option<serde_json::Value>>;

    // === Slash Commands ===

    /// Run a slash command (e.g., /docs, /search)
    fn run_slash_command(
        &self,
        command: SlashCommand,
        args: Vec<String>,
        worktree: Option<&Worktree>,
    ) -> Result<SlashCommandOutput, String>;

    /// Provide completions for slash command arguments
    fn complete_slash_command_argument(
        &self,
        command: SlashCommand,
        args: Vec<String>,
    ) -> Result<Vec<SlashCommandArgumentCompletion>, String>;

    // === Debug Adapters ===

    /// Get the binary for a debug adapter.
    /// Accepts the adapter name, task definition, optional user-supplied path,
    /// and the active worktree.
    fn get_dap_binary(
        &mut self,
        adapter_name: String,
        config: DebugTaskDefinition,
        user_provided_debug_adapter_path: Option<String>,
        worktree: &Worktree,
    ) -> Result<DebugAdapterBinary, String>;

    // === Context Servers (MCP) ===

    /// Returns the command used to start a context server.
    /// `Project` is a WIT-generated type.
    fn context_server_command(
        &mut self,
        context_server_id: &ContextServerId,
        project: &Project,
    ) -> Result<Command>;
}
```

> **Note:** `Worktree` and `Project` here are WIT-generated types re-exported from the
> WASM component ABI — not Rust trait objects. All methods have default implementations
> that return an error, so extension authors only override what they need.

---

## 4) Delegates: Sandboxed Access

The extension host (in `crates/extension/src/extension.rs`) defines internal async traits
that bridge the WASM boundary to actual Zed internals. Extension authors receive
WIT-generated `Worktree` / `Project` values; these delegate traits are what lives
on the host side.

### WorktreeDelegate

```rust
#[async_trait]
pub trait WorktreeDelegate: Send + Sync + 'static {
    fn id(&self) -> u64;
    fn root_path(&self) -> String;
    // Note: async, takes RelPath (relative path inside worktree)
    async fn read_text_file(&self, path: &RelPath) -> Result<String>;
    // Returns the binary's path as a String if found
    async fn which(&self, binary_name: String) -> Option<String>;
    async fn shell_env(&self) -> Vec<(String, String)>;
}
```

Extensions can read files and locate binaries, but only within the worktree.
All methods are **async** — they go through the host message-passing layer.

### ProjectDelegate

```rust
pub trait ProjectDelegate: Send + Sync + 'static {
    fn worktree_ids(&self) -> Vec<u64>;
    // No direct worktree accessor — callers resolve worktrees via WorktreeDelegate
}
```

### KeyValueStoreDelegate

```rust
pub trait KeyValueStoreDelegate: Send + Sync + 'static {
    // Used exclusively by the `index_docs` extension method
    // to write indexed documentation into Zed's docs store.
    fn insert(&self, key: String, docs: String) -> Task<Result<()>>;
}
```

> **Important:** `KeyValueStoreDelegate` is **not** a general persistence store.
> It is a specialized interface for the `/docs` slash command's indexing pipeline.
> Extensions cannot persist arbitrary data between calls.

---

## 5) Extension Index

All installed extensions are cataloged in an index:

```rust
pub struct ExtensionIndex {
    pub extensions: BTreeMap<Arc<str>, ExtensionIndexEntry>,
    pub themes: BTreeMap<Arc<str>, ExtensionIndexThemeEntry>,
    pub icon_themes: BTreeMap<Arc<str>, ExtensionIndexIconThemeEntry>,
    pub languages: BTreeMap<LanguageName, ExtensionIndexLanguageEntry>,
}
```

### Extension Entry

```rust
pub struct ExtensionIndexEntry {
    pub manifest: Arc<ExtensionManifest>,  // reference-counted, not owned
    pub dev: bool,                         // true for local dev extensions
}

pub struct ExtensionManifest {
    pub id: Arc<str>,
    pub name: String,
    pub version: Arc<str>,
    pub schema_version: SchemaVersion,   // newtype over i32, not a bare u32

    pub description: Option<String>,
    pub repository: Option<String>,
    pub authors: Vec<String>,
    pub lib: LibManifestEntry,           // WASM binary metadata

    // Paths to directories/files — resolved at load time
    pub themes: Vec<PathBuf>,
    pub icon_themes: Vec<PathBuf>,
    pub languages: Vec<PathBuf>,

    // Named maps for structured entries
    pub grammars: BTreeMap<Arc<str>, GrammarManifestEntry>,
    pub language_servers: BTreeMap<LanguageServerName, LanguageServerManifestEntry>,
    pub context_servers: BTreeMap<Arc<str>, ContextServerManifestEntry>,
    pub agent_servers: BTreeMap<Arc<str>, AgentServerManifestEntry>,
    pub slash_commands: BTreeMap<Arc<str>, SlashCommandManifestEntry>,

    // Additional capabilities
    pub snippets: Option<ExtensionSnippets>,
    pub capabilities: Vec<ExtensionCapability>,
    pub debug_adapters: BTreeMap<Arc<str>, DebugAdapterManifestEntry>,
    pub debug_locators: BTreeMap<Arc<str>, DebugLocatorManifestEntry>,
    pub language_model_providers: BTreeMap<Arc<str>, LanguageModelProviderManifestEntry>,
}
```

---

## 6) What Extensions Can Provide

### Languages

Each language lives in its own **subdirectory** under `languages/` and contains
a `config.toml`. The top-level `extension.toml` simply lists the paths:

```toml
# extension.toml
languages = ["languages/my-lang"]

[grammars.my-lang]
repository = "https://github.com/..."
rev = "<git-sha>"
```

The `languages/my-lang/config.toml` defines the language metadata:

```toml
name = "My Language"
grammar = "my-lang"
path_suffixes = ["myl"]
line_comments = ["# "]
```

Tree-sitter query files (`highlights.scm`, `brackets.scm`, `indents.scm`, etc.)
also live in the language subdirectory.

### Themes

```toml
# extension.toml
themes = ["themes/my-theme.json"]
```

JSON theme files following Zed's theme schema.

### Language Servers

```toml
[language_servers.my-lsp]
name = "My LSP"
languages = ["My Language"]
```

Plus the `language_server_command()` implementation in the Extension trait.

### Slash Commands

Custom commands for the AI assistant:

```toml
[slash_commands.docs]
description = "Search documentation"
requires_argument = true
```

### Context Servers (MCP)

```toml
[context_servers.my-mcp-server]
# configuration fields
```

Model Context Protocol servers for AI features.

---

## 7) Extension Lifecycle

### Installation

```
User searches extensions in UI
    ↓
Download .tar.gz from Zed extension registry
    ↓
Extract to platform data directory:
  macOS  → ~/Library/Application Support/Zed/extensions/
  Linux  → $XDG_DATA_HOME/zed/extensions/  (~/.local/share/zed/extensions/)
  Windows→ %LOCALAPPDATA%\Zed\extensions\
    ↓
Build ExtensionIndex from manifests
    ↓
Load WASM binary into WasmHost
    ↓
Extension ready
```

### Development Mode

```
Command palette → "zed: install dev extension" → pick local directory
    ↓
Extension loaded from local path (marked dev: true in index)
    ↓
File watcher on extension directory
    ↓
On change → press "Rebuild" in Extensions panel → hot-reload
```

### Updates

```
Zed checks registry periodically
    ↓
New version available?
    ↓
Download → replace → reload
    ↓
No restart needed (WASM can be swapped)
```

---

## 8) WASM Sandboxing Details

### What the Sandbox Prevents

- No direct filesystem access (only through `WorktreeDelegate`)
- No arbitrary process spawning (only binaries discovered via `which()`)
- No shared memory with other extensions
- No access to other Zed internals (UI, buffers, settings)

### What the Sandbox Allows

- **HTTP requests** — via `http_client::fetch()` / `fetch_stream()` (WIT-exported by the host)
- Pure computation (parsing, formatting, analysis)
- Reading worktree files and environment through delegates
- GitHub release queries, Node.js package management helpers (host-side utilities)
- Returning structured data (completions, commands, labels, etc.)

### Performance

WASM runs at near-native speed through Wasmtime's JIT compiler. The overhead is primarily in the marshaling layer (serializing/deserializing data across the WASM boundary).

---

## 9) Extension API Versioning

Versioning uses **semver** (via Wasmtime's component model), not a single integer constant.
The host checks the WASM binary's embedded API version against a supported range:

```rust
// crates/extension_host/src/wasm_host/wit.rs
pub fn wasm_api_version_range(release_channel: ReleaseChannel) -> RangeInclusive<Version> {
    let max_version = match release_channel {
        // Dev / Nightly can test unreleased API versions
        ReleaseChannel::Dev | ReleaseChannel::Nightly => latest::MAX_VERSION,
        // Stable / Preview only accept the released range
        ReleaseChannel::Stable | ReleaseChannel::Preview => since_v0_6_0::MAX_VERSION,
    };
    since_v0_0_1::MIN_VERSION..=max_version
}
```

- Extensions whose API version falls **outside** the supported range are rejected at load time.
- New API features are staged on Dev/Nightly first, then promoted to Stable.
- The `extension_api` crate (published to crates.io as `zed_extension_api`) is the public
  API surface that extension authors target.

---

## 10) Anti-Patterns

### Doing Too Much Work in Extension Calls

```rust
// BAD: reading a large file just to extract one value
fn language_server_command(&mut self, id: &LanguageServerId, worktree: &Worktree) -> Result<Command> {
    let config = worktree.read_text_file("huge_config.json")?;
    // ... parse everything
    Ok(command)
}

// GOOD: do minimal work — let the LSP itself handle configuration
fn language_server_command(&mut self, _id: &LanguageServerId, worktree: &Worktree) -> Result<Command> {
    let binary = worktree.which("my-lsp").ok_or("my-lsp not found")?;
    Ok(Command { command: binary, args: vec![], env: vec![] })
}
```

### Relying on Global State

```rust
// BAD: mutable statics are undefined behaviour in WASM across invocations
static mut CACHE: Option<Data> = None;

// GOOD: for docs indexing, use the KeyValueStore passed to index_docs()
fn index_docs(&self, provider: String, package: String, db: &KeyValueStore) -> Result<(), String> {
    db.insert(format!("{}/{}", provider, package), docs_text)?;
    Ok(())
}
// NOTE: there is no general-purpose KV store for arbitrary extension state
```

---

## 11) Mental Model

```
Extension System
│
├─── ExtensionStore (app-level manager)
│    └─ ExtensionIndex
│       ├─ Extensions (id → manifest + WASM)
│       ├─ Themes (name → theme file)
│       ├─ Languages (name → grammar + queries)
│       └─ Language Servers (name → adapter)
│
├─── WasmHost (Wasmtime runtime)
│    └─ Per-extension WASM instance
│       ├─ Extension trait implementation
│       ├─ Sandboxed: no direct FS/process access
│       ├─ HTTP requests allowed via host-exported fetch()
│       └─ Communicates via WIT/delegates only
│
├─── Delegates (host-side async traits)
│    ├─ WorktreeDelegate → read files (async), find binaries (async)
│    ├─ ProjectDelegate  → list worktree IDs
│    └─ KeyValueStoreDelegate → insert indexed docs (docs pipeline only)
│
├─── Extension Capabilities
│    ├─ Languages + Tree-sitter grammars
│    ├─ Themes
│    ├─ Language server adapters
│    ├─ Slash commands
│    ├─ Context servers (MCP)
│    └─ Debug adapter configs
│
└─── Lifecycle
     ├─ Install from registry (download .tar.gz)
     ├─ Load WASM into sandbox
     ├─ Hot-reload in dev mode
     └─ Auto-update from registry
```

**Core insight:** Extensions run in WASM sandboxes with no direct filesystem or process access. They interact with Zed through well-defined WIT-exported interfaces (delegates, HTTP client, Node.js helpers). This provides strong security isolation while maintaining near-native performance. The extension API is versioned with semver — new API features are staged on Dev/Nightly before reaching Stable.
