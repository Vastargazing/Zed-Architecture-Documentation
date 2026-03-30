# Zed Architecture Documentation

A comprehensive guide to the Zed code editor's architecture, organized as a series of **architectural atoms** — self-contained documents that each explain one fundamental layer of the system.

---

## Reading Order

The atoms are ordered from the lowest-level foundation to the highest-level features. Each atom builds on concepts from previous ones.

| # | Atom | What It Covers |
|---|------|---------------|
| **1** | [GPUI](1_GPUI.md) | Custom GPU UI framework: entity system, rendering pipeline, actions, events, focus, platform abstraction |
| **2** | [Workspace & Pane](2_Workspace-Pane.md) | Window layout: split trees, tab management, dock panels, persistence, focus routing |
| **3** | [Project & Worktree](3_Project-Worktree.md) | File system layer: background scanning, snapshots, file watching, buffer store, dual local/remote architecture |
| **4** | [Editor & Buffer](4_Editor-Buffer.md) | Text editing engine: Rope data structure, display map pipeline (5 transform layers + CreaseMap), selections, anchors, undo/redo, syntax highlighting |
| **5** | [Settings & Registry](5_Settings-Registry.md) | Configuration system: hierarchical settings, keymap binding, settings traits, per-language/per-file overrides |
| **6** | [LSP & Language Server](6_LSP-Language-Server.md) | Editor intelligence: LSP client, server lifecycle, language registry, multi-server coordination |
| **7** | [Extensions](7_Extensions.md) | Plugin architecture: WASM sandboxing, Extension trait, delegates, extension index, hot reload |
| **8** | [Collaboration & CRDT](8_Collaboration-CRDT.md) | Real-time editing: CRDT, Lamport clocks, version vectors, RPC protocol, collab server |
| **9** | [Terminal & Tasks](9_Terminal-Tasks.md) | Execution environment: terminal emulation (Alacritty), PTY, task templates, variable substitution, runnables |
| **10** | [Agent & AI](10_Agent-AI.md) | AI assistance: model providers, tool system, inline assistant, context management, MCP |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Zed Application                           │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Agent (AI)        │  Extensions (WASM)                   │   │
│  │  Model providers   │  Languages, themes, LSP adapters     │   │
│  └────────┬───────────┴──────────┬───────────────────────────┘   │
│           │                      │                               │
│  ┌────────▼──────────────────────▼───────────────────────────┐   │
│  │  Workspace                                                │   │
│  │  ├─ PaneGroup (split tree)                                │   │
│  │  ├─ Dock (panels: left, right, bottom)                    │   │
│  │  └─ WorkspaceDb (persistence)                             │   │
│  └────────┬──────────────────────────────────────────────────┘   │
│           │                                                      │
│  ┌────────▼──────────┐  ┌───────────────────┐  ┌────────────┐   │
│  │  Editor            │  │  Terminal          │  │  Settings   │   │
│  │  ├─ MultiBuffer    │  │  ├─ Alacritty      │  │  ├─ Store   │   │
│  │  ├─ DisplayMap     │  │  ├─ PTY            │  │  ├─ Keymap  │   │
│  │  ├─ Selections     │  │  └─ Tasks          │  │  └─ Scoped  │   │
│  │  └─ Anchors        │  │                    │  │            │   │
│  └────────┬───────────┘  └────────────────────┘  └────────────┘   │
│           │                                                      │
│  ┌────────▼──────────────────────────────────────────────────┐   │
│  │  Project                                                  │   │
│  │  ├─ WorktreeStore (file trees with background scanning)   │   │
│  │  ├─ BufferStore (open text buffers)                       │   │
│  │  ├─ LspStore (language server lifecycle)                  │   │
│  │  ├─ GitStore (version control)                            │   │
│  │  ├─ TaskStore (task execution)                            │   │
│  │  └─ DapStore (debugger)                                   │   │
│  └────────┬──────────────────────────────────────────────────┘   │
│           │                                                      │
│  ┌────────▼──────────────────────────────────────────────────┐   │
│  │  Text / Language / CRDT                                   │   │
│  │  ├─ Rope (SumTree<Chunk>) — text storage                 │   │
│  │  ├─ Fragments (CRDT edit history)                         │   │
│  │  ├─ Lamport clocks + version vectors                      │   │
│  │  └─ Tree-sitter syntax maps                               │   │
│  └────────┬──────────────────────────────────────────────────┘   │
│           │                                                      │
│  ┌────────▼──────────────────────────────────────────────────┐   │
│  │  GPUI (UI Framework)                                      │   │
│  │  ├─ Entity system (state management)                      │   │
│  │  ├─ Context hierarchy (App → Context<T> → Window)         │   │
│  │  ├─ Element/Render (3-phase rendering pipeline)           │   │
│  │  ├─ Actions + DispatchTree (input handling)               │   │
│  │  ├─ EventEmitter + Subscriptions (reactivity)             │   │
│  │  └─ Platform abstraction (macOS, Linux, Windows)          │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Collaboration (RPC)                                      │   │
│  │  ├─ Protobuf messages over WebSocket                      │   │
│  │  ├─ Collab server (relay, not resolver)                   │   │
│  │  └─ Dual-mode: Local / Remote for every store             │   │
│  └───────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Architectural Principles

### 1. Everything is an Entity

State lives in entities managed by GPUI's entity map. Access goes through typed handles (`Entity<T>`, `WeakEntity<T>`). No global mutable state — all mutations go through `entity.update(cx, ...)`.

