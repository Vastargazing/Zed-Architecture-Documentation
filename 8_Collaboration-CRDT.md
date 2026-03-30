# 8. Collaboration & CRDT: Real-Time Editing

The eighth architectural atom — how multiple users edit the same document simultaneously without conflicts. This covers **CRDT (Conflict-free Replicated Data Types), Lamport clocks, version vectors, the RPC protocol, and the collaboration server**.

---

## 1) The Problem: Concurrent Edits

Two users editing the same file at the same time:

```
Initial state: "Hello World"

User A (replica 0):  Insert "!" at offset 11    → "Hello World!"
User B (replica 1):  Insert " Beautiful" at 5   → "Hello Beautiful World"

Both changes happen simultaneously. What's the correct result?
  → "Hello Beautiful World!"

But neither user saw the other's edit when they made theirs.
How do we converge to the same result on both machines?
```

**Answer: CRDT** — a mathematical framework where any order of applying operations converges to the same state.

---

## 2) Zed's CRDT Model

### Core Data Structure

```rust
pub struct Buffer {   // text crate
    snapshot: BufferSnapshot,
    history: History,
    deferred_ops: OperationQueue<Operation>,  // Ops waiting for causal dependencies
    lamport_clock: clock::Lamport,
}

pub struct BufferSnapshot {
    visible_text: Rope,                       // Current document text
    fragments: SumTree<Fragment>,             // All edits ever made
    insertions: SumTree<InsertionFragment>,   // Insertion tracking
    version: clock::Global,                   // Causality frontier
    replica_id: ReplicaId,                    // This replica's ID
}
```

### Fragment Model

Every character ever inserted in the document exists as a Fragment. Deletion doesn't remove fragments — it marks them invisible (tombstoning):

```
Document history:
1. Initial: "Hello"
   Fragment A: { text: "Hello", visible: true, timestamp: (0, 0) }

2. User 0 inserts " World" at offset 5:
   Fragment A: { text: "Hello", visible: true, timestamp: (0, 0) }
   Fragment B: { text: " World", visible: true, timestamp: (1, 0) }

3. User 1 deletes "llo" (offset 2..5):
   Fragment A1: { text: "He", visible: true, timestamp: (0, 0) }
   Fragment A2: { text: "llo", visible: FALSE, timestamp: (0, 0) }  ← tombstoned
   Fragment B:  { text: " World", visible: true, timestamp: (1, 0) }

Visible text: "He World"
Full history: all fragments preserved
```

### Why Tombstoning?

- **Undo** = flip visibility back to true (no data loss)
- **Merge** = fragments provide stable insertion points for concurrent edits
- **History** = can reconstruct any past version

---

## 3) Lamport Clocks: Causal Ordering

### What is a Lamport Clock?

A logical timestamp: `(counter, replica_id)`. Every operation gets a unique timestamp.

```rust
pub type TransactionId = clock::Lamport;

pub struct Lamport {
    pub value: u32,      // Monotonically increasing counter
    pub replica_id: ReplicaId,  // Unique per participant
}
```

### Rules

1. Before performing an operation: increment counter
2. When receiving a remote operation: set counter = max(local, remote) + 1
3. Tie-breaking: if same counter value, compare replica_id

### Why Lamport Clocks?

They provide a **total order** across all operations from all replicas, without requiring synchronized wall clocks.

```
User A (replica 0):  Edit at Lamport(1, 0)
User B (replica 1):  Edit at Lamport(1, 1)

Total order: (1, 0) < (1, 1)  (same counter, lower replica wins)
```

---

## 4) Version Vector: Causality Frontier

A version vector tracks "which operations have I seen?"

```rust
pub struct Global {
    // Flat array: index == replica_id.0, value == highest seq seen
    // SmallVec<[u32; 4]> avoids heap allocation for up to 4 replicas
    values: SmallVec<[u32; 4]>,
}
```

Example:

```
Version { replica_0: 5, replica_1: 3, replica_2: 7 }

Means: "I have seen operations up to Lamport(5,0), Lamport(3,1), and Lamport(7,2)"
```

### Causal Dependency

An operation is **causally ready** if all operations it depends on have been applied. The version vector at the operation's creation time tells us its dependencies.

```
Operation: { timestamp: (4, 1), start_version: { replica_0: 3, replica_1: 3 } }

This operation depends on:
  - All ops from replica 0 up to Lamport(3, 0)
  - All ops from replica 1 up to Lamport(3, 1)

If we haven't seen Lamport(3, 0) yet, this operation goes into deferred_ops
until we receive it.
```

---

## 5) Operation Types

```rust
pub enum Operation {
    Buffer(text::Operation),
    UpdateDiagnostics { server_id, diagnostics, lamport_timestamp },
    UpdateSelections { selections, lamport_timestamp, line_mode, cursor_shape },
    UpdateCompletionTriggers { triggers, lamport_timestamp, server_id },
    UpdateLineEnding { line_ending, lamport_timestamp },
}
```

