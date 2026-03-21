# M1 — Tree + Shell: Design Spec

> Milestone 1 for Yord. Scope: Tauri 2 + Svelte 5 scaffold, ECS storage, tree UI, terminal PTY, kill propagation, session restore.

---

## 1. What M1 Delivers

A desktop app where you create projects, add terminal nodes, and manage everything in a tree. Processes are tracked, killed cleanly, and restored on restart.

**In scope:**
- Tauri 2 + Svelte 5 + TypeScript scaffold
- ECS storage layer (entities + components tables, rusqlite)
- Tree panel UI (buildTree() from flat data, collapsible, context menu)
- Entity CRUD (create/delete/rename projects and terminals)
- PTY (spawn per Pty component, xterm.js rendering, write/resize)
- Kill propagation (depth-first by component type, process group kill, respect Pinned)
- Session restore (progressive 4-phase, respawn Pty.cmd in Pty.cwd)

**Out of scope:**
- Tiling/split layout engine (M2)
- Browser/editor/filemanager nodes (M3)
- Agent nodes (M4)
- Plugin system, workspace switcher, quick nav, scrollback persistence

---

## 2. Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| Frontend | Svelte 5 + TypeScript | Fine-grained reactivity, no vdom |
| Desktop | Tauri 2 | Native webview, no Chromium bundle |
| Backend | Rust | Process management, PTY, SQLite |
| Terminal | xterm.js (canvas default, WebGL fallback up) | Industry standard |
| PTY | portable-pty 0.9+ | Cross-platform, from wezterm |
| Storage | rusqlite | Single-writer + WAL, in-memory cache |
| IPC commands | `#[tauri::command]` + manual types | taurpc doesn't support Channel\<T\>; evaluate at M2 |
| IPC streaming | Tauri 2 Channel\<T\> | Per-PTY high-throughput data |
| Package manager | Bun | Fast installs |
| CSS | Plain CSS + CSS variables | Scoped Svelte styles |

---

## 3. Data Model

### 3.1 ECS from day one

Entity IDs are UUID v7 (time-sortable, unique). Generated via the `uuid` crate with `v7` feature.

Two SQLite tables:

```sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA busy_timeout = 5000;
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 268435456;
PRAGMA cache_size = -16000;

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

CREATE INDEX idx_component_type ON components(component_type);
CREATE INDEX idx_belongs_to ON components(component_type, json_extract(data, '$.parent_id'))
  WHERE component_type = 'BelongsTo';
```

**Single-writer architecture** (see `docs/research/storage/sqlite-single-writer.md` for full implementation):
- Dedicated `std::thread` for writes (not `spawn_blocking` — Tokio would parallelize, defeating single-writer)
- `ThreadLocal<Connection>` for readers (one per OS thread, no checkout/return overhead, Zed pattern)
- Write closures sent via `mpsc::unbounded_channel`, results returned via `oneshot`
- `SQLITE_OPEN_NO_MUTEX` on all connections (we guarantee single-thread access)
- Reader connections opened with `SQLITE_OPEN_READ_ONLY` (defense in depth)
- Simple append-only migration system with `_migrations` version table
- Use `INSERT OR REPLACE` / `ON CONFLICT` for idempotent component upserts (prevents constraint error spam)
- Verify bundled SQLite >= 3.51.3 (WAL-reset data race fix)
- On startup: if WAL file exists and is large (>1MB), run checkpoint before `integrity_check` to recover data first
- Run `PRAGMA wal_checkpoint(TRUNCATE)` on writer during shutdown

### 3.2 M1 Components

```
// Structural
Label:          { text, icon?, color? }
BelongsTo:      { parent_id }              // single parent for M1
Order:          { index }

// Identity
Project:        { cwd }
Pty:            { cmd, cwd, env? }

// Lifecycle
Running:        { pid, pgid, started_at }
Crashed:        { exit_code, stderr_tail }
RestoreOnStart: {}
Pinned:         {}
```

Lifecycle states derived from component presence (no stored state field).

---

## 4. Swappable Boundary Traits

