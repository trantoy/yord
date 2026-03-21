# Reference Projects — Research Report

> Compiled 2026-03-21. Source repos cloned to `/refs/` (gitignored).

---

## Overview

| Project | License | Stack | Can Take Code? | Relevance |
|---------|---------|-------|----------------|-----------|
| **tauri-plugin-pty** | MIT | Tauri 2 + Rust + portable-pty | Yes | Tauri PTY plugin, multi-session |
| **tauri-terminal** | None (unlicensed) | Tauri 1 + Rust + portable-pty | No (ref only) | Single-PTY demo |
| **canopy** | MIT | Tauri 2 + React 19 + portable-pty + SQLite | Yes | Closest to Yord's M1 scope |
| **Clif-Code** | MIT | Tauri 2 + SolidJS + portable-pty + Monaco | Yes | IDE with PTY + file tree |
| **wezterm** | MIT | Rust native, portable-pty origin | Yes (crate dep) | PTY crate source, process mgmt |
| **zellij** | MIT | Rust native, raw openpty via nix | Yes | Session restore, layout, kill patterns |
| **waveterm** | Apache-2.0 | Go + Electron + React + SQLite | Patterns only | Workspace model, xterm.js, scrollback |
| **zed** | GPL-3.0 | Rust native, GPUI, alacritty_terminal | No (inspiration) | Entity model, kill propagation, SQLite |
| **helix** | MPL-2.0 | Rust native, TUI | No (inspiration) | SlotMap tree, view/document split |

---

## 1. PTY Integration Patterns

### How each project spawns and manages PTY

**portable-pty** (used by tauri-plugin-pty, canopy, Clif-Code, tauri-terminal):
- `native_pty_system().openpty(PtySize)` returns `PtyPair { master, slave }`
- `slave.spawn_command(CommandBuilder)` forks the child
- `master.try_clone_reader()` / `master.take_writer()` for I/O
- Unix: `setsid()` + `TIOCSCTTY` in `pre_exec` — child becomes session leader
- Windows: ConPTY via `CreatePseudoConsole`
- `ChildKiller` trait separated from `Child` — can kill from one thread while another blocks on `wait()`

**zellij** (raw nix):
- `nix::pty::openpty()` + `libc::login_tty()` in `pre_exec`
- `close_fds::close_open_fds(3, &[])` for fd hygiene
- `tokio::AsyncFd` wrapping the master fd for epoll-based async reads
- Direct `libc::ioctl(TIOCSWINSZ)` for resize

**zed** (alacritty_terminal as library):
- `alacritty_terminal::tty::new()` creates the PTY
- `Term<ZedListener>` for terminal state machine (VTE parsing, grid, scrolling)
- `EventLoop` on dedicated I/O thread
- Events bridged through `UnboundedSender<AlacTermEvent>` channel

**waveterm** (Go):
- `creack/pty` Go library: `pty.StartWithSize()`
- 4KB read loop in goroutine → circular blockfile (SQLite) + pub/sub event
- WebSocket IPC to Electron frontend

### Tauri↔PTY IPC patterns (most relevant for Yord)

| Project | Transport | Direction |
|---------|-----------|-----------|
| **canopy** | Tauri v2 `Channel<TerminalEvent>` | PTY→frontend (push) |
| **tauri-plugin-pty** | Blocking `invoke("read")` in async loop | PTY→frontend (poll) |
| **tauri-terminal** | `requestAnimationFrame` + `invoke("read")` | PTY→frontend (poll) |
| **Clif-Code** | Tauri `emit_to()` events | PTY→frontend (push) |

**Canopy's approach is best for Yord.** It uses Tauri v2's `Channel<T>` (high-throughput, per-entity stream) with a dedicated `std::thread::spawn` reader thread per PTY. This matches Yord's design doc specification of `Channel<PtyOutput>` per entity. The blocking-invoke polling in tauri-plugin-pty ties up Tokio threads.

### Reader thread pattern (canopy — recommended)

```
std::thread::spawn → read 4KB chunks from master reader
  → Channel<TerminalEvent>::send(Output { data: Vec<u8> })
  → on EOF/error: send Exit { code }, remove from state map
```

### Reader thread pattern (Clif-Code — alternative)

```
std::thread::spawn → read 32KB chunks from master reader
  → app_handle.emit_to(window_label, "terminal-output", data)
  → kill flag (Arc<Mutex<bool>>) checked each iteration
```

