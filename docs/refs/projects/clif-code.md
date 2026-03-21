# Clif-Code

> Monorepo: ClifPad (Tauri 2 IDE with Monaco + terminal) and ClifCode (TUI AI coding agent). MIT. [GitHub](https://github.com/DLhugly/Clif-Code)

## Overview

Two products: ClifPad is a ~20MB native code editor/IDE (Tauri 2 + SolidJS + Monaco + portable-pty 0.8 + xterm.js v6). ClifCode is a standalone TUI AI coding agent (pure Rust, no runtime). ClifPad supports multi-window, file watching (notify 7), and AI agent integration.

Relevant to Yord: alternative PTY pattern with kill flag + window-scoped PTY tracking. 32KB read buffer (vs canopy's 4KB). Uses `emit_to()` events instead of Channel. File tree component with drag-and-drop.

## Architecture Decisions

**SolidJS over React/Svelte** -- ClifPad chose SolidJS for its 7KB runtime (vs React ~40KB+, Svelte 5 ~15-20KB). Compiles to direct DOM mutations with no virtual DOM; meaningful for a Tauri app where every KB counts toward binary size and RAM. State uses `createSignal` and `createStore` for fine-grained reactivity -- each store mutation only updates the exact DOM node that depends on it. For Yord: Svelte 5 runes (`$state`, `$derived`) provide equivalent fine-grained reactivity; the architectural decision validates the "no virtual DOM" approach for desktop IDE UIs.

**`emit_to()` events instead of `Channel<T>`** -- All backend-to-frontend streaming (PTY output, AI, file watcher, git state) uses `app.emit_to(&label, "event-name", payload)` scoped to specific window labels. When a window is destroyed, `on_window_event(WindowEvent::Destroyed)` cleans up for that label. `Channel<T>` is point-to-point and tied to a single invoke call, so it can't broadcast scoped events. Downside: every listener gets ALL events of that name for the window, requiring manual `session_id` filtering -- mild fan-out inefficiency. For Yord: `emit_to` for multi-window scoping, `Channel<T>` for single-session high-throughput flows (PTY I/O). Hybrid approach recommended.

**Kill flag pattern (`Arc<Mutex<bool>>`) for PTY cleanup** -- The PTY reader (`pty.rs`) uses `std::thread::spawn` with a blocking `reader.read()` loop. `portable-pty`'s reader is synchronous `Box<dyn Read>` -- can't select/poll. The kill flag is checked at loop top; `child.kill()` causes EOF/EIO, unblocking the read. Mutex poison recovery (`unwrap_or_else(|e| e.into_inner())`) prevents cascading panics. EIO handling is a platform workaround: macOS returns `ErrorKind::Other` (EIO), Linux returns `BrokenPipe` -- both treated as normal exit. For Yord: adopt directly; consider `tokio::sync::watch` channel instead of `Arc<Mutex<bool>>` for async idiomatic Rust, and wrap in a `PtyHandle` struct with `Drop` to auto-set the kill flag.

**32KB read buffer** -- `let mut buf = [0u8; 32768]` reduces IPC round-trips for bursty output (cat, htop). Tradeoff: single events can carry up to 32KB serialized as JSON via `String::from_utf8_lossy`. Claude Code integration uses 4KB buffer since its output is always human-readable text. For Yord: 32KB is reasonable for PTY; consider adaptive sizing or backpressure if memory is a concern.

**Multi-window PTY scoping by window label** -- `PtyState` stores sessions in `Mutex<HashMap<String, PtySession>>` keyed by UUID session ID; each `PtySession` stores a `window_label`. `kill_all_for_window()` iterates and kills matching sessions. Necessary because Tauri 2's `.manage()` registers global state, not per-window. For Yord: use `HashMap<EntityId, Vec<PtySession>>` keyed by entity rather than window label.

**Monaco over CodeMirror** -- Monaco provides 70+ language modes, IntelliSense, multi-cursor, minimap, diff view out of the box. CodeMirror 6 would be ~50KB vs Monaco's ~2-3MB tree-shaken. For an IDE targeting "VS Code-quality," Monaco is pragmatic. For Yord: irrelevant for M1 (workspace manager, not code editor); if editing is added later, CodeMirror 6 fits the small-binary philosophy better.

**State management: one store file per domain** -- Uses SolidJS primitives exclusively (no external library). `createSignal` for scalars, `createStore` for arrays/objects, `createMemo` for derived state, `createRoot` for module-scoped singletons. Each domain has its own store file (`fileStore.ts`, `gitStore.ts`, `terminalStore.ts`, etc.) exporting named functions and signals. Cross-store deps via direct imports. Settings persisted to `~/.clif/settings.json` via Tauri commands. For Yord: the pattern maps cleanly to Svelte 5 runes; Yord's SQLite ECS store replaces file-based persistence.

## Bugs Encountered & How Solved

- **macOS PTY EIO on child exit** (`pty.rs` lines 156-159): macOS returns `ErrorKind::Other` (EIO) when the child exits; Linux returns `BrokenPipe`. Both handled as normal exit conditions. Without this, spurious errors on every shell session end.
- **Mutex poison recovery** (`pty.rs` line 138): `kill_flag_clone.lock().unwrap_or_else(|e| e.into_inner())` recovers from poisoned mutex if the kill-flag-setting thread panicked. Prevents cascading panics in the reader thread.
- **npm/fnm environment conflicts** (`pty.rs` line 92): `cmd.env_remove("npm_config_prefix")` before spawning shells. Node version managers (fnm, nvm) set this, causing `npm` to fail or use the wrong node version in spawned terminals.
- **Claude Code nesting detection** (`pty.rs` line 90): `cmd.env_remove("CLAUDECODE")` prevents the `claude` CLI from detecting it's inside another Claude Code session (which makes it refuse to start).
- **Transient read errors** (`pty.rs` lines 161-162): On transient I/O errors (not EIO/BrokenPipe), the reader thread sleeps 10ms before retrying. Prevents tight-loop CPU burn on flaky PTY states.
- **Shell session auto-restart** (`TerminalPanel.tsx` lines 91-97): On PTY exit, auto-respawns after 500ms with `[Session ended -- restarting...]`. Guard (`inst.alive`) prevents respawn during intentional cleanup.
- **File change debouncing** (`fileStore.ts` lines 218-231): File watcher events debounced at 500ms before refreshing file tree and git status. Prevents dozens of redundant refreshes during `git checkout` or bulk operations.
- **Git state change debouncing** (`gitStore.ts` lines 228-233): `.git/` directory changes debounced at 300ms. Git operations like `commit` touch multiple files (index, HEAD, COMMIT_EDITMSG) in rapid succession.
- **Large file diff avoidance** (`fileStore.ts` lines 10, 121, 196, 492): Files over 512KB skip automatic diff view to prevent UI freezes from Monaco's diff computation. Shows a toast warning.

## Current Bugs & Tech Debt

- **Global static session maps for Claude Code / Agent** (`claude_code.rs`, `agent.rs`): Use `static SESSIONS: LazyLock<Arc<Mutex<HashMap<...>>>>` instead of Tauri-managed state. Not window-scoped -- sessions aren't cleaned up on window close, leaking processes.
- **No PTY session persistence / restore**: PTY sessions live in memory only. App crash = all terminal state lost. No scrollback persistence, no reconnection logic.
- **PTY output event fan-out**: Each `"pty-output"` event delivered to all N terminal tab listeners. With 32KB buffer and 10 tabs, up to 320KB of unnecessary JSON deserialization per read cycle.
- **`git_commit` always force-stages everything**: Runs `git add -A` before committing, ignoring the user's selective staging from the UI.
- **File tree hardcoded filters** (`fs.rs` lines 37-43): Hidden files, `node_modules`, `target`, `dist` unconditionally skipped. No `.gitignore` parsing or user configuration.
- **`read_file` fails on binary files** (`fs.rs` line 90): Uses `fs::read_to_string` with no binary detection or graceful fallback.
- **API keys stored in plaintext**: Written to `~/.clif/api_keys.json` as plaintext JSON. No OS keychain integration.
- **Silenced emit failures**: 40+ instances of `let _ = app.emit_to(...)`. Event delivery failures silently swallowed; difficult to debug.
- **Naive search** (`search.rs`): Linear scan, reads every file into memory, `to_lowercase().find()`. No regex, no ripgrep integration. 500-result cap.
- **Glob matching exponential worst case** (`search.rs` lines 142-169): Backtracking on `*` wildcards; technically unbounded time complexity.
- **Duplicated home directory helper**: `get_home_dir()` / `dirs_next()` implemented identically in `ai.rs` and `settings.rs`.
- **No SIGTERM/SIGINT before SIGKILL**: `session.child.kill()` sends SIGKILL immediately. Child processes (npm dev servers, etc.) don't get a chance to clean up.
- **Agent `change_directory` is a no-op** (`agent.rs` lines 360-368): Checks if path is a directory and returns success, but never updates `workspace_dir`. Subsequent tool calls use the original directory.
- **Claude Code `send` is a non-functional stub** (`claude_code.rs` line 186): Always returns an error because Claude Code runs in `-p` (print) mode which doesn't accept stdin.

## What To Learn

- **Kill flag pattern for PTY reader threads** (`pty.rs` lines 112-175): Create `Arc<Mutex<bool>>`, clone for reader thread; blocking read loop checks flag at top; `child.kill()` causes EOF/EIO to unblock; poison recovery via `unwrap_or_else`. For Yord: adopt directly, consider wrapping in a `PtyHandle` struct that implements `Drop`.
- **Window-scoped PTY management** (`pty.rs`, `lib.rs` lines 86-99): `PtyState` registered globally via `.manage()`, `pty_spawn` extracts window label, `emit_to(&label, ...)` for scoped events, `on_window_event(Destroyed)` triggers `kill_all_for_window(&label)`. For Yord: replace window labels with entity IDs; when a node is deleted from the tree, kill its sessions.
- **File watcher integration** (`services/file_watcher.rs`): `notify::RecommendedWatcher` with `mpsc::channel`; smart filtering (`.git/` mutations go to separate "git-changed" event channel for state tracking); ignore list; one watcher per window, old watcher dropped on restart. The `.git/` state-change detection (`is_git_state_change`) watches specific subpaths to detect commits, staging, and branch switches without running `git status`.
- **File tree with drag-and-drop** (`FileTree.tsx`, `FileTreeItem.tsx`): Items are `draggable` with custom MIME type; directories accept drops; self-drop and subtree-drop prevention; `fs::rename` on backend. Lazy-loaded children on first expansion; expand state in `createStore<Record<string, boolean>>` keyed by path. For Yord: replace filesystem paths with entity IDs; drag-and-drop becomes entity reparenting.
- **AI agent integration patterns** (`commands/agent.rs`, `stores/agentStore.ts`): Three tiers: simple SSE chat, Claude Code subprocess, built-in tool-calling agent. Agent supports up to 200 turns with automatic context compaction at ~80K tokens (3-tier: truncate large results > stub old results > drop old turns). Tool results capped at 12KB. Cancellation checked between tool executions and SSE chunks. For Yord: the context compaction strategy is worth studying if adding AI integration.

## Issues

- **PTY session leak on Claude Code / Agent window close**: Global `LazyLock` statics not cleaned up in `on_window_event(Destroyed)` handler. Closing a window with a running AI session leaves the child process running until app exit. **Yord M1 lesson**: ALL child processes must be in state that participates in cleanup. Use Tauri's `.manage()` for everything, or implement a central process registry with entity-scoped cleanup.
- **No SIGTERM before SIGKILL**: `child.kill()` sends SIGKILL. Dev servers don't close cleanly; ports remain bound, temp files not cleaned. **Yord M1**: send SIGTERM first, wait, then SIGKILL. Use process group management (`setsid` + `killpg`) to catch grandchild processes.
- **File watcher thread leak potential** (`file_watcher.rs`): Spawns a thread per watcher with no stored `JoinHandle`. If `notify` holds internal references keeping the channel alive, the thread could hang with no way to verify it exited. **Yord**: store `JoinHandle`s, join with timeout on cleanup. Consider `tokio::task::spawn_blocking` instead of raw `std::thread::spawn`.
- **Terminal output event fan-out**: N terminal tabs = N deserialization passes per PTY read. At 32KB * 10 tabs = 320KB of unnecessary JSON parsing per read cycle. **Yord**: use `Channel<T>` or session-specific event names (e.g., `pty-output-{session_id}`) to avoid fan-out.
- **`git commit` force-stages everything**: UI shows selective staging but commit ignores it. **Yord**: keep staging and committing as separate operations; never implicitly stage in the commit path.
- **No process group management**: `child.kill()` kills only the direct child (the shell), not descendants. Orphaned grandchild processes accumulate. **Yord M1 core feature**: use `setsid()` + `killpg()` for process groups; on Windows, use Job Objects.
- **Settings and API keys not per-project**: All config stored globally in `~/.clif/`. No per-project settings. **Yord advantage**: ECS/SQLite design naturally supports per-workspace settings as components on workspace entities.

## Key Files

| File | Why |
|------|-----|
| `refs/Clif-Code/clif-pad-ide/src-tauri/src/commands/pty.rs` | PTY with kill flag, window scoping, 32KB buffer |
| `refs/Clif-Code/clif-pad-ide/src-tauri/src/lib.rs` | Tauri setup with multi-window, menu building, dual state |
| `refs/Clif-Code/clif-pad-ide/src/components/terminal/TerminalPanel.tsx` | Terminal component with tab management, auto-restart |
| `refs/Clif-Code/clif-pad-ide/src/stores/terminalStore.ts` | SolidJS store for terminal tab state |
| `refs/Clif-Code/clif-pad-ide/src/lib/tauri.ts` | Complete IPC wrapper layer |
| `refs/Clif-Code/clif-code-tui/src/session.rs` | Session persistence and context compaction |