```rust
trait EntityStore {
    fn create(&self, components: Vec<ComponentData>) -> Result<EntityId>;
    fn delete(&self, id: &EntityId) -> Result<()>;
    fn get(&self, id: &EntityId) -> Result<EntityWithComponents>;
    fn get_all(&self) -> Result<Vec<EntityWithComponents>>;
    fn set_component(&self, id: &EntityId, component: ComponentData) -> Result<()>;
    fn remove_component(&self, id: &EntityId, comp_type: &str) -> Result<()>;
    fn query(&self, filters: &[ComponentFilter]) -> Result<Vec<EntityWithComponents>>;
}

trait PtyBackend {
    fn spawn(&self, cmd: &[String], cwd: &str, env: &HashMap<String, String>) -> Result<PtyHandle>;
    fn write(&self, handle: &PtyHandle, data: &[u8]) -> Result<()>;
    fn resize(&self, handle: &PtyHandle, cols: u16, rows: u16) -> Result<()>;
    fn kill(&self, handle: &PtyHandle) -> Result<()>;
}

trait ProcessManager {
    fn kill_group(&self, pgid: u32, signal: Signal) -> Result<()>;
    fn is_alive(&self, pid: u32) -> bool;
    fn children(&self, pid: u32) -> Result<Vec<u32>>;
}
```

M1 implementations: `SqliteEntityStore`, `PortablePtyBackend`, `UnixProcessManager` / `WindowsProcessManager`.

---

## 5. Rust Backend

### 5.1 Systems

- **LifecycleSystem** — spawn/kill/restart PTY processes
- **RestoreSystem** — progressive 4-phase restore on startup
- **KillProjectSystem** — depth-first by component type, process group kill
- **PersistSystem** — debounced write to SQLite with WAL mode (1s debounce for component changes, `PRAGMA optimize` on shutdown)

### 5.2 PTY Spawning

- `portable-pty` 0.9+: `openpty()` → `slave.spawn_command()` → drop slave immediately
- `setsid()` for new process group (Linux/macOS)
- `CREATE_NEW_PROCESS_GROUP` + Job Objects (Windows)
- `PR_SET_PDEATHSIG(SIGTERM)` on Linux for crash safety
- `close_fds` crate for fd hygiene after fork
- `take_writer()` even if unused (prevents child blocking on stdin forever)
- `std::thread::spawn` per PTY reader (not Tokio — avoids thread pool starvation)
- Wrap reader thread body in `std::panic::catch_unwind` — on panic, still clean up PTY state and send exit notification
- Kill flag (`Arc<AtomicBool>`) + `ChildKiller` handle for clean shutdown
- Implement `Drop` on PTY handle with 100ms graceful kill timeout as defense-in-depth
- **Shell detection fallback chain:** `$SHELL` → `/etc/passwd` entry → `/bin/sh` (Unix) / `COMSPEC` → `cmd.exe` (Windows). Validate shell exists before spawn.
- Set `TERM=xterm-256color` and `TERM_PROGRAM=yord` in PTY environment
- Remove `npm_config_prefix` env, filter `CLAUDECODE*`, `ALACRITTY_*`, stale terminal-specific env vars
- macOS PATH augmentation (`~/.cargo/bin`, Homebrew paths, nvm/fnm)
- **Windows PATH:** inherit current PATH as-is, do not reconstruct. Expand `%VAR%` syntax in `Pty.cwd`/`Pty.cmd`. Handle `PATHEXT` empty entries (`;;`). Normalize paths (backslashes, valid drive letter).
- EIO handling: macOS `ErrorKind::Other`, Linux `BrokenPipe` — both are normal EOF
- **Input vs output backpressure:** Never drop user input (writes to PTY stdin). Backpressure applies only to PTY output (reads from stdout). Write path must be unbuffered or flushed immediately.

### 5.3 PTY Output Streaming

Single async Tokio task per PTY with ring buffer + timer flush. See `docs/research/pty/output-buffering.md` for full design.

