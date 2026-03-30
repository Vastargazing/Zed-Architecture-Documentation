# 10. Agent & AI: Intelligent Assistance

The tenth architectural atom — how Zed integrates AI models for code generation, editing, and conversation. This covers **the agent panel, model providers, tool system, inline assistant, and context management**.

---

## 1) Architecture Overview

```
┌─────────────────────────────────────────────────┐
│  Agent Panel (UI)                                │
│  ├─ Chat interface                               │
│  ├─ Tool invocations (file edit, terminal, etc.) │
│  └─ Context attachments (files, selections)      │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  Session (Entity<Session>)                       │
│  ├─ Thread: conversation history                 │
│  ├─ AcpThread: Agent Client Protocol             │
│  └─ Profiles: model + tool configuration         │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  LanguageModels (provider registry)              │
│  ├─ Anthropic (Claude)                           │
│  ├─ OpenAI (GPT)                                 │
│  ├─ Google AI (Gemini)                           │
│  ├─ Copilot Chat                                 │
│  ├─ Ollama (local)                               │
│  ├─ Bedrock, Deepseek, Codestral, etc.           │
│  └─ Cloud LLM (Zed's proxy)                     │
└──────────────────────┬──────────────────────────┘
                       │
              HTTP/SSE to provider APIs
```

---

## 2) Agent Settings

```rust
pub struct AgentSettings {
    pub enabled: bool,
    pub default_model: Option<LanguageModelSelection>,
    pub inline_assistant_model: Option<LanguageModelSelection>,
    pub favorite_models: Vec<LanguageModelSelection>,
    pub model_parameters: Vec<LanguageModelParameters>,  // Per-model temperature
    pub tool_permissions: ToolPermissions,
    pub profiles: IndexMap<AgentProfileId, AgentProfileSettings>,
}
```

Users can:
- Choose default models for chat vs inline assist
- Set per-model generation parameters (temperature, max tokens)
- Configure which tools the agent can use
- Create profiles (different model + tool combinations)

---

## 3) Sessions and Threads

### Session

```rust
pub struct Session {
    thread: Entity<Thread>,              // Conversation history
    acp_thread: Entity<AcpThread>,       // Agent Client Protocol thread
    project_id: EntityId,
    pending_save: Task<Result<()>>,      // Auto-save to disk
}
```

A Session represents one conversation. It persists to disk so users can resume conversations across Zed restarts.

### Thread

The Thread holds the actual conversation messages:

```
Thread
├── Message[0]: User "Fix the type error in main.rs"
├── Message[1]: Assistant (thinking + tool calls)
│   ├── ToolCall: read_file("src/main.rs")
│   ├── ToolResult: <file contents>
│   ├── ToolCall: edit_file("src/main.rs", ...)
│   └── ToolResult: <success>
├── Message[2]: Assistant "I've fixed the type error..."
└── Message[3]: User "Thanks, now run the tests"
```

---

## 4) Model Providers

### LanguageModels Registry

```rust
pub struct LanguageModels {
    models: HashMap<ModelId, Arc<dyn LanguageModel>>,
    model_list: AgentModelList,   // Grouped by provider
    refresh_models_rx: watch::Receiver<()>,
    refresh_models_tx: watch::Sender<()>,
}
```

### Supported Providers

Each provider has its own crate:

| Provider | Crate | Protocol |
|----------|-------|----------|
| Anthropic (Claude) | `crates/anthropic/` | HTTP + SSE streaming |
| OpenAI (GPT) | `crates/open_ai/` | HTTP + SSE streaming |
| Google AI (Gemini) | `crates/google_ai/` | HTTP + SSE streaming |
| Copilot Chat | `crates/copilot_chat/` | HTTP |
| Bedrock (AWS) | `crates/bedrock/` | AWS SDK |
| Deepseek | `crates/deepseek/` | HTTP + SSE |
| Codestral (Mistral) | `crates/codestral/` | HTTP + SSE |
| Ollama (local) | `crates/ollama/` | HTTP |
| OpenRouter | `crates/open_router/` | HTTP + SSE |
| Cloud LLM | `crates/cloud_llm_client/` | Zed proxy |

### The LanguageModel Trait

