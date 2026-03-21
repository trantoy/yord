# tauri-plugin-pty

> Tauri v2 plugin exposing PTY functionality with node-pty-compatible API. MIT. [GitHub](https://github.com/Tnze/tauri-plugin-pty)

## Overview

Tauri v2 plugin providing multi-session PTY management. Uses portable-pty 0.9, exposes spawn/read/write/resize/kill commands via Tauri plugin system. TypeScript API mirrors node-pty's `IPty` interface. Sessions tracked in `BTreeMap<u32, Arc<Session>>` behind `RwLock`.

Relevant to Yord: closest existing Tauri v2 PTY plugin. Multi-session pattern directly applicable.

## Architecture Decisions

- **`RwLock<BTreeMap<u32, Arc<Session>>>` for sessions.** RwLock allows concurrent reads (most operations only look up sessions). BTreeMap gives deterministic iteration order for debugging. Arc lets sessions outlive the map lock — clone the Arc, drop the read lock, then operate on the session's inner Mutexes independently.
- **Separate ChildKiller from Child.** portable-pty's `clone_killer()` returns an independent kill handle. Without this, `kill()` and `wait()` would deadlock: `exitstatus` holds the child lock waiting for exit, `kill` can't acquire the lock to send the signal.
- **Blocking read in async command.** portable-pty gives `Box<dyn Read + Send>`, not async. Wrapping would require `spawn_blocking` or an async adapter. The shortcut works at small scale but starves Tokio at `num_cpus` concurrent sessions.
- **No Tauri events/channels for streaming.** TS polls via tight `for(;;)` loop calling `invoke('plugin:pty|read')`. Simpler to implement, and IPC overhead is masked by the blocking read (no busy-spinning). But adds latency — each data chunk requires a full IPC round-trip including JSON serialization.
- **EventEmitter2 from node-pty.** Ported from Microsoft's node-pty (MIT) to maintain API compatibility. Avoids coupling consumer API to Tauri's event system.
- **Reader/writer extraction order matters.** `take_writer()` before `try_clone_reader()` — writer is moved out (destructive), reader is cloned. Master retains reference needed for `resize()`.

## Bugs Encountered & How Solved

- **`default.json~` backup file.** Example app has both `default.json` and `default.json~` in capabilities — author iterated on plugin deps (changed from `opener:default` to `os:default`).
- **`convertEol: true` in xterm.js.** Workaround for PTY output sending bare `\n` without `\r`, causing staircase-stepping. Enabled at the xterm layer.
- **Environment passthrough.** `cmd.env(k, v)` without `env_clear()` — correct for terminals (shells need inherited PATH, HOME), but no way to *remove* inherited env vars. `undefined` values in TS are filtered by Tauri serialization, never reaching Rust.

## Current Bugs & Tech Debt

- **Sessions never removed from map.** No `close`/`destroy`/`cleanup` command. PTY file descriptors, reader/writer handles remain allocated for app lifetime. Resource leak.
- **`dispose()`, `pause()`, `resume()` throw "not implemented".** `clear()` logs warning. No cleanup path — calling dispose doesn't kill process or stop read loop.
- **Blocking reads/waits on Tokio threads.** `read()` and `exitstatus` (`child.wait()`) block worker threads. At `num_cpus` concurrent sessions, entire Tokio runtime starved — all Tauri IPC frozen.
- **`"Unavaliable pid"` typo** in 4 error paths. Bare string errors with no context (which PID? how many sessions?). No structured error types — everything is `Result<T, String>`.
- **`kill()` silently swallows errors.** No `.catch()` on TS side. Signal parameter accepted but never passed to Rust. portable-pty's `ChildKiller::kill()` only sends default signal (SIGHUP Unix, TerminateProcess Windows).
- **`process` property never set.** `IPty.process` is always `undefined`. No Rust command to query process name.
- **Data always raw bytes** despite `encoding` parameter. `onData` fires `Uint8Array`, not string. 4096 bytes → ~16KB JSON (`[72,101,108,...]`), 4x overhead.
- **TODO: flow control params unused.** `handle_flow_control`, `flow_control_pause`, `flow_control_resume` all assigned to `_`.
- **Race: operations before spawn completes.** If `spawn` rejects, chained `.then()` on `write()`/`resize()` silently never execute.
- **Race: readData() vs wait().** Both start simultaneously. `onExit` can fire before output buffer drained.
- **CI only tests Windows.** Linux/macOS not in matrix.

## What To Learn

- **node-pty-compatible API surface.** Mirrors `IPty`, `IPtyForkOptions`, `IEvent<T>`, `IDisposable`. Yord should decide: maintain compatibility or build thinner interface.
- **Tauri 2 plugin pattern.** `build.rs` declares commands → auto-generates permission TOML. `init<R: Runtime>()` with `Builder::<R>::new("pty")`. Consumer adds `.plugin(init())` + `"pty:default"` capability. Canonical pattern.
- **Session handle pattern.** Opaque `u32` returned to frontend. Note: called `pid` in TS but it's NOT the OS PID — it's an internal counter.
- **Spawn sequence.** `native_pty_system()` → `openpty(PtySize)` → `take_writer()` → `try_clone_reader()` → `CommandBuilder::new(file)` → `slave.spawn_command(cmd)` → `child.clone_killer()`.

## Issues

- **Thread starvation (#1 risk).** Blocking read + blocking wait = `num_cpus` max concurrent sessions before deadlock.
- **No graceful shutdown.** No mechanism to kill all sessions on exit, clean up PTY fds, stop read loops.
- **No backpressure.** TS read loop fires as fast as data arrives. `pause()`/`resume()` unimplemented.
- **No process group management.** Only direct child killed.
- **4x JSON overhead on binary data.** `Vec<u8>` serialized as JSON number array.
- **`_init` promise masks spawn failures.** Rejected spawn → dead terminal with no feedback.
- **readData() vs wait() race.** `onExit` can fire before output fully drained.

## Key Files

| File | Why |
|------|-----|
| `refs/tauri-plugin-pty/src/lib.rs` | Entire Rust plugin (216 lines): session management, spawn/read/write/resize/kill/exitstatus commands |
| `refs/tauri-plugin-pty/api/index.ts` | TypeScript API (309 lines): TauriPty class with node-pty-compatible interface |
| `refs/tauri-plugin-pty/api/eventEmitter2.ts` | Minimal event emitter (48 lines) |
| `refs/tauri-plugin-pty/build.rs` | Tauri v2 plugin build script |
| `refs/tauri-plugin-pty/examples/vanilla/src/index.js` | Example app showing xterm.js integration |
| `refs/tauri-plugin-pty/examples/vanilla/src-tauri/capabilities/default.json` | Tauri v2 permissions config |
