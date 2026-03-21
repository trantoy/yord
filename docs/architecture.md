# Yord — Architecture: Hybrid ECS + Tree

---

## The Problem with a Pure Tree

Intuitively, a workspace is a tree. A project contains tools, tools contain sub-tools. It's easy to draw, easy to explain, easy to render with a recursive component.

But a tree is a constraint. Each node lives in exactly one place. Hierarchy is rigidly embedded in the data structure. This creates real problems as the product grows.

---

## The Problem with Pure ECS

ECS (Entity-Component-System) is an architecture from game dev. No hierarchy, no objects with methods. Just:

- **Entity** — simply a unique ID
- **Component** — pure data without logic
- **System** — logic that queries entities with the required set of components and processes them

ECS solves all tree problems but creates new ones: UI rendering is non-obvious, node ordering is non-trivial, the learning curve is steeper. For a dev tool where the main UI is a tree, pure ECS is overkill.

---

## Hybrid: Flat Storage, Tree UI

The idea is simple: **store data as ECS, display as a tree.**

```
┌─────────────────────────────────────┐
│           UI Layer (Svelte)         │
│         renders the tree            │
│     builds it from flat data        │
└──────────────┬──────────────────────┘
               │ buildTree()
┌──────────────▼──────────────────────┐
│         Storage Layer (Rust)        │
│    flat entities + components       │
│    queryable, extensible            │
└─────────────────────────────────────┘
```

The tree is a **view** over flat data, not the data itself.

---

## What It Looks Like

### Storage (flat ECS)

```rust
// Each node is an entity with a set of components
// M1: single parent_id. Future: parent_ids[] for shared nodes.
Entity {
  id: "term-1",
  components: {
    Label:      { text: "terminal" },
    BelongsTo:  { parent_id: "project-saas" },     // M1: scalar
    Order:      { index: 0 },
    Pty:        { cmd: ["zsh"], cwd: "~/saas" },
    Running:    { pid: 4821 },
  }
}

Entity {
  id: "agent-1",
  components: {
    Label:      { text: "architect agent" },
    BelongsTo:  { parent_id: "project-saas" },     // M1: single parent
    // Future: { parent_ids: ["project-saas", "project-sdk"] } for shared nodes
    Agent:      { model: "claude-opus", context_node_ids: ["term-1"] },
    Running:    { pid: 4900 },
  }
}
```

### UI (builds tree from flat data)

```typescript
function buildTree(entities: Entity[]): TreeNode[] {
  // 1. find root nodes (BelongsTo is empty or absent)
  // 2. group by parent_id
  // 3. sort by Order.index
  // 4. return nested structure for rendering
}
```

This function is called reactively on any change to entities. The UI is always up to date, the tree is always consistent.

---

## Why This Is Better Than a Pure Tree

### 1. Shared nodes

One agent, two projects. No need for "links" or duplication.

```
// Pure tree — can't do this:
project-saas
  └── agent ← node can only be here

project-sdk
  └── agent ← had to duplicate

// Hybrid — native (future, with parent_ids[]):
entity:agent  [BelongsTo: [project-saas, project-sdk]]
```

In the UI, the agent appears under both projects with a visual "shared" badge.

### 2. Extensibility without schema changes

Want to add "pinned" nodes — ones that don't get killed on project kill.

```rust
// Pure tree: add a field to the base Node type
struct Node {
  ...
  pinned: bool,  // ← change the model, DB migration, update all callsites
}

// Hybrid: add a component
Entity {
  components: {
    ...
    Pinned: {}  // ← just a new component, nothing breaks
  }
}
```

KillSystem: `if entity.has(Pinned) { skip }`. Nothing else changes.

### 3. Powerful queries

```rust
// All running terminals across all projects
query![Pty, Running]

// All nodes of project X that need to be restored
query![BelongsTo(project_x), RestoreOnStart]

// All agents that have access to a specific terminal
query![Agent, ContextOf(terminal_id)]
```

In a pure tree — traversal with filtering. In ECS — indexed queries, O(1).

### 4. Layout as components

Layout doesn't need a separate data structure. It's just components on the same entities:

```rust
Entity {
  id: "term-1",
  components: {
    Pty: { ... },
    LayoutLeaf: { pane_id: "pane-3" },       // which pane it's displayed in
    LayoutVisible: { workspace_id: "ws-1" },  // which workspace it's visible in
  }
}
```

Single source of truth. No synchronization between process tree and layout tree.

### 5. New node types without refactoring

Adding Docker support:

```rust
// Pure tree: add NodeType::Docker, update switch/match everywhere

// Hybrid: create a new component
Component::DockerCompose { compose_path: String, auto_up: bool }

// DockerSystem processes entities that have this component
// Everything else (BelongsTo, Running, Label) — already works
```

---

## Why This Is Better Than Pure ECS

### 1. The UI stays simple

The tree renders recursively. The user sees a familiar hierarchy. No need to explain ECS to the user — it's an implementation detail.

### 2. Node ordering is obvious

