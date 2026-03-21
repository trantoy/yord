# canopy

> Tauri v2 workspace manager for Claude Code sessions with terminal multiplexing. MIT. [GitHub](https://github.com/The-Banana-Standard/canopy)

## Overview

Desktop app for managing Claude Code CLI sessions and plain terminals across project workspaces. Tauri v2 + React 19 + TypeScript + portable-pty 0.8 + SQLite (via tauri-plugin-sql) + xterm.js v6. Includes daily planner, GitHub dashboard, skills manager, session persistence.

Relevant to Yord: closest stack match (Tauri 2 + PTY + SQLite + xterm.js). Uses `Channel<TerminalEvent>` for PTY streaming (same approach Yord plans). Tab session persistence via localStorage. PATH augmentation for macOS GUI apps.

## Architecture Decisions

- **React 19 with hooks-only state (no state library).** All state in custom hooks, threaded through `App.tsx` via prop drilling. Makes `App.tsx` a 565-line god component. **Yord should NOT follow** — use Svelte 5 stores.
- **tauri-plugin-sql (frontend SQL) instead of rusqlite.** All SQL in TypeScript. Simpler to iterate but no Rust-side validation, no background Rust DB tasks, ad-hoc migrations. **Yord should use rusqlite** — ECS needs Rust-side access.
- **localStorage for tab persistence (not SQLite).** Simple but fragile. **Yord should NOT follow** — store everything in SQLite.
- **`Channel<TerminalEvent>` for PTY streaming.** Per-terminal typed channel, no global event multiplexing. **Right pattern for Yord.**
- **Single `Mutex<HashMap<String, TerminalInstance>>` for all terminals.** Write can block on full PTY buffer, freezing all terminals. **Yord should use per-terminal state.**
- **`std::thread::spawn` for PTY readers (not tokio).** Correct — avoids blocking Tokio workers.
- **Synchronous Rust commands for filesystem I/O.** Fine for small workspaces, blocks IPC for large ones.

## Bugs Encountered & How Solved

- **macOS PATH problem.** GUI apps get minimal PATH. **Fix:** `ensure_full_path()` augments with ~10 directories + nvm/fnm globbing. Yord will hit this.
- **Initial command timing.** Command sent before prompt ready → silently dropped. **Fix:** `AtomicBool` output detection + 2000ms Claude settle time. Still a heuristic.
- **Closing tab in split mode jumped to Home.** **Fix:** Compute adjacent tab index.
- **Daily Planner losing text on tab switch.** **Fix:** `display: none` instead of conditional rendering.
- **File drops targeting wrong terminal in split.** **Fix:** `getBoundingClientRect()` hit-testing + 5s safety timeout.
- **Nested Claude Code errors.** **Fix:** Filter `CLAUDECODE*`/`CLAUDE_CODE*` env vars before spawn.

## Current Bugs & Tech Debt

- **No graceful shutdown.** `child.kill()` = SIGKILL only. No SIGTERM, no timeout.
- **No process group management.** Only direct child killed. Subprocesses orphaned.
- **Single Mutex contention.** All terminal ops serialize on one lock.
- **Ignored errors.** `let _ = on_event.send(...)`, `let _ = term.child.kill()`. No logging.
- **No resize debouncing.** ResizeObserver fires on every event.
- **No database migrations.** `CREATE TABLE IF NOT EXISTS` on every launch.
- **No scrollback buffer.** Frontend slow → output lost. No replay.
- **No Channel error recovery.** Broken channel → reader loops forever.
- **Dead code.** `project_path`, `is_claude_session`, `has_output` stored but never read.
- **curl for HTTP.** Runtime dep instead of Rust HTTP client.

## What To Learn

- **PTY lifecycle.** `openpty()` → `spawn_command()` → `drop(slave)` immediately → `try_clone_reader()` in `std::thread::spawn` → 4KB read loop → `Channel.send(Output)` → on EOF: remove from map, `child.wait()`, send `Exit`. Dropping slave critical for EOF detection.
- **`Channel<T>` usage.** Frontend creates channel, sets `onmessage`, passes to Rust. Per-terminal, typed, no multiplexing.
- **PATH augmentation.** `ensure_full_path()`: `~/.local/bin`, `~/.cargo/bin`, `/usr/local/bin`, `/opt/homebrew/{bin,sbin}`, `~/.nvm/versions/node/*/bin`, `~/.fnm/aliases/default/bin`. Each checked with `exists()`.
- **Window destroy cleanup.** `WindowEvent::Destroyed` → `try_state()` (no panic) → lock → `child.kill()` each → clear map.
- **Terminal status detection.** 6 statuses. "Waiting" via `onBell()` + 500-char ANSI-stripped buffer matched against prompt patterns. 5s cooldown. Triggers dock bounce, badge, toast, notification.
- **Env var filtering.** Filter `CLAUDECODE*`/`CLAUDE_CODE*` before spawn.
- **Security patterns.** Keyring allowlist, path traversal prevention, URL allowlist, automated security tests.

## Issues

- **No process tree tracking.** Yord's `kill(-pgid)` solves this.
- **No graceful shutdown.** Yord needs SIGTERM → timeout → SIGKILL.
- **Single-Mutex contention.** Yord should use per-entity state.
- **Lossy session restore.** Only tab metadata in localStorage. Yord's SQLite persistence is better.
- **Frontend/backend split.** SQLite in TS, PTY in Rust — no coordination. Yord's Rust-side ECS fixes this.
- **Reader thread leak on panic.** Yord should wrap in `catch_unwind`.
- **Channel error not checked.** Yord should check send result and clean up.

## Key Files

| File | Why |
|------|-----|
| `refs/canopy/src-tauri/src/commands/terminal.rs` | PTY lifecycle: spawn, reader thread, Channel streaming, cleanup |
| `refs/canopy/src-tauri/src/state.rs` | AppState with terminal instance map |
| `refs/canopy/src-tauri/src/lib.rs` | Tauri app setup, plugin registration, PTY cleanup on window destroy |
| `refs/canopy/src/hooks/useTerminal.ts` | Tab management, session persistence, status tracking |
| `refs/canopy/src/components/Terminal/TerminalView.tsx` | xterm.js integration, attention detection, session restore |
| `refs/canopy/src/services/terminal-service.ts` | IPC wrappers for PTY commands using Tauri Channel |
| `refs/canopy/src/services/database-service.ts` | SQLite schema and CRUD operations |
| `refs/canopy/src-tauri/src/commands/projects.rs` | Workspace scanning, tech stack detection |
