# 1. GPUI: The UI Framework

The first architectural atom — Zed's custom GPU-accelerated UI framework. This covers **entity management, rendering pipeline, event dispatch, and platform abstraction**.

---

## 1) What is GPUI and Why Custom?

GPUI is Zed's homegrown UI framework, purpose-built for a code editor. Instead of using Electron, Qt, or native toolkit bindings, Zed renders everything through the GPU with its own layout engine (Taffy/flexbox) and scene graph.

**Why not use existing frameworks?**

- **Performance**: Zero-copy rendering, no DOM diffing, direct GPU scene submission
- **Control**: Full ownership of event dispatch, focus model, and rendering pipeline
- **Consistency**: Same pixel-perfect output on macOS, Linux, Windows
- **Integration**: Tight coupling with CRDT text buffers and async Rust

**Core crate**: `crates/gpui/`

---

## 2) Entity System: State Management

### The Entity Model

GPUI manages all application state through **Entities** — strong, typed references to heap-allocated structs inside an entity map.

```
EntityMap (SlotMap)
├── EntityId(0) → Box<Workspace>
├── EntityId(1) → Box<Pane>
├── EntityId(2) → Box<Editor>
├── EntityId(3) → Box<Buffer>
└── ...
```

### Core Types

```rust
// Strong reference (ref-counted)
pub struct Entity<T> {
    any_entity: AnyEntity,       // ID + ref count handle
    entity_type: PhantomData<T>, // Compile-time type tag
}

// Weak reference (non-retaining)
pub struct WeakEntity<T> {
    any_entity: AnyWeakEntity,
    entity_type: PhantomData<T>,
}

// Type-erased reference (for heterogeneous collections)
pub struct AnyEntity {
    entity_id: EntityId,
    entity_type: TypeId,
    entity_map: Weak<RwLock<EntityRefCounts>>,
}
```

### Read/Update Pattern

All entity access goes through the context — you never hold a direct `&mut T`:

```rust
// Read (immutable borrow from the map)
let value = entity.read(cx);

// Update (exclusive mutable borrow, with context)
entity.update(cx, |state: &mut MyStruct, cx: &mut Context<MyStruct>| {
    state.count += 1;
    cx.notify(); // schedule re-render
});
```

**Why this pattern?**

- Prevents aliased mutable references (Rust borrow checker)
- The `Lease` mechanism temporarily moves the entity out of the map during `update`, preventing re-entrant borrows
- Context provides access to the rest of the app while holding the mutable borrow

### EntityMap Internals

```rust
pub(crate) struct EntityMap {
    entities: SecondaryMap<EntityId, Box<dyn Any>>,  // The actual storage
    ref_counts: Arc<RwLock<EntityRefCounts>>,         // Ref counting
}

// Key operations:
impl EntityMap {
    fn reserve<T>() -> Slot<T>;           // Reserve ID before construction
    fn insert<T>(slot, entity) -> Entity<T>; // Fill the slot
    fn lease<T>(entity) -> Lease<T>;      // Move out for mutation
    fn end_lease<T>(lease);               // Move back in
    fn read<T>(entity) -> &T;             // Immutable access
}
```

**The Lease pattern** is critical: during `update()`, the entity is temporarily removed from the map (leased out), mutated, then put back. This guarantees no other code can access it during mutation.

---

## 3) Context Hierarchy

GPUI has a layered context system. Each layer adds capabilities:

```
App (root context, owns everything)
 └── Context<T> (entity-scoped, adds notify/observe/subscribe)
      └── Window (adds rendering, focus, input dispatch)
```

### App — The Root

```rust
pub struct App {
    platform: Rc<dyn Platform>,          // OS abstraction
    entities: EntityMap,                  // All entities live here
    windows: SlotMap<WindowId, Window>,   // All windows
    keymap: Rc<RefCell<Keymap>>,          // Global keybindings
    actions: Rc<ActionRegistry>,         // Action type registry
    background_executor: BackgroundExecutor,
    foreground_executor: ForegroundExecutor,
    // ... observers, globals, pending effects
}
```