```rust
pub trait LanguageModel: Send + Sync {
    fn id(&self) -> LanguageModelId;
    fn name(&self) -> LanguageModelName;
    fn provider_id(&self) -> LanguageModelProviderId;
    fn provider_name(&self) -> LanguageModelProviderName;
    fn max_token_count(&self) -> u64;

    fn supports_tools(&self) -> bool;
    fn supports_images(&self) -> bool;
    fn supports_thinking(&self) -> bool { false }

    fn count_tokens(
        &self,
        request: LanguageModelRequest,
        cx: &App,
    ) -> BoxFuture<'static, Result<u64>>;

    fn stream_completion(
        &self,
        request: LanguageModelRequest,
        cx: &AsyncApp,
    ) -> BoxFuture<'static, Result<BoxStream<'static, Result<LanguageModelCompletionEvent, LanguageModelCompletionError>>, LanguageModelCompletionError>>;
}
```

---

## 5) Tool System

### What Are Tools?

Tools allow the AI agent to take actions in the editor — read files, edit code, run commands, search, etc.

### Built-in Tools

| Tool | Purpose |
|------|---------|
| `read_file` | Read file contents |
| `edit_file` | Apply edits to a file |
| `create_file` | Create a new file |
| `list_directory` | List files in a directory |
| `search` | Search across the project |
| `run_terminal_command` | Execute shell commands |
| `diagnostics` | Get LSP diagnostics |
| `symbol_info` | Get symbol information |
| `find_references` | Find references to a symbol |

### Tool Permission Model

```rust
pub enum ToolPermissionMode {
    Allow,    // Auto-approve without prompting
    Deny,     // Auto-reject with an error
    Confirm,  // Always prompt for confirmation (default)
}

pub struct ToolPermissions {
    /// Global default when no specific rule matches
    pub default: ToolPermissionMode,
    /// Per-tool rules, keyed by tool name
    pub tools: HashMap<Arc<str>, ToolRules>,
}

pub struct ToolRules {
    pub default: Option<ToolPermissionMode>,
    /// Regex patterns — matching args are auto-allowed
    pub always_allow: Vec<CompiledRegex>,
    /// Regex patterns — matching args are auto-denied
    pub always_deny: Vec<CompiledRegex>,
    /// Regex patterns — matching args always prompt
    pub always_confirm: Vec<CompiledRegex>,
}
```

Permissions are regex-based: rules match against the tool's arguments, not just the tool name. Dangerous tools (file editing, terminal commands) default to `Confirm`.

### Tool Execution Flow

```
Agent decides to use a tool
    ↓
Check tool_permissions
    ↓
Permission = Always? → execute immediately
Permission = Ask? → show confirmation dialog → user approves → execute
Permission = Never? → tool call rejected
    ↓
Tool executes (e.g., reads a file)
    ↓
Result sent back to model as tool_result message
    ↓
Model continues generating with tool result context
```

---

## 6) Context Management

### What is Context?

Context is information the agent uses to understand the codebase:

```
Agent prompt includes:
├── System prompt (agent instructions)
├── Project rules (.zed/rules.md)
├── Attached files (user-selected)
├── Selected code regions
├── Active diagnostics
├── Tool results from conversation
└── Conversation history
```

### Project Context

```rust
struct ProjectState {
    project: Entity<Project>,
    project_context: Entity<ProjectContext>,   // Codebase indexing
    context_server_registry: Entity<ContextServerRegistry>,
}
```

### Context Servers (MCP)

Zed supports the **Model Context Protocol (MCP)** — a standard for providing context to AI models:

```
Context Server (MCP)
├── Provides: documents, resources, prompts
├── Protocol: JSON-RPC over stdin/stdout
├── Examples: documentation lookup, API reference, database schema
└── Configured via: extensions or settings
```

---

## 7) Inline Assistant

Separate from the agent panel, the inline assistant allows AI editing directly in the editor:

```
User selects code → Ctrl+Enter → "Refactor this to use iterators"
    ↓
Selection + instruction sent to model
    ↓
Model generates replacement code
    ↓
Diff shown inline (old vs new)
    ↓
User accepts or rejects
```

### Inline vs Agent Panel

| Feature | Inline Assistant | Agent Panel |
|---------|-----------------|-------------|
| Scope | Single selection/region | Full conversation |
| Tools | No tools | Full tool access |
| History | No history | Persistent sessions |
| Model | May use different model | Default model |
| Use case | Quick edits | Complex tasks |

---

## 8) Agent Profiles

Profiles bundle model selection + tool configuration:

```json
{
  "profiles": {
    "code-review": {
      "name": "Code Review",
      "model": { "provider": "anthropic", "model": "claude-sonnet-4-20250514" },
      "tools": {
        "edit_file": false,
        "run_terminal_command": false
      }
    },
    "full-agent": {
      "name": "Full Agent",
      "model": { "provider": "anthropic", "model": "claude-sonnet-4-20250514" },
      "tools": {
        "edit_file": true,
        "run_terminal_command": true
      }
    }
  }
}
```

