# 4. Editor & Buffer: Text Editing Engine

The fourth architectural atom — how text is stored, transformed, and rendered on screen. This covers **Rope data structure, display map pipeline, selections, anchors, undo/redo, and syntax highlighting**.

---

## 1) Core Architecture Overview

```
             ┌─────────────────────────────┐
             │   Editor (Entity<Editor>)     │
             │   UI controller + bindings    │
             └──────────────┬───────────────┘
                            │
             ┌──────────────▼───────────────┐
             │  MultiBuffer (Entity)         │
             │  Excerpt aggregator           │
             └──────────────┬───────────────┘
                            │
             ┌──────────────▼───────────────┐
             │  Buffer (language crate)      │
             │  Syntax + LSP + diagnostics   │
             └──────────────┬───────────────┘
                            │
             ┌──────────────▼───────────────┐
             │  TextBuffer (text crate)      │
             │  CRDT + Rope + history        │
             └──────────────┬───────────────┘
                            │
             ┌──────────────▼───────────────┐
             │  Rope (SumTree<Chunk>)        │
             │  Raw text storage             │
             └──────────────────────────────┘
```

<img width="1440" height="890" alt="image" src="https://github.com/user-attachments/assets/4867a0a7-7c0a-4c4f-83c4-e9bba28f340f" />


Each layer adds capabilities on top of the previous one.

---

## 2) Rope: Text Storage

### What is a Rope?

A Rope is a balanced tree of text chunks. Instead of one continuous `String`, text is split across many small chunks (64–128 bytes each), stored in a SumTree.

```rust
pub struct Rope {
    chunks: SumTree<Chunk>,  // B-tree of text pieces
}
```

### Why Not a String?

| Operation | String | Rope |
|-----------|--------|------|
| Insert at position | O(n) — shift all bytes after | O(log n) — split + insert |
| Delete range | O(n) — shift bytes | O(log n) — split + merge |
| Get line count | O(n) — scan for `\n` | O(1) — stored in summary |
| Get byte at offset | O(1) | O(log n) — tree walk |
| Clone | O(n) — copy all | O(1) — Arc shared nodes |

For a 100MB file, insert operations go from ~100ms (String) to ~1μs (Rope).

### SumTree Properties

Each SumTree node tracks summaries:
- Byte count
- UTF-16 offset count (for LSP compatibility)
- Line count
- Point (row, col) position

This means converting between offset, point, UTF-16 offset is O(log n) via tree seek.

### Key Operations

```rust
impl Rope {
    fn append(&mut self, other: Rope);
    fn replace(&mut self, range: Range<usize>, text: &str);
    fn slice(&self, range: Range<usize>) -> Rope;
    fn offset_to_point(&self, offset: usize) -> Point;
    fn point_to_offset(&self, point: Point) -> usize;
    fn line_len(&self, row: u32) -> u32;
    fn len(&self) -> usize;
    fn chars(&self) -> impl Iterator<Item = char>;
}
```

---

## 3) TextBuffer: CRDT Layer

TextBuffer wraps Rope with collaboration support.

```rust
pub struct Buffer {   // text crate
    snapshot: BufferSnapshot,
    history: History,                        // Undo/redo stacks
    deferred_ops: OperationQueue<Operation>, // CRDT ops waiting for dependencies
    lamport_clock: clock::Lamport,           // Logical timestamp
}

pub struct BufferSnapshot {
    visible_text: Rope,                       // Current document text
    deleted_text: Rope,                       // Tombstoned text (kept for CRDT undo/merge)
    fragments: SumTree<Fragment>,             // All edits, ordered by Lamport timestamp
    insertions: SumTree<InsertionFragment>,   // Insertion tracking
    version: clock::Global,                   // Causality frontier
    replica_id: ReplicaId,                    // This replica's ID
}
```

### Fragment Model

Every edit in the document's history is a Fragment:

```
Document: "Hello World"

Fragments:
├── Fragment { text: "Hello ", visible: true, timestamp: (1, 0) }
├── Fragment { text: "Beautiful ", visible: true, timestamp: (3, 1) }  ← insertion
└── Fragment { text: "World", visible: true, timestamp: (2, 0) }

Undo "Beautiful ":
├── Fragment { text: "Hello ", visible: true }
├── Fragment { text: "Beautiful ", visible: FALSE }  ← tombstoned
└── Fragment { text: "World", visible: true }
```

