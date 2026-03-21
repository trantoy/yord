# zellij

> Terminal workspace and multiplexer with plugin system (WASM). MIT. [GitHub](https://github.com/zellij-org/zellij)

## Overview

Terminal multiplexer with tiling/floating panes, tabs, sessions, and WASM plugin system. Client-server architecture over Unix domain sockets. 5 main crates: zellij-server (PTY, screen, plugins), zellij-client (input, IPC), zellij-utils (types, layout, serialization), zellij-tile (plugin API), zellij-tile-utils.

Relevant to Yord: session serialization/restore to KDL files. Raw `nix::pty::openpty()` with tokio `AsyncFd`. Layout tree structure (TiledPaneLayout recursive tree, but flat BTreeMap for runtime pane storage — same hybrid as Yord). Signal escalation: SIGHUP then SIGKILL. Per-thread architecture with crossbeam channels.

## Architecture Decisions

**Client-server over single process** — A long-lived server holds all PTYs alive; short-lived clients connect via Unix domain socket. Enables session persistence (detach/reattach), multi-client attach (each tracked by `ClientId: u16`), and crash isolation (client crash never kills processes). The `server_listener` thread accepts connections on `ZELLIJ_SOCK_DIR` and spawns a `server_router` per client.

**Raw `nix::openpty()` instead of portable-pty** — `os_input_output_unix.rs` calls `openpty()` directly for precise control over:
- `login_tty()` in `pre_exec()` to set the controlling terminal on the slave side
- `close_fds::close_open_fds(3, &[])` to prevent FD leaks from the server to children
- Termios inheritance via `tcgetattr(0)` passed to `openpty()`
- `O_NONBLOCK` set on master FD before handing to tokio's `AsyncFd`

Windows uses a separate `WindowsPtyBackend`. Going raw avoids a portability crate that adds complexity for the primary platform.

**Tokio `AsyncFd` for PTY reads** — `RawFdAsyncReader` wraps the master PTY FD with tokio's `AsyncFd`, using `epoll` for async notification without a thread per PTY. Lazily initialized: `AsyncFd` registration deferred to first `read()` because `spawn_terminal` runs outside the tokio runtime. Each terminal gets a `TerminalBytes` task sending bytes to Screen via `task::spawn_blocking`.

**Crossbeam channels between threads** — `zellij-utils/src/channels.rs` re-exports `crossbeam::channel`. Chosen for:
- Multi-producer multi-consumer: the `Bus` struct uses `crossbeam::Select` to drain from multiple receivers simultaneously
- Error context propagation: every message is `(T, ErrorContext)` so receivers know call-stack origin via `OPENCALLS` thread-local
- Bounded channels for backpressure: PTY-to-Screen uses `bounded(50)`, client-server buffer uses `bounded(5000)`; when full, the client is disconnected rather than OOM-killing the session

**KDL for session serialization (not JSON/SQLite)** — Session state serialized to KDL (same format as layouts/config). Benefits:
- Round-trip fidelity: serialized KDL is a valid layout file loadable directly for resurrection
- Human-editable: users can tweak saved layouts before resurrecting
- Already a dependency (KDL is the config format)

Cost: no random-access or querying. Acceptable because session state is written as periodic complete snapshots via `write_session_state_to_disk()`.

**Per-thread architecture (screen, pty, plugin, pty_writer, background)** — `init_session()` spawns 5 named threads, each with a typed instruction enum (`PtyInstruction`, `ScreenInstruction`, etc.) as its message protocol. Main loop is `loop { match bus.recv() { ... } }` with exhaustive matching for compile-time guarantees. `ThreadSenders` holds `Option<SenderWithContext<T>>` for cross-thread communication.

`pty_writer` is separate from `pty` because some programs deadlock if you write to STDIN while reading STDOUT (notably vim). The writer handles backpressure with per-terminal `VecDeque<PendingWrite>` and a 10MB cap.

**Recursive tree for layout config, flat `BTreeMap` for runtime** — Config side: `TiledPaneLayout` is a recursive tree with `children: Vec<TiledPaneLayout>`, natural for KDL parsing. Runtime side: `HashMap<PaneId, Box<dyn Pane>>` flat map for O(1) lookup, efficient pane creation/deletion, and compatibility with the cassowary constraint solver (operates on flat `Span` vectors). Bridge: `get_tiled_panes_layout_from_panegeoms()` reconstructs the tree from flat geometry data heuristically (can fail, returns `None`).

**WASM for plugins** — Uses `wasmi` (interpreter, not JIT). Provides sandboxing (explicit `PluginPermission` grants), language agnosticity (Rust/Go/any WASM target), hot-reloading, and crash isolation. Single `wasmi::Engine` per plugin thread, each plugin gets its own WASM instance.

## Bugs Encountered & How Solved

- **PTY spawn FD leaks** — `handle_openpty()` calls `login_tty(pid_secondary)` in `pre_exec()` followed by `close_open_fds(3, &[])` to prevent children from inheriting server FDs (including other PTY master sides). Without this, a child could write to another pane's PTY.
- **Signal shutdown hangs** — `handle_command_exit()` implements 3-attempt graceful shutdown: send SIGTERM up to 3 times with 10ms polling, then escalate to `child.kill()` (SIGKILL). Solves shells that trap SIGTERM and take time to clean up.
- **PTY write deadlocks (vim)** — Dedicated `pty_writer` thread with `PtyWriteInstruction::Write` messages and per-terminal queuing. When buffer exceeds `MAX_PENDING_BYTES` (10MB), the queue is dropped entirely rather than blocking.
- **Async reader panic on wrong thread** — `RawFdAsyncReader` uses two-phase initialization: stores the `File` in `pending` and promotes to `AsyncFd` on first `read()` call (inside tokio task), because `AsyncFd::new()` requires a live reactor but `spawn_terminal` runs on the plain PTY thread.
- **Terminal exit during shutdown (FIXME)** — `terminal_bytes.rs:81`: pane reader tasks fail to send `ScreenInstruction::Render` when Screen channel is already closed on Ctrl+Q. Silently ignored; FIXME notes the ideal fix is detecting shutdown state.
- **Client buffer overflow** — `ClientSender` buffer was 50, overflowed regularly. Increased to 5000. Comment notes this is arbitrary; proper fix would be a redraw-on-backpressure mechanism.
- **Pane resizer layout corruption (HACK)** — `pane_resizer.rs:134`: `is_layout_valid()` prevents cascading corruption when geometry is in a bad state (e.g., stacked panes too tall). If validation fails, resize is silently abandoned.
- **Resize batching** — `StartCachingResizes` / `ApplyCachedResizes` exist because Screen emits multiple resize instructions per operation, causing glitches in programs that debounce resize signals. Fix: cache all resizes, apply only the final size per pane.

## Current Bugs & Tech Debt

- **Session rename race** (`screen.rs:7630`) — Socket file and session info folder renamed non-atomically. Background jobs thread can re-create the old folder between rename and next write cycle.
- **Scroll region inefficiency** — Multiple TODOs in `grid.rs` (lines 1578, 1593, 1649, 1778, 1788, 1967, 1990): `update_all_lines()` called where only scroll region lines need updating. Every scroll operation re-renders the entire viewport.
- **Tab close race with pending state** (`screen.rs:2344`) — Closing a tab with pending layout application causes a silent `Tab with index X not found` error.
- **ModeInfo duplication** (`screen.rs:1216`) — Screen carries both `mode_info: BTreeMap<ClientId, ModeInfo>` and `default_mode_info: ModeInfo`, structural duplication making mode management error-prone.
- **Sixel cache clearing** (`sixel.rs:286`) — Entire image cache cleared on eviction rather than evicting individual entries.
- **Search whole_word_only unimplemented** (`search.rs:99`) — Field exists but matching logic not implemented.
- **Pane trait object indirection** — Panes are `Box<dyn Pane>`, requiring dynamic dispatch. FIXMEs at `terminal_pane.rs:124` and `pane_resizer.rs:23` note the design should be reworked.
- **Error context boilerplate** (`errors.rs:212`) — Manually implemented `From<&XxxInstruction>` impls; FIXME suggests using strum's `EnumDiscriminants` derive.
- **HTTP client timeout** (`background_jobs.rs:147`) — Plugin web requests have no timeout; hung requests leak tokio tasks.
- **Tabstop width assumption** (`grid.rs:27`) — Hardcoded `TABSTOP_WIDTH: usize = 8`; some terminals use 4.
- **Force render mystery** (`tiled_panes/mod.rs:256,434,467`) — "TODO: why do we need this?" on `set_force_render()` calls, suggesting render invalidation gaps.

## What To Learn

- **Session serialization/restore** — Background job triggers `SerializeLayoutForResurrection` every 60s. Screen collects `SessionLayoutMetadata` from all tabs/panes, produces KDL via `session_serialization::serialize_session_layout()`, writes to `ZELLIJ_SESSION_INFO_CACHE_DIR/<session>/layout`. Captures tab names, pane geometries, running commands, CWDs, plugin states, focus state. Dirty detection (`is_dirty()`) avoids redundant writes. Resurrection loads cached KDL as a normal layout. **Yord takeaway**: SQLite gives incremental updates KDL cannot, but periodic snapshots with dirty detection is solid. Consider SQLite for entity/component storage with periodic checkpoint for full tree state.
- **Signal escalation pattern** — Three-level API: `UnixPtyBackend` (kill->SIGHUP, force_kill->SIGKILL, send_sigint->SIGINT), `ServerOsApi` trait (wraps backend), `Pty` struct (close_pane, send_sigint_to_pane). Gap: `close_pane()` sends single SIGHUP with no timeout-based SIGKILL fallback. **Yord takeaway**: Implement proper escalation with kill state machine per process.
- **Typed instruction enums** — Each thread has a single enum as its message protocol (`PtyInstruction` 31 variants, `ScreenInstruction` ~100, `BackgroundJob` 15). Main loop is exhaustive `match` with compile-time guarantees. **Yord takeaway**: Maps well to Tauri commands + internal channels; each Yord manager could have its own typed instruction enum and message loop.
- **Layout tree to flat pane map hybrid** — Recursive `TiledPaneLayout` for config/serialization, flat `HashMap<PaneId, Box<dyn Pane>>` for runtime with `PaneGeom` as source of truth. Cassowary constraint solver operates on flat `Span` vectors. **Yord takeaway**: Validates Yord's ECS + tree approach. Avoid needing to reconstruct the tree from flat data; maintain both representations synchronized.
- **Background job with dual-frequency persistence** — `ReadAllSessionInfosOnMachine` loops every 1s for metadata, every 60s for full layout serialization. **Yord takeaway**: Use WAL mode with periodic checkpoints; frequent lightweight updates, infrequent full serialization.
- **Client buffering with backpressure** — `ClientSender` wraps `bounded(5000)` crossbeam channel with dedicated drain thread. On overflow: warn, propagate error, disconnect client, send `Exit(Disconnect)`. Prevents slow client from blocking all rendering. **Yord takeaway**: Use bounded channels for Rust-to-frontend event streams in Tauri; drop frames rather than blocking backend.

## Issues

- **No SIGKILL escalation on pane close** — `close_pane()` sends single SIGHUP and removes tracking. Processes ignoring SIGHUP (nohup, daemons) become orphans. **Yord**: implement kill state machine with SIGHUP -> poll (100ms intervals, 3s timeout) -> SIGKILL.
- **PTY writer buffer drop is silent data loss** — When `PendingWrite` exceeds 10MB, entire queue cleared silently. No user notification. **Yord**: notify UI when input is being dropped.
- **Session serialization not crash-safe** — `write_session_state_to_disk()` uses `File::create()` + `write!()` directly; mid-write crash truncates the file. No atomic write (write-to-temp + rename). **Yord**: SQLite WAL mode provides crash recovery natively.
- **Flat-to-tree layout reconstruction is lossy** — `get_tiled_panes_layout_from_panegeoms()` heuristically reconstructs tree from flat geometries. Complex nested splits may not reconstruct perfectly, breaking swap layouts. **Yord**: store tree structure directly in database.
- **Terminal ID counter never recycles** — `next_terminal_id_counter` is `AtomicU32`, monotonically increasing. IDs never reused. Unlikely to exhaust but architecturally inelegant. **Yord**: use recycling or UUIDs.
- **`handle_command_exit` has dead code** — Child exit handler calls `child.wait()` then `handle_command_exit` which calls `try_wait()` again. The `should_exit` signal handling code is unreachable after `wait()` succeeds.
- **Resize debounce crosses thread boundaries** — Resize caching state lives on `ServerOsInputOutput` behind a `Mutex`, crossing thread boundaries via shared mutable state rather than being contained within PtyWriter's own state.
- **No process group tracking** — Only individual child PIDs tracked (`id_to_child_pid: HashMap<u32, u32>`). Grandchild processes don't receive signals on pane close. **Yord**: use `setsid()` for process groups and `killpg()` for group signals.

## Key Files

| File | Why |
|------|-----|
| `refs/zellij/zellij-server/src/os_input_output_unix.rs` | PTY spawn, kill, resize, signal handling |
| `refs/zellij/zellij-server/src/terminal_bytes.rs` | Async read loop: PTY bytes → ScreenInstruction::PtyBytes |
| `refs/zellij/zellij-server/src/pty.rs` | Pty struct, PtyInstruction enum, spawn/close orchestration |
| `refs/zellij/zellij-server/src/lib.rs` | Server startup, thread spawning, SessionMetaData::drop cleanup |
| `refs/zellij/zellij-utils/src/session_serialization.rs` | Session → KDL layout serialization |
| `refs/zellij/zellij-server/src/session_layout_metadata.rs` | Runtime metadata collection for persistence |
| `refs/zellij/zellij-utils/src/input/layout.rs` | Layout, TiledPaneLayout, SplitDirection |
| `refs/zellij/zellij-utils/src/pane_size.rs` | PaneGeom, Dimension, Constraint |
