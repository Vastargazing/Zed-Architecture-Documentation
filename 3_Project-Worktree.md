# 3. Project & Worktree: File System and Data Layer

The third architectural atom — how Zed manages files, directories, and the entire data layer. This covers **worktree scanning, snapshot consistency, file watching, and the project coordination model**.

---

## 1) What is a Project?

A `Project` is the central data coordinator in Zed. It owns everything non-UI: files, buffers, language servers, git state, debuggers, tasks, and environment.

```rust
pub struct Project {
    worktree_store: Entity<WorktreeStore>,       // File trees
    buffer_store: Entity<BufferStore>,           // Open text buffers
    lsp_store: Entity<LspStore>,                 // Language servers
    git_store: Entity<GitStore>,                 // Git metadata
    dap_store: Entity<DapStore>,                 // Debug adapters
    breakpoint_store: Entity<BreakpointStore>,
    task_store: Entity<TaskStore>,               // Build/run tasks
    environment: Entity<ProjectEnvironment>,     // Shell env vars
    agent_server_store: Entity<AgentServerStore>,
    context_server_store: Entity<ContextServerStore>,
    // ... subscriptions, channels
}
```

### Two Modes

| Mode | Description |
|------|-------------|
| **Local** | Direct filesystem access, full control over LSP, files, git |
| **Remote** | Guest in a collaborative session, proxies everything via RPC |

This dual-mode pattern repeats throughout: `WorktreeStore`, `BufferStore`, `LspStore` — all have Local and Remote variants.

---

## 2) WorktreeStore: Multi-Root Registry

A project can have multiple root directories (monorepo, multi-folder workspace). `WorktreeStore` manages them all.

```rust
pub struct WorktreeStore {
    next_entry_id: Arc<AtomicUsize>,
    worktrees: Vec<WorktreeHandle>,
    loading_worktrees: HashMap<Arc<SanitizedPath>, Shared<Task<...>>>,
    initial_scan_complete: (watch::Sender<bool>, watch::Receiver<bool>),
}
```

### Events

```rust
pub enum WorktreeStoreEvent {
    WorktreeAdded(Entity<Worktree>),
    WorktreeRemoved(EntityId, WorktreeId),
    WorktreeReleased(EntityId, WorktreeId),   // Handle dropped, worktree de-registered
    WorktreeOrderChanged,                      // Worktrees reordered by user
    WorktreeUpdateSent(Entity<Worktree>),      // Collab: update serialized and sent to guests
    WorktreeUpdatedEntries(WorktreeId, UpdatedEntriesSet),
    WorktreeUpdatedGitRepositories(WorktreeId, UpdatedGitRepositoriesSet),
    WorktreeDeletedEntry(WorktreeId, ProjectEntryId),
}
```

### Key Operations

- **`find_or_create_worktree(path)`** — opens directory if not already loaded
- **`create_worktree(path)`** — adds new worktree with ID allocation
- **`wait_for_initial_scan()`** — blocks until ALL worktrees finish their initial FS scan

---

## 3) Worktree: The File Tree

### Two Variants

```rust
pub enum Worktree {
    Local(LocalWorktree),    // Watches real filesystem
    Remote(RemoteWorktree),  // Receives updates via RPC
}
```

### LocalWorktree

```rust
pub struct LocalWorktree {
    snapshot: LocalSnapshot,
    scan_requests_tx: channel::Sender<ScanRequest>,
    path_prefixes_to_scan_tx: channel::Sender<PathPrefixScanRequest>,
    _background_scanner_tasks: Vec<Task<()>>,  // Detached async scanner
    is_scanning: (watch::Sender<bool>, watch::Receiver<bool>),
    visible: bool,
}
```

The key insight: **the worktree has a background scanner running in a separate async task**. It continuously watches the filesystem and updates the snapshot.

---

## 4) Background Scanner: How File Watching Works

### State Machine

```rust
enum BackgroundScannerPhase {
    InitialScan,                        // First full directory walk
    EventsReceivedDuringInitialScan,    // Events caught while scanning
    Events,                              // Normal operation (steady state)
}
```

### Scanner Flow