`App` provides:
- `new()` — create entities
- `update_entity()` — mutate entities
- `read_entity()` — read entities
- `update_window()` — dispatch into a window context
- `spawn()` — async tasks on background executor
- Global state (set_global, global, try_global)

### Context\<T\> — Entity-Scoped

When inside an `entity.update()`, you get `Context<T>`:

```rust
pub struct Context<'a, T> {
    app: &'a mut App,            // Full app access (via Deref)
    entity_state: WeakEntity<T>, // "self" reference
}
```

It adds:
- **`notify()`** — mark this entity as changed, schedule re-renders for observers
- **`observe(other)`** — call a closure when another entity calls `notify()`
- **`subscribe(other)`** — call a closure when another entity emits an event
- **`on_release()`** — cleanup when this entity is dropped
- **`spawn()`** — async task with weak self reference
- **`listener()`** — create event handler closure that captures weak self

### AppContext Trait

```rust
pub trait AppContext {
    fn new<T>(&mut self, build: impl FnOnce(&mut Context<T>) -> T) -> Entity<T>;
    fn update_entity<T, R>(&mut self, handle: &Entity<T>, f: impl FnOnce(&mut T, &mut Context<T>) -> R) -> R;
    fn read_entity<T, R>(&self, handle: &Entity<T>, f: impl FnOnce(&T, &App) -> R) -> R;
}
```

### VisualContext Trait

For contexts that also have a window:

```rust
pub trait VisualContext: AppContext {
    fn window_handle(&self) -> AnyWindowHandle;
    fn new_window_entity<T>(&mut self, build: impl FnOnce(&mut Window, &mut Context<T>) -> T) -> Entity<T>;
    fn replace_root_view<V: Render>(&mut self, build: impl FnOnce(&mut Window, &mut Context<V>) -> V) -> Entity<V>;
}
```

---

## 4) Rendering: Element & Render Traits

### The Render Trait

Any entity that can be displayed implements `Render`:

```rust
pub trait Render: 'static + Sized {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement;
}
```

Every frame, GPUI calls `render()` on visible views, which returns an element tree. **This is retained-mode with immediate-mode feel** — you rebuild the element tree each frame, but GPUI diffs and caches underneath.

### The Element Trait

Low-level building block for rendering:

```rust
pub trait Element: 'static + IntoElement {
    type RequestLayoutState: 'static;
    type PrepaintState: 'static;

    fn id(&self) -> Option<ElementId>;

    // Phase 1: Request layout from Taffy
    fn request_layout(&mut self, id, inspector_id, window, cx)
        -> (LayoutId, Self::RequestLayoutState);

    // Phase 2: Commit bounds for hit testing
    fn prepaint(&mut self, id, inspector_id, bounds, request_layout_state, window, cx)
        -> Self::PrepaintState;

    // Phase 3: Emit GPU primitives
    fn paint(&mut self, id, inspector_id, bounds, request_layout_state, prepaint_state, window, cx);
}
```

### RenderOnce — Stateless Components

```rust
pub trait RenderOnce: 'static {
    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement;
}
```

Consumed on render (no retained state). Used for helper components, buttons, icons.

### Three-Phase Rendering Pipeline

```
Frame start
    │
    ▼
┌─ Phase 1: Request Layout ──────────────────────┐
│  Walk element tree, call request_layout()        │
│  Each element registers with Taffy layout engine │
│  Returns LayoutId for later bounds resolution    │
└──────────────────────────────────────────────────┘
    │
    ▼
┌─ Phase 2: Prepaint ────────────────────────────┐
│  Taffy resolves all layouts → concrete bounds    │
│  Call prepaint() with resolved Bounds<Pixels>    │
│  Elements register hitboxes for mouse events     │
└──────────────────────────────────────────────────┘
    │
    ▼
┌─ Phase 3: Paint ───────────────────────────────┐
│  Call paint() on each element                    │
│  Elements emit Scene primitives:                 │
│  - Quads (rectangles, borders)                   │
│  - Shadows                                       │
│  - Text (sprites from glyph cache)               │
│  - Paths (vector shapes)                         │
│  - Surfaces (embedded views)                     │
└──────────────────────────────────────────────────┘
    │
    ▼
Scene submitted to GPU via PlatformWindow::submit_frame()
```