- 256KB ring buffer per PTY (bounded memory, overwrites oldest on overflow — xterm.js has its own scrollback)
- `tokio::select!` with `biased` over: PTY read (priority), 4ms timer tick, command channel
- Flush when timer fires OR buffer >75% full (adaptive: don't wait under load)
- 32KB read buffer (drains kernel PTY buffer in 1-2 reads)
- One Tauri 2 `Channel<Vec<u8>>` per entity
- Raw `Vec<u8>` / `Uint8Array` (not string conversion — handles non-UTF-8)
- Check `channel.send()` result — if broken (e.g., frontend reload), stop reading and clean up. Don't loop into a dead channel.
- No socketpair (simpler than wezterm — we don't parse escape sequences on Rust side)

### 5.4 PTY Process Management (Per-PTY Actor Pattern)

One `tokio::spawn` task per PTY (not one global actor). See `docs/research/pty/actor-patterns.md` for full design.

**Registry:**
```rust
struct PtyRegistry {
    ptys: RwLock<HashMap<EntityId, PtyHandle>>,
}

struct PtyHandle {
    cmd_tx: mpsc::Sender<PtyCommand>,
}
```

**Per-PTY actor commands:**
```rust
enum PtyCommand {
    Write(Vec<u8>),
    Resize { rows: u16, cols: u16 },
    Kill,
    GetStatus(oneshot::Sender<PtyStatus>),
}
```

Each actor task owns its PTY state and runs a `tokio::select!` loop over: PTY read, pending writes, and command channel. Actor self-removes from registry on exit (EOF, error, or kill).

**Backpressure:** 10MB pending-bytes cap per PTY (zellij pattern). If exceeded, pending writes are dropped with a warning — prevents OOM. Write blocking concern addressed: writes go through the actor's pending buffer and are drained via `select!`, never blocking command processing.

**Why per-PTY actors (not one global)?** A stuck PTY blocks only its own actor task. Tokio can run thousands of tasks on a few threads. Global mutex (canopy pattern) would freeze all terminals on one stuck PTY.

### 5.5 Kill Propagation

Depth-first by component type:
1. Entities with `Agent` component
2. Entities with `Pty` component
3. Entities with `Editor` component
4. Entities with `Docker` component
5. Entities with `Webview` component

M1 implements kill for `Pty` only. Other component types listed for forward compatibility.

Entities with `Pinned` component are skipped.

Per-entity kill sequence (PTY — two-step, from zed):
1. `killpg(tcgetpgrp(pty_fd), SIGHUP)` — kill foreground process group first (e.g., `cargo build`)
2. Poll `try_wait()` at 50ms intervals for 250ms grace
3. Kill shell process group if foreground kill succeeded
4. `SIGKILL` to process group if still alive
5. Remove `Running` component after confirmed exit

Non-shell processes (Docker, agents): `SIGTERM` with 5s timeout.

**Windows kill sequence:** `GenerateConsoleCtrlEvent(CTRL_C_EVENT)` → wait → `TerminateProcess`. Verify Job Object assignment with `IsProcessInJob()` after spawn. Note: `TerminateProcess` returns nonzero on *success* (wezterm has an inverted error check bug here — write fresh with correct semantics).

**SIGHUP handler on Yord process:** Install a handler so external SIGHUP (e.g., from a parent terminal) triggers clean shutdown (kill propagation + WAL checkpoint), not abrupt exit.

**After SIGKILL:** Call `waitpid` (blocking, short timeout) to reap zombies. The `try_wait()` polling handles the grace period, but after SIGKILL do a final blocking reap.

### 5.6 App Shutdown

- Custom quit handler + `RunEvent::ExitRequested` (Tauri's default Quit skips destructors)
- `on_window_event(Destroyed)` to kill PTY children — use `try_state()` (returns Option), not `state()` (panics if unavailable during abnormal shutdown)
- On startup: sweep SQLite for stale `Running` PIDs from previous crashes

### 5.7 Session Restore (Progressive)

See `docs/research/storage/session-restore.md` for exhaustive edge case table.

**Phase 1 (0–200ms):** Load entities from SQLite. Run `PRAGMA integrity_check` (if corrupt, copy to `.bak`, start fresh). Check `_migrations` max version — if DB is from a newer Yord version, show error and optionally start fresh with backup. If `Order.index` values have gaps/duplicates, re-normalize. Show tree with skeleton UI. Use `tauri-plugin-window-state` for window geometry restore (on Wayland: position restore is a no-op, restore size only).

**Phase 2 (200ms–1s):** Stale PID cleanup:
- Store `(pid, started_at)` pairs — `kill(pid, 0)` only tells you a process exists, not that it's YOURS. Compare `started_at` against `/proc/<pid>/stat` start time on Linux. If mismatch, treat as dead.
- Also kill stale process groups (`killpg`) to clean up child processes.
- Write `last_clean_shutdown` timestamp to DB on clean exit. On startup, if missing/stale, run aggressive cleanup.
- Spawn focused terminal only.

**Phase 3 (1–5s):** Validate and spawn remaining `RestoreOnStart` entities:
- Validate `Pty.cwd` exists (fallback: parent project CWD → `$HOME`). Show warning badge if fallback used.
- Validate `Pty.cmd` on PATH. If missing, show as `Crashed` with "Command not found" — don't spawn.
- **Default shell detection:** If `Pty.cmd` matches previously-detected default shell, use CURRENT `$SHELL` instead (don't hardcode paths).
- **Bounded concurrency:** Semaphore (4 concurrent spawns) to prevent thundering herd. Priority: focused → visible → background.
- **Per-entity mutex** to prevent double-spawn race (user clicks "start" during Phase 3).
- **Per-entity `catch_unwind`** around each entity's restore — one bad entity must not crash the entire restore.
- **Crash loop prevention:** If process exits within <2s of spawn during restore, don't auto-restart. Show as `Crashed`. Also check if the port it was using is already bound — provide specific error: "Port 3000 already in use."
- If `Pty.cmd` is not a shell and not found on PATH, fall back to spawning the default shell in the same CWD rather than showing `Crashed`.
- Filter session-specific env vars on restore: `TERM_SESSION_ID`, `TMUX`, `STY`, `WINDOWID`, `SHELL_SESSION_ID`.

**Phase 4 (5s+):** `PRAGMA optimize`. Show restore summary toast: "5 terminals restored, 1 failed (missing directory)".

On terminal restore: prepend `\033c` reset to clear stale terminal state.

### 5.8 Foreground Process Tracking (M1 stretch goal)

`tcgetpgrp(pty_fd)` returns the foreground process PID. Combined with `sysinfo` crate, show dynamic labels in the tree: "zsh" → "npm run dev" → "node server.js". Cache with 300ms TTL (wezterm pattern). Low-hanging fruit for better UX — skip if time-constrained.

---

## 6. IPC

### 6.1 Commands (`#[tauri::command]`)

Plain Tauri commands with manual TypeScript types. `taurpc` doesn't support `Channel<T>` (Yord's most critical IPC mechanism). Evaluate taurpc at M2 when IPC surface grows.

```rust
// Entity management
#[tauri::command] async fn create_entity(components: Vec<ComponentData>) -> Result<EntityId, YordError>;
#[tauri::command] async fn delete_entity(id: String) -> Result<(), YordError>;
#[tauri::command] async fn get_entities() -> Result<Vec<EntityWithComponents>, YordError>;

// Component management
#[tauri::command] async fn set_component(entity_id: String, component: ComponentData) -> Result<(), YordError>;
#[tauri::command] async fn remove_component(entity_id: String, component_type: String) -> Result<(), YordError>;

// Lifecycle
#[tauri::command] async fn start_entity(id: String, channel: Channel<Vec<u8>>) -> Result<(), YordError>;
#[tauri::command] async fn stop_entity(id: String) -> Result<(), YordError>;
#[tauri::command] async fn restart_entity(id: String, channel: Channel<Vec<u8>>) -> Result<(), YordError>;
#[tauri::command] async fn kill_project(id: String) -> Result<(), YordError>;

// PTY
#[tauri::command] async fn pty_write(entity_id: String, data: Vec<u8>) -> Result<(), YordError>;
#[tauri::command] async fn pty_resize(entity_id: String, cols: u16, rows: u16) -> Result<(), YordError>;
// allocate_port / release_port deferred to M3
```

Note: `start_entity` and `restart_entity` accept a `Channel<Vec<u8>>` parameter for PTY output streaming. The frontend creates the channel and passes it in the invoke call.

### 6.2 Events and Channels

```
// Events (low-frequency)
"entity:component-changed"  { entity_id, component_type, data }
"entity:removed"            { entity_id }

// Channels (per-PTY, high-frequency)
Channel<PtyOutput> { entity_id, data: Vec<u8> }
```

Cleanup: every `listen()` and Channel `onmessage` cleaned up on component destroy. Delete `channel.onmessage` explicitly (known memory leak #13133).

---

## 7. Frontend

### 7.1 Tree Panel

See `docs/research/frontend/flat-to-tree.md` for full `buildTree()` implementation.

- `buildTree()`: O(n) single-pass algorithm. Index entities by `parent_id`, sort by `Order.index`, recursive assembly. Returns fresh `TreeNode[]` for `$state.raw()` reassignment.
- Handles orphans (promote to root — visible, not silently dropped), cycles (visited set), missing parents
- Use `$state.raw()` for entity/tree data (NOT `$state()` — 5,000x perf cliff with deep proxies)
- Called on: initial load, `entity:component-changed` event, `entity:removed` event
- Debounce rapid events with `queueMicrotask()` to avoid redundant rebuilds
- Collapsible, color-coded status dots (green=Running, red=Crashed, grey=idle)
- Right-click context menu: start / stop / restart / rename / delete / add child
- Drag to reorder (updates `Order` component)
- Drag onto project to reparent (updates `BelongsTo` component)

### 7.2 Terminal View

- All xterm.js instances in main Svelte webview (no separate Tauri webviews)
- Renderer fallback: WebGL → Canvas → DOM. Handle `webglcontextlost` event at runtime — fall back to Canvas renderer on context loss (macOS sleep, GPU driver reset). Re-attempt WebGL after delay.
- Lazy instantiation (only visible terminals get DOM-mounted)
- `@xterm/addon-fit`, `@xterm/addon-search` (`Cmd+F`)
- Default scrollback: 5,000 lines
- Ship bundled monospace font (system monospace fallback)
- **Escape sequence filtering (two-layer defense)** — see `docs/research/pty/escape-filtering.md`:
  - **Layer 1 (Rust, `vte` crate):** Strip CSI 20t/21t (title reports, RCE vector), all window manipulation CSI t, DCS sequences (except XTGETTCAP — allow through for terminal capability detection by programs like Neovim), DECRQSS. Validate OSC 7 (file: protocol only, 1024 byte cap). Block OSC 52 clipboard queries. Enforce OSC 52 size limits (75KB decoded, 128KB raw). Document specific ConPTY artifact sequences to filter on Windows.
  - **Layer 2 (xterm.js handlers):** OSC 52 focus gating (only write clipboard when terminal focused) + user notification. OSC 7 CWD tracking. Paste de-fanging (strip bracketed paste markers `\x1b[200~`/`\x1b[201~` from clipboard content).

### 7.3 Focus & Keyboard

- Tracked in Svelte store with visual border on active pane
- Global shortcuts use `Cmd`/`Super` — never intercept `Ctrl+C/D/Z/L/R`
- Check `event.isComposing` before acting (IME safety)

### 7.4 M1 Layout (Simple)

Tree panel on left, single terminal view on right. Click a terminal node to view it. No tiling/splitting (M2).

### 7.5 Security

- Enable CSP in `tauri.conf.json` — allow `blob:`, `data:`, `wasm-unsafe-eval` for xterm.js
- Audit capability files in `src-tauri/capabilities/`

---

## 8. Error Handling

Use `thiserror` for a structured `YordError` enum:

```rust
#[derive(thiserror::Error, Debug)]
enum YordError {
    #[error("Entity not found: {0}")]
    EntityNotFound(String),
    #[error("Component not found: {entity_id}/{component_type}")]
    ComponentNotFound { entity_id: String, component_type: String },
    #[error("PTY error: {0}")]
    Pty(String),
    #[error("Database error: {0}")]
    Db(#[from] rusqlite::Error),
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}
```

Errors serialized to frontend as JSON string via Tauri's `Result` return type. Frontend displays errors in the tree node (Crashed state with stderr tail) or as toast notifications.

---

## 9. Testing Strategy

- **EntityStore**: in-memory `HashMap`-backed implementation of the `EntityStore` trait for unit tests. No SQLite dependency in unit tests.
- **PtyBackend**: mock implementation that simulates spawn/write/kill without real processes.
- **Integration tests**: real SQLite + real PTY on CI. Test spawn, write, resize, kill propagation, session restore.
- **Frontend**: vitest for `buildTree()` logic and store derivations. No E2E in M1.
- **CI**: GitHub Actions matrix: Ubuntu, macOS, Windows.

---

## 10. Directory Structure

```
yord/
├── src-tauri/src/
│   ├── main.rs
│   ├── commands/          (#[tauri::command] handlers)
│   ├── ecs/
│   │   ├── entity.rs      (Entity struct)
│   │   ├── component.rs   (Component types + ComponentData enum)
│   │   ├── traits.rs      (EntityStore, PtyBackend, ProcessManager)
│   │   └── store.rs       (SqliteEntityStore)
│   ├── systems/
│   │   ├── lifecycle.rs   (LifecycleSystem)
│   │   ├── restore.rs     (RestoreSystem)
│   │   ├── kill.rs        (KillProjectSystem)
│   │   └── persist.rs     (PersistSystem)
│   ├── db/
│   │   ├── mod.rs
│   │   └── migrations/
│   └── process/
│       ├── pty.rs         (PortablePtyBackend, PTY spawning)
│       ├── actor.rs       (PtyRegistry, PtyHandle, per-PTY actor loop)
│       ├── ring_buffer.rs (256KB ring buffer for output batching)
│       ├── escape_filter.rs (vte-based escape sequence filter)
│       └── port_pool.rs
├── src/
│   ├── lib/stores/        (entities, tree, focus)
│   ├── lib/types/         (entity, tree, component types)
│   ├── lib/ipc/           (typed invoke wrappers)
│   ├── components/tree/   (TreePanel, TreeNode, ContextMenu)
│   ├── components/views/  (TerminalView)
│   └── App.svelte
└── package.json
```

---

## 11. Platforms

Linux, Windows, macOS, Android, iOS. Browser deferred.

- Windows: minimum 10 1809+ (ConPTY)
- Linux: WebKitGTK 2.42+ recommended
- Dev/test on Linux first

---

## 12. Cross-Platform Notes

- Avoid `backdrop-filter` CSS (broken on WebKitGTK)
- Test CJK characters and emoji rendering on all platforms
- ConPTY injects unsolicited escape sequences — account for in filtering
- PTY resize: debounce 50-100ms, fresh resize when hidden terminal becomes visible

---

## 13. Research References

- `docs/research/landscape/ide-pain-points.md` — VS Code, Neovim, tmux/zellij pain points
- `docs/research/landscape/ide-path.md` — IDE architecture lessons (Xi, Atom, VS Code, Zed, Lapce)
- `docs/research/landscape/tauri-pitfalls.md` — Tauri 2 bugs, security, performance traps
- `docs/research/pty/` — actor patterns, output buffering, escape filtering
- `docs/research/storage/` — SQLite single-writer, session restore edge cases
- `docs/research/frontend/` — flat-to-tree (buildTree) implementation
- `docs/research/from-issues/known-pitfalls.md` — 30+ pitfalls with spec coverage analysis
- `docs/research/from-issues/spec-validation.md` — 592 issues cross-referenced against 8 design decisions
- `docs/research/from-issues/cross-platform.md` — 75 platform-specific bugs (Windows, macOS, Linux)
- `docs/refs/report.md` — 9 reference project survey
- `docs/refs/projects/` — per-project deep research

---

*Spec version 3.0 — 2026-03-21*
*v2: 7 research dives. v3: 592-issue cross-reference, 30+ pitfall catalog, 75 platform bugs*
