# zed

> High-performance code editor built in Rust with GPUI framework. GPL-3.0 (inspiration only). [GitHub](https://github.com/zed-industries/zed)

## Overview

224-crate Rust monorepo. Custom GPUI framework with built-in entity system and GPU rendering. Uses `alacritty_terminal` as a library for integrated terminal. Custom `sqlez` SQLite wrapper with domain-based migrations.

Relevant to Yord: GPUI entity model (typed entity store with reactive observation — EntityMap via SlotMap, Entity<T> handles, read(cx)/update(cx) access, observe/subscribe reactivity). Two-step kill: `killpg()` foreground process group then shell. Foreground process tracking via `tcgetpgrp()` + sysinfo. Domain-based SQLite migrations. Pane group as recursive enum.

## Architecture Decisions

**Custom GPUI instead of existing UI framework**
Needed a framework where every piece of UI state is an entity owned by a single `App` context, with all rendering on one foreground thread. This centralized ownership enables:
- Deterministic effect scheduling (notify/emit queues flushed in order)
- Leak detection in tests (track every handle creation/release with backtraces)
- Window invalidation driven by entity access tracking (framework records which entities a window reads during render, invalidates only those windows when an entity notifies)

No existing Rust UI framework provided this combination.

**SlotMap + SecondaryMap entity model with typed Entity\<T\> handles**
`EntityRefCounts` uses `SlotMap<EntityId, AtomicUsize>` for ref counts; `EntityMap` stores data in `SecondaryMap<EntityId, Box<dyn Any>>`. Separation lets ref counts live behind `Arc<RwLock<...>>` shared across threads while entity data stays single-threaded. `Entity<T>` wraps `AnyEntity` with `PhantomData<fn(T) -> T>` (invariant in T) for compile-time type safety over type-erased storage; `downcast_ref`/`downcast_mut` happens inside `read()`/`lease()`.

**Context-gated access (read/update) to prevent reentrancy**
Entity state is owned by `App`, not the handle. `entity.update(cx, |state, cx| ...)` leases the entity (physically removes it from `SecondaryMap` onto the stack). If the callback re-enters the same entity, `double_lease_panic` fires with a typed error message. The lease pattern also means no observer can see partial state during an update.

**Dual reactive mechanisms: observe/notify vs subscribe/emit**
- `observe`/`notify`: pull-based "something changed," no payload. Observers read new state themselves. Used for re-renders and window invalidation.
- `subscribe`/`emit`: push-based "something happened," typed event payload (`impl EventEmitter<MyEvent>`). Used for semantic events (CloseTerminal, TitleChanged). Dispatched via `pending_effects` queue by `TypeId`.

Having both avoids forcing every state change to allocate an event object while still allowing rich payloads when needed.

**alacritty_terminal as library instead of portable-pty**
`alacritty_terminal` provides a full terminal emulator (`Term<T>`, EventLoop, grid/cell model, selection, search, vi mode), not just a PTY fd. `portable-pty` only gives a PTY descriptor — you still need the entire VT100/xterm state machine. Tradeoff: coupling to alacritty's internal APIs (see `unsafe append_text_to_term` bug below).

**Two-step killpg() for terminal processes**
1. `killpg(tcgetpgrp(pty_fd), SIGKILL)` — kills the foreground process group (e.g., `cargo build` and all children). The shell survives because it's in a different process group.
2. `kill_child_process()` — kills the shell so the terminal can fully exit.

On `Drop`: send `Msg::Shutdown` to the event loop, wait 100ms, then force-kill as fallback.

**Custom sqlez instead of rusqlite/sqlx**
Wraps `libsqlite3-sys` directly (raw C FFI). Provides:
- Domain-based migrations (not supported by rusqlite/sqlx out of the box)
- `ThreadSafeConnection`: per-thread read connections via `ThreadLocal`, serialized writes through a single background thread channel
- Typed statement bindings (`Bind`/`Column` traits with `StaticColumnCount`)
- Savepoint-based nested transactions
- Eager execution for migrations (`sqlite3_exec` for multi-statement DDL)
- Fallback to in-memory DB if file can't be opened

**Domain-based migrations**
`Domain` trait: `NAME: &str` + `MIGRATIONS: &[&str]`. Each domain owns its migration sequence, stored in a `migrations` table as `(domain, step, migration)`. Benefits:
- Independent evolution per domain sharing a SQLite file
- Change detection: stored migration text compared to code on startup; mismatch panics with a diff unless `should_allow_migration_change` returns true
- Composability: `Migrator` implemented for tuples of up to 5 domains
- Retry logic (up to 10 times) for concurrent process scenarios; foreign keys disabled during migration

**PaneGroup as recursive enum over tree data structure**
`Member::Pane(Entity<Pane>)` vs `Member::Axis(PaneAxis)` where `PaneAxis` contains `Vec<Member>`. Advantages:
- Natural pattern matching for every operation (split, remove, render, serialize)
- Flexible n-ary splits (not limited to binary)
- Flex-based sizing with `Arc<Mutex<Vec<f32>>>` shared with the element system for drag-resize
- Serialization mirrors structure (`SerializedPaneGroup` → SQLite adjacency-list with `parent_group_id` + `position`)

## Bugs Encountered & How Solved

- **Entity double-lease panic**: `lease`/`end_lease` in `entity_map.rs` catches reentrancy where an update callback tries to update the same entity. `Lease`'s `Drop` impl panics if the lease isn't properly ended (unless already panicking). Test `test_entity_map_weak_upgrade_before_cleanup` verifies weak handles can't upgrade after strong handle drop but before `take_dropped` cleanup — prevents use-after-free of entity slots.

- **Unsafe `append_text_to_term`**: After a task finishes, Zed appends a summary to the terminal using alacritty_terminal internals in unsupported ways. Workaround: manually call `term.newline()` and reset `cursor.point.column = Column(0)` between lines. The code comment explicitly states "there could be more consequences."

- **PTY signal handling and thread affinity**: On Unix, PTY creation must happen on the foreground thread because `fork()`/`exec()` inherits signal handlers. Running it on a background thread causes signal issues. Solution: `cx.spawn()` on non-Windows, `cx.background_spawn()` on Windows (no `fork()`).

- **Terminal event batching**: Without batching, high-throughput output (`cat /dev/urandom`) floods the UI thread. Solution: 4ms timer coalescing events, or flush at 100 events, whichever comes first.

- **Windows `GetProcessId` returning zero**: A zero PID from `GetProcessId` causes stack overflow in sysinfo's process traversal. Fix: fall back to child PID; if that's also zero, return `None`.

- **Terminal Drop graceful shutdown**: `Drop` sends `Msg::Shutdown` to the event loop, waits 100ms via detached background spawn, then `kill_child_process()`. The detached spawn ensures the kill fires even during app shutdown cleanup.

## Current Bugs & Tech Debt

**Terminal:**
- Unbounded channel between alacritty event loop and Zed could grow without bound (TODO at terminal.rs:573)
- `TerminalContent` struct unnecessarily public (TODO at terminal.rs:787)
- Logic in the render path that should be extracted (TODO at terminal_view.rs:1197)
- Wrapper div needed to prevent `TerminalElement` from stealing events — workaround for event propagation bug (TODO at terminal_view.rs:1254)
- DANGER: spawned I/O thread from `event_loop.spawn()` is not joined on drop, could leak threads (terminal.rs:602)

**GPUI:**
- Window bounds bug: opening a window with the currently active window on the stack erroneously falls back to `None` (TODO/BUG at window.rs:1051)
- Tooltip occlusion bug: absolute bounds used during prepaint, so tooltip doesn't know if its hitbox is occluded (TODO at div.rs:3045)
- Window ordering: `next_window`/`previous_window` use `HashMap::keys()` which is non-deterministic (TODO at app.rs:317, 342)

**Workspace:**
- HACK: empty string child needed to force drop target to appear despite `min_w_6()` — layout engine quirk (pane.rs:3555, 3601)
- Two `Option<T>` settings that are never `None`, should be plain `T` (workspace_settings.rs:47, 55)

## What To Learn

- **Entity\<T\> handle pattern with ref counting**: Three-tier ownership — `EntityRefCounts` in `Arc<RwLock<...>>` (thread-shared atomic counts), `Entity<T>` handles (clone increments, drop decrements, zero pushes to `dropped_entity_ids`), `EntityMap` (single-threaded data, periodically calls `take_dropped()`). The lease pattern prevents reentrancy without runtime borrow checking overhead. Yord adaptation: SQLite owns entities so no ref counting needed, but a `HashSet<EntityId>` of currently-updating entities can replicate the lease safety for in-memory caches.

- **Context-mediated state access**: `Context<'a, T>` wraps `&'a mut App` + `WeakEntity<T>`. Provides scoped methods: `cx.notify()`, `cx.emit(event)`, `cx.spawn(async |weak_self, cx| ...)`, `cx.observe(other, ...)`, `cx.subscribe(other, ...)`. Yord adaptation: create a `NodeContext` wrapping the Tauri `AppHandle` + current node ID for similar scoped access.

- **Two-step kill propagation**: `tcgetpgrp(pty_fd)` → foreground group → `killpg(pgid, SIGKILL)`, then `pty.child().id()` → shell → `sysinfo process.kill()`. Yord adaptation: identical pattern. For session restore, persist shell command but not the foreground process (can't be restored anyway).

- **Foreground process tracking**: `tcgetpgrp(pty_fd)` for PID, `sysinfo::System::refresh_processes_specifics` for name/cwd/argv, cached in `RwLock<Option<ProcessInfo>>`. Falls back to shell PID if `tcgetpgrp` returns -1. Yord adaptation: use for tab titles showing current command. Consider polling less aggressively than Zed (every terminal wakeup event).

- **Domain-based SQLite migrations**: Run eagerly (not prepared) for multi-statement DDL; stored in `migrations` table and compared on startup; wrapped in savepoints; foreign keys disabled during migration. `ThreadSafeConnection` provides per-thread reads + serialized background writes. Yord adaptation: domain-per-component-type migration strategy — each component type (Shell, TreeNode, TerminalState) as a domain with its own history.

- **PaneGroup recursive serialization**: In-memory tree serializes to adjacency-list in SQLite (`pane_groups` table with `group_id, parent_group_id, position, axis, flexes`; `center_panes` table with `pane_id, parent_group_id, position`). Flexes stored as JSON text, validated on load — count mismatch or sum deviation resets to uniform `1.0`. Groups with no children are filtered out; single-child groups are collapsed. Yord adaptation: `parent_id` + `position` columns map directly to the planned EAV schema.

## Issues

- **Entity identity**: Zed's `Entity<T>` is typed and ref-counted in memory. Yord's SQLite-backed entities won't need ref counting (DB owns everything), but a `NodeHandle<T>` wrapping an entity ID is worth adopting for the frontend for type-safe access through Tauri commands.

- **Lease pattern for update safety**: Worth adopting for any in-memory caches. In the Rust backend, SQLite transactions naturally provide isolation.

- **Observer vs subscriber split**: Maps to Yord's architecture — "notify" for reactive UI updates (node changed, re-render tree), "emit" for semantic events (terminal exited, shell error). Use Tauri's `emit_to` for targeted notifications vs `emit` for broadcast.

- **Process group kill**: Critical for M1 terminal. Two-step kill (foreground group then shell) is the correct Unix approach. Implement graceful shutdown with timeout on terminal close.

- **Recursive tree as enum for view layer**: Yord's tree is a flat ECS store with `parent_id` relationships, but rendering needs an in-memory tree. `Member::Pane | Member::Axis` is a clean pattern for this view layer, derived from the flat store.

- **Migration strategy**: Adopt domain-based migrations from day one. With EAV, new component types can be added without migrations, but structural changes need a migration system. The `should_allow_migration_change` hook is valuable during rapid prototyping while enforcing immutability in production.

- **Write serialization**: Use `ThreadSafeConnection` pattern — read from any thread, write from one background thread. Use WAL mode (`PRAGMA journal_mode=WAL`) as Zed does.

## Key Files

| File | Why |
|------|-----|
| `refs/zed/crates/gpui/src/app/entity_map.rs` | Entity store: SlotMap + ref counting + typed handles |
| `refs/zed/crates/gpui/src/app/context.rs` | Context methods: observe, subscribe, notify, emit |
| `refs/zed/crates/gpui/src/_ownership_and_data_flow.rs` | Reactive system documentation |
| `refs/zed/crates/terminal/src/terminal.rs` | PTY integration via alacritty_terminal |
| `refs/zed/crates/terminal/src/pty_info.rs` | Foreground process tracking via tcgetpgrp + sysinfo |
| `refs/zed/crates/workspace/src/pane_group.rs` | Recursive enum: Member::Pane / Member::Axis |
| `refs/zed/crates/workspace/src/persistence.rs` | Workspace → SQLite serialization |
| `refs/zed/crates/sqlez/src/domain.rs` | Domain-based migration pattern |