Visible text = concatenation of all visible fragments. Undo = flip visibility, not delete.

---

## 4) Language Buffer: Syntax Layer

```rust
pub struct Buffer {   // language crate
    text: TextBuffer,
    syntax_map: Mutex<SyntaxMap>,
    diagnostics: TreeMap<LanguageServerId, DiagnosticSet>,
    remote_selections: TreeMap<ReplicaId, SelectionSet>,
    language: Option<Arc<Language>>,
    capability: Capability,  // ReadWrite / ReadOnly
    file: Option<Arc<dyn File>>,
}
```

Adds:
- **Syntax highlighting** via Tree-sitter
- **Diagnostics** from LSP servers
- **Remote selections** for collaboration
- **Language configuration** (indentation, brackets, comments)
- **File association** (path, mtime, dirty state)

---

## 5) MultiBuffer: Excerpt Aggregation

### Why MultiBuffer?

An Editor doesn't always show a single file. It can show:
- One file (normal editing)
- Multiple excerpts from different files (search results, diagnostics view)
- Multiple ranges from the same file (collapsed view)

```rust
pub struct MultiBuffer {
    snapshot: RefCell<MultiBufferSnapshot>,
    buffers: BTreeMap<BufferId, BufferState>,
    excerpts_by_path: BTreeMap<PathKey, Vec<ExcerptId>>,
    paths_by_excerpt: HashMap<ExcerptId, PathKey>,
    diffs: HashMap<BufferId, DiffState>,
    singleton: bool,   // Optimization: true if just one buffer
    history: History,   // MultiBuffer-level undo/redo
}
```

### Excerpt Model

```
MultiBuffer (diagnostics view)
├── Excerpt[0]: file_a.rs lines 10-15 (where error is)
├── Excerpt[1]: file_a.rs lines 40-45 (another error)
├── Excerpt[2]: file_b.rs lines 5-10  (third error)
└── Excerpt[3]: file_c.rs lines 1-3   (warning)
```

Each excerpt has:
- `ExcerptId` — unique identifier
- Source `Buffer` + `Range<Anchor>` — which part of which file
- Context padding lines before/after

### Singleton Optimization

Most editors show one file. When `singleton: true`, MultiBuffer skips the excerpt indirection and delegates directly to the underlying Buffer. This avoids overhead for the common case.

---

## 6) Display Map: Text-to-Pixels Pipeline

### The Six Layers

Text goes through six transformation layers before reaching the screen:

```
Buffer Text (MultiBufferOffset / Point)
    │
    ▼ CreaseMap ── Tracks explicitly foldable ranges (LSP fold ranges, supersede auto-indent detection)
    │
    ▼ InlayMap ─── Injects inlay hints (type annotations, parameter names)
    │
    ▼ FoldMap ──── Collapses code folds ("⋯")
    │
    ▼ TabMap ───── Expands tabs to spaces
    │
    ▼ WrapMap ──── Soft-wraps long lines at window edge
    │
    ▼ BlockMap ─── Inserts custom block decorations (diagnostics, diff hunks)
    │
Display Text (DisplayRow / DisplayPoint)
```

Each layer:
- Has its own **coordinate space** (InlayPoint, FoldPoint, TabPoint, WrapPoint, DisplayPoint)
- Has a **Snapshot** (immutable state at a point in time)
- Has **coordinate converters** (A_point → B_point)
- Has a **sync()** method to apply buffer edits through the layer

### Layer 1: InlayMap

Inserts zero-width or short text that doesn't exist in the buffer:

```
Buffer: |let x = calculate()|
Inlay:  |let x: i32 = calculate(value: 42)|
                ^^^^^              ^^^^^^^^^^
               type hint         parameter hint
```

Transform: `Isomorphic | Inlay`

### Layer 2: FoldMap

Collapses regions (functions, comments, imports) into placeholders:

```
Before fold:
|fn main() {          |
|    let x = 1;       |
|    let y = 2;       |
|    println!("{}", x);|
|}                     |

After fold:
|fn main() {⋯}        |
```

