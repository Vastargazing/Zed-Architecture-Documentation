# 2. Workspace & Pane: Layout and Navigation

The second architectural atom — managing visibility, navigation, and window layout. This covers **split trees, tab management, dock panels, persistence, and focus routing**.

---

## 1) Fundamental Separation: Workspace vs Pane vs Panel

### Ownership Hierarchy

```
Workspace (Entity)
├── Center (PaneGroup)
│   └── Pane (Entity<Pane>)
│       └── PaneItem / Tab (buffer, editor, terminal, etc.)
│
└── Dock
    ├── Panel left (Project panel, Diagnostics, etc.)
    ├── Panel right (Chat, Outline, etc.)
    └── Panel bottom (Terminal, Output, etc.)
```

**Key idea:** Workspace = container/coordinator, Pane = tab collection, Panel = fixed-position sidebar.

**Why this separation:**

- `Pane` — a collection of tabs in one split region (one focus timeline)
- `Panel` — a fixed position element, toggled show/hide
- `Workspace` — the main coordinator (knows which Pane is active, which Panels are visible)

---

## 2) PaneGroup: Tree of Splits (Not a Linear List)

### Why a Tree Structure?

A user can create arbitrary split layouts:

```
┌─ Editor 1 ─┬─ Editor 3 ─┐
│             │             │
├─ Editor 2 ──┤ Editor 4 ──┤
│             │             │
└─ Editor 5 ─┴─ Editor 6 ─┘
```

This is not a flat array. It's a recursive tree of splits.

**PaneGroup as a tree:**

```
// PaneGroup is a wrapper with a single root: Member
// Member is a recursive enum:

Member::Axis(PaneAxis {
    axis: Horizontal,
    flexes: [1.0, 2.0],   // relative weights, NOT fractions (sum = members.len default)
    members: [
        Member::Axis(PaneAxis {
            axis: Vertical,
            flexes: [1.0, 1.0, 1.0],
            members: [Member::Pane(Pane1), Member::Pane(Pane2), Member::Pane(Pane5)]
        }),
        Member::Axis(PaneAxis {
            axis: Vertical,
            flexes: [1.0, 1.0, 1.0],
            members: [Member::Pane(Pane3), Member::Pane(Pane4), Member::Pane(Pane6)]
        })
    ]
})
```

**Rule:** Each split has an axis (Horizontal/Vertical) and flexes — **relative weights** (default `1.0` per pane, equal split). A `[2.0, 1.0]` pair means 2:1 ratio.

**Complexity when closing a Pane:**

1. Remove from tree
2. Rebalance neighbor flexes
3. If remaining subtree is empty → collapse the `Axis`

---

## 3) Active Path: How the System Knows Where Events Go

### Focus is a Path Through the Tree

```
Workspace
├── Center (PaneGroup { root: Member::Axis(...) })
│   ├── left child (Member::Axis(PaneAxis))
│   │   └── active_pane → Member::Pane(Pane[1])  ← FOCUS IS HERE
│   └── right child (Member::Axis(PaneAxis))
│       └── Member::Pane(Pane[3])
│
└── Dock
    └── left (ProjectPanel) — hidden
```

**active_pane** = a single path to a leaf in the tree.

**When a key is pressed:**

1. Event goes to `active_pane` (which holds an Editor or other item)
2. The item (e.g., Editor) gets focus
3. If the item doesn't consume the event — it bubbles up (Pane → Workspace)

**Invariant:** There is always exactly one active Pane in Center at any moment.

---

## 4) Tab Lifecycle: Open → Activate → Close

### 4.1: Open

```
User: "Open file.rs"
  ↓
Workspace::open_path(project_path, cx)         // or add_item_to_active_pane(item, cx)
  ├─ Project::open_buffer(path) → creates Entity<Buffer>
  ├─ active_pane.add_item(box_item, cx)
  ├─ WorkspaceDb::global(cx).save_items(...)    // persists open tabs
  └─ cx.notify()  ← re-render with new tab
```

**Failure modes:**
- File doesn't exist (Project returns error)
- Buffer not persisted to state (loss of open_tabs on restart)

### 4.2: Activate

```
User: clicks tab2
  ↓
EditorElement::on_click() (in pane_ui)
  ├─ pane.activate_item(tab2)
  └─ cx.notify()  ← re-render, tab2 is now highlighted, Editor2 is visible
```

**Invariant:** After activate, the active tab in `active_pane` = the tab the user sees.

### 4.3: Close

```
User: closes tab1
  ↓
pane.close_item(tab1)
  ├─ If tab1 = active → need to pick another (usually adjacent)
  ├─ If this was the last tab in Pane → can the Pane be empty?
  │  (Policy: Center Pane CANNOT be empty)
  ├─ If Pane becomes empty → collapse it from PaneGroup
  └─ WorkspaceDb.save()
```