### AnyView — Type-Erased Views

```rust
pub struct AnyView {
    entity: AnyEntity,
    render: fn(&AnyView, &mut Window, &mut App) -> AnyElement,
    cached_style: Option<Rc<StyleRefinement>>,
}
```

Allows heterogeneous collections of views (e.g., Pane tabs can be Editor, Terminal, etc.)

### Scene Primitives

```rust
pub struct Scene {
    paint_operations: Vec<PaintOperation>,
    shadows: Vec<Shadow>,
    quads: Vec<Quad>,
    paths: Vec<Path<ScaledPixels>>,
    underlines: Vec<Underline>,
    monochrome_sprites: Vec<MonochromeSprite>,   // Icons
    subpixel_sprites: Vec<SubpixelSprite>,       // Text glyphs
    polychrome_sprites: Vec<PolychromeSprite>,   // Color images
    surfaces: Vec<PaintSurface>,                 // Embedded content
}
```

---

## 5) Action System: Input Dispatch

### What is an Action?

Actions are named, typed commands that can be triggered by keybindings, menus, or code.

```rust
pub trait Action: Any + Send {
    fn boxed_clone(&self) -> Box<dyn Action>;
    fn name(&self) -> &'static str;
    fn build(value: serde_json::Value) -> Result<Box<dyn Action>>;
}

// Define actions with the macro:
actions!(editor, [MoveUp, MoveDown, MoveLeft, MoveRight, Newline, Undo, Redo]);

// Actions with parameters:
#[derive(Clone, PartialEq, Action)]
#[action(namespace = "editor")]
struct SelectTo {
    position: usize,
}
```

### Keymap → Action Resolution

```
User presses Ctrl+Z
    │
    ▼
Platform captures keystroke
    │
    ▼
Window::dispatch_keystroke()
    │
    ▼
DispatchTree walks from focused node → root
    │
    ▼
At each node, check KeyContext matches binding:
    keymap.json: { "bindings": { "ctrl-z": "editor::Undo" } }
    context: "Editor && mode == normal"
    │
    ▼
Match found → deserialize Action → dispatch
    │
    ▼
Two-phase dispatch:
  1. Capture (root → focused): listeners can intercept
  2. Bubble (focused → root): handlers process
    │
    ▼
First handler that consumes the action stops propagation
```

### DispatchTree

The dispatch tree mirrors the element tree's focus structure:

```rust
pub(crate) struct DispatchTree {
    nodes: Vec<DispatchNode>,
    focusable_node_ids: FxHashMap<FocusId, DispatchNodeId>,
    view_node_ids: FxHashMap<EntityId, DispatchNodeId>,
    keymap: Rc<RefCell<Keymap>>,
    action_registry: Rc<ActionRegistry>,
}

pub(crate) struct DispatchNode {
    key_listeners: Vec<KeyListener>,
    action_listeners: Vec<DispatchActionListener>,
    context: Option<KeyContext>,   // e.g., "Editor && vim_mode == normal"
    focus_id: Option<FocusId>,
    parent: Option<DispatchNodeId>,
}
```

### Dispatch Phases

```rust
pub enum DispatchPhase {
    Capture, // root → focused (for intercepting)
    Bubble,  // focused → root (for handling)
}
```

---

## 6) Event System: EventEmitter + Subscriptions

### EventEmitter

Entities can emit typed events:

```rust
pub trait EventEmitter<E: Any>: 'static {}

// Declare that Buffer emits Edit events:
impl EventEmitter<EditEvent> for Buffer {}

// Emit inside an update:
entity.update(cx, |this, cx| {
    cx.emit(EditEvent { ranges: vec![...] });
});
```