### 2. Dual Local/Remote Architecture

Every data store (Worktree, Buffer, LSP) has two variants — Local (direct access) and Remote (RPC proxy). This enables seamless collaboration without special-casing.

### 3. Snapshot-Based Consistency

Background tasks (file scanning, syntax parsing, soft-wrapping) produce immutable snapshots. UI code reads snapshots for consistent, non-blocking access. Writers and readers never contend.

### 4. CRDT for Collaboration

Text buffers use a CRDT with Lamport clocks, ensuring all replicas converge to the same state regardless of operation order. The collab server acts as a relay for live project sessions (no conflict resolution); channel buffers are additionally persisted server-side in a database.

### 5. WASM Sandboxing for Extensions

Extensions run in WASM sandboxes with no direct filesystem or process access. They interact with Zed through WIT-exported interfaces (delegates, HTTP client, Node.js helpers). HTTP requests are allowed via a host-exported `fetch()`. This provides strong security isolation while keeping extensions capable.

### 6. SumTree Everywhere

Zed's custom SumTree (a B-tree with aggregate summaries) powers both Rope (text storage) and Worktree (file indexing). It provides O(log n) operations for all common editor operations.

### 7. Layered Display Pipeline

Text goes through 5 coordinate-transformation layers before rendering: `InlayMap` → `FoldMap` → `TabMap` → `WrapMap` → `BlockMap`. Each layer has its own coordinate space and snapshot, making the pipeline composable and debuggable. `DisplayMap` sits on top as the coordinator (adding highlights; `DisplayPoint` is a newtype of `BlockPoint`). `CreaseMap` is a metadata structure that tracks foldable regions (feeds into `FoldMap`), not a pipeline layer itself.

### 8. Settings Cascade

Settings merge hierarchically in this order (each layer overrides the previous):

```
default (compiled-in defaults)
  → extension (settings contributed by extensions)
  → global (global_settings — e.g. in SSH remote projects)
  → user (<platform-config-dir>/zed/settings.json, +release-channel/OS/profile variants)
    macOS:   ~/Library/Application Support/Zed/settings.json
    Linux:   $XDG_CONFIG_HOME/zed/settings.json  (~/.config/zed/settings.json)
    Windows: %APPDATA%\Zed\settings.json
  → server (pushed from the remote SSH machine when in a remote project)
  → project (.zed/settings.json, stacked per directory depth)
```

Language-specific and file/path-specific overrides are resolved at read time within the project layer via `SettingsLocation { worktree_id, path }`.

---

## Crate Map

Key crates and their roles:

| Crate | Role |
|-------|------|
| `gpui` | UI framework core |
| `gpui_platform` | Platform trait definitions |
| `gpui_macos`, `gpui_linux`, `gpui_windows` | Platform backends |
| `gpui_wgpu` | GPU rendering backend |
| `workspace` | Window layout, panes, panels |
| `project` | Data coordination (worktrees, buffers, LSP) |
| `worktree` | File system scanning and snapshots |
| `editor` | Text editor (UI + controls) |
| `multi_buffer` | Excerpt aggregation for editors |
| `language` | Buffer + syntax (Tree-sitter) |
| `language_core` | Core language types shared across crates |
| `language_model` | Abstraction layer for AI model providers |
| `language_models` | Concrete provider registration (registers all providers on startup) |
| `text` | Rope + CRDT core |
| `rope` | SumTree-based rope |
| `sum_tree` | B-tree with aggregate summaries; backbone of Rope and Worktree |
| `lsp` | LSP JSON-RPC client |
| `settings` | Settings store + traits |
| `extension`, `extension_host`, `extension_api` | Extension system |
| `collab` | Collaboration server |
| `client` | RPC client |
| `rpc` | RPC peer/client, WebSocket connection handling |
| `proto` | Protocol buffer definitions (cloud API) |
| `terminal` | Terminal emulation |
| `task` | Task templates and execution |
| `agent` | AI agent core: tools, templates, edit_agent, local LLM server |
| `agent_ui` | AI agent panel UI, inline assistant, conversation view |
| `agent_settings` | AI configuration |
| `anthropic`, `open_ai`, `google_ai`, `copilot_chat`, `ollama`, `bedrock`, `deepseek`, `codestral`, `open_router`, `cloud_llm_client`, ... | Model providers |
| `theme` | Theme loading and application |
| `fs` | Filesystem abstraction |
| `git` | Git integration |
| `dap` | Debug Adapter Protocol |

---

## Where to Start

**If you're new to the codebase:**
1. Start with [1_GPUI.md](1_GPUI.md) — understand the entity/context/render model
2. Read [2_Workspace-Pane.md](2_Workspace-Pane.md) — understand the UI layout
3. Read [4_Editor-Buffer.md](4_Editor-Buffer.md) — understand the editing engine

**If you're working on a specific area:**
- Language support → [6_LSP-Language-Server.md](6_LSP-Language-Server.md) + [7_Extensions.md](7_Extensions.md)
- Collaboration → [8_Collaboration-CRDT.md](8_Collaboration-CRDT.md)
- AI features → [10_Agent-AI.md](10_Agent-AI.md)
- Terminal/Tasks → [9_Terminal-Tasks.md](9_Terminal-Tasks.md)
- Configuration → [5_Settings-Registry.md](5_Settings-Registry.md)
