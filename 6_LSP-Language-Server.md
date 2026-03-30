# 6. LSP & Language Server: Editor Intelligence

The sixth architectural atom — how Zed communicates with language servers to provide completions, diagnostics, go-to-definition, and more. This covers **the LSP client, server lifecycle, language registry, and multi-server coordination**.

---

## 1) Architecture Layers

```
┌────────────────────────────────────────────┐
│ Editor / UI                                 │
│ (completions menu, diagnostics gutter, etc.)│
└──────────────────────┬─────────────────────┘
                       │
┌──────────────────────▼─────────────────────┐
│ LspStore (Entity)                           │
│ Lifecycle, routing, deduplication            │
│ Variants: LocalLspStore / RemoteLspStore    │
└──────────────────────┬─────────────────────┘
                       │
┌──────────────────────▼─────────────────────┐
│ LanguageRegistry                            │
│ Maps languages → LSP adapters               │
└──────────────────────┬─────────────────────┘
                       │
┌──────────────────────▼─────────────────────┐
│ LanguageServer (lsp crate)                  │
│ JSON-RPC process wrapper                    │
└──────────────────────┬─────────────────────┘
                       │
              Subprocess (stdin/stdout)
              ┌────────▼────────────┐
              │ rust-analyzer       │
              │ typescript-language │
              │ pyright             │
              │ ...                 │
              └─────────────────────┘
```

---

## 2) LanguageServer: The RPC Client

The `lsp` crate provides a low-level JSON-RPC wrapper for communicating with language server processes.

```rust
pub struct LanguageServer {
    server_id: LanguageServerId,
    capabilities: RwLock<ServerCapabilities>,
    notification_handlers: Arc<Mutex<HashMap<&'static str, NotificationHandler>>>,
    response_handlers: Arc<Mutex<HashMap<RequestId, ResponseHandler>>>,
    io_tasks: Mutex<Option<(Task<...>, Task<...>)>>,
    // ... process handle, root path, etc.
}
```

### Message Flow

```
         Zed (client)                  Language Server (process)
              │                              │
              │──── Request (id, method) ───>│
              │                              │
              │<── Response (id, result) ────│
              │                              │
              │<── Notification (no id) ─────│
              │     (diagnostics, progress)  │
              │                              │
              │──── Notification ────────────>│
              │     (didOpen, didChange)     │
```

### Request/Response Correlation

Each request gets a unique `RequestId`. When a response arrives, it's matched with the pending request handler:

```rust
// Send request
let response = language_server
    .request::<lsp::request::Completion>(params)
    .await?;

// Under the hood:
// 1. Serialize {jsonrpc: "2.0", id: 42, method: "textDocument/completion", params}
// 2. Store ResponseHandler for id=42
// 3. When {id: 42, result: ...} arrives, invoke handler
// 4. Return deserialized result
```

### Capabilities

Each server reports its capabilities during initialization:

```rust
pub struct ServerCapabilities {
    pub completion_provider: Option<CompletionProvider>,
    pub hover_provider: Option<HoverProvider>,
    pub definition_provider: Option<OneOf<bool, DefinitionOptions>>,
    pub references_provider: Option<...>,
    pub rename_provider: Option<...>,
    pub code_action_provider: Option<...>,
    pub document_formatting_provider: Option<...>,
    // ... many more
}
```

Zed checks capabilities before sending requests. If the server doesn't support hover, no hover requests are sent.

---

## 3) LspStore: Server Lifecycle

`LspStore` manages the lifecycle of all language servers for a project.

### Two Variants

| Variant | Role |
|---------|------|
| `LocalLspStore` | On the project host — manages actual server processes |
| `RemoteLspStore` | On collaboration guests — proxies via RPC to host |

### Server Deduplication: LanguageServerSeed

Multiple files can use the same language server (e.g., all `.rs` files share one `rust-analyzer`). The dedup key is:

```
LanguageServerSeed = (worktree_id, server_name, toolchain, settings_hash)
```

If two files resolve to the same seed, they share one server instance.

### Server Lifecycle