---

## 2. Process Management & Kill Propagation

### Signal strategies

| Project | Strategy | Process Groups? |
|---------|----------|-----------------|
| **wezterm** | SIGHUP → 250ms grace → SIGKILL | No (relies on PTY close → kernel SIGHUP) |
| **zellij** | SIGHUP (or SIGKILL for force) per PID | No (individual PIDs) |
| **zed** | `killpg(pid, SIGKILL)` for foreground + shell | Yes (process groups) |
| **waveterm** | SIGHUP → 400ms → SIGKILL; `kill(-pgid)` for jobs | Yes (for job manager) |
| **canopy** | `child.kill()` on window destroy | No |

**Yord's design** specifies process group kill (`kill(-pgid, SIGTERM)` → timeout → `kill(-pgid, SIGKILL)`). This is the most thorough approach. Zed's two-step pattern is worth noting: kill the foreground process group first, then the shell.

### Crash safety

| Mechanism | Used by |
|-----------|---------|
| `PR_SET_PDEATHSIG(SIGTERM)` (Linux) | Yord design (planned) |
| `setsid()` session leader | portable-pty, zellij |
| Job Objects (Windows) | Yord design (planned), wezterm (ConPTY) |
| PID persistence for orphan cleanup | waveterm, Yord design (planned) |
| `on_window_event(Destroyed)` cleanup | canopy, Clif-Code |
| `Drop` impl on pane | zed |

### Foreground process tracking

Both zed and wezterm use `libc::tcgetpgrp(pty_fd)` to identify the foreground process group in a PTY. Combined with `sysinfo` crate for process name/CWD/args. Wezterm caches with 300ms TTL.

---

## 3. Session Persistence & Restore

| Project | Storage | What's persisted | Scrollback? |
|---------|---------|-----------------|-------------|
| **canopy** | localStorage (tabs) + SQLite (projects) | Tab list, label, project path, Claude session ID | No |
| **zellij** | KDL files on disk (periodic, 60s) | Layout tree, per-pane command + CWD + geometry + focus + pane contents | Partial (pane contents) |
| **waveterm** | SQLite (JSON blobs per table) | Full object hierarchy + circular blockfile for terminal output | Yes (2MB circular buffer + xterm.js serialize addon) |
| **zed** | SQLite (sqlez, domain migrations) | Pane groups + items + working directories | No |
| **wezterm** | None (mux server keeps processes alive) | N/A | N/A |
| **helix** | None | N/A | N/A |

### Waveterm's scrollback persistence (most sophisticated)

1. PTY output appended to circular blockfile (2MB cap) in SQLite filestore
2. Periodically, `@xterm/addon-serialize` creates a terminal state snapshot → `cache:term:full`
3. On restore: load snapshot, replay only data written after the snapshot's `ptyoffset`
4. Result: fast restore without replaying entire history

### Yord's planned approach (from design doc)

M1: respawn command in saved CWD, no scrollback. This aligns with canopy's approach. Post-M1, waveterm's circular buffer + serialize addon pattern is the proven path for full scrollback restore.

---

## 4. Data Models & Storage

### waveterm (JSON blobs per table)

```
Client → Window → Workspace → Tab → Block (+ LayoutState)
```

Each type gets its own SQLite table with `oid`, `version`, `data` (JSON). Generic CRUD via Go generics. Optimistic concurrency via version field. Separate filestore DB for large binary data.

**vs Yord's EAV**: Waveterm's approach is simpler (one table per type, JSON blob) but less flexible. Yord's two-table EAV (`entities` + `components`) allows adding new component types without schema changes. Waveterm differentiates block types via `Meta.view` and `Meta.controller` keys — similar to Yord's component-based identity.

### zed's sqlez (domain-based migrations)

Each crate declares a `Domain` with `NAME` and ordered `MIGRATIONS` (SQL strings). Domains compose via tuple types. Migrations stored in a `migrations` table and verified at startup.

**Worth adopting**: The domain migration pattern is cleaner than a global migrations folder for modular codebases. Yord could have each system declare its own migrations.

### zellij (KDL files)

`GlobalLayoutManifest` → tabs → panes with geometry, commands, CWDs. Serialized to KDL text files. Not applicable to Yord (SQLite is better).

---

## 5. Layout Management

### Patterns across projects