Profile `tools` is a `HashMap<tool_name, bool>` — each tool is simply enabled (`true`) or disabled (`false`). Tool-level permission granularity (Allow/Deny/Confirm) is configured separately via `tool_permissions` in the global settings, not inside profiles. Built-in profiles: `write`, `ask`, `minimal`.

---

## 9) Streaming and Cost

### Streaming Responses

Model responses are streamed token-by-token via Server-Sent Events (SSE):

```
Model starts generating
    ↓
Token "I" → displayed immediately
Token "'" → displayed
Token "ll" → displayed
Token " fix" → displayed
...
    ↓
Stream completes
    ↓
Final message assembled
```

### Cost Tracking

```
Each request:
├── Input tokens (prompt + context)
├── Output tokens (response)
├── Model pricing (varies by provider)
└── Displayed in UI per-message
```

---

## 10) Edit Prediction (Copilot-style)

Separate from the agent, Zed also has edit prediction — inline code suggestions as you type:

```
Crates:
├── crates/edit_prediction/          ← core prediction engine
├── crates/edit_prediction_cli/      ← CLI interface
├── crates/edit_prediction_context/  ← context building
├── crates/edit_prediction_types/    ← shared types
└── crates/edit_prediction_ui/       ← ghost text rendering
```

This provides Copilot-like ghost text suggestions, using either:
- GitHub Copilot (if configured)
- Zed's own prediction models
- Other compatible providers

---

## 11) Rules Files

Project-level AI instructions are loaded from well-known filenames at the project root:

```
.rules
.cursorrules
.windsurfrules
.clinerules
.github/copilot-instructions.md
AGENT.md
AGENTS.md
CLAUDE.md
GEMINI.md
```

Example (e.g. `AGENTS.md` or `.rules`):

```markdown
# Project Rules

- Use TypeScript for all new code
- Follow the existing pattern of functional components
- Always add error handling with proper error types
- Write tests using vitest
```

Zed scans each worktree root for these files and automatically includes the found content in the system prompt for all AI interactions within the project. There is no `.zed/rules.md` — the supported filenames are listed above.

---

## 12) Anti-Patterns

### Sending Too Much Context

```
// BAD: attach entire codebase as context
attach_all_files_in_project(); // Exceeds token limit, costs $$$

// GOOD: the agent uses tools to read files on-demand
// Start with minimal context, let the model request what it needs
```

### Allowing Unrestricted Tool Access

```json
// BAD: all tools auto-approved
{ "tool_permissions": { "run_terminal_command": "always" } }
// Agent could run destructive commands without user review

// GOOD: dangerous tools require confirmation
{ "tool_permissions": { "run_terminal_command": "ask" } }
```

### Ignoring Model Capabilities

```rust
// BAD: request tool use from model that doesn't support it
if model.supports_tools() == false {
    // Still sending tool definitions → wasted tokens, possible errors
}

// GOOD: check capabilities
if model.supports_tools() {
    request.tools = available_tools;
}
```

---

## 13) Mental Model

```
Agent & AI System
│
├─── Agent Panel (UI)
│    ├─ Chat interface (messages + tool calls)
│    ├─ Context attachments (files, selections)
│    └─ Profiles (model + tool config)
│
├─── Session (Entity<Session>)
│    ├─ Thread (conversation history)
│    ├─ AcpThread (Agent Client Protocol)
│    └─ Persisted to disk (resume across restarts)
│
├─── LanguageModels (provider registry)
│    ├─ Anthropic, OpenAI, Google, Copilot, Ollama...
│    ├─ LanguageModel trait (stream_completion, supports_tools)
│    └─ Per-model parameters (temperature, max tokens)
│
├─── Tool System
│    ├─ Built-in: read_file, edit_file, terminal, search...
│    ├─ Permission model: Always / Ask / Never
│    └─ Extension-provided tools
│
├─── Context
│    ├─ System prompt + project rules (.zed/rules.md)
│    ├─ Attached files and selections
│    ├─ Context servers (MCP protocol)
│    └─ Tool results from conversation
│
├─── Inline Assistant
│    ├─ Selection-based editing
│    ├─ Diff preview (accept/reject)
│    └─ Separate model configuration
│
└─── Edit Prediction
     ├─ Copilot-style ghost text
     ├─ Multiple provider support
     └─ Context-aware suggestions
```

**Core insight:** Zed's AI system is provider-agnostic with a pluggable model registry. The agent uses a tool-based architecture where the AI model decides which tools to invoke (read files, edit code, run commands). Permissions control what the agent can do autonomously. Context management (MCP, rules files, attachments) keeps the model informed without overwhelming it with tokens.