**Focus fallback on close:**

1. Adjacent tab in same Pane → focus that
2. No tabs left → move focus to neighboring Pane
3. Last Pane in Center → it stays (empty or with placeholder)

---

## 5) Panel: Dock Management

### Panel vs Pane

```
Pane (Center):           Panel (Dock):
├─ Multiple tabs          ├─ Single UI (or none)
├─ User-created           ├─ App-provided (Project, Diagnostics, Chat)
└─ Text/editor content    └─ Specialized UI

Workspace knows:
├─ active Center Pane (one)
├─ Panel[left], Panel[right], Panel[bottom] (each show/hide)
```

### Panel Lifecycle

```
RegisterPanel("project_panel", ProjectPanel, "left")
  → at app startup

User toggles Panel:
  ├─ toggle_panel("project_panel", cx)
  ├─ if visible: set hidden, notify
  ├─ if hidden: set visible, notify
  └─ WorkspaceDb.save({ dock_state })
```

**Complexity:** A Panel can be resizable (width changes on left dock), but resizing AFFECTS Center width (reduces space for Panes).

---

## 6) Persistence: WorkspaceDb

### Why?

User closes Zed, opens again → should see the same layout.

### What Gets Saved?

```json
{
  "workspace": {
    "open_tabs": [
      { "pane": 0, "item": "buffer_id", "position": 0 },
      { "pane": 1, "item": "buffer_id", "position": 100 }
    ],
    "pane_group": {
      "axis": "horizontal",
      "flexes": [1.0, 1.0],   // relative weights (equal split by default)
      "members": [...]
    },
    "active_pane": 0,
    "dock": {
      "left": { "panel": "project", "visible": true, "width": 300 },
      "bottom": { "panel": "terminal", "visible": false }
    }
  }
}
```

### How Load Works

```
Zed starts:
  ├─ WorkspaceDb::load()
  ├─ Reconstruct PaneGroup from saved JSON
  ├─ Create Entity<Pane> for each Pane in tree
  ├─ open_tab(buffer_id) for each tab
  ├─ Set active_pane from JSON
  └─ cx.notify() → render

Pitfall: Buffer may disappear (file deleted)
  → graceful: close tab or show placeholder
```

---

## 7) Activation vs Focus

### Active ≠ Focused

```
Active = "this is the Pane the user intends to work in"
Focused = "this is the exact element receiving key events"

Example:
├─ active_pane = Pane[left] (user clicked it)
│  ├─ active_item = File1
│  └─ focus = Editor.cursor (inside File1)
│
└─ Pane[right] (not active, but visible)
   └─ File2 (visible, but no focus)

Focus events go to → Editor.cursor
Pane switching goes to → active_pane selector
```

**Why separated:**

- Activation = "this is the primary Pane"
- Focus = "this specific input field"
- Different events route to different targets

---

## 8) Split/Unsplit: Tree Mutation

### User Presses "Split Right"

```
Current tree:
├─ Member::Pane(Pane[1])  ← active

User: Workspace::split_pane(active_pane, SplitDirection::Right, cx)

New tree:
├─ Member::Axis(PaneAxis {
       axis: Horizontal,
       flexes: [1.0, 1.0],   // equal by default
       members: [
           Member::Pane(Pane[1]),  // old
           Member::Pane(Pane[2]),  // new, contains copy of buffer
       ]
   })
```

**Complexity:**

- Find which `Axis` contains the active `Pane`
- Replace `Member::Pane` with `Member::Axis`, add sibling
- Recalculate flexes (default `1.0` each = equal split)

**Pitfall:** If PaneGroup tree is incorrectly represented in memory → split breaks invariants.

---

## 9) Invariants

### Invariant 1: Center Always Has At Least One Pane

```rust
// NEVER:
center.is_empty()  // → panic

// ALWAYS:
center.panes().len() > 0
```

Why: what would you render with no Panes?

### Invariant 2: active_pane Always Exists

```rust
// NEVER:
workspace.active_pane()  // None → panic

// ALWAYS:
workspace.active_pane()  // Some(pane_entity)
```

### Invariant 3: PaneGroup Tree is Balanced

```rust
// Flexes must sum to members.len() (relative weights, default 1.0 per member)
// Validation on load: if sum deviates → reset to vec![1.0; members.len()]
// Axis must be consistent with content (Vertical axis → stacked vertically)
// No empty Axis nodes (collapse them)
```

### Invariant 4: Closed Pane Has No Tabs

```rust
// NEVER:
pane.tabs.len() > 0 && pane.is_closed()

// ALWAYS:
// Closed Pane = removed from tree, all tabs closed
```

---

## 10) Dock Positioning

