# Yord — Design Document
> Project-scoped workspace manager. Every tool is a child of a project, not the other way around.

---

## 1. Problem

Modern development workflows span multiple tools: editor, terminal, browser, file manager, agent. These tools are unaware of each other. The developer is the glue — mentally tracking which terminal belongs to which project, which browser tab is for which service, which agent is working on what.

The OS and existing IDEs organize around tools. You open VSCode, then inside VSCode you open a project. You open a terminal, then `cd` into a project. The project is implicit. It lives only in the developer's head.

**Yord inverts this.** The project is the root. Tools are children. When you kill a project, you kill its children. When you restore a session, everything comes back exactly as you left it.

---

## 2. Goals

- Project is the organizing primitive, not the tool
- Every running process belongs to a node in the tree
- Layout (how things are arranged on screen) is separate from process state (what is running)
- Everything is restorable across sessions
- The user chooses their tools — Yord doesn't force an editor or terminal
- No Electron, no Chromium bundle — native webview, small binary

---

## 3. Non-Goals

- Not an IDE — no language server, no debugger built-in (delegate to tools)
- Not a window manager — does not replace i3, sway, or the OS WM
- Not a browser — the webview nodes are embedded contexts, not a full browser
- Not a cloud product — local-first, no sync in v1

---

## 4. Core Concepts

> **Terminology**: In Rust/storage code, the data unit is called an **entity**. In the UI/Svelte layer, entities are called **nodes** and displayed as a tree. `buildTree()` converts entities to tree nodes for rendering.

### 4.1 Entity + Components

Everything in Yord is an **entity** — a unique ID with a set of components attached. Components are pure data. An entity's behavior is determined by which components it has: an entity with a `Pty` component is a terminal, one with an `Agent` component is an agent.

In the UI, entities are called **nodes** and displayed as a tree. The tree is a view over flat data, not the data itself. See `docs/architecture.md` for the full rationale.

```
Entity {
  id:         string (uuid)
  created_at: integer
  updated_at: integer
}

// Identity comes from components, not a type enum:
// - Project:       { cwd }                        → root node
// - Pty:           { cmd, cwd, env }               → terminal / TUI / dev server
// - Webview:       { url }                         → browser
// - Editor:        { kind, port, cwd }             → code-server etc
// - Agent:         { model, system_prompt }         → LLM agent
// - Docker:        { compose_path, auto_up }        → docker compose
//
// Structural:
// - Label:         { text, icon, color }
// - BelongsTo:     { parent_id }                   → single parent (M1), parent_ids[] later
// - Order:         { index }                       → sibling order
//
// Lifecycle:
// - Running:       { pid, started_at }
// - Crashed:       { exit_code, stderr_tail }
// - RestoreOnStart: {}
// - Pinned:        {}                              → don't kill on project kill
//
// Layout (M2):
// - LayoutLeaf:    { pane_id }
// - LayoutVisible: { workspace_id }
```

**Lifecycle states** are derived from components, not stored as a field:
- No `Running` or `Crashed` component → **idle**
- `Running` component present → **running**
- `Crashed` component present → **crashed**
- (An entity that had `Running` removed after a clean exit → **stopped**, inferred from absence)

### 4.2 Process Tree vs Layout Tree

Two separate concerns. They reference each other but are not coupled.

**Process tree** — derived from `BelongsTo` + `Order` components. Source of truth for what exists and what is running. Built by `buildTree()` in Svelte from flat entity data.

**Layout tree** — describes how the screen is divided. A recursive structure stored as a `Layout` component on the project entity:

```typescript
type LayoutNode =
  | { type: "split";  direction: "h" | "v"; ratio: number; children: [LayoutNode, LayoutNode] }
  | { type: "tabbed"; activeIndex: number;  children: LayoutNode[] }
  | { type: "leaf";   entityId: string }     // references an entity
  | { type: "empty" }                        // placeholder pane
```

This is stored as a JSON blob in a `Layout` component on the project entity — not in a separate table. Each entity also has a `LayoutLeaf` component pointing to which pane it's displayed in, enabling fast queries in both directions.

An entity can exist without appearing in the layout (running in background). A layout leaf can point to a stopped entity (shows a restart prompt).

### 4.3 Workspace

A workspace is a project entity viewed with a specific layout. The `Layout` component on the project entity stores the current arrangement. Multiple saved layouts per project may be supported later.