```rust
pub struct FoldPlaceholder {
    pub render: Arc<dyn Fn(FoldId, Range<Anchor>, &mut App) -> AnyElement>,
    pub constrain_width: bool,
    pub merge_adjacent: bool,
    pub collapsed_text: Option<SharedString>,  // LSP-provided text
}
```

Transform: `Isomorphic | Fold(placeholder)`

### Layer 3: TabMap

Converts hard tabs (`\t`) to visual spaces. Configurable tab width.

### Layer 4: WrapMap

Soft-wraps lines that exceed the editor's pixel width:

```rust
struct Transform {
    summary: TransformSummary,
    display_text: Option<&'static str>,  // None = no wrap, Some = "\n" + indent
}
```

- `wrap_width: Option<Pixels>` — target width (None = no wrapping)
- Rewrapping happens in a **background task** for large files
- Pending edits are queued while rewrap is in progress

### Layer 5: BlockMap

Injects custom block elements between lines:

```
|fn main() {                    |
|  ┌─ error: mismatched types ─┐|  ← BlockMap injection
|  │  expected i32, found &str │|
|  └───────────────────────────┘|
|    let x: i32 = "hello";     |
|}                               |
```

Block types: `AboveLine(height)`, `BelowLine(height)`

Used for: diagnostics inline, git diff hunks, code lenses.

---

## 7) Selections and Cursors

### Selection Model

```rust
pub struct Selection<T> {
    id: usize,
    start: T,        // Can be Anchor, Point, or Offset
    end: T,
    reversed: bool,  // Cursor at start (true) or end (false)
    goal: SelectionGoal,
}
```

**SelectionGoal**: Remembers the intended column when moving vertically through lines of different lengths.

```
Cursor at column 50, line has 80 chars → column 50
Move down, next line has 30 chars → column 30 (clamped)
Move down again, next line has 80 chars → column 50 (restored from goal)
```

### SelectionsCollection

```rust
pub struct SelectionsCollection {
    disjoint: Arc<[Selection<Anchor>]>,    // Non-overlapping selections
    pending: Option<PendingSelection>,      // Mouse drag in progress
    select_mode: SelectMode,                // Character / Line / Word
    line_mode: bool,                        // Vim visual line
    is_extending: bool,                     // Shift+Click extend
}
```

**Multiple cursors**: `disjoint` is an array — Zed natively supports multiple simultaneous selections/cursors.

**Invariant**: Selections in `disjoint` are always sorted and non-overlapping. After any mutation, they are merged if they overlap.

---

## 8) Anchors: Stable Positions

### The Problem

When text is inserted before your cursor, your cursor position (as an offset) becomes wrong:

```
Before: "Hello|World" (cursor at offset 5)
Insert "!!" at offset 3: "Hel!!lo|World"
Cursor should be at offset 7, not 5
```

### The Solution: Anchors

An Anchor references a specific edit operation (Lamport timestamp) rather than a byte offset:

```rust
// TextBuffer-level anchor
// Note: timestamp is stored as two inline fields (replica_id + value) to save 8 bytes
pub struct Anchor {
    pub(crate) timestamp_replica_id: clock::ReplicaId,
    pub(crate) timestamp_value: clock::Seq,
    pub offset: u32,            // Offset within that edit's text
    pub bias: Bias,             // Left or Right
    pub buffer_id: Option<BufferId>,  // Which buffer this anchor belongs to
}

// MultiBuffer-level anchor (adds excerpt ID)
pub struct Anchor {
    pub excerpt_id: ExcerptId,
    pub text_anchor: text::Anchor,
    pub diff_base_anchor: Option<text::Anchor>,
}
```

### Bias

When text is inserted exactly at an anchor's position, should the anchor move left or right?

```
Text: "ab|cd"  (anchor at position between b and c)
Insert "XY" at that position:

Bias::Left:  "ab|XYcd"  (anchor stays before insertion)
Bias::Right: "abXY|cd"  (anchor stays after insertion)
```

Selections use `Bias::Left` for start and `Bias::Right` for end, so they grow when text is inserted at their boundaries.

---

## 9) Coordinate Spaces

Each display map layer introduces its own coordinate space:

```
Offset         → byte offset in buffer
Point          → (row, col) in buffer
InlayPoint     → after inlay injection
FoldPoint      → after code folding
TabPoint       → after tab expansion
WrapPoint      → after soft wrapping
DisplayPoint   → final screen coordinates (DisplayRow, DisplayCol)
```

Conversion utilities:

```rust
pub trait ToOffset { fn to_offset(&self, snapshot: &BufferSnapshot) -> usize; }
pub trait ToPoint { fn to_point(&self, snapshot: &BufferSnapshot) -> Point; }
pub trait ToOffsetUtf16 { fn to_offset_utf16(&self, snapshot) -> OffsetUtf16; }
pub trait ToPointUtf16 { fn to_point_utf16(&self, snapshot) -> PointUtf16; }
```

UTF-16 support is needed for LSP protocol compatibility (LSP uses UTF-16 offsets).

---

## 10) Undo/Redo: Transaction System

### Data Structures

```rust
pub struct Transaction {
    pub id: TransactionId,           // Lamport timestamp
    pub edit_ids: Vec<clock::Lamport>, // All edits in this transaction
    pub start: clock::Global,        // Version vector at transaction start
}

pub struct HistoryEntry {
    transaction: Transaction,
    first_edit_at: Instant,
    last_edit_at: Instant,
    suppress_grouping: bool,
}

struct History {
    base_text: Rope,                                    // Original text before any local edits
    operations: TreeMap<clock::Lamport, Operation>,     // All CRDT ops (for collaborative replay)
    undo_stack: Vec<HistoryEntry>,
    redo_stack: Vec<HistoryEntry>,
    transaction_depth: usize,
    group_interval: Duration,  // ~300ms auto-grouping
}
```

### Flow

```
1. start_transaction()  → push entry on undo_stack
2. User types "abc"     → three edits, all tracked by same TransactionId
3. end_transaction()    → entry becomes immutable, redo stack cleared
4. Undo                 → pop from undo_stack, push to redo_stack
5. Redo                 → pop from redo_stack, push to undo_stack
```

### Auto-Grouping

Edits within **300ms** of each other are automatically grouped into one undo step:

```
Type "H" at T=0ms
Type "e" at T=100ms    → same group (100 < 300)
Type "l" at T=200ms    → same group
Type "l" at T=250ms    → same group
Type "o" at T=800ms    → new group (800 - 250 > 300)
```

Result: Undo undoes "Hell" as one step, then "o" as another.

### MultiBuffer Transaction Coordination

When editing in a MultiBuffer (e.g., search-and-replace across files), the undo system wraps all buffer transactions into one MultiBuffer transaction. Undoing one undo step reverts changes across all affected files.

```rust
// MultiBuffer tracks which buffers were modified per transaction
HashMap<BufferId, text::TransactionId>
```

---

## 11) Syntax Highlighting: Tree-sitter Integration

### SyntaxMap

```rust
pub struct SyntaxSnapshot {
    // SumTree of internal SyntaxLayerEntry items (not Vec!)
    // SyntaxLayer<'a> is a public lifetime-bounded view over these entries
    layers: SumTree<SyntaxLayerEntry>,
    parsed_version: clock::Global,
    interpolated_version: clock::Global,
    language_registry_version: usize,
    update_count: usize,
}

// Public view — returned by iterators, holds refs into the SumTree
pub struct SyntaxLayer<'a> {
    pub language: &'a Arc<Language>,
    pub(crate) depth: usize,
    tree: &'a tree_sitter::Tree,
    pub offset: (usize, tree_sitter::Point),
}
```

### Highlighting Flow

```
Buffer text changes
    ↓
Tree-sitter incremental re-parse (only changed regions)
    ↓
SyntaxMap updated with new tree
    ↓
Query highlights_config against tree
    ↓
Captures → (Range<usize>, HighlightId) pairs
    ↓
HighlightMap → maps HighlightId to theme colors
    ↓
Renderer uses colors for text painting
```

### Language Injection

A single file can have multiple languages (HTML + CSS + JS, Markdown + code blocks):