`Order.index` — explicit, persistent, drag-to-reorder is trivial. In pure ECS, ordering is a non-trivial problem.

### 3. Clear mental model for developers

"It's a tree, but stored flat" — easy to explain to a new contributor. "It's ECS where layout is also a system" — takes time to understand.

### 4. Migration from v1

You can start with a pure tree in v1. The hybrid is the same model but with the "single parent" constraint removed. This is not a breaking change in the storage layer if done right from the start.

---

## Storage Layer

SQLite. Two tables instead of one. Requires `PRAGMA foreign_keys = ON` at connection time for cascading deletes.

```sql
PRAGMA foreign_keys = ON;

-- Entities
CREATE TABLE entities (
  id         TEXT PRIMARY KEY,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

-- Components (EAV pattern)
CREATE TABLE components (
  entity_id      TEXT NOT NULL REFERENCES entities(id) ON DELETE CASCADE,
  component_type TEXT NOT NULL,
  data           TEXT NOT NULL,  -- JSON
  PRIMARY KEY (entity_id, component_type)
);

-- Indexes for fast queries
CREATE INDEX idx_component_type ON components(component_type);
CREATE INDEX idx_belongs_to ON components(component_type, json_extract(data, '$.parent_id'))
  WHERE component_type = 'BelongsTo';
```

Queries in Rust:

```rust
// All children of a project (M1: single parent_id)
fn query_children(parent_id: &str) -> Vec<Entity> {
  db.query(
    "SELECT entity_id FROM components
     WHERE component_type = 'BelongsTo'
     AND json_extract(data, '$.parent_id') = ?",
    [parent_id]
  )
}

// All children of a project (future: parent_ids array)
fn query_children(parent_id: &str) -> Vec<Entity> {
  db.query(
    "SELECT c.entity_id FROM components c, json_each(c.data, '$.parent_ids') j
     WHERE c.component_type = 'BelongsTo' AND j.value = ?",
    [parent_id]
  )
}

// All running processes
fn query_running() -> Vec<Entity> {
  db.query(
    "SELECT entity_id FROM components WHERE component_type = 'Running'"
  )
}
```

---

## Components (full v1 list)

| Component | Data | Purpose |
|-----------|------|---------|
| `Label` | `{ text, icon, color }` | Display in UI |
| `BelongsTo` | `{ parent_id }` (M1) / `{ parent_ids: [] }` (future) | Hierarchy |
| `Order` | `{ index }` | Order among siblings |
| `Project` | `{ cwd }` | Root node |
| `Pty` | `{ cmd, cwd, env }` | Terminal/TUI process |
| `Webview` | `{ url }` | Browser node |
| `Editor` | `{ kind, port, cwd }` | code-server etc |
| `Agent` | `{ model, system_prompt }` | LLM agent |
| `ContextOf` | `{ node_ids: [] }` | Agent can see these nodes |
| `Docker` | `{ compose_path, auto_up }` | Docker compose |
| `Running` | `{ pid, pgid, started_at }` | Live process |
| `Crashed` | `{ exit_code, stderr_tail }` | Crashed process |
| `RestoreOnStart` | `{}` | Restore on app launch |
| `Pinned` | `{}` | Don't kill on project kill |
| `Layout` | `{ tree: LayoutNode }` | Full layout tree (on project entities) |
| `LayoutLeaf` | `{ pane_id }` | Position in layout |
| `LayoutVisible` | `{ workspace_id }` | Which workspace entity is visible in |

---

## Systems (Rust)

```
LifecycleSystem    — query[Pty | Webview | Editor | Agent] → spawn/kill/restart
RestoreSystem      — query[RestoreOnStart] → respawn on app launch
KillProjectSystem  — query[BelongsTo(X)] → recursive kill, respect Pinned
PortSystem         — query[Editor | Docker] → allocate/release ports
LayoutSystem       — query[LayoutLeaf] → build layout tree for renderer
PersistSystem      — debounced write of changes to SQLite
```

---

## Starting Simple (M1 → multi-parent later)

M1 uses single-parent `BelongsTo: { parent_id }`. The ECS schema is in place from day one — the only simplification is one parent instead of many. Upgrading to multi-parent later requires:

1. Change `BelongsTo` data from `{ parent_id }` to `{ parent_ids: [] }`
2. Update `buildTree()` — handle multiple parents
3. Update UI — display shared nodes with a visual badge

The storage schema does not change. This is a deliberate decision — start with single-parent, but the ECS storage layer doesn't bake that constraint in.

---

## Summary

The hybrid is not a compromise. It's the right model for this problem:

- **User sees a tree** — familiar, intuitive
- **Data is stored flat** — flexible, queryable, extensible
- **New features = new components** — no refactoring of existing code
- **Shared nodes** — native, not a hack
- **Layout is not a separate structure** — just components

This is what Bevy (game engine) does, this is what modern Unity does. Hierarchy as a view, not as a data structure.

---

*Document version 0.1*
*Related to: design.md*