### Three Positions, Three Policies

```
Left Dock:          Center:         Right Dock:
┌──────────────┬──────────────┬──────────────┐
│  Project     │   Editors    │    Chat      │
│  (resizable) │ (remaining)  │ (resizable)  │
└──────────────┴──────────────┴──────────────┘

Bottom Dock:
└──────────────────────────────────────────────┐
│  Terminal (height resizable)                  │
└──────────────────────────────────────────────┘
```

**Complexity:**

- Left + Right → reduce Center width
- Bottom → reduces Center height
- Resizing Left can affect Right (if both are narrow)
- All three need coordination

**Policy:**

1. Load sizes from DB
2. On each resize → save size to DB
3. On next launch → restore sizes

---

## 11) Navigation & Commands

### Built-in Navigation

Actions (dispatched via keybindings or `cx.dispatch_action`):

```
ActivatePaneLeft    → Pane to the left gets focus
ActivatePaneRight   → Pane to the right
ActivatePaneUp      → Pane above
ActivatePaneDown    → Pane below
SwapPaneLeft / SwapPaneRight / SwapPaneUp / SwapPaneDown  → swap positions
MovePaneLeft / MovePaneRight / MovePaneUp / MovePaneDown  → move to edge

// Split:
Workspace::split_pane(pane, SplitDirection::Right, cx)
```

**Why built-in?**

- Keyboard navigation must be quick-access
- Independent of Pane content
- Works the same for Editor, Chat, Terminal

---

## 12) Collaborative Implications

### Policy for Collaboration

```
User1: opens file X
  → broadcast OpenItem(file_id, user_id)

User2: receives, does open_item on their Workspace
  → tabs synchronized

Pitfall: User1 splits while User2 closes simultaneously
  → possible inconsistencies in PaneGroup tree
  → needs versioning / conflict resolution
```

Typical approach:
- Each user maintains their own layout (not synchronized)
- Only "which files are open" is synced (not "which layout")

---

## 13) Architecture Decision: Workspace vs Project

```
Decision:
  Workspace manages UI layer (Panes, Panels, tabs)
  Project manages data layer (files, LSP, buffers)

Consequence:
  Workspace does NOT know about Project content (shouldn't filter by type)
  Project does NOT know about UI layout (shouldn't choose which Pane)
```

This is critical: Workspace = pure UI, Project = pure data.

---

## 14) Anti-Patterns

### Workspace Knowing About LSP

```rust
// BAD
impl Workspace {
    fn close_pane_if_no_diagnostics(&mut self, cx: &mut Context<Self>) {
        let diagnostics = self.project.get_diagnostics(); // LSP data!
        if diagnostics.is_empty() { /* close */ }
    }
}

// GOOD
impl Workspace {
    fn close_pane(&mut self, cx: &mut Context<Self>) {
        // Pure UI logic, no domain knowledge
    }
    // Project notifies Workspace via events if diagnostics change
}
```

### Race Between Split and Close

```rust
// DANGEROUS
User clicks "split" while "close" async is in flight:
  ├─ Split fires: PaneGroup.split(pane)
  ├─ Close completes: PaneGroup.close(pane) ← but pane was just split!
  └─ Invariant violated

// CORRECT
// Synchronize split/close through a single transaction or mutex
```

### Forgetting to Persist

```rust
// BUG: layout lost on restart
fn close_pane(&mut self, cx: &mut Context<Self>) {
    self.pane_group.remove(pane);
    cx.notify();  // ← render updated
    // FORGOT: schedule serialization
}

// CORRECT
fn close_pane(&mut self, cx: &mut Context<Self>) {
    self.pane_group.remove(pane);
    // WorkspaceDb is a global singleton, not a field:
    let db = WorkspaceDb::global(cx);
    db.save_workspace(self, cx);  // ← persist layout
    cx.notify();
}
```

---

## 15) Mental Model

```
Workspace (Entity<Workspace>)
│
├─── Active Path
│    └─ Points to one Leaf (Pane) in Center
│       (focus goes here on keyboard event)
│
├─── Center: PaneGroup (tree of Pane splits)
│    │  PaneGroup { root: Member, is_center: bool }
│    ├─ Member::Axis(PaneAxis { axis, flexes, members })
│    │  └─ Can recursively nest
│    └─ Member::Pane(Entity<Pane>)
│       └─ Contains tab list
│
├─── Dock (left, right, bottom)
│    └─ Show/hide toggle
│    └─ Resizable
│
├─── WorkspaceDb Persistence
│    └─ Saves tree structure, tab list, active pane
│
└─── Invariants
     └─ Center never empty
     └─ active_pane always exists
     └─ PaneGroup tree always valid
```

**Core insight:** Workspace = UI tree of splits + panels, completely separated from data logic.