All projects with layout management use the same fundamental pattern: **recursive split tree**.

| Project | Data Structure | Storage |
|---------|---------------|---------|
| **zed** | `Member::Pane \| Member::Axis { children, flexes }` | SQLite (relational, parent_group_id) |
| **helix** | `SlotMap<ViewId, Node>` with `Content::View \| Content::Container` | Not persisted |
| **zellij** | `TiledPaneLayout { children, split_direction }`, runtime: flat `BTreeMap<PaneId, Pane>` | KDL file |
| **waveterm** | `LayoutState` tree in Block metadata | SQLite JSON blob |
| **Yord (planned)** | `LayoutNode` recursive enum as `Layout` component on project entity | SQLite JSON blob in components table |

**Helix's SlotMap tree** is notable: nodes stored in a flat SlotMap with explicit parent pointers, O(1) access by ID, no pointer chasing. `recalculate()` walks from root dividing available Rect. This is the simplest correct implementation.

**Zellij's hybrid**: layout tree for structure, flat `BTreeMap` for runtime pane state. This is exactly Yord's "hybrid ECS + tree" — tree as a view, flat storage as source of truth.

---

## 6. Entity/Component Patterns

### Zed's GPUI entity model (inspiration — GPL-3.0)

Not a game ECS. It's a **typed entity store with reactive observation**:

- `EntityMap` uses `slotmap::SlotMap` for ID gen + `SecondaryMap` for `Box<dyn Any>` storage
- `Entity<T>` typed handles with ref counting (strong/weak)
- Access only through context: `entity.read(cx)` / `entity.update(cx, |state, cx| ...)`
- Two reactive mechanisms:
  - **Observe/Notify**: callback on state change (entity calls `cx.notify()`)
  - **Subscribe/Emit**: typed event channels (entity implements `EventEmitter<E>`)
- Views = entities that implement `Render`

**Relevance to Yord**: Yord's backend entity management could adopt a similar context-mediated access pattern. But since Yord's reactivity lives in Svelte (frontend), the Rust backend doesn't need GPUI's full reactive system. The key insight: typed handles + context-gated access prevent reentrancy bugs.

### Helix's document/view split

`DocumentId` and `ViewId` are separate. Multiple views can reference the same document. This confirms Yord's design where entities (documents) and layout leaves (views) are independent.

---

## 7. xterm.js Integration

### Addon usage across projects

| Addon | canopy | Clif-Code | waveterm | Yord (planned) |
|-------|--------|-----------|---------|----------------|
| `@xterm/addon-fit` | Yes | Yes | Yes (custom fork) | Yes |
| `@xterm/addon-search` | Yes | No | Yes | Yes |
| `@xterm/addon-web-links` | Yes | No | Yes | Future |
| `@xterm/addon-webgl` | No | No | Yes (primary) | Yes (with fallback) |
| `@xterm/addon-canvas` | No | No | Yes (fallback) | Yes (fallback) |
| `@xterm/addon-serialize` | No | No | Yes (for restore) | Future (post-M1) |

### Data format

- **canopy**: raw `Vec<u8>` via Channel — correct, handles non-UTF-8
- **Clif-Code**: `String::from_utf8_lossy` via events — lossy, may corrupt binary output
- **waveterm**: base64-encoded bytes via WebSocket — correct but overhead
- **tauri-plugin-pty**: raw `Vec<u8>` via invoke response — correct

**Yord should use raw bytes** (`Vec<u8>` / `Uint8Array`) via Tauri Channel. xterm.js handles raw bytes natively.

---

## 8. Key Takeaways for Yord M1

### Must-use

1. **`portable-pty` 0.9+** as cargo dependency (from crates.io, MIT). Provides cross-platform PTY with `setsid()`, `ChildKiller` separation, and ConPTY on Windows.