### Subscribing to Events

```rust
// In Context<T>:
cx.subscribe(&buffer_entity, |self_state, buffer, event: &EditEvent, cx| {
    // React to buffer edits
    self_state.refresh_diagnostics(cx);
});
```

### Observer vs Subscriber

| Pattern | Trigger | Use Case |
|---------|---------|----------|
| `observe(entity)` | entity calls `cx.notify()` | Re-read state, refresh derived data |
| `subscribe(entity)` | entity calls `cx.emit(event)` | React to specific typed events |

### Subscription Lifetime

```rust
pub struct Subscription {
    unsubscribe: Option<Box<dyn FnOnce()>>,
}

// Drop = unsubscribe (RAII)
// .detach() = keep alive until entity drops
let sub = cx.observe(&other, |...|{});
sub.detach(); // won't unsubscribe when sub goes out of scope
```

---

## 7) Focus System

### FocusHandle

```rust
pub struct FocusHandle {
    id: FocusId,
    handles: Arc<FocusMap>,
    tab_index: isize,
    tab_stop: bool,
}
```

### Focus Model

```
Window
├── focus: Option<FocusId>  ← exactly one focused element
│
├── DispatchTree
│   ├── Node (Workspace) [focusable]
│   │   ├── Node (Pane) [focusable]
│   │   │   └── Node (Editor) [focusable] ← focused
│   │   └── Node (Pane) [focusable]

Focus path: [Workspace, Pane, Editor]
```

### Focus Operations

```rust
// Check focus
focus_handle.is_focused(window)         // exact match
focus_handle.contains_focused(window)   // self or descendant

// Move focus
window.focus(focus_handle)

// Focus events
cx.on_focus_in(|self, window, cx| { ... })   // gained focus
cx.on_focus_out(|self, window, cx| { ... })  // lost focus
```

### Focus and Key Dispatch Interaction

When a key is pressed:
1. Find the focused `FocusId` in the window
2. Look up the `DispatchNodeId` for that focus
3. Walk **up** the dispatch tree (Bubble phase) or **down** (Capture phase)
4. At each node, match `KeyContext` against keymap bindings
5. If binding matches, dispatch the Action

---

## 8) Platform Abstraction

### The Platform Trait

```rust
pub trait Platform: 'static {
    fn run(&self, on_finish_launching: Box<dyn FnOnce()>);
    fn quit(&self);
    fn displays(&self) -> Vec<Rc<dyn PlatformDisplay>>;
    fn open_window(&self, handle, options) -> Result<Box<dyn PlatformWindow>>;
    fn set_menus(&self, menus: Vec<Menu>, keymap: &Keymap);
    fn open_url(&self, url: &str);
    fn keyboard_layout(&self) -> Box<dyn PlatformKeyboardLayout>;
    // ... clipboard, file dialogs, thermal state, etc.
}
```

### Platform Implementations

```
crates/gpui/         ← core framework (platform-agnostic)
crates/gpui_macos/   ← macOS: Cocoa/Metal backend
crates/gpui_linux/   ← Linux: Wayland/X11 backend
// Windows uses platform code inside gpui itself
```

### PlatformWindow

```rust
pub trait PlatformWindow: Any {
    fn bounds(&self) -> Bounds<Pixels>;
    fn scale_factor(&self) -> f32;
    fn set_title(&mut self, title: SharedString);
    fn show(&mut self);
    fn close(&mut self);
    fn submit_frame(&mut self, frame: RenderedFrame) -> Result<()>;
    // ... resize, fullscreen, minimize, etc.
}
```

---

## 9) Async Execution Model

### Two Executors

```
BackgroundExecutor          ForegroundExecutor
├── Thread pool             ├── Single main thread
├── CPU-bound work          ├── UI updates
├── File I/O                ├── Entity mutations
├── Tree-sitter parsing     ├── Rendering
└── LSP communication       └── Event dispatch
```

### Spawning Tasks

