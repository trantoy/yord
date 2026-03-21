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
Running:       { pid: number, pgid: number, started_at: number }
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

**Process group kill**: Signals are sent to the entire process group (`kill(-pgid, SIGTERM)`), not just the shell PID. This ensures child processes (e.g., `node server.js` spawned by `npm run dev`) are killed too. Sequence per entity:
1. Send `SIGTERM` to process group
2. Wait configurable timeout (default 5s, Docker gets more)
3. Send `SIGKILL` to process group if still alive
4. Remove `Running` component only after confirmed process exit

**Crash safety**: On Linux, child processes are spawned with `PR_SET_PDEATHSIG(SIGTERM)` so they die if the Yord process crashes. On Windows, Job Objects group child processes for reliable cleanup. PIDs/PGIDs are persisted in SQLite so the next launch can detect and clean up orphans.

### 6.2 Session Restore

Progressive restore on application start:

**Phase 1 (0–200ms):** Load entities from SQLite. Restore window geometry. Show tree panel with skeleton UI. Window starts hidden, shows after layout is ready.

**Phase 2 (200ms–1s):** Stale PID cleanup — check each `Running` component's PID (`kill(pid, 0)` on Unix, `OpenProcess` on Windows). If dead, remove `Running` and add `Crashed` if exit was non-zero. Spawn the focused/active terminal only.