The core text operation:

```rust
// text crate operations
pub enum Operation {
    Edit(EditOperation),
    Undo(UndoOperation),
}

pub struct EditOperation {
    pub timestamp: clock::Lamport,
    pub version: clock::Global,          // Causal dependencies
    pub ranges: Vec<Range<FullOffset>>,  // What was replaced (FullOffset counts visible+deleted bytes)
    pub new_text: Vec<Arc<str>>,         // Replacement text
}
```

---

## 6) The Collaboration Protocol

### Architecture

```
┌──────────────┐        ┌──────────────┐
│  User A      │        │  User B      │
│  (host)      │        │  (guest)     │
│              │        │              │
│  Project     │        │  Project     │
│  (Local)     │        │  (Remote)    │
│              │        │              │
│  Buffer      │        │  Buffer      │
│  (replica 0) │        │  (replica 8) │  ← first collab ID
└──────┬───────┘        └──────┬───────┘
       │                       │
       │    ┌──────────────┐   │
       └────┤  Collab      ├───┘
            │  Server      │
            │  (relay)     │
            └──────────────┘
```

### RPC Protocol

```rust
pub const PROTOCOL_VERSION: u32 = 68;
// Transport: WebSocket (async_tungstenite)
// Serialization: Protocol Buffers
```

### Client

```rust
pub struct Client {
    peer: Arc<Peer>,                      // RPC peer
    http: Arc<HttpClientWithUrl>,         // API requests
    cloud_client: Arc<CloudApiClient>,
    state: RwLock<ClientState>,           // Connection state
    handler_set: ProtoMessageHandlerSet,  // Message routing
}

actions!(client, [SignIn, SignOut, Reconnect]);
```

### Message Flow: Edit Propagation

```
User A types "x"
    ↓
Buffer::edit() → create EditOperation with Lamport timestamp
    ↓
Apply locally (immediate, no latency)
    ↓
Serialize Operation → proto::Operation
    ↓
Send via RPC to collab server
    ↓
Server relays to all other participants
    ↓
User B receives proto::Operation
    ↓
Deserialize → check causal dependencies (version vector)
    ↓
Dependencies met? → apply immediately
    ↓
Dependencies missing? → queue in deferred_ops
    ↓
When missing ops arrive → apply deferred ops in order
```

---

## 7) Shared State vs Private State

### Shared (Synchronized)

