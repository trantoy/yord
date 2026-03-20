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

### 4.1 Node

Everything in Yord is a node. Nodes form a tree. The root nodes are projects. Children are tools.

```
Node {
  id:        string (uuid)
  type:      NodeType
  label:     string
  parentId:  string | null
  children:  string[]   // ordered list of child ids
  state:     NodeState
  config:    NodeConfig  // type-specific
}
```

**NodeType** values:
- `project` — logical grouping, no process of its own
- `terminal` — PTY process, rendered via xterm.js
- `editor` — code-server or other web editor, rendered via webview
- `browser` — webview pointed at a URL
- `filemanager` — TUI (yazi/lf) in xterm.js, or filebrowser web app
- `agent` — Claude or other LLM agent process, may have sub-agents as children
- `docker` — docker compose project, tracks container lifecycle
- `devserver` — long-running process (vite, next dev, etc.), output in terminal view

**NodeState**:
- `idle` — defined but not running
- `starting` — process is spawning
- `running` — process is alive
- `stopped` — process exited cleanly
- `crashed` — process exited with error

### 4.2 Process Tree vs Layout Tree

Two separate data structures. They reference each other but are not coupled.

**Process Tree** — source of truth for what exists and what is running.

```typescript
type ProcessNode = {
  id: string
  type: NodeType
  label: string
  parentId: string | null
  children: string[]
  state: NodeState
  pid?: number
  exitCode?: number
  config: NodeConfig
}
```

**Layout Tree** — describes how the screen is divided. A layout node is either a split, a tab group, or a leaf that points to a process node.

```typescript
type LayoutNode =
  | { type: "split";  direction: "h" | "v"; ratio: number; children: [LayoutNode, LayoutNode] }
  | { type: "tabbed"; activeIndex: number;  children: LayoutNode[] }
  | { type: "leaf";   nodeId: string }       // references ProcessNode.id
  | { type: "empty" }                        // placeholder pane
```

A process node can exist in the process tree without appearing in the layout (running in background). A layout leaf can point to a stopped process (shows a restart prompt).

### 4.3 Workspace

A workspace is a saved combination of:
- A subset of the process tree (a project and all its descendants)
- A layout tree for that project

Workspaces can be switched instantly. Switching a workspace does not kill processes — they keep running in the background unless explicitly stopped.

---

## 5. Data Model (Full)

### 5.1 NodeConfig (per type)

```typescript
type TerminalConfig = {
  cmd: string[]         // e.g. ["zsh"] or ["npm", "run", "dev"]
  cwd: string           // working directory
  env?: Record<string, string>
  restoreOnStart: boolean
}

type EditorConfig = {
  kind: "code-server" | "openvscode-server" | "custom"
  port: number          // assigned at runtime, persisted
  cwd: string           // project root for the editor
  restoreOnStart: boolean
}

type BrowserConfig = {
  url: string
  restoreOnStart: boolean
}

type FileManagerConfig = {
  kind: "yazi" | "lf" | "filebrowser" | "custom"
  cwd: string
  port?: number         // if web-based
  restoreOnStart: boolean
}

type AgentConfig = {
  kind: "claude" | "custom"
  model?: string
  systemPrompt?: string
  contextNodeIds: string[]   // which sibling nodes the agent can see
  restoreOnStart: boolean
}

type DockerConfig = {
  composePath: string
  autoUp: boolean
}

type DevServerConfig = {
  cmd: string[]
  cwd: string
  port?: number
  restoreOnStart: boolean
}

type ProjectConfig = {
  cwd: string           // root directory of the project
  color?: string        // accent color in UI
  icon?: string         // emoji or icon name
}
```

### 5.2 Persistence Format

Stored as SQLite via Tauri's built-in SQLite plugin. Three tables:

```sql
CREATE TABLE nodes (
  id         TEXT PRIMARY KEY,
  type       TEXT NOT NULL,
  label      TEXT NOT NULL,
  parent_id  TEXT,
  position   INTEGER NOT NULL,   -- order among siblings
  state      TEXT NOT NULL DEFAULT 'idle',
  config     TEXT NOT NULL,      -- JSON blob of NodeConfig
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE layouts (
  id          TEXT PRIMARY KEY,
  node_id     TEXT NOT NULL,     -- which project this layout belongs to
  layout_json TEXT NOT NULL,     -- serialized LayoutTree
  updated_at  INTEGER NOT NULL
);

CREATE TABLE sessions (
  id          TEXT PRIMARY KEY,
  node_id     TEXT NOT NULL,     -- process node
  pid         INTEGER,
  port        INTEGER,
  started_at  INTEGER,
  state       TEXT NOT NULL
);
```

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

**Parent-child kill propagation**: When a project node is stopped, Yord sends stop signals depth-first to all descendants before stopping the project itself. Order:
1. Agents (graceful shutdown, flush context)
2. Dev servers
3. Editors
4. Terminals
5. Docker (compose down)
6. File managers
7. Browsers (just close webview)

### 6.2 Session Restore

On application start:
1. Load all nodes from SQLite
2. Load last session states
3. For each node where `restoreOnStart = true` and last state was `running`: spawn process
4. Assign new PIDs, update sessions table
5. Reconstruct layout from last saved layout per project
6. Open the last active workspace

Processes that crashed on previous session: shown as `crashed`, user must manually restart.

### 6.3 Port Management

Web-based nodes (editor, filebrowser) need a port. Yord manages a port pool:
- Range: 49152–65535 (ephemeral range)
- Assignment: at spawn time, find next available port
- Persist: port stored in sessions table, reused on restore if still available

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
│         xterm.js instances (per terminal/TUI node)       │
│         webview iframes (per browser/editor node)        │
└────────────────────────┬────────────────────────────────┘
                         │ Tauri IPC (commands + events)
┌────────────────────────▼────────────────────────────────┐
│                     Rust Core                            │
│   ProcessManager  │  LayoutStore  │  PortPool            │
│   PtyBridge       │  NodeStore    │  EventRouter         │
│   SessionManager  │  ConfigStore  │                      │
└─────────────────────────────────────────────────────────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
           SQLite    OS Processes  PTY
```

### 7.3 Tauri Commands (Rust → TS interface)

```rust
// Node management
#[tauri::command] fn create_node(config: NodeCreateRequest) -> Result<Node>
#[tauri::command] fn delete_node(id: String) -> Result<()>
#[tauri::command] fn update_node(id: String, patch: NodePatch) -> Result<Node>
#[tauri::command] fn get_tree() -> Result<Vec<Node>>

// Lifecycle
#[tauri::command] fn start_node(id: String) -> Result<()>
#[tauri::command] fn stop_node(id: String) -> Result<()>
#[tauri::command] fn restart_node(id: String) -> Result<()>
#[tauri::command] fn kill_project(id: String) -> Result<()>  // recursive stop

// PTY (terminal nodes)
#[tauri::command] fn pty_write(node_id: String, data: Vec<u8>) -> Result<()>
#[tauri::command] fn pty_resize(node_id: String, cols: u16, rows: u16) -> Result<()>

// Layout
#[tauri::command] fn save_layout(project_id: String, layout: LayoutNode) -> Result<()>
#[tauri::command] fn get_layout(project_id: String) -> Result<LayoutNode>