```
1. INITIAL SCAN PHASE
   ├─ Recursively enumerate directory tree
   ├─ Discover git repositories
   ├─ Build .gitignore stack
   ├─ Load worktree settings
   └─ Mark initial_scan_complete

2. EVENT PROCESSING PHASE (steady state)
   ├─ Watch filesystem via fs::Watcher
   ├─ Batch PathEvent notifications
   ├─ Prioritize explicit scan requests over raw FS events
   └─ Apply updates to SumTree snapshot
```

### File System Watcher Integration

```
Filesystem change detected (inotify/FSEvents/ReadDirectoryChangesW)
    ↓
FS_WATCH_LATENCY (~100ms) debounce
    ↓
Scanner collects batch of PathEvents
    ↓
Scanner updates entries_by_path SumTree
    ↓
Emits WorktreeStoreEvent::WorktreeUpdatedEntries(UpdatedEntriesSet)
    ↓
Project receives event → notifies UI
    ↓
File tree panel, diagnostics, etc. refresh
```

---

## 5) Snapshot: Consistent Reads

### Why Snapshots?

The background scanner runs concurrently. UI code needs a consistent view of the file tree. Solution: **immutable, cloneable snapshots**.

```rust
pub struct Snapshot {
    id: WorktreeId,
    abs_path: Arc<SanitizedPath>,
    root_name: Arc<RelPath>,
    entries_by_path: SumTree<Entry>,     // Indexed by path
    entries_by_id: SumTree<PathEntry>,   // Indexed by ID
    scan_id: usize,                       // Current scan version
    completed_scan_id: usize,             // Last confirmed completed scan
}
```

### Dual Indexing

Two SumTree indexes for different access patterns:

| Index | Key | Use Case |
|-------|-----|----------|
| `entries_by_path` | file path | Directory traversal, tree display |
| `entries_by_id` | `ProjectEntryId` | Lookup after renames (ID stable, path changes) |

### Scan ID Coordination

```
scan_id:           Increments every time scanning begins
completed_scan_id: Only advances when ALL preceding scans complete

wait_for_snapshot(scan_id):  Blocks until specific scan finishes
```

This enables causally-ordered reads: "Give me the snapshot that includes the file I just created."

### LocalSnapshot (Extends Snapshot)

```rust
pub struct LocalSnapshot {
    snapshot: Snapshot,
    ignores_by_parent_abs_path: HashMap<Arc<Path>, (Arc<Gitignore>, bool)>,
    repo_exclude_by_work_dir_abs_path: HashMap<Arc<Path>, (Arc<Gitignore>, bool)>,
    git_repositories: TreeMap<ProjectEntryId, LocalRepositoryEntry>,
    root_file_handle: Option<Arc<dyn fs::FileHandle>>,
}
```

Adds .gitignore awareness and git repo metadata on top of the base snapshot.

---

## 6) Entry: File System Node

```rust
pub struct Entry {
    pub id: ProjectEntryId,
    pub kind: EntryKind,           // UnloadedDir, PendingDir, Dir, File
    pub path: Arc<RelPath>,
    pub inode: u64,                // For rename detection
    pub mtime: Option<MTime>,
    pub is_ignored: bool,          // .gitignore
    pub is_hidden: bool,
    pub is_always_included: bool,  // Override exclusions
    pub is_external: bool,         // Symlink outside worktree
    pub is_private: bool,          // .env files
    pub size: u64,
    pub char_bag: CharBag,         // For fuzzy file finder
    pub is_fifo: bool,
}
```

### Entry Kind States

```
UnloadedDir → PendingDir → Dir
                              ↘
                          File (leaf)
```

| State | Meaning |
|-------|---------|
| `UnloadedDir` | Not yet scanned (collapsed in UI) |
| `PendingDir` | Currently being scanned |
| `Dir` | Fully loaded directory |
| `File` | Regular file |

### Path Change Events

```rust
pub enum PathChange {
    Added,           // New entry created
    Removed,         // Entry deleted
    Updated,         // Metadata changed
    AddedOrUpdated,  // Uncertain (wasn't pre-loaded)
    Loaded,          // Discovered during initial scan
}

pub type UpdatedEntriesSet = Arc<[(Arc<RelPath>, ProjectEntryId, PathChange)]>;
```