- **Buffer text** (via CRDT operations)
- **Selections/cursors** (each user's selections visible to others)
- **Diagnostics** (host relays from language servers)
- **Open files** (which files are being edited)

### Private (Not Synchronized)

- **Window layout** (split arrangement, dock state)
- **Scroll position** (each user scrolls independently)
- **Fold state** (what's folded)
- **Settings** (each user's preferences)
- **Undo history** (each user has their own undo stack)

---

## 8) Conflict Resolution

### Insert-Insert Conflict

Two users insert at the same position:

```
User A inserts "X" at offset 5, timestamp (1, 0)
User B inserts "Y" at offset 5, timestamp (1, 1)

CRDT resolution: order by timestamp
(1, 0) < (1, 1)  →  "X" comes before "Y"

Result: "HelloXY World" (deterministic on both machines)
```

### Delete-Insert Conflict

One user deletes a range while another inserts inside it:

```
Text: "Hello World"
User A deletes "lo Wo" (offset 3..8)
User B inserts "X" at offset 6 (inside the deleted range)

Resolution: "X" survives! It's a new insertion.
Tombstone the deleted fragments, but X is a new fragment.

Result: "HelXrld"
```

### Undo Semantics

Each user's undo only undoes their own operations:

```
User A types "Hello"
User B types "World"
User A presses Ctrl+Z

Result: "World" remains. Only User A's "Hello" is undone.
```

---

## 9) The Collab Server

### Purpose

The collab server (`crates/collab/`) is a **relay and coordinator**, not a conflict resolver. CRDT resolution happens on each client.

### Server Responsibilities

- **User authentication** (GitHub OAuth)
- **Room management** (create/join/leave rooms)
- **Message relaying** (forward operations between participants)
- **Presence** (who's in the room, what files they have open)
- **Channel management** (persistent collaborative spaces)
- **Channel buffer persistence** (stores snapshots + operations for collaborative channels in DB)

### What the Server Does NOT Do

- Resolve text conflicts (CRDT handles this on each client)
- Store document contents for live project sessions (ephemeral — only relayed)
- Make editing decisions (pure relay)

> Note: Channel buffers (collaborative spaces) **are** persisted server-side in `buffer_snapshot` / `buffer_operation` DB tables. Only peer-to-peer project sessions are truly ephemeral.

---

## 10) Remote Project: The Guest Experience

When a guest joins a collaboration session:

```
Guest connects
    ↓
Server sends project metadata (worktree structure, open files)
    ↓
Guest creates Project (Remote variant)
    ↓
Guest creates RemoteWorktree (receives file tree via RPC)
    ↓
Guest opens file → RPC request to host → host sends buffer content
    ↓
Guest's Buffer created with host's content + replica_id
    ↓
Edits flow bidirectionally via CRDT operations
    ↓
LSP requests proxied through host (guest has no language servers)
```

---

## 11) Replica Management

```
Reserved IDs (clock::ReplicaId constants):
  0 = LOCAL          — the local user / host
  1 = REMOTE_SERVER  — SSH remote server
  2 = AGENT          — AI agent
  3 = LOCAL_BRANCH   — local branch operations
  4–7               — reserved
  8+ = collab guests — assigned sequentially from FIRST_COLLAB_ID

Each replica maintains its own:
├── Lamport clock (independent counter)
├── Undo history (only own operations)
├── Selection state (own cursors)
└── Version vector (what's been seen)
```

### Replica ID Assignment

The collab server assigns replica IDs to guests when they join. The host always uses `ReplicaId::LOCAL` (0). Guest IDs start at `ReplicaId::FIRST_COLLAB_ID` (8) and increment to avoid conflicts with reserved IDs 1–7.

---

## 12) Deferred Operations Queue

Operations may arrive out of order due to network conditions:

```rust
// Tuple struct wrapping a SumTree (B-tree with aggregated summaries)
// Ordered by Lamport timestamp; supports efficient range queries and dedup
pub struct OperationQueue<T: Operation>(SumTree<OperationItem<T>>);
```

When an operation arrives whose causal dependencies aren't met (its start version references operations we haven't seen yet), it goes into the deferred queue. Once the missing operations arrive, deferred operations are applied.

**Invariant:** After applying all received operations, every replica converges to the same document state. This is guaranteed by the CRDT math.

---

## 13) Anti-Patterns

### Assuming Instant Delivery

```rust
// BAD: assuming operations arrive in order
fn on_remote_edit(op: Operation) {
    self.buffer.apply(op); // may fail if dependencies not met!
}

// GOOD: check causal readiness
fn on_remote_edit(op: Operation) {
    if self.buffer.can_apply(&op) {
        self.buffer.apply(op);
        self.buffer.flush_deferred();
    } else {
        self.buffer.defer(op);
    }
}
```

### Synchronizing UI Layout

```rust
// BAD: trying to sync splits across users
fn on_remote_split(pane_config: PaneGroupConfig) {
    self.workspace.apply_layout(pane_config); // Don't! Each user owns their layout
}

// GOOD: only sync data, not UI
// Each user's workspace is independent
```

### Using Wall Clock for Ordering

```rust
// BAD: wall clocks differ between machines
let timestamp = SystemTime::now();

// GOOD: use Lamport clock
let timestamp = self.lamport_clock.tick();
```

---

## 14) Mental Model

```
Collaboration System
│
├─── CRDT Layer (per Buffer)
│    ├─ Fragments: SumTree of all edits ever made
│    ├─ Lamport Clock: (counter, replica_id) for ordering
│    ├─ Version Vector: "what have I seen" frontier
│    ├─ Deferred Ops: queue for out-of-order arrivals
│    └─ Guarantee: all replicas converge to same text
│
├─── RPC Layer
│    ├─ Client (WebSocket → Collab Server)
│    ├─ Protobuf serialization
│    ├─ Request/Response + Notification patterns
│    └─ Protocol version: 68
│
├─── Collab Server
│    ├─ Relay (forwards operations)
│    ├─ Room management
│    ├─ User authentication (GitHub OAuth)
│    ├─ Channel buffer persistence (snapshots + ops in DB)
│    └─ NOT a conflict resolver (CRDT is client-side)
│
├─── Shared State
│    ├─ Buffer text (CRDT)
│    ├─ Selections/cursors
│    ├─ Diagnostics
│    └─ Open files
│
├─── Private State (per user)
│    ├─ Window layout
│    ├─ Scroll/fold state
│    ├─ Settings
│    └─ Undo history
│
└─── Remote Project
     ├─ Guest gets RemoteWorktree (file tree via RPC)
     ├─ Buffers synced via CRDT operations
     ├─ LSP proxied through host
     └─ Replica IDs assigned by server (guests start at ID 8+)
```

**Core insight:** Zed uses a CRDT with Lamport clocks for conflict-free real-time collaboration. Every edit is a Fragment in a SumTree. Tombstoning (not deletion) enables undo. Version vectors ensure causal consistency. The collab server is a pure relay — all intelligence is in the client-side CRDT.