```rust
// From Context<T>:
cx.spawn(async move |weak_self, cx| {
    let result = expensive_computation().await;
    weak_self.update(cx, |this, cx| {
        this.apply_result(result);
        cx.notify();
    })?;
    Ok(())
});

// From App:
cx.background_executor().spawn(async { ... });
cx.foreground_executor().spawn(async { ... });
```

**Key rule**: Entity mutations must happen on the foreground executor. Background tasks return results, foreground tasks apply them.

---

## 10) Invariants

### Entity Invariants

- **Entity\<T\> is always valid** while held — the entity exists in the map
- **WeakEntity\<T\>.upgrade()** returns `None` if entity was dropped
- **During update()**, no other code can access the entity (lease pattern)
- **notify()** is idempotent per frame — multiple calls schedule one re-render

### Rendering Invariants

- **render() is pure** — side effects go in update(), not render()
- **Element lifecycle is one frame** — elements are recreated each render
- **ElementId enables cross-frame state** — same ID = same element state slot
- **Layout is synchronous** — no async in the render path

### Focus Invariants

- **At most one focused element per window**
- **Focus path is always valid** — if focused element is removed, focus moves up
- **Key dispatch respects focus path** — events bubble from focused to root

---

## 11) Anti-Patterns

### Holding Entity References Across Await

```rust
// BAD: entity could be dropped during await
let data = entity.read(cx).clone_data();
let result = some_async().await;
entity.update(cx, |e, cx| e.apply(result)); // entity may be gone!

// GOOD: use WeakEntity
let weak = entity.downgrade();
let result = some_async().await;
weak.update(cx, |e, cx| e.apply(result))?; // returns Err if dropped
```

### Nested Updates

```rust
// BAD: will panic (double lease)
entity.update(cx, |this, cx| {
    entity.update(cx, |this, cx| { // PANIC: already leased
        ...
    });
});

// GOOD: use cx (which Derefs to App) for other entities
entity_a.update(cx, |a, cx| {
    entity_b.update(cx, |b, cx| { // OK: different entity
        ...
    });
});
```

### Side Effects in render()

```rust
// BAD: render should be pure
fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    self.count += 1; // Mutation during render!
    div().child(format!("{}", self.count))
}

// GOOD: mutations in event handlers or observers
fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    div()
        .child(format!("{}", self.count))
        .on_click(cx.listener(|this, _event, _window, cx| {
            this.count += 1;
            cx.notify();
        }))
}
```

---

## 12) Mental Model

```
GPUI Architecture
│
├─── Entity System (State)
│    ├── EntityMap: SlotMap<EntityId, Box<dyn Any>>
│    ├── Entity<T>: strong typed handle
│    ├── WeakEntity<T>: weak typed handle
│    └── Lease pattern: exclusive mutation
│
├─── Context Hierarchy (Access)
│    ├── App: root, owns everything
│    ├── Context<T>: notify, observe, subscribe
│    └── Window: rendering, focus, input
│
├─── Rendering (Output)
│    ├── Render trait → element tree
│    ├── Element trait → 3-phase pipeline
│    ├── Taffy → flexbox layout
│    └── Scene → GPU primitives
│
├─── Actions & Input (Events)
│    ├── Action trait → typed commands
│    ├── Keymap → keystroke → Action resolution
│    ├── DispatchTree → focus-aware routing
│    └── Capture/Bubble phases
│
├─── Subscriptions (Reactivity)
│    ├── observe() → on notify
│    ├── subscribe() → on emit
│    └── RAII Subscription → auto cleanup
│
├─── Focus (Navigation)
│    ├── FocusHandle + FocusId
│    ├── One focused element per window
│    └── Focus path = dispatch path
│
└─── Platform (OS Integration)
     ├── Platform trait → OS abstraction
     ├── PlatformWindow → native window
     └── macOS / Linux / Windows backends
```

**Core insight**: GPUI is an ECS-like framework where entities hold state, contexts provide scoped access, elements are transient render output, and actions/subscriptions form the event backbone.