These are **immutable, atomic bundles** — sent as events to downstream consumers.

---

## 7) SumTree: The Core Data Structure

SumTree is Zed's custom B-tree that powers both the file tree and the text buffer. It provides:

- **O(log n) lookup** by any summary dimension
- **Efficient cloning** (shared internal nodes via Arc)
- **Range queries** across multiple dimensions simultaneously

For worktrees:

```
SumTree<Entry>
├── Seek by path → directory traversal
├── Seek by id → constant-time-ish lookups
├── Merge updates → batch apply scanner results
└── Clone → create immutable snapshot cheaply
```

---

## 8) RemoteWorktree: Collaboration

For collaborative projects, the guest doesn't scan the filesystem — they receive updates via RPC:

```rust
pub struct RemoteWorktree {
    snapshot: Snapshot,
    background_snapshot: Arc<Mutex<(Snapshot, Vec<proto::UpdateWorktree>)>>,
    project_id: u64,
    client: AnyProtoClient,
    replica_id: ReplicaId,
    file_scan_inclusions: PathMatcher,  // Filters which paths to sync
}
```

### Update Pipeline

```
Host scanner detects change
    ↓
Serialize as proto::UpdateWorktree
    ↓
Send via RPC to all guests
    ↓
Guest: background task applies to background_snapshot
    ↓
Foreground task copies to main snapshot
    ↓
Emits Event::UpdatedEntries
    ↓
Guest UI refreshes
```

---

## 9) BufferStore: Open File Management

```rust
pub struct BufferStore {
    state: BufferStoreState,                          // Local | Remote
    loading_buffers: HashMap<ProjectPath, Shared<Task<...>>>,
    worktree_store: Entity<WorktreeStore>,
    opened_buffers: HashMap<BufferId, OpenBuffer>,
    path_to_buffer_id: HashMap<ProjectPath, BufferId>,
}
```

### Buffer Opening Pipeline

```
User opens file.rs
    ↓
BufferStore::open_buffer(ProjectPath)
    ↓
Check path_to_buffer_id cache → hit? return existing
    ↓
Miss → resolve ProjectPath to Worktree
    ↓
Worktree::load_file() → read content from disk (or RPC)
    ↓
Create Entity<Buffer> with content
    ↓
Register in opened_buffers and path_to_buffer_id
    ↓
Return Entity<Buffer>
```

**Deduplication:** If two tabs open the same file, they share the same Buffer entity. The `loading_buffers` map prevents duplicate concurrent loads.

---

## 10) Worktree Settings

```rust
pub struct WorktreeSettings {
    pub project_name: Option<String>,
    pub file_scan_exclusions: PathMatcher,       // .gitignore-like patterns
    pub file_scan_inclusions: PathMatcher,       // Override exclusions
    pub parent_dir_scan_inclusions: PathMatcher, // Ancestor paths
    pub private_files: PathMatcher,              // .env, secrets
    pub hidden_files: PathMatcher,
    pub read_only_files: PathMatcher,
}
```

Loaded from `.zed/settings.json` in the project. The scanner checks these patterns to:
- Skip excluded subtrees during initial scan (performance)
- Mark private files (hidden in file tree but accessible)
- Mark read-only files (prevent edits)

---

## 11) Project Environment

```rust
pub struct ProjectEnvironment {
    cli_environment: Option<HashMap<String, String>>,
    local_environments: HashMap<(Shell, Arc<Path>), Shared<Task<...>>>,
    remote_environments: HashMap<(Shell, Arc<Path>), Shared<Task<...>>>,
}
```

- Inherits CLI environment if opened via `zed` CLI command
- Caches shell environments per (shell, path) pair
- Supports direnv and other environment hooks
- Separate handling for local and remote projects

---

## 12) Integration Flow: Adding a Worktree