```
File opened with language X
    ↓
LanguageRegistry: lookup adapters for language X
    ↓
Resolve LanguageServerSeed
    ↓
Already running? → reuse existing server
    ↓
Not running? → start new process
    ↓
LanguageServer::new(binary_path, args, root_dir)
    ↓
Send "initialize" request
    ↓
Receive ServerCapabilities
    ↓
Send "initialized" notification
    ↓
Send "textDocument/didOpen" for all relevant buffers
    ↓
Server is ready for requests
```

### Server Shutdown

```
Last file using server X is closed (or project closes)
    ↓
Send "shutdown" request
    ↓
Wait for response
    ↓
Send "exit" notification
    ↓
Terminate process
```

---

## 4) LanguageRegistry: Mapping Languages to Servers

```rust
pub struct LanguageRegistry {
    languages: Vec<Arc<Language>>,
    lsp_adapters: HashMap<LanguageName, Vec<CachedLspAdapter>>,
    // ... loading state, pending languages
}
```

### LspAdapter

Each language server type has an adapter that knows how to:

```rust
pub trait LspAdapter {
    fn name(&self) -> LanguageServerName;
    
    /// Get the binary to run
    fn get_language_server_command(&self, cx) -> Result<LanguageServerCommand>;
    
    /// Customize initialization options
    fn initialization_options(&self, cx) -> Option<Value>;
    
    /// Process completions before showing to user
    fn process_completions(&self, completions) -> Vec<Completion>;
    
    /// Convert LSP diagnostic to Zed diagnostic
    fn process_diagnostics(&self, diagnostics) -> Vec<Diagnostic>;
    
    /// Workspace configuration for this server
    fn workspace_configuration(&self, cx) -> Option<Value>;
}
```

This abstraction allows extensions to provide custom server adapters.

---

## 5) Document Synchronization

### How Zed Keeps Servers in Sync

When the user edits a buffer:

```
User types character
    ↓
Buffer updated (text crate)
    ↓
LspStore receives buffer change event
    ↓
For each server attached to this buffer:
    ├─ Full sync server → send entire document text
    └─ Incremental sync server → send only the edit delta
    ↓
Server re-parses, updates diagnostics
    ↓
Server sends textDocument/publishDiagnostics notification
    ↓
Zed receives, updates Buffer.diagnostics
    ↓
Editor UI re-renders with new squiggly lines
```

### Sync Modes

| Mode | Payload | When |
|------|---------|------|
| Full | Entire document text | Simple servers, or on open |
| Incremental | Edit delta (range + new text) | After each change (efficient) |

---

## 6) Key LSP Features in Zed

### Completions

```
User types "vec."
    ↓
Editor triggers completion request
    ↓
LspStore.request::<Completion>(position)
    ↓
Server returns CompletionList
    ↓
LspAdapter.process_completions() (normalize, sort)
    ↓
Completion menu rendered
    ↓
User selects → apply CompletionItem.textEdit
```

### Go to Definition

```
User Ctrl+clicks on symbol
    ↓
Editor → LspStore.request::<GotoDefinition>(position)
    ↓
Server returns Location (file, range)
    ↓
Workspace opens file, moves cursor to range
```

### Diagnostics

```
Server pushes textDocument/publishDiagnostics
    ↓
LspStore receives notification
    ↓
Buffer.diagnostics updated (TreeMap<LanguageServerId, DiagnosticSet>)
    ↓
Editor renders:
├─ Squiggly underlines in text
├─ Gutter icons (error/warning)
└─ Diagnostics panel updated
```

### Code Actions

```
User places cursor on diagnostic
    ↓
Editor → LspStore.request::<CodeAction>(range)
    ↓
Server returns CodeActionList (fix imports, extract function, etc.)
    ↓
Light bulb icon rendered
    ↓
User clicks → apply WorkspaceEdit
```

### Rename

```
User: F2 on symbol
    ↓
Editor → LspStore.request::<PrepareRename>(position)
    ↓
Server confirms: "this symbol can be renamed"
    ↓
User types new name
    ↓
Editor → LspStore.request::<Rename>(position, new_name)
    ↓
Server returns WorkspaceEdit (changes across multiple files)
    ↓
Apply all file edits atomically
```