```
Markdown buffer:
├── Layer 0: Markdown (entire file)
├── Layer 1: Rust (within ```rust blocks)
├── Layer 2: JavaScript (within ```js blocks)
└── Layer 3: Python (within ```python blocks)
```

Each layer gets its own Tree-sitter tree and highlight queries.

### Grammar Configuration

```rust
pub struct Grammar {
    id: GrammarId,
    highlights_config: Option<HighlightsConfig>,  // Syntax colors
    brackets_config: Option<BracketsConfig>,      // () [] {}
    outline_config: Option<OutlineConfig>,        // Symbol outline
    indents_config: Option<IndentConfig>,         // Auto-indent rules
    injection_config: Option<InjectionConfig>,    // Language injection
    ts_language: tree_sitter::Language,
}
```

---

## 12) Editor Entity: The UI Controller

```rust
pub struct Editor {
    buffer: Entity<MultiBuffer>,
    display_map: Entity<DisplayMap>,
    selections: SelectionsCollection,
    scroll_manager: ScrollManager,
    blink_manager: Entity<BlinkManager>,
    
    // Visual decorations
    background_highlights: BTreeMap<TypeId, BackgroundHighlights>,
    gutter_highlights: TreeMap<TypeId, GutterHighlights>,
    
    // LSP integration
    diagnostics: Vec<DiagnosticEntry>,
    code_actions_task: Option<Task<()>>,
    
    // Multi-cursor modes
    columnar_selection_state: Option<ColumnarSelectionState>,
    add_selections_state: Option<AddSelectionsState>,
}
```

The Editor entity orchestrates everything:
- Receives Actions (MoveUp, MoveDown, Newline, Undo, etc.)
- Updates selections via SelectionsCollection
- Triggers display map updates
- Coordinates with LSP for completions, hover, diagnostics

---

## 13) Anti-Patterns

### Confusing Coordinate Spaces

```rust
// BAD: using buffer offset where display point is needed
let offset = editor.selections.newest().head();
paint_cursor_at(offset); // WRONG! Doesn't account for folds, wraps

// GOOD: convert through display map
let display_point = snapshot.offset_to_display_point(offset);
paint_cursor_at(display_point);
```

### Holding Snapshots Too Long

```rust
// BAD: snapshot becomes stale
let snapshot = buffer.read(cx).snapshot();
// ... lots of code, maybe async ...
let text = snapshot.text(); // May not match current buffer!

// GOOD: take snapshot close to use
let text = buffer.read(cx).snapshot().text();
```

### Modifying Selections Without Merging

```rust
// BAD: overlapping selections after manual manipulation
selections.push(Selection { start: 5, end: 10 });
selections.push(Selection { start: 8, end: 15 }); // overlaps!

// GOOD: always go through SelectionsCollection which auto-merges
```

---

## 14) Mental Model

```
Editor (Entity<Editor>)
│
├─── MultiBuffer (Entity<MultiBuffer>)
│    ├─ Excerpts (regions from one or more Buffers)
│    └─ MultiBuffer-level undo/redo
│
├─── Buffer (language crate)
│    ├─ TextBuffer (text crate)
│    │  ├─ Rope (SumTree<Chunk>) ← actual text bytes
│    │  ├─ Fragments (SumTree<Fragment>) ← CRDT edit history
│    │  ├─ Lamport clock ← causal ordering
│    │  └─ History (undo/redo stacks)
│    ├─ SyntaxMap (Tree-sitter layers)
│    ├─ Diagnostics (per LSP server)
│    └─ Remote selections (for collab)
│
├─── DisplayMap (Entity<DisplayMap>)
│    ├─ InlayMap → inlay hints
│    ├─ FoldMap → code folding
│    ├─ TabMap → tab expansion
│    ├─ WrapMap → soft wrapping
│    └─ BlockMap → block decorations
│
├─── SelectionsCollection
│    ├─ disjoint: sorted, non-overlapping
│    ├─ pending: mouse drag in progress
│    └─ Anchors for stability across edits
│
└─── Coordinate Spaces
     Offset ↔ Point ↔ InlayPoint ↔ FoldPoint
     ↔ TabPoint ↔ WrapPoint ↔ DisplayPoint
```

**Core insight:** Text is stored as a Rope (SumTree) for O(log n) edits. Anchors keep positions stable across edits. The display map pipeline transforms text through 5 layers before rendering. Everything is snapshot-based for thread-safe reads.