2. **Tauri v2 `Channel<T>`** for PTY output streaming (canopy's pattern). One `std::thread::spawn` reader per PTY, sending `Vec<u8>` chunks. Avoids Tokio thread pool starvation.

3. **Process group kill**: `kill(-pgid, SIGTERM)` → configurable timeout → `kill(-pgid, SIGKILL)`. Store both PID and PGID in `Running` component.

4. **`on_window_event(Destroyed)` cleanup** (canopy/Clif-Code pattern). Kill all PTY children when the window closes.

### Should-adopt

5. **Canopy's PATH augmentation** (`ensure_full_path()`). GUI apps on macOS don't inherit shell PATH — must manually add `~/.cargo/bin`, Homebrew paths, etc.

6. **Clif-Code's kill flag** (`Arc<AtomicBool>`) for clean PTY reader thread shutdown, rather than relying solely on EOF.

7. **Wezterm's `ChildKiller` separation**. Clone a killer handle for the tree manager thread while a waiter thread blocks on `child.wait()`.

8. **Foreground process tracking** via `tcgetpgrp(pty_fd)` + `sysinfo` crate (zed/wezterm pattern). Useful for showing what's running in terminal tab labels.

9. **`close_fds` crate** (zellij) for closing inherited file descriptors after fork.

### Future (post-M1)

10. **Scrollback persistence** via circular buffer + `@xterm/addon-serialize` snapshots (waveterm pattern).

11. **Domain-based migrations** (zed's sqlez pattern) if Yord's codebase becomes modular enough to warrant it.

12. **`alacritty_terminal` as library** (zed) for server-side terminal state when needed (e.g., reading terminal content for agent context).

---

## 9. Key Files Index

### Tauri PTY integration (start here)

| File | Why |
|------|-----|
| `refs/canopy/src-tauri/src/commands/terminal.rs` | Best PTY lifecycle reference: spawn, reader thread, Channel streaming, cleanup |
| `refs/canopy/src-tauri/src/lib.rs` | Tauri v2 app setup, plugin registration, window destroy cleanup |
| `refs/canopy/src/hooks/useTerminal.ts` | Tab management, session persistence, status tracking |
| `refs/canopy/src/services/terminal-service.ts` | IPC wrappers for PTY commands using Tauri Channel |
| `refs/tauri-plugin-pty/src/lib.rs` | Multi-session PTY plugin (216 lines), session map pattern |
| `refs/tauri-plugin-pty/api/index.ts` | node-pty-compatible TypeScript API |

### PTY internals

| File | Why |
|------|-----|
| `refs/wezterm/pty/src/lib.rs` | portable-pty API: all traits (PtySystem, MasterPty, Child, ChildKiller) |
| `refs/wezterm/pty/src/unix.rs` | Unix PTY: openpty, setsid, signal reset, fd cleanup |
| `refs/wezterm/pty/src/win/conpty.rs` | Windows ConPTY implementation |
| `refs/wezterm/pty/examples/bash.rs` | Minimal portable-pty usage example |

### Process management

| File | Why |
|------|-----|
| `refs/zed/crates/terminal/src/pty_info.rs` | Foreground process tracking via tcgetpgrp + sysinfo |
| `refs/wezterm/procinfo/src/linux.rs` | Full process tree from /proc |
| `refs/waveterm/pkg/shellexec/conninterface.go` | Graceful kill: SIGHUP → grace → SIGKILL |
| `refs/zellij/zellij-server/src/os_input_output_unix.rs` | Signal handling, PTY spawn, kill |

### Layout trees

| File | Why |
|------|-----|
| `refs/helix/helix-view/src/tree.rs` | Cleanest split tree: SlotMap-backed, parent pointers, recalculate |
| `refs/zed/crates/workspace/src/pane_group.rs` | Recursive enum: Member::Pane / Member::Axis |
| `refs/zellij/zellij-utils/src/input/layout.rs` | Layout definition with TiledPaneLayout tree |

### Session persistence

| File | Why |
|------|-----|
| `refs/waveterm/pkg/filestore/blockstore.go` | Circular buffer for terminal scrollback |
| `refs/zellij/zellij-utils/src/session_serialization.rs` | Session → KDL layout serialization |
| `refs/zed/crates/workspace/src/persistence.rs` | Workspace → SQLite serialization |
| `refs/zed/crates/sqlez/src/domain.rs` | Domain-based migration pattern |

### Data models

| File | Why |
|------|-----|
| `refs/waveterm/pkg/waveobj/wtype.go` | Object hierarchy: Client → Window → Workspace → Tab → Block |
| `refs/waveterm/pkg/wstore/wstore_dbops.go` | Generic SQLite CRUD for wave objects |
| `refs/zed/crates/gpui/src/app/entity_map.rs` | Entity store: SlotMap + ref counting + typed handles |

---

*Report based on shallow clones (--depth 1) as of 2026-03-21.*