Workspaces can be switched instantly. Switching does not kill processes — they keep running in the background unless explicitly stopped.

---

## 5. Data Model (Full)

### 5.1 Component Data

Each component type has a specific data shape stored as JSON. See `docs/architecture.md` for the full components table.

Key component data shapes for M1:

```typescript
// Structural
Label:         { text: string, icon?: string, color?: string }
BelongsTo:     { parent_id: string }           // M1: single parent
Order:         { index: number }

// Identity (determines what kind of node this is)
Project:       { cwd: string }
Pty:           { cmd: string[], cwd: string, env?: Record<string, string> }
Webview:       { url: string }
Editor:        { kind: string, port: number, cwd: string }
Agent:         { model: string, system_prompt?: string }
ContextOf:     { node_ids: string[] }           // agent sees these nodes
Docker:        { compose_path: string, auto_up: boolean }

// Lifecycle
Running:       { pid: number, started_at: number }
Crashed:       { exit_code: number, stderr_tail: string }
RestoreOnStart: {}
Pinned:        {}

// Layout (M2)
Layout:        { tree: LayoutNode }             // on project entities, the full layout tree
LayoutLeaf:    { pane_id: string }              // on leaf entities, which pane they're in
LayoutVisible: { workspace_id: string }
```

A dev server is modeled as `Pty` + `Label` (no dedicated component needed). A TUI file manager (yazi) is `Pty` + `Label`. A web file manager (filebrowser) is `Webview` + `Label`. The component combination determines behavior.

### 5.2 Persistence Format