```
Project::local() creates WorktreeStore
    ↓
Project.open_path() → WorktreeStore.find_or_create_worktree()
    ↓
WorktreeStore.create_worktree() → Worktree::local()
    ↓
LocalWorktree spawns background scanner task
    ↓
Scanner runs initial scan, emits UpdatedEntries events
    ↓
WorktreeStoreEvent::WorktreeAdded emitted
    ↓
Project.on_worktree_added() subscribes to Worktree events
    ↓
Project::Event::WorktreeAdded sent to UI
```

### File Change Detection Flow

```
Filesystem watcher detects change
    ↓
100ms debounce (FS_WATCH_LATENCY)
    ↓
BackgroundScanner batches and processes events
    ↓
WorktreeStoreEvent::WorktreeUpdatedEntries(UpdatedEntriesSet)
    ↓
Project.on_worktree_store_event() handles batch
    ↓
Telemetry + project-level Event::WorktreeUpdatedEntries
    ↓
UI refreshes: file tree, buffers reloaded if needed
```

---

## 13) Architectural Patterns

### Pattern 1: Snapshot-Based Consistency

- Immutable `Snapshot` for read-only access
- SumTree enables efficient indexing (by path, by ID)
- Scan IDs ensure causally ordered updates
- Multiple concurrent readers, single writer (scanner)

### Pattern 2: Dual Local/Remote

Every store has two variants — one for direct filesystem, one for RPC proxy. The interface is the same, the implementation differs.

### Pattern 3: Background Scanning with Event Batching

- Scanner runs in background async task
- FS events are debounced and batched
- Updates are atomic (entire UpdatedEntriesSet at once)
- No partial state visible to readers

### Pattern 4: Path-Stable IDs

- `ProjectEntryId` is stable across renames (assigned by scanner)
- `Path` changes when file is renamed
- Consumers that track files by ID survive renames transparently

---

## 14) Anti-Patterns

### Reading Snapshot During Mutation

```rust
// BAD: snapshot may be stale
let snapshot = worktree.read(cx).snapshot();
// ... time passes, scanner updates ...
let entry = snapshot.entry_for_path("file.rs"); // may be stale!

// GOOD: read fresh snapshot when needed
worktree.read(cx).snapshot().entry_for_path("file.rs")
```

### Blocking on Initial Scan in UI Thread

```rust
// BAD: blocks entire UI
worktree_store.wait_for_initial_scan().await; // blocks foreground!

// GOOD: spawn and update when ready
cx.spawn(async move |weak, cx| {
    worktree_store.wait_for_initial_scan().await;
    weak.update(cx, |this, cx| {
        this.scan_complete = true;
        cx.notify();
    });
});
```

### Hardcoding Paths Instead of Using IDs

```rust
// BAD: breaks on rename
let my_file = "src/main.rs";

// GOOD: use ProjectEntryId
let entry_id = entry.id;
// later: worktree.entry_for_id(entry_id) → works after rename
```

---

## 15) Mental Model

```
Project (Entity<Project>)
│
├─── WorktreeStore (Entity<WorktreeStore>)
│    ├─ Worktree[0] (Local)
│    │  ├─ BackgroundScanner (async task)
│    │  │  ├─ Initial scan → walk directory
│    │  │  └─ Event loop → watch FS changes
│    │  ├─ Snapshot (immutable, cloneable)
│    │  │  ├─ entries_by_path: SumTree<Entry>
│    │  │  └─ entries_by_id: SumTree<PathEntry>
│    │  └─ LocalSnapshot (+ gitignore, git repos)
│    │
│    └─ Worktree[1] (Remote, for collab)
│       └─ RPC-synced snapshot
│
├─── BufferStore (Entity<BufferStore>)
│    ├─ opened_buffers: Map<BufferId, OpenBuffer>
│    ├─ path_to_buffer_id: dedup cache
│    └─ Loading dedup via Shared<Task>
│
├─── LspStore (language servers, per worktree)
├─── GitStore (git state, per repo)
├─── TaskStore (build/run tasks)
├─── DapStore (debugger)
└─── ProjectEnvironment (shell env vars)
```

**Core insight:** Project = data coordinator. Worktree = filesystem snapshot with background scanning. Everything is dual-mode (local/remote) for seamless collaboration.