**Phase 3 (1–5s):** For remaining entities with `RestoreOnStart`: validate `Pty.cwd` exists and `Pty.cmd` is on PATH. If valid, spawn. If invalid, show error in tree (don't silently skip). Assign new PIDs/PGIDs, update `Running` components.

**Phase 4 (5s+):** Run `PRAGMA optimize` on SQLite.

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
| UI framework | Svelte 5 + TypeScript | Minimal overhead, fine-grained reactivity, no vdom |
| Desktop shell | Tauri 2 | Rust core, native webview, no Chromium bundle |
| Backend | Rust | Process management, PTY, file system, SQLite |
| Terminal renderer | xterm.js | Industry standard, handles all TUI apps |
| PTY bridge | portable-pty (Rust crate) | Cross-platform PTY, from wezterm project |
| Storage | rusqlite | Direct Rust control, in-memory + WAL persist |
| IPC (commands) | taurpc + specta | Typed IPC, auto-generates TS types from Rust traits |
| IPC (streaming) | Tauri 2 Channel\<T\> | High-throughput PTY data, lower overhead than events |
| Package manager | Bun | Fast installs, native TS execution |
| CSS | Plain CSS + CSS variables | Scoped Svelte styles, precise layout control |

### 7.2 Swappable Boundaries (Traits)

Three traits at dependency boundaries. Trait where you'd swap a dep, not at every layer.

```rust
// Storage — swap SQLite for bevy_ecs, sled, etc.
trait EntityStore {
    fn create(&self, components: Vec<ComponentData>) -> Result<EntityId>;
    fn delete(&self, id: &EntityId) -> Result<()>;
    fn get(&self, id: &EntityId) -> Result<EntityWithComponents>;
    fn get_all(&self) -> Result<Vec<EntityWithComponents>>;
    fn set_component(&self, id: &EntityId, component: ComponentData) -> Result<()>;
    fn remove_component(&self, id: &EntityId, comp_type: &str) -> Result<()>;
    fn query(&self, filters: &[ComponentFilter]) -> Result<Vec<EntityWithComponents>>;
}

// PTY — swap portable-pty for wezterm-term, custom impl, etc.
trait PtyBackend {
    fn spawn(&self, cmd: &[String], cwd: &str, env: &HashMap<String, String>) -> Result<PtyHandle>;
    fn write(&self, handle: &PtyHandle, data: &[u8]) -> Result<()>;
    fn resize(&self, handle: &PtyHandle, cols: u16, rows: u16) -> Result<()>;
    fn kill(&self, handle: &PtyHandle) -> Result<()>;
}

// Process management — swap kill strategy per platform
trait ProcessManager {
    fn kill_group(&self, pgid: u32, signal: Signal) -> Result<()>;
    fn is_alive(&self, pid: u32) -> bool;
    fn children(&self, pid: u32) -> Result<Vec<u32>>;
}
```

M1 implementations: `SqliteEntityStore`, `PortablePtyBackend`, `UnixProcessManager` / `WindowsProcessManager`. Future milestones can swap any of these without touching the rest of the codebase.

### 7.3 Process Layers

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

### 7.4 Tauri Commands (taurpc trait → auto-gen TS)

```rust
#[taurpc::procedures]
trait YordApi {
    // Entity management
    fn create_entity(components: Vec<ComponentData>) -> Result<EntityId>;
    fn delete_entity(id: String) -> Result<()>;
    fn get_entities() -> Result<Vec<EntityWithComponents>>;

    // Component management
    fn set_component(entity_id: String, component: ComponentData) -> Result<()>;
    fn remove_component(entity_id: String, component_type: String) -> Result<()>;

    // Lifecycle
    fn start_entity(id: String) -> Result<()>;
    fn stop_entity(id: String) -> Result<()>;
    fn restart_entity(id: String) -> Result<()>;
    fn kill_project(id: String) -> Result<()>;  // recursive stop

    // PTY (entities with Pty component)
    fn pty_write(entity_id: String, data: Vec<u8>) -> Result<()>;
    fn pty_resize(entity_id: String, cols: u16, rows: u16) -> Result<()>;

    // Port
    fn allocate_port() -> Result<u16>;
    fn release_port(port: u16) -> Result<()>;
}
```

### 7.5 Tauri Events and Channels (Rust → UI push)

```rust
// Events (low-frequency state changes)
"entity:component-changed"  { entity_id, component_type, data }
"entity:removed"            { entity_id }
"entity:port-ready"         { entity_id, port }

// Channels (high-frequency streaming — one Channel<T> per PTY entity)
// PTY data is NOT sent via events. Each PTY gets a dedicated Tauri 2 Channel
// that streams batched output directly to a JS callback.
Channel<PtyOutput> { entity_id, data: Vec<u8> }
```

Component changes cover all state transitions — `Running` added means started, `Running` removed means stopped, `Crashed` added means crashed. No separate state-change event needed.

**PTY output batching**: Rust reads PTY in a tight loop on a dedicated thread per entity. Output is accumulated in a ring buffer (256KB per PTY) and flushed to the Channel every 16ms (60fps) or when the buffer exceeds 4KB — whichever comes first. This prevents UI freezes from output floods (e.g., `cat /dev/urandom`, verbose builds).

### 7.6 Implementation Notes (from research)

**PTY spawning:**
- Use `setsid()` (Linux/macOS) to create a new process group per PTY
- On Windows, use `CREATE_NEW_PROCESS_GROUP` flag + Job Objects
- Store both PID and PGID in `Running` component
- Set `PR_SET_PDEATHSIG(SIGTERM)` on Linux for crash safety
- Respect `$SHELL` and launch as interactive shell by default
- PTY reads are blocking — use `spawn_blocking` thread per PTY, never read in async task directly
- EOF detection differs: Unix drops slave side; Windows needs explicit EOT (`\x04`)

**PTY process management (actor pattern):**
- Use an actor (mpsc channel) instead of `Arc<Mutex<HashMap>>` for the PTY registry
- Commands: Spawn, Write, Resize, Kill sent via channel to a single owner task
- Eliminates lock contention and prevents deadlocks with Tokio runtime

**App shutdown:**
- Tauri's `MenuItem::Quit` calls `exit(0)` immediately, skipping destructors
- Must use custom quit handler + `RunEvent::ExitRequested` for cleanup
- On startup, sweep for orphaned PIDs from previous crashes (stored in SQLite)

**xterm.js:**
- All terminal instances render in the main Svelte webview (no separate Tauri webviews)
- Renderer fallback chain: WebGL → Canvas → DOM (WebGL fails on older Linux WebKitGTK)
- Lazy instantiation: only create xterm.js instance when entity is visible in layout
- Hidden terminals: PTY output buffered in Rust ring buffer, xterm.js detached from DOM
- Ship a bundled monospace font (system monospace as fallback)
- Integrate `@xterm/addon-search` for scrollback search (`Cmd+F` in terminal panes)
- Default scrollback: 5,000 lines (configurable per entity)
- **Escape sequence sanitization**: Whitelist allowed sequences. Strip OSC 52 (clipboard write), DECRQSS responses, title set/report, OSC 5113 (Kitty file transfer). Real CVEs with CVSS 9.8 demonstrate RCE via unsanitized terminal output.

**Keyboard:**
- Global shortcuts use `Cmd`/`Super` key — never intercept `Ctrl+C/D/Z/L/R`
- Check `event.isComposing` before acting on key events (IME safety)
- Focus tracked in Svelte store with visual border indicator on active pane

**SQLite (single-writer architecture):**
- Dedicated writer connection + reader pool (single-writer is ~20x faster than pooled writes)
- PRAGMAs on every connection:
  ```sql
  PRAGMA journal_mode = WAL;
  PRAGMA synchronous = NORMAL;
  PRAGMA busy_timeout = 5000;
  PRAGMA foreign_keys = ON;
  PRAGMA temp_store = MEMORY;
  PRAGMA mmap_size = 268435456;
  PRAGMA cache_size = -16000;
  ```
- Use transactions for atomic entity+component persist
- Verify PIDs alive before persisting `Running` state
- Verify bundled SQLite ≥ 3.51.3 (WAL-reset data race fix)

**PTY resize:**
- Debounce resize events (50-100ms) during pane divider drags
- Send fresh resize when a hidden terminal becomes visible

**Scrollback persistence (future, post-M1):**
- Rust ring buffer (256KB per PTY) enables re-feeding xterm.js after macOS WKWebView jettison or session restore

**Svelte performance:**
- Use `$state.raw()` (not `$state()`) for entity/tree data from Rust — deep proxies cause 5,000x slowdown on tree structures
- Reassign to trigger updates instead of relying on deep reactivity
- Consider virtual scrolling for large trees (17,000+ nodes in <100ms with flat rendering)

**Tauri security:**
- Enable CSP in `tauri.conf.json` — not enabled by default, leaving app open to XSS → full IPC access
- Allow `blob:`, `data:`, `wasm-unsafe-eval` for xterm.js in CSP
- Audit capability files in `src-tauri/capabilities/` — all are auto-enabled

**Tauri event/channel cleanup:**
- Every `listen()` and Channel `onmessage` must be cleaned up on component destroy
- Channel has a known memory leak (#13133): delete `channel.onmessage` explicitly on unmount
- After 2M events without cleanup, frontend reaches ~1.1GB

**Cross-platform:**
- Minimum Windows 10 1809+ (ConPTY support)
- Minimum WebKitGTK 2.42+ recommended for Linux
- Avoid `backdrop-filter` CSS (broken on WebKitGTK)
- Test CJK characters and emoji rendering on all platforms

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
│   │   │   ├── traits.rs      -- EntityStore, PtyBackend, ProcessManager traits
│   │   │   └── store.rs       -- SqliteEntityStore (impl EntityStore)
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
*Target platforms: Linux, Windows, macOS, Android, iOS (browser deferred — see docs/backlog/browser-platform.md)*