Stored as SQLite. Two tables (EAV pattern). Requires `PRAGMA foreign_keys = ON` at connection time for `ON DELETE CASCADE` to work.

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE entities (
  id         TEXT PRIMARY KEY,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

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

Example queries:

```sql
-- All children of a project (M1: parent_id is a scalar)
SELECT entity_id FROM components
WHERE component_type = 'BelongsTo'
  AND json_extract(data, '$.parent_id') = ?;

-- All running entities
SELECT entity_id FROM components WHERE component_type = 'Running';
```

Runtime state (pid, port) lives in `Running` components, not a separate sessions table. Layout lives in `Layout` components on project entities, not a separate layouts table.

---

## 6. Lifecycle

### 6.1 Node Lifecycle

```
[defined] → start() → [starting] → process spawned → [running]
                                                     → [crashed]  (non-zero exit)
[running] → stop()  → SIGTERM    → [stopped]
[running] → kill()  → SIGKILL    → [stopped]
[crashed / stopped] → restart()  → [starting] → ...
```

**Parent-child kill propagation**: When a project entity is stopped, Yord sends stop signals depth-first to all descendants before stopping the project itself. Kill order is determined by component type:
1. Entities with `Agent` component (graceful shutdown, flush context)
2. Entities with `Pty` component (terminals, dev servers, TUI file managers)
3. Entities with `Editor` component
4. Entities with `Docker` component (compose down)
5. Entities with `Webview` component (browsers, web file managers — just close)

Entities with the `Pinned` component are skipped.

### 6.2 Session Restore

On application start:
1. Load all entities and components from SQLite
2. For each entity with `RestoreOnStart` component that also has a `Running` component (from previous session): spawn process
3. Assign new PIDs, update `Running` components
4. Reconstruct layout from `Layout` components on project entities
5. Open the last active workspace

Entities that crashed in the previous session (have `Crashed` component) are shown as crashed — user must manually restart.

**M1 restore scope**: Respawns the configured command (`Pty.cmd`) in the configured directory (`Pty.cwd`). Terminal scrollback is **not** preserved. Shell state (env vars, history, background jobs) does not survive PTY restart. "Restore" means "respawn the same process," not "resume the exact terminal state."

### 6.3 Port Management

Web-based entities (editor, filebrowser) need a port. Yord manages a port pool:
- Range: 49152–65535 (ephemeral range)
- Assignment: at spawn time, find next available port
- Persist: port stored in the entity's `Editor` or `Webview` component, reused on restore if still available

---

## 7. Architecture

### 7.1 Technology Stack

| Layer | Technology | Reason |
|-------|-----------|--------|
| UI framework | Svelte 5 | Minimal overhead, fine-grained reactivity, no vdom |
| Desktop shell | Tauri 2 | Rust core, native webview, no Chromium bundle |
| Backend | Rust | Process management, PTY, file system, SQLite |
| Terminal renderer | xterm.js | Industry standard, handles all TUI apps |
| PTY bridge | portable-pty (Rust crate) | Cross-platform PTY, clean Tauri command API |
| Storage | SQLite via tauri-plugin-sql | Structured, queryable, no external dep |
| IPC | Tauri commands + events | Typed, bidirectional |

### 7.2 Process Layers

```
┌─────────────────────────────────────────────────────────┐
│                    Svelte UI (WebView)                   │
│   Tree panel │ Layout engine │ Toolbar │ Status bar      │
│              │               │         │                 │
│         xterm.js instances (per terminal/TUI entity)     │
│         webview iframes (per browser/editor entity)      │
└────────────────────────┬────────────────────────────────┘
                         │ Tauri IPC (commands + events)
┌────────────────────────▼────────────────────────────────┐
│                     Rust Core (Systems)                   │
│   LifecycleSystem │  LayoutSystem  │  PortSystem         │
│   PtyBridge       │  EntityStore   │  EventRouter        │
│   RestoreSystem   │  PersistSystem │                     │
└─────────────────────────────────────────────────────────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
           SQLite    OS Processes  PTY
```

### 7.3 Tauri Commands (Rust → TS interface)

```rust
// Entity management
#[tauri::command] fn create_entity(components: Vec<ComponentData>) -> Result<EntityId>
#[tauri::command] fn delete_entity(id: String) -> Result<()>
#[tauri::command] fn get_entities() -> Result<Vec<EntityWithComponents>>

// Component management
#[tauri::command] fn set_component(entity_id: String, component: ComponentData) -> Result<()>
#[tauri::command] fn remove_component(entity_id: String, component_type: String) -> Result<()>

// Lifecycle
#[tauri::command] fn start_entity(id: String) -> Result<()>
#[tauri::command] fn stop_entity(id: String) -> Result<()>
#[tauri::command] fn restart_entity(id: String) -> Result<()>
#[tauri::command] fn kill_project(id: String) -> Result<()>  // recursive stop

// PTY (entities with Pty component)
#[tauri::command] fn pty_write(entity_id: String, data: Vec<u8>) -> Result<()>
#[tauri::command] fn pty_resize(entity_id: String, cols: u16, rows: u16) -> Result<()>

// Port
#[tauri::command] fn allocate_port() -> Result<u16>
#[tauri::command] fn release_port(port: u16) -> Result<()>
```

### 7.4 Tauri Events (Rust → UI push)

```rust
// Emitted to frontend
"entity:component-changed"  { entity_id, component_type, data }
"entity:removed"            { entity_id }
"entity:pty-data"           { entity_id, data: Vec<u8> }
"entity:port-ready"         { entity_id, port }
```

Component changes cover all state transitions — `Running` added means started, `Running` removed means stopped, `Crashed` added means crashed. No separate state-change event needed.

---

## 8. UI

### 8.1 Layout

```
┌──────────┬────────────────────────────────────────────┐
│          │                                            │
│   Tree   │              Content Area                  │
│  Panel   │         (tiling layout engine)             │
│          │                                            │
│ project  │  ┌─────────────────┬──────────────────┐   │
│  ├ term  │  │                 │                  │   │
│  ├ edit  │  │    terminal     │    browser       │   │
│  └ brows │  │                 │                  │   │
│          │  ├─────────────────┴──────────────────┤   │
│ project  │  │                                    │   │
│  └ term  │  │           editor                   │   │
│          │  │                                    │   │
└──────────┴──┴────────────────────────────────────┴───┘
```

### 8.2 Tree Panel

- Collapsible tree built by `buildTree()` from flat entity data
- Node icons by component type (emoji or icon set)
- Color dot = lifecycle state (green=Running, red=Crashed, grey=idle)
- Right-click context menu: start / stop / restart / rename / delete / add child
- Drag to reorder siblings (updates `Order` component)
- Drag onto a project to reparent (updates `BelongsTo` component)

### 8.3 Content Area (Tiling Engine)

The content area renders a `LayoutNode` tree recursively:

- `split` → CSS flexbox with a draggable divider
- `tabbed` → tab bar + single visible child
- `leaf` → renders the appropriate view for that node type:
  - `terminal/tui` → xterm.js instance
  - `browser/editor/filemanager(web)` → `<webview>` (Tauri native webview tag)
  - `project` → dashboard (children as clickable cards)
  - `agent` → chat-like thread view

Empty panes show a "+" button to attach a node.

### 8.4 Node Views

**Terminal node**
```
┌─────────────────────────────────────────────────────┐
│ ● terminal  [zsh]  ~/projects/saas    ✕ kill  ↺ restart │
├─────────────────────────────────────────────────────┤
│                                                     │
│  $ npm run dev                                      │
│  > ready on http://localhost:3000                   │
│  _                                                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Browser node**
- URL bar at top (editable)
- `<webview>` fills the rest
- Back / forward / reload buttons

**Editor node**
- Thin header with editor name and status
- `<webview>` pointed at localhost:PORT
- Port shown, click to copy

**Agent node**
- Shows model, status
- Conversation thread
- Context: which sibling nodes are visible to the agent
- Streaming responses rendered as they come

### 8.5 Workspace Switcher

Triggered by `Cmd+K` or a button in the toolbar. Overlay showing:
- All projects as cards (name, color, running child count)
- Search/filter
- Click to switch active workspace
- Switching does not kill processes

### 8.6 Quick Nav (`Cmd+P`)

Fuzzy search over all nodes in the tree. Navigate to any node instantly. Inspired by VSCode `Cmd+P` but tree-aware.

---

## 9. Event Routing

The trickiest part. When the user types, keyboard events need to go to the focused node, not broadcast to all terminals.

### 9.1 Focus Model

- One node is "focused" at a time (tracked in Svelte store)
- xterm.js instances: only the focused instance has `element.focus()` called
- webview instances: Tauri webview focus API
- Clicking a pane sets focus
- `Ctrl+Tab` cycles focus within the active project
- Keyboard shortcuts (e.g. `Cmd+K`) are caught at the app level before propagating to focused node

### 9.2 Global Shortcuts

Intercepted by Tauri's global shortcut plugin before they reach any node:

| Shortcut | Action |
|----------|--------|
| `Cmd+K` | Workspace switcher |
| `Cmd+P` | Quick nav |
| `Cmd+N` | New node (contextual) |
| `Cmd+W` | Close/detach current pane |
| `Cmd+\` | Split pane vertically |
| `Cmd+-` | Split pane horizontally |
| `Cmd+[1-9]` | Focus nth pane |

---

## 10. Agent Nodes

Agents are first-class nodes. They:
- Live in the tree like any other node
- Have a `contextNodeIds` list — which siblings they can observe (read terminal output, file content, browser URL)
- Can spawn sub-agent children (sub-agents are just child nodes of type `agent`)
- Communicate via the Anthropic API (or other providers)

```
project-agent-sdk
  ├── agent (orchestrator)
  │     ├── sub-agent (openai integration)
  │     └── sub-agent (google integration)
  ├── terminal (logs)
  └── browser (api docs)
```

The orchestrator agent's `contextNodeIds` includes the terminal and browser. Sub-agents have context scoped to their task.

Agent nodes persist conversation history in SQLite. On restore, the conversation is resumed (no re-execution, just context available).

---

## 11. Milestones

### M1 — Tree + Shell (current target)
- [ ] Tauri 2 + Svelte 5 scaffold
- [ ] Entity/component data model in Rust (ECS storage layer)
- [ ] SQLite persistence (entities + components tables)
- [ ] Tree panel UI (Svelte, `buildTree()` from flat data)
- [ ] Create/delete/rename project and terminal entities
- [ ] Spawn PTY process per entity with `Pty` component
- [ ] xterm.js renders PTY output
- [ ] Kill propagation (kill project → kills children, respect `Pinned`)
- [ ] Session restore for terminal entities (respawn command, no scrollback)

### M2 — Layout Engine
- [ ] LayoutNode data model (`Layout` component on project entities)
- [ ] Tiling split/tabbed renderer in Svelte
- [ ] Draggable dividers
- [ ] Attach/detach entities to/from panes (`LayoutLeaf` component)
- [ ] Layout persistence via `Layout` component
- [ ] Workspace switcher UI

### M3 — Web Nodes
- [ ] Tauri webview component
- [ ] Browser entity (`Webview` component, URL bar + webview)
- [ ] Port pool in Rust (`PortSystem`)
- [ ] Editor entity (`Editor` component, code-server / openvscode-server)
- [ ] Auto-launch code-server with correct `cwd`
- [ ] File manager entity (yazi via `Pty` or filebrowser via `Webview`)

### M4 — Agent Nodes
- [ ] `Agent` component + `ContextOf` component
- [ ] Anthropic API integration (streaming)
- [ ] Context wiring (read sibling terminal output via `ContextOf`)
- [ ] Sub-agent spawning (child entities with `Agent` component)
- [ ] Conversation history persistence

### M5 — Polish
- [ ] Quick nav `Cmd+P`
- [ ] Drag-to-reparent in tree (update `BelongsTo`)
- [ ] Entity config editor UI
- [ ] Crash reporting (`Crashed` component with stderr tail)
- [ ] Theming / accent colors per project (`Label.color`)
- [ ] Onboarding / empty state

---

## 12. Open Questions

1. **TUI file manager integration depth** — yazi has a great plugin API. Should Yord talk to yazi over its IPC to know which file is focused (to open it in the editor node)? Or is that over-engineering for v1?

2. **Multi-window** — should a project be detachable into its own OS window? Useful for multi-monitor. Deferred but needs to be compatible with the layout model.

3. **Remote projects** — SSH into a server, run terminal/editor nodes there. Node config could have a `remote: SSHConfig` field. Architecture supports it but not in scope yet.

4. **Plugin system** — new node types defined as plugins (WASM? JS?). Needed for long-term extensibility. The ECS model naturally supports this — plugins just register new component types and systems.

5. **Collaborative mode** — two developers sharing a workspace tree over a relay. Very far future but the local-first SQLite model would need to become CRDT-based.

---

## 13. Directory Structure (proposed)

```
yord/
├── src-tauri/
│   ├── src/
│   │   ├── main.rs
│   │   ├── commands/
│   │   │   ├── mod.rs
│   │   │   ├── entities.rs    -- entity/component CRUD
│   │   │   ├── lifecycle.rs   -- start/stop/restart
│   │   │   ├── pty.rs         -- PTY bridge
│   │   │   └── ports.rs       -- port pool
│   │   ├── ecs/
│   │   │   ├── mod.rs
│   │   │   ├── entity.rs      -- Entity struct
│   │   │   ├── component.rs   -- Component types + ComponentData enum
│   │   │   └── store.rs       -- EntityStore (in-memory + SQLite)
│   │   ├── systems/
│   │   │   ├── mod.rs
│   │   │   ├── lifecycle.rs   -- LifecycleSystem
│   │   │   ├── restore.rs     -- RestoreSystem
│   │   │   ├── kill.rs        -- KillProjectSystem
│   │   │   └── persist.rs     -- PersistSystem
│   │   ├── db/
│   │   │   ├── mod.rs
│   │   │   └── migrations/
│   │   └── process/
│   │       ├── pty.rs         -- PTY abstraction
│   │       └── port_pool.rs
│   └── Cargo.toml
├── src/
│   ├── lib/
│   │   ├── stores/
│   │   │   ├── entities.ts    -- entity/component store
│   │   │   ├── tree.ts        -- buildTree() reactive derivation
│   │   │   └── focus.ts       -- focus state
│   │   ├── types/
│   │   │   ├── entity.ts      -- Entity, Component types
│   │   │   └── tree.ts        -- TreeNode (UI view type)
│   │   └── ipc/
│   │       ├── commands.ts    -- typed wrappers around invoke()
│   │       └── events.ts      -- typed event listeners
│   ├── components/
│   │   ├── tree/
│   │   │   ├── TreePanel.svelte
│   │   │   ├── TreeNode.svelte
│   │   │   └── ContextMenu.svelte
│   │   ├── layout/            -- M2
│   │   │   ├── LayoutRenderer.svelte
│   │   │   ├── SplitPane.svelte
│   │   │   ├── TabbedPane.svelte
│   │   │   └── Leaf.svelte
│   │   └── views/
│   │       ├── TerminalView.svelte
│   │       ├── BrowserView.svelte  -- M3
│   │       ├── EditorView.svelte   -- M3
│   │       └── AgentView.svelte    -- M4
│   └── App.svelte
└── package.json
```

---

*Document version: 0.1 — pre-implementation*
*Stack: Tauri 2 + Svelte 5 + Rust + SQLite*
*Target platform: macOS first, cross-platform by M3*