---

## 7) Multi-Server Coordination

A single file can have multiple language servers:

```
file.tsx → [typescript-language-server, eslint, tailwindcss]
```

### How Requests Are Routed

- **Completions**: merged from all servers, deduplicated
- **Diagnostics**: each server's diagnostics stored separately, displayed together
- **Go-to-definition**: try each server, return first success
- **Formatting**: user configures which server handles formatting

### Server Priority

When multiple servers can handle the same request, Zed uses:
1. The user-configured formatter (for format requests)
2. The primary language server (first registered)
3. Fallback to others

---

## 8) LSP and Collaboration

### Local Host

```
Host machine:
├── Buffer changes → sent to LanguageServer process
├── Diagnostics received → stored in Buffer
├── Also sent to all connected guests via RPC
```

### Remote Guest

```
Guest machine:
├── Buffer changes → sent via RPC to host
├── Host relays to LanguageServer
├── Host receives diagnostics → relays to guest via RPC
├── Guest LspStore (Remote variant) stores diagnostics
```

The guest never talks to language servers directly — everything proxies through the host.

---

## 9) Inlay Hints

A special form of LSP response that shows inline type annotations:

```
let x = calculate();        ← buffer text
let x: i32 = calculate();   ← with inlay hint
          ^^^^
     InlayHint from server
```

Inlay hints are injected through the **InlayMap** layer of the display pipeline (see Editor/Buffer atom).

---

## 10) Anti-Patterns

### Sending Requests Without Checking Capabilities

```rust
// BAD: server may not support hover
let hover = server.request::<Hover>(params).await; // may error

// GOOD: check capabilities first
if server.capabilities().hover_provider.is_some() {
    let hover = server.request::<Hover>(params).await;
}
```

### Holding Server Lock During Async

```rust
// BAD: lock held across await point
let handlers = server.notification_handlers.lock();
let result = some_async().await; // other tasks blocked!
drop(handlers);

// GOOD: lock briefly, release before async
let handler = {
    let handlers = server.notification_handlers.lock();
    handlers.get("textDocument/publishDiagnostics").cloned()
};
if let Some(handler) = handler {
    handler(notification).await;
}
```

### Ignoring Server Crashes

```rust
// BAD: server crashed, no recovery
// User sees no completions forever

// GOOD: LspStore monitors server process
// On crash → restart with backoff
// Notify user of restart
```

---

## 11) Mental Model

```
LspStore (Entity<LspStore>)
│
├─── Server Registry
│    ├─ LanguageServerId(0) → rust-analyzer (for *.rs)
│    ├─ LanguageServerId(1) → typescript-language-server (for *.ts)
│    └─ LanguageServerId(2) → eslint (for *.ts, *.js)
│
├─── LanguageRegistry
│    ├─ "Rust" → [RustAnalyzerAdapter]
│    ├─ "TypeScript" → [TsServerAdapter, EslintAdapter]
│    └─ "Python" → [PyrightAdapter]
│
├─── Per-Server State
│    ├─ LanguageServer (JSON-RPC wrapper)
│    │  ├─ capabilities: what it supports
│    │  ├─ response_handlers: pending requests
│    │  └─ notification_handlers: event listeners
│    ├─ Attached buffers (which files use this server)
│    └─ Server process handle
│
├─── Document Sync
│    ├─ Buffer edit → didChange notification
│    ├─ File open → didOpen notification
│    └─ File close → didClose notification
│
└─── Request/Response
     ├─ Completion, Hover, Definition, References
     ├─ Rename, CodeAction, Formatting
     ├─ Diagnostics (pushed from server)
     └─ InlayHints (requested by editor)
```

**Core insight:** Zed acts as a multi-server LSP client. Each language can have multiple servers. The LspAdapter abstraction allows extensions to plug in new servers. Everything proxies through RPC in collaboration mode. Capabilities are checked before requests.
