# tauri-terminal

> Minimal Tauri v1 terminal demo with single PTY and xterm.js. No license. [GitHub](https://github.com/marc2332/tauri-terminal)

## Overview

Proof-of-concept single-PTY terminal in Tauri v1. Uses portable-pty 0.8, requestAnimationFrame polling for PTY output. Entire backend is 125 lines, frontend is 62 lines. App exits when the shell exits.

Relevant to Yord: minimal reference for portable-pty + xterm.js integration. Tauri v1 (not v2), so IPC patterns don't directly apply. Primarily useful as a catalog of anti-patterns to avoid.

## Architecture Decisions

- **Single PTY, no multi-session.** `AppState` holds one `PtyPair`, one writer, one reader — no session ID, no HashMap. Demo simplicity.
- **requestAnimationFrame polling.** Frontend calls `invoke("async_read_from_pty")` in a rAF loop (~60fps). Avoids push complexity but wastes IPC round-trips when idle and adds latency when busy.
- **`AsyncMutex` for state.** Uses `tokio::sync::Mutex` (correct for async commands) but the actual I/O inside the lock (`fill_buf()`) is blocking, defeating the purpose.
- **`fill_buf()` for reading.** Returns whatever is in the buffer without separate allocation. But it's a blocking call — if no data, it blocks the Tokio worker thread.
- **`exit()` on child exit.** When shell dies, `std::process::exit()` kills the entire app. Skips all destructors, Drop impls, and Tauri shutdown hooks.
- **UTF-8 string conversion.** PTY output converted to `String` via `from_utf8()` for JSON serialization. Silently drops data on non-UTF-8 bytes or partial multi-byte sequences at buffer boundaries.

**Decisions Yord should NOT follow:** All of the above. Every choice optimizes for demo simplicity at the cost of correctness.

## Bugs Encountered & How Solved

- **Linux `spawn_command` EPERM.** `spawn_command()` returns "Operation not permitted (os error 1)" on Linux but the shell works anyway. Likely related to `TIOCSCTTY`/`setsid()` in portable-pty's child setup. Workaround: `.catch()` the error and continue. Yord should investigate the root cause rather than swallowing the error.
- **Platform-conditional shell.** `#[cfg(target_os = "windows")]` selects `powershell.exe` vs `bash` and sets `TERM` to `cygwin` vs `xterm-256color`. Avoids terminfo issues on Windows.
- **`PtySize` pixel dimensions zeroed.** `pixel_width: 0, pixel_height: 0` — avoids pixel-accurate sizing by relying on row/col counts only.

## Current Bugs & Tech Debt

- **No cleanup on exit.** `exit()` leaks master PTY fd, skips Tauri's `RunEvent::ExitRequested`, leaves shell subprocesses as orphans parented to PID 1.
- **Blocking `fill_buf()` in async context.** Can exhaust Tokio's thread pool with multiple concurrent PTYs. One blocked worker per PTY.
- **UTF-8 boundary corruption.** `fill_buf()` can split multi-byte UTF-8 characters across reads. First read fails `from_utf8()`, partial bytes consumed and lost. Characters silently dropped.
- **No resize debouncing.** `addEventListener("resize", fitTerminal)` fires dozens of times during drag-resize, causing mutex contention and unnecessary `TIOCSWINSZ` ioctls.
- **No error handling.** All errors mapped to `()` or `String`. Fire-and-forget `invoke()` for writes/resizes. If PTY dies, frontend shows frozen terminal with no indication.
- **Tauri 1.x only.** Won't compile against Tauri 2 without migration. Uses v1 `allowlist` config schema.
- **CSP disabled.** `"csp": null` in config. Terminal rendering arbitrary PTY output with no CSP is a security risk.

## What To Learn

**Minimal integration topology (valid, but replace every piece for production):**
1. PTY creation: `native_pty_system().openpty()` → `try_clone_reader()` + `take_writer()`
2. Shell spawn: `slave.spawn_command(cmd)` with platform-conditional shell + `$TERM`
3. Tauri state: `Arc<AsyncMutex<T>>` in managed state, `State<'_, AppState>` in commands
4. xterm.js: `Terminal` + `FitAddon`, `term.onData()` for input, `term.write()` for output
5. Resize: `fitAddon.fit()` → IPC → `master.resize()`

**What NOT to do (anti-pattern catalog):**
- Poll-based reading → use Tauri Channels
- Blocking I/O in async commands → use `spawn_blocking` or dedicated thread
- `exit()` on child exit → graceful shutdown with signal propagation
- UTF-8 string serialization → binary Channels (`Vec<u8>` → `ArrayBuffer`)
- Single global PTY → session registry from day one
- No error types → `thiserror` enum with structured errors
- No process groups → `process_group(0)` + `kill(-pgid)`

## Issues

- **60 useless IPC calls/sec per terminal** when idle due to rAF polling. With 10 terminals: 600 calls/sec doing nothing. Channels eliminate this.
- **Tokio thread pool exhaustion** from blocking reads. N PTYs = N blocked workers. Default pool is 4-8 threads.
- **No session persistence.** Terminal state lost entirely on exit. No CWD, no scrollback, no env saved.
- **Orphan processes.** No `process_group(0)`, no group kill. Shell subprocesses survive app exit.
- **Frozen terminal on PTY death.** Frontend has no error detection — user sees a dead terminal with no feedback.

## Key Files

| File | Why |
|------|-----|
| `refs/tauri-terminal/src-tauri/src/main.rs` | Entire Rust backend (125 lines) |
| `refs/tauri-terminal/src/main.ts` | Entire frontend (62 lines) |