// Port
#[tauri::command] fn allocate_port() -> Result<u16>
#[tauri::command] fn release_port(port: u16) -> Result<()>
```

### 7.4 Tauri Events (Rust → UI push)

```rust
// Emitted to frontend
"node:state-changed"    { node_id, state, exit_code? }
"node:pty-data"         { node_id, data: Vec<u8> }
"node:port-ready"       { node_id, port }
"node:process-crashed"  { node_id, exit_code, stderr_tail }
"tree:changed"          { nodes: Vec<Node> }
```

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

- Collapsible tree
- Node icons by type (emoji or icon set)
- Color dot = node state (green=running, red=crashed, grey=idle)
- Right-click context menu: start / stop / restart / rename / delete / add child
- Drag to reorder siblings
- Drag onto a project to reparent

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
- [ ] Tauri + Svelte scaffold
- [ ] Process tree data model in Rust
- [ ] SQLite persistence (nodes table)
- [ ] Tree panel UI (Svelte component)
- [ ] Create/delete/rename project and terminal nodes
- [ ] Spawn PTY process per terminal node
- [ ] xterm.js renders PTY output
- [ ] Kill propagation (kill project → kills children)
- [ ] Session restore for terminal nodes

### M2 — Layout Engine
- [ ] LayoutNode data model
- [ ] Tiling split/tabbed renderer in Svelte
- [ ] Draggable dividers
- [ ] Attach/detach nodes to/from panes
- [ ] Layout persistence per project
- [ ] Workspace switcher UI

### M3 — Web Nodes
- [ ] Tauri webview component
- [ ] Browser node (URL bar + webview)
- [ ] Port pool in Rust
- [ ] Editor node (code-server / openvscode-server)
- [ ] Auto-launch code-server with correct `cwd`
- [ ] File manager node (yazi in xterm.js + filebrowser option)

### M4 — Agent Nodes
- [ ] Agent node type
- [ ] Anthropic API integration (streaming)
- [ ] Context wiring (read sibling terminal output)
- [ ] Sub-agent spawning
- [ ] Conversation history persistence

### M5 — Polish
- [ ] Quick nav `Cmd+P`
- [ ] Drag-to-reparent in tree
- [ ] Node config editor UI
- [ ] Crash reporting (stderr tail on crash)
- [ ] Theming / accent colors per project
- [ ] onboarding / empty state

---

## 12. Open Questions

1. **TUI file manager integration depth** — yazi has a great plugin API. Should Yord talk to yazi over its IPC to know which file is focused (to open it in the editor node)? Or is that over-engineering for v1?

2. **Multi-window** — should a project be detachable into its own OS window? Useful for multi-monitor. Deferred but needs to be compatible with the layout model.

3. **Remote projects** — SSH into a server, run terminal/editor nodes there. Node config could have a `remote: SSHConfig` field. Architecture supports it but not in scope yet.

4. **Plugin system** — new node types defined as plugins (WASM? JS?). Needed for long-term extensibility. Not in v1 but the `NodeType` enum should not be too hardcoded.

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
│   │   │   ├── nodes.rs       -- CRUD commands
│   │   │   ├── lifecycle.rs   -- start/stop/restart
│   │   │   ├── pty.rs         -- PTY bridge
│   │   │   ├── layout.rs      -- layout persistence
│   │   │   └── ports.rs       -- port pool
│   │   ├── models/
│   │   │   ├── node.rs
│   │   │   ├── layout.rs
│   │   │   └── session.rs
│   │   ├── db/
│   │   │   ├── mod.rs
│   │   │   └── migrations/
│   │   └── process/
│   │       ├── manager.rs     -- ProcessManager
│   │       ├── pty.rs         -- PTY abstraction
│   │       └── port_pool.rs
│   └── Cargo.toml
├── src/
│   ├── lib/
│   │   ├── stores/
│   │   │   ├── tree.ts        -- process tree store
│   │   │   ├── layout.ts      -- layout store
│   │   │   └── focus.ts       -- focus state
│   │   ├── types/
│   │   │   ├── node.ts
│   │   │   └── layout.ts
│   │   └── ipc/
│   │       ├── commands.ts    -- typed wrappers around invoke()
│   │       └── events.ts      -- typed event listeners
│   ├── components/
│   │   ├── tree/
│   │   │   ├── TreePanel.svelte
│   │   │   ├── TreeNode.svelte
│   │   │   └── ContextMenu.svelte
│   │   ├── layout/
│   │   │   ├── LayoutRenderer.svelte
│   │   │   ├── SplitPane.svelte
│   │   │   ├── TabbedPane.svelte
│   │   │   └── Leaf.svelte
│   │   └── nodes/
│   │       ├── TerminalView.svelte
│   │       ├── BrowserView.svelte
│   │       ├── EditorView.svelte
│   │       └── AgentView.svelte
│   └── App.svelte
└── package.json
```

---

*Document version: 0.1 — pre-implementation*
*Stack: Tauri 2 + Svelte 5 + Rust + SQLite*
*Target platform: macOS first, cross-platform by M3*
