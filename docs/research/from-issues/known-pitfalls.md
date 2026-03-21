# Known Pitfalls — M1 Tree + Shell

> Based on analysis of reference project issue trackers. See [docs/refs/README.md](../../refs/README.md) for rules.

Organized by M1 area. Each pitfall lists the symptom, root cause, which projects hit it, the fix applied, and whether Yord's M1 spec already handles it.

---

## 1. PTY Spawning Pitfalls

### 1.1 Drop-based cleanup leaves orphaned children

**Symptom:** Closing a terminal leaves shell child processes running indefinitely. CPU and memory grow over time.

**Root cause:** Relying on the PTY master fd's `Drop` impl to signal child exit. `Drop` closes the fd, which may send SIGHUP — but children that ignore or catch SIGHUP survive. No explicit `kill()` or `waitpid()` is called.

**Refs:** canopy [#23](https://github.com/The-Banana-Standard/canopy/issues/23), zed [#9482](https://github.com/zed-industries/zed/issues/9482), [#34932](https://github.com/zed-industries/zed/issues/34932), [#47412](https://github.com/zed-industries/zed/issues/47412)

**Fix applied:** Explicit `kill(pid, SIGTERM)` + `waitpid()` before dropping PTY handle. Zed added `killpg()` for process group cleanup.

**Yord relevance:** Handled. M1 spec has explicit two-step kill (SIGHUP foreground group, then shell group, then SIGKILL fallback) plus `Drop` impl with 100ms graceful timeout as defense-in-depth.

### 1.2 portable-pty reader thread never returns on Windows

**Symptom:** Reader thread hangs after child process exits. PTY handle cannot be cleaned up. Accumulates threads over time.

**Root cause:** `portable-pty` on Windows (ConPTY) does not reliably report EOF when the child exits. The blocking `read()` never returns.

**Refs:** wezterm [#4718](https://github.com/wezterm/wezterm/issues/4718), [#463](https://github.com/wezterm/wezterm/issues/463), [#5107](https://github.com/wezterm/wezterm/issues/5107)

**Fix applied:** Kill flag (`Arc<AtomicBool>`) checked by reader loop. When kill is requested, reader breaks out even if `read()` would block.

**Yord relevance:** Handled. M1 spec includes kill flag + `ChildKiller` handle pattern. Test this on Windows CI specifically.

### 1.3 Deadlock at 4+ concurrent PTY instances

**Symptom:** Application freezes when spawning multiple PTY instances. Only reproducible with 4+ terminals open.

**Root cause:** Global mutex in PTY management code serializes all operations. When multiple PTYs spawn simultaneously, the mutex contention causes a deadlock or severe stall.

**Refs:** tauri-plugin-pty [#2](https://github.com/Tnze/tauri-plugin-pty/issues/2)

**Fix applied:** Move to per-PTY state management instead of a single global lock.

**Yord relevance:** Handled. M1 spec uses per-PTY actor pattern (one `tokio::spawn` task per PTY) specifically to avoid global mutex. A stuck PTY blocks only its own task.

### 1.4 Shell not found / shell detection failure

**Symptom:** PTY spawn fails entirely. Error: "Cannot find shell" or "SHELL variable: NotPresent".

**Root cause:** `$SHELL` is unset (containers, some Linux distros, `su` without `-l`), or points to a shell not on `$PATH` in the current context (e.g., nix environments).

**Refs:** zellij [#1722](https://github.com/zellij-org/zellij/issues/1722), [#2006](https://github.com/zellij-org/zellij/issues/2006), [#1943](https://github.com/zellij-org/zellij/issues/1943), [#3272](https://github.com/zellij-org/zellij/issues/3272)

**Fix applied:** Fallback chain: `$SHELL` -> `/etc/passwd` entry -> `/bin/sh`. Validate shell exists before spawn.

**Yord relevance:** Partially handled. M1 spec says "respect `$SHELL`, launch as interactive shell by default" but does not define a fallback chain when `$SHELL` is unset or invalid. **Add:** fallback to `/etc/passwd` -> `/bin/sh` (Unix) or `cmd.exe` (Windows) if `$SHELL` is missing.

### 1.5 EIO on PTY read misinterpreted as fatal error

**Symptom:** PTY read returns `EIO` or `BrokenPipe` after child exits. Application treats it as an error and shows crash UI instead of normal exit.

**Root cause:** On Linux/macOS, reading from a PTY master after the slave side closes produces `EIO` (macOS: `ErrorKind::Other`; Linux: `BrokenPipe`). This is normal EOF signaling, not an error.

**Refs:** waveterm [#630](https://github.com/wavetermdev/waveterm/issues/630), zellij [#979](https://github.com/zellij-org/zellij/issues/979)

**Fix applied:** Treat `EIO`/`BrokenPipe` on PTY read as EOF, not as error.

**Yord relevance:** Handled. M1 spec explicitly documents EIO handling: "macOS `ErrorKind::Other`, Linux `BrokenPipe` — both are normal EOF."

### 1.6 ConPTY injects unsolicited escape sequences

**Symptom:** On Windows, terminal output contains unexpected escape sequences not produced by the running program. Extra blank lines, corrupted prompts, mispositioned cursor.

**Root cause:** Windows ConPTY acts as a translation layer and injects its own CSI sequences (cursor repositioning, screen clearing) that terminal emulators don't expect.

**Refs:** wezterm [#4784](https://github.com/wezterm/wezterm/issues/4784), [#3531](https://github.com/wezterm/wezterm/issues/3531), [#3624](https://github.com/wezterm/wezterm/issues/3624), zed [#42345](https://github.com/zed-industries/zed/issues/42345), [#46514](https://github.com/zed-industries/zed/issues/46514)

**Fix applied:** Strip or reinterpret ConPTY-injected sequences. Some projects filter known ConPTY artifacts.

**Yord relevance:** Partially handled. M1 spec notes "ConPTY injects unsolicited escape sequences -- account for in filtering" but does not specify which sequences or how. **Add:** document specific ConPTY artifacts to filter (cursor save/restore, ED sequences) in `escape_filter.rs`.

### 1.7 File descriptor leak from PTY spawning

**Symptom:** "Too many open files" error after long-running sessions or many spawn/close cycles. Eventually PTY spawn fails.

**Root cause:** File descriptors inherited by forked child processes or not closed after PTY master/slave setup. Over hundreds of spawn cycles, fds accumulate.

**Refs:** wezterm [#2466](https://github.com/wezterm/wezterm/issues/2466), zellij [#796](https://github.com/zellij-org/zellij/issues/796), [#4496](https://github.com/zellij-org/zellij/issues/4496)

**Fix applied:** Use `close_fds` crate or `CLOEXEC` on all fds. Explicitly close slave fd after spawn. Audit fd inheritance.

**Yord relevance:** Handled. M1 spec includes "`close_fds` crate for fd hygiene after fork" and "drop slave immediately" after spawn.

### 1.8 portable-pty PATH corruption on Windows

**Symptom:** Commands that rely on custom PATH entries fail inside the PTY. Environment variables not properly expanded.

**Root cause:** `portable-pty` on Windows strips or reorders PATH entries, removing user-added directories.

**Refs:** wezterm [#4205](https://github.com/wezterm/wezterm/issues/4205), [#2980](https://github.com/wezterm/wezterm/issues/2980), [#6783](https://github.com/wezterm/wezterm/issues/6783)

**Fix applied:** Manually preserve and pass through the full PATH. Expand environment variables before passing to ConPTY.

**Yord relevance:** Partially handled. M1 spec mentions macOS PATH augmentation but not Windows PATH preservation. **Add:** test that Windows PTY inherits complete PATH, and expand `%VARIABLES%` before spawn.

---

## 2. Kill Propagation Edge Cases

### 2.1 Zombie processes from child process groups

**Symptom:** After closing a terminal, `ps` shows zombie or orphaned processes (especially language servers, build tools, node processes). Memory usage grows.

**Root cause:** Child process creates its own process group or forks a grandchild. Killing the shell PID does not reach the grandchild. `waitpid()` not called, leaving zombies.

**Refs:** helix [#4068](https://github.com/helix-editor/helix/issues/4068), zed [#46474](https://github.com/zed-industries/zed/issues/46474), [#45211](https://github.com/zed-industries/zed/issues/45211), [#47455](https://github.com/zed-industries/zed/issues/47455), [#48722](https://github.com/zed-industries/zed/issues/48722), zellij [#518](https://github.com/zellij-org/zellij/issues/518), [#1286](https://github.com/zellij-org/zellij/issues/1286), [#3024](https://github.com/zellij-org/zellij/issues/3024), [#4054](https://github.com/zellij-org/zellij/issues/4054), wezterm [#5479](https://github.com/wezterm/wezterm/issues/5479)

**Fix applied:** Use `killpg()` to kill entire process groups. Call `tcgetpgrp()` to find the foreground group first. Always `waitpid()` after kill. Zed uses SIGHUP -> wait -> SIGKILL sequence on the whole group.

**Yord relevance:** Handled. M1 spec has explicit kill sequence: `killpg(tcgetpgrp(pty_fd), SIGHUP)` -> poll `try_wait()` -> kill shell group -> SIGKILL. Also includes `setsid()` for process group isolation.

### 2.2 setsid() omission causes signal leakage

**Symptom:** Killing a terminal in a multiplexer/parent also kills sibling processes. Clipboard operations in nested sessions fail or corrupt state.

**Root cause:** Child process inherits parent's session and process group. Signals sent to the group propagate to siblings. Without `setsid()`, the PTY child shares the parent's controlling terminal.

**Refs:** helix [#5424](https://github.com/helix-editor/helix/issues/5424), [#13853](https://github.com/helix-editor/helix/issues/13853)

**Fix applied:** Call `setsid()` in the child before `exec()` to create a new session and process group.

**Yord relevance:** Handled. M1 spec explicitly includes "`setsid()` for new process group (Linux/macOS)" and `CREATE_NEW_PROCESS_GROUP` + Job Objects for Windows.

### 2.3 Orphaned processes on project switch / app quit

**Symptom:** Switching to a different project or quitting the app leaves processes running. Node.js, LSP servers, and build tools persist indefinitely.

**Root cause:** App quit bypasses destructors (Tauri's default quit behavior). Project switch does not enumerate and kill associated processes. No shutdown hook cleans up.

**Refs:** zed [#34932](https://github.com/zed-industries/zed/issues/34932), [#37458](https://github.com/zed-industries/zed/issues/37458), [#51155](https://github.com/zed-industries/zed/issues/51155), [#43864](https://github.com/zed-industries/zed/issues/43864)

**Fix applied:** Custom quit handler that iterates all managed processes and kills them. Store PID/PGID for cleanup. Stale PID sweep on startup.

**Yord relevance:** Handled. M1 spec includes custom quit handler + `RunEvent::ExitRequested`, `on_window_event(Destroyed)` with `try_state()`, and startup stale PID sweep.

### 2.4 Windows: Job Objects required for tree kill

**Symptom:** On Windows, killing a process only kills the immediate process, not its children. Build tools spawn dozens of child processes that survive.

**Root cause:** Windows has no process groups like Unix. `TerminateProcess()` only kills the target PID.

**Refs:** zed [#36934](https://github.com/zed-industries/zed/issues/36934), [#48722](https://github.com/zed-industries/zed/issues/48722)

**Fix applied:** Create a Win32 Job Object, assign the shell process to it, set `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`. When the job handle is closed, all processes in the job are terminated.

**Yord relevance:** Handled. M1 spec mentions "`CREATE_NEW_PROCESS_GROUP` + Job Objects (Windows)".

### 2.5 PR_SET_PDEATHSIG only works for direct children on Linux

**Symptom:** After a Yord crash, grandchild processes (e.g., `node` spawned by `npm run dev`) survive because they were not direct children.

**Root cause:** `PR_SET_PDEATHSIG` delivers the signal when the *direct parent thread* dies, not when the process group leader dies. Grandchildren never get the signal.

**Refs:** zellij [#1414](https://github.com/zellij-org/zellij/issues/1414), zed [#20223](https://github.com/zed-industries/zed/issues/20223)

**Fix applied:** Combine `PR_SET_PDEATHSIG` with process group kills and stale PID cleanup on restart.

**Yord relevance:** Handled. M1 spec includes `PR_SET_PDEATHSIG(SIGTERM)` AND startup stale PID sweep as complementary strategies.

### 2.6 SIGHUP kills tmux/screen sessions unintentionally

**Symptom:** Closing a terminal that was running tmux kills the tmux server, destroying all sessions.

**Root cause:** SIGHUP sent to the PTY process group propagates into tmux, which interprets it as a detach-and-die signal.

**Refs:** wezterm [#5101](https://github.com/wezterm/wezterm/issues/5101), zellij [#1747](https://github.com/zellij-org/zellij/issues/1747), [#1326](https://github.com/zellij-org/zellij/issues/1326)

**Fix applied:** Kill only the foreground process group via `tcgetpgrp()`, not the shell's entire session. Let the shell handle cleanup of background jobs.

**Yord relevance:** Handled. M1 spec's kill sequence starts with `killpg(tcgetpgrp(pty_fd), SIGHUP)` -- foreground group first, not the whole session.

---

## 3. Session Restore Failures

### 3.1 CWD no longer exists after restore

**Symptom:** Terminal opens to `$HOME` or fails to spawn. Previous working directory was on a deleted branch, temp dir, or unmounted volume.

**Root cause:** Session stores the CWD path at save time. By restore time, the directory may be gone (deleted git branch worktree, ephemeral container, unmounted drive).

**Refs:** zellij [#3374](https://github.com/zellij-org/zellij/issues/3374), waveterm [#1662](https://github.com/wavetermdev/waveterm/issues/1662), wezterm [#2092](https://github.com/wezterm/wezterm/issues/2092)

**Fix applied:** Validate CWD before spawn. Fallback chain: saved CWD -> parent project CWD -> `$HOME`.

**Yord relevance:** Handled. M1 spec Phase 3 explicitly validates `Pty.cwd` with fallback chain and warning badge.

### 3.2 Stale PIDs from previous crash matched to wrong process

**Symptom:** On startup, app thinks a previous terminal is still running and either skips respawn or tries to interact with an unrelated process.

**Root cause:** PIDs are recycled by the OS. `kill(pid, 0)` returns success because the PID was reused by a completely different process.

**Refs:** zed [#36934](https://github.com/zed-industries/zed/issues/36934), zellij [#3731](https://github.com/zellij-org/zellij/issues/3731), [#3918](https://github.com/zellij-org/zellij/issues/3918)

**Fix applied:** Store `(pid, started_at)` pairs and compare against `/proc/<pid>/stat` start time. If mismatch, treat as dead.

**Yord relevance:** Handled. M1 spec Phase 2 stores `(pid, started_at)` and validates against `/proc/<pid>/stat` start time. Also stores `last_clean_shutdown` timestamp for aggressive cleanup detection.

### 3.3 Version upgrade breaks session format

**Symptom:** After upgrading the app, previous sessions cannot be loaded. Session list shows sessions but attaching crashes or shows empty state.

**Root cause:** Session serialization format changes between versions with no migration path. Old data cannot be deserialized by new code.

**Refs:** zellij [#1255](https://github.com/zellij-org/zellij/issues/1255), [#3280](https://github.com/zellij-org/zellij/issues/3280), [#3371](https://github.com/zellij-org/zellij/issues/3371), [#4350](https://github.com/zellij-org/zellij/issues/4350), zed [#47543](https://github.com/zed-industries/zed/issues/47543)

**Fix applied:** Versioned schema with migration system. Zed checks DB version on startup; zellij had to add backward-compat handling.

**Yord relevance:** Handled. M1 spec uses SQLite with an append-only migration system and `_migrations` version table. Phase 1 runs `PRAGMA integrity_check` and backs up corrupt DBs.

### 3.4 Thundering herd on restore -- all terminals spawn at once

**Symptom:** App startup is slow or unresponsive. System load spikes. Some terminals fail to spawn due to resource exhaustion.

**Root cause:** All `RestoreOnStart` entities spawn simultaneously with no concurrency limit. This overwhelms fd limits, CPU, and memory.

**Refs:** zellij [#176](https://github.com/zellij-org/zellij/issues/176), zed [#43224](https://github.com/zed-industries/zed/issues/43224) (slow startup), [#49442](https://github.com/zed-industries/zed/issues/49442)

**Fix applied:** Bounded concurrency with priority ordering. Spawn visible/focused terminals first.

**Yord relevance:** Handled. M1 spec Phase 3 uses semaphore (4 concurrent spawns) with priority: focused -> visible -> background.

### 3.5 Double-spawn race during restore

**Symptom:** Two instances of the same shell appear. One may be invisible but consuming resources.

**Root cause:** User clicks "start" on a terminal while the restore system is also spawning it. No per-entity locking prevents concurrent spawns for the same entity.

**Refs:** zellij [#4058](https://github.com/zellij-org/zellij/issues/4058), [#4873](https://github.com/zellij-org/zellij/issues/4873)

**Fix applied:** Per-entity mutex/lock that prevents concurrent spawn operations.

**Yord relevance:** Handled. M1 spec Phase 3 includes "per-entity mutex to prevent double-spawn race."

### 3.6 Window position/geometry restore fails on Wayland

**Symptom:** Window opens at wrong position or size after restart. Sometimes opens off-screen.

**Root cause:** Wayland does not allow clients to set their own window position (by design). APIs that work on X11/macOS fail silently on Wayland.

**Refs:** zed [#15959](https://github.com/zed-industries/zed/issues/15959), [#43952](https://github.com/zed-industries/zed/issues/43952), wezterm [#5901](https://github.com/wezterm/wezterm/issues/5901)

**Fix applied:** Accept Wayland's positioning limitations. Restore size only; skip position restore on Wayland. Detect compositor type at startup.

**Yord relevance:** Partially handled. M1 spec uses `tauri-plugin-window-state` for geometry restore but does not mention Wayland limitations. **Add:** test on Wayland; expect position restore to be a no-op. Ensure size-only restore works.

### 3.7 Crash loop on restore -- fast-exit process auto-respawns

**Symptom:** A terminal that crashes immediately after spawn gets auto-restarted in an infinite loop, consuming CPU and filling logs.

**Root cause:** Restore system respawns failed terminals without checking how quickly they exited. A broken command (missing binary, bad env) exits instantly and is respawned.

**Refs:** zellij [#4413](https://github.com/zellij-org/zellij/issues/4413), [#3151](https://github.com/zellij-org/zellij/issues/3151)

**Fix applied:** If process exits within N seconds of spawn during restore, mark as `Crashed` and stop auto-restart.

**Yord relevance:** Handled. M1 spec Phase 3: "Crash loop prevention: If process exits within <2s of spawn during restore, don't auto-restart. Show as `Crashed`."

---

## 4. SQLite Issues

### 4.1 Database corruption from unclean shutdown

**Symptom:** App fails to start. Error: "Failed to load database file" or integrity check fails.

**Root cause:** WAL file left in inconsistent state after crash or forced kill. Especially common on Windows where file locks behave differently.

**Refs:** zed [#36839](https://github.com/zed-industries/zed/issues/36839), [#49241](https://github.com/zed-industries/zed/issues/49241), [#49585](https://github.com/zed-industries/zed/issues/49585)

**Fix applied:** `PRAGMA integrity_check` on startup. If corrupt, back up the DB file and start fresh. WAL checkpoint on clean shutdown.

**Yord relevance:** Handled. M1 spec Phase 1 runs `PRAGMA integrity_check`, copies corrupt DB to `.bak`, and starts fresh. Shutdown runs `PRAGMA wal_checkpoint(TRUNCATE)`.

### 4.2 Database version mismatch after downgrade

**Symptom:** App crashes on startup when opening a database that was migrated by a newer version. No error message -- just a hang or crash.

**Root cause:** Forward migrations are applied but there is no handling for downgrade. Newer schema features (columns, indexes) cause queries to fail.

**Refs:** zed [#47543](https://github.com/zed-industries/zed/issues/47543)

**Fix applied:** Check migration version on startup. If DB version is newer than app version, show a clear error message instead of crashing.

**Yord relevance:** Not explicitly handled. **Add:** on startup, compare `_migrations` max version against app's known max version. If DB is from a newer version, show error toast and optionally start fresh with backup.

### 4.3 Database constraint errors flood logs

**Symptom:** Log files grow to gigabytes with repeated SQLite constraint violation errors. Performance degrades.

**Root cause:** Duplicate insert attempts (e.g., inserting a component that already exists) without `INSERT OR REPLACE` or proper conflict resolution. Race between concurrent operations.

**Refs:** zed [#47109](https://github.com/zed-industries/zed/issues/47109)

**Fix applied:** Use `INSERT OR REPLACE` or `ON CONFLICT` clauses. Reduce log verbosity for expected constraint violations.

**Yord relevance:** Partially handled. M1 spec uses single-writer architecture which eliminates write races, but should still use `INSERT OR REPLACE` for idempotent component upserts. **Add:** use `ON CONFLICT` clause in component upserts.

### 4.4 WAL data race in SQLite < 3.51.3

**Symptom:** Intermittent database corruption under concurrent read/write workloads.

**Root cause:** Bug in SQLite WAL implementation fixed in 3.51.3. Concurrent readers and writers can race on WAL reset.

**Refs:** (discovered during Yord research, corroborated by SQLite changelog)

**Fix applied:** Upgrade to SQLite >= 3.51.3.

**Yord relevance:** Handled. M1 spec explicitly says "Verify bundled SQLite >= 3.51.3 (WAL-reset data race fix)."

---

## 5. xterm.js Rendering Bugs

### 5.1 WebGL context loss after sleep/screen lock

**Symptom:** Terminal content freezes for several minutes after macOS sleep or screen lock. Sometimes shows black rectangle until interaction.

**Root cause:** WebGL context is lost when GPU is suspended. xterm.js WebGL renderer does not automatically recover. The canvas becomes stale.

**Refs:** waveterm [#2895](https://github.com/wavetermdev/waveterm/issues/2895), zed [#33191](https://github.com/zed-industries/zed/issues/33191), wezterm [#4501](https://github.com/wezterm/wezterm/issues/4501)

**Fix applied:** Detect WebGL context loss and fall back to canvas renderer. Some projects auto-retry WebGL after a delay.

**Yord relevance:** Partially handled. M1 spec has renderer fallback chain "WebGL -> Canvas -> DOM" but does not mention runtime recovery from context loss. **Add:** listen for `webglcontextlost`/`webglcontextrestored` events on the xterm.js canvas; fall back to canvas renderer on loss.

### 5.2 Synchronized Output not supported -- TUI animations flicker

**Symptom:** TUI applications (htop, btop, etc.) flicker or scroll instead of animating in-place. Output appears to "tear."

**Root cause:** xterm.js does not support the synchronized output escape sequence (DCS `=1s` / `=2s`). Each write is rendered immediately, so partial frames are visible.

**Refs:** waveterm [#2787](https://github.com/wavetermdev/waveterm/issues/2787)

**Fix applied:** No upstream fix in xterm.js at time of writing. Workaround: buffer output and flush on a timer (which Yord's ring buffer + 4ms timer already does partially).

**Yord relevance:** Partially handled. M1 spec's ring buffer + 4ms timer flush provides some batching, which mitigates flickering. For full support, **monitor** xterm.js for synchronized output support and adopt when available.

### 5.3 Memory leak from high-throughput output

**Symptom:** Memory grows unboundedly when a terminal runs a high-output command like `yes` or `cat /dev/urandom`.

**Root cause:** xterm.js accumulates data faster than it can render. If the write path has no backpressure, the JS heap grows until the tab crashes.

**Refs:** zed [#23008](https://github.com/zed-industries/zed/issues/23008), zellij [#525](https://github.com/zellij-org/zellij/issues/525), [#803](https://github.com/zellij-org/zellij/issues/803)

**Fix applied:** Bounded buffer on the Rust side. Drop oldest data when buffer is full (ring buffer). Backpressure via pending-bytes cap.

**Yord relevance:** Handled. M1 spec uses 256KB ring buffer per PTY (overwrites oldest on overflow) and 10MB pending-bytes backpressure cap. xterm.js has its own scrollback limit (5,000 lines).

### 5.4 Resize causes rendering artifacts

**Symptom:** After resizing the window, terminal content is misaligned, lines overlap, or the prompt appears in the wrong position.

**Root cause:** Resize event timing mismatch between the xterm.js instance and the PTY. If resize is sent to the PTY before xterm.js has recalculated its dimensions (or vice versa), the rows/cols are out of sync.

**Refs:** zed [#42365](https://github.com/zed-industries/zed/issues/42365), zellij [#4389](https://github.com/zellij-org/zellij/issues/4389), [#2828](https://github.com/zellij-org/zellij/issues/2828)

**Fix applied:** Debounce resize events (50-100ms). Send fresh resize when a hidden terminal becomes visible. Use `@xterm/addon-fit`.

**Yord relevance:** Handled. M1 spec: "PTY resize: debounce 50-100ms, fresh resize when hidden terminal becomes visible." Uses `@xterm/addon-fit`.

---

## 6. Keyboard / Focus / IME

### 6.1 Ctrl+C/D/Z intercepted by app instead of sent to terminal

**Symptom:** Standard shell shortcuts do not work in the integrated terminal. `Ctrl+C` triggers an app action instead of sending SIGINT.

**Root cause:** App-level keybinding system captures the key event before it reaches the terminal. No distinction between "terminal focused" and "editor focused" context.

**Refs:** zellij [#3002](https://github.com/zellij-org/zellij/issues/3002), [#3890](https://github.com/zellij-org/zellij/issues/3890), zed [#30132](https://github.com/zed-industries/zed/issues/30132)

**Fix applied:** When terminal has focus, pass through `Ctrl+C/D/Z/L/R` to the PTY. Use only `Cmd`/`Super` for app shortcuts.

**Yord relevance:** Handled. M1 spec: "Global shortcuts use `Cmd`/`Super` -- never intercept `Ctrl+C/D/Z/L/R`."

### 6.2 IME composition events trigger premature actions

**Symptom:** Typing CJK characters or using IME sends partial/incorrect input to the terminal. Characters are doubled or wrong.

**Root cause:** Keydown handler fires during IME composition. The `isComposing` flag is not checked, so each composition step is treated as a final keystroke.

**Refs:** zed [#21042](https://github.com/zed-industries/zed/issues/21042), wezterm [#3837](https://github.com/wezterm/wezterm/issues/3837)

**Fix applied:** Check `event.isComposing` (or `keyCode === 229`) before processing keyboard input. Ignore events during composition.

**Yord relevance:** Handled. M1 spec: "Check `event.isComposing` before acting (IME safety)."

### 6.3 Characters missed when typing quickly

**Symptom:** Fast typing drops characters. Output appears truncated or garbled.

**Root cause:** PTY write path is not flushing fast enough, or input events are batched and some are lost. In some cases, the write channel has backpressure that drops input.

**Refs:** tauri-terminal [#5](https://github.com/marc2332/tauri-terminal/issues/5)

**Fix applied:** Ensure writes to PTY are unbuffered or flushed immediately. Do not apply backpressure to stdin writes (only to stdout reads).

**Yord relevance:** Partially handled. M1 spec's per-PTY actor receives `PtyCommand::Write` via `mpsc::Sender`, which should be fast. **Add:** ensure write path does not drop input data -- backpressure should only apply to output, never to user input.

### 6.4 Keyboard protocol mismatch breaks arrow keys in containers

**Symptom:** Arrow keys print escape sequences (`^[[A`, `^[[B`) instead of moving the cursor. Happens in devcontainers or SSH sessions.

**Root cause:** The PTY is spawned with TERM settings that don't match the remote environment's expectations. Kitty keyboard protocol or DECCKM mode conflicts.

**Refs:** zed [#48355](https://github.com/zed-industries/zed/issues/48355), wezterm [#510](https://github.com/wezterm/wezterm/issues/510), [#6994](https://github.com/wezterm/wezterm/issues/6994)

**Fix applied:** Set `TERM=xterm-256color` by default. Ensure escape sequence generation matches what xterm.js expects.

**Yord relevance:** Not explicitly addressed. **Add:** set `TERM=xterm-256color` in PTY environment. Document that Yord's terminal type matches xterm.js expectations.

---

## 7. Security (Escape Sequences)

### 7.1 Title report sequences enable command injection

**Symptom:** Malicious terminal output sets window title, then a title query response is injected into stdin, executing arbitrary commands.

**Root cause:** CSI 20t/21t sequences report the window title back to the application via the input stream. An attacker can set the title to a command, then trigger a title query.

**Refs:** (well-known terminal security issue, motivating escape filtering across wezterm, zed, zellij)

**Fix applied:** Strip CSI 20t/21t and all window manipulation CSI t sequences.

**Yord relevance:** Handled. M1 spec Layer 1 (Rust, `vte` crate): "Strip CSI 20t/21t (title reports, RCE vector), all window manipulation CSI t."

### 7.2 OSC 52 clipboard write without focus gating

**Symptom:** A background terminal silently writes to the system clipboard. User pastes unexpected/malicious content.

**Root cause:** OSC 52 clipboard write is processed regardless of whether the terminal is focused. Any background process can overwrite the clipboard.

**Refs:** waveterm [#2845](https://github.com/wavetermdev/waveterm/issues/2845), wezterm [#836](https://github.com/wezterm/wezterm/issues/836), [#1662](https://github.com/wezterm/wezterm/issues/1662), [#3848](https://github.com/wezterm/wezterm/issues/3848), [#4791](https://github.com/wezterm/wezterm/issues/4791)

**Fix applied:** Only process OSC 52 clipboard writes when the terminal pane is focused. Show a notification when clipboard is written.

**Yord relevance:** Handled. M1 spec Layer 2: "OSC 52 focus gating (only write clipboard when terminal focused) + user notification."

### 7.3 OSC 52 size limit bypass causes memory exhaustion

**Symptom:** A crafted escape sequence sends a massive base64 payload via OSC 52, consuming all available memory.

**Root cause:** No size limit on OSC 52 data. A malicious or buggy program can send megabytes of clipboard data.

**Refs:** zellij [#1709](https://github.com/zellij-org/zellij/issues/1709) (1024 byte cutoff was too low), wezterm (undocumented internal limit)

**Fix applied:** Enforce a size limit on OSC 52 payloads. Zellij originally capped at 1024 bytes but this broke legitimate use. A higher limit (75-128KB) is more practical.

**Yord relevance:** Handled. M1 spec: "Enforce OSC 52 size limits (75KB decoded, 128KB raw)."

### 7.4 Bracketed paste markers in pasted content

**Symptom:** Pasting text that contains embedded `\x1b[200~` / `\x1b[201~` sequences can break out of bracketed paste mode, allowing injected input to be interpreted as commands.

**Root cause:** Clipboard content is not sanitized before pasting. If the content includes bracketed paste markers, the terminal thinks the paste has ended and starts interpreting subsequent bytes as commands.

**Refs:** wezterm [#1296](https://github.com/wezterm/wezterm/issues/1296), [#6291](https://github.com/wezterm/wezterm/issues/6291)

**Fix applied:** Strip bracketed paste markers from clipboard content before sending to the PTY.

**Yord relevance:** Handled. M1 spec Layer 2: "Paste de-fanging (strip bracketed paste markers from clipboard content)."

---

## 8. Memory Leaks / Resource Management

### 8.1 Event listener leaks on frontend component destroy

**Symptom:** Memory grows steadily as terminals are opened and closed. Tauri event listeners accumulate. Eventually the app becomes sluggish or crashes.

**Root cause:** `listen()` handlers and Channel `onmessage` callbacks are not cleaned up when the Svelte component is destroyed. Each terminal creates listeners that persist after the terminal view is removed.

**Refs:** zed [#13133](https://github.com/zed-industries/zed/issues/13133) (known memory leak in channel cleanup), waveterm [#2731](https://github.com/wavetermdev/waveterm/issues/2731)

**Fix applied:** Clean up all `listen()` and Channel callbacks in component `onDestroy`. Explicitly delete `channel.onmessage`.

**Yord relevance:** Handled. M1 spec: "every `listen()` and Channel `onmessage` cleaned up on component destroy. Delete `channel.onmessage` explicitly (known memory leak #13133)."

### 8.2 Thread leak from PTY spawn/close cycles

**Symptom:** Thread count grows with each terminal open/close cycle. After hundreds of cycles, the process hits OS thread limits.

**Root cause:** Reader threads spawned per PTY are not joined after the PTY is closed. `std::thread::spawn` creates a detached thread that runs until the read call returns -- but on some platforms, the read never returns (see pitfall 1.2).

**Refs:** wezterm [#6116](https://github.com/wezterm/wezterm/issues/6116), zellij [#1948](https://github.com/zellij-org/zellij/issues/1948)

**Fix applied:** Kill flag to break reader loop. Join or abort reader thread on PTY close. Wrap reader in `catch_unwind` for panic safety.

**Yord relevance:** Handled. M1 spec: kill flag (`Arc<AtomicBool>`) for reader thread, `catch_unwind` wrapper, and actor self-removes from registry on exit.

### 8.3 GPU/VRAM memory leak from image/render cache

**Symptom:** VRAM usage grows over time. Eventually GPU runs out of memory, causing rendering artifacts or crashes.

**Root cause:** Rendered textures or image cache entries are not evicted when no longer visible. Each terminal's rendered content is cached but never freed.

**Refs:** wezterm [#4826](https://github.com/wezterm/wezterm/issues/4826), [#7400](https://github.com/wezterm/wezterm/issues/7400), zed [#27414](https://github.com/zed-industries/zed/issues/27414)

**Fix applied:** LRU cache for images. Evict textures when terminal is closed or hidden. Cap total VRAM budget.

**Yord relevance:** Not directly applicable to M1 (no image support), but relevant for xterm.js WebGL renderer. **Monitor:** xterm.js WebGL renderer memory usage in long-running sessions during integration testing.

### 8.4 File descriptor leak from /proc/pid/stat polling

**Symptom:** After running for many hours, the app accumulates thousands of open file descriptors to `/proc/*/stat`, eventually hitting the fd limit.

**Root cause:** Foreground process tracking polls `/proc/<pid>/stat` on a timer but does not close the file descriptors reliably.

**Refs:** zed [#49776](https://github.com/zed-industries/zed/issues/49776)

**Fix applied:** Ensure file handles are closed after each poll. Use RAII patterns (`File` struct with `Drop`).

**Yord relevance:** Relevant if M1 stretch goal (foreground process tracking via `tcgetpgrp` + `sysinfo`) is implemented. **Add:** if polling `/proc`, ensure fd is closed after each read. Use `sysinfo` crate's API which handles this internally rather than raw `/proc` reads.

### 8.5 Scrollback buffer unbounded growth

**Symptom:** Long-running terminals with high output consume hundreds of MB of memory for scrollback history.

**Root cause:** No cap on scrollback line count. Each terminal accumulates all output since spawn.

**Refs:** wezterm [#2453](https://github.com/wezterm/wezterm/issues/2453), [#3815](https://github.com/wezterm/wezterm/issues/3815), zellij [#1789](https://github.com/zellij-org/zellij/issues/1789), [#1625](https://github.com/zellij-org/zellij/issues/1625)

**Fix applied:** Cap scrollback lines (typically 3,000-10,000). Discard oldest lines when cap is reached.

**Yord relevance:** Handled. M1 spec: "Default scrollback: 5,000 lines." xterm.js enforces this on the frontend side.

### 8.6 Deadlock from blocking I/O on main/UI thread

**Symptom:** App freezes completely. UI is unresponsive. Must force-kill.

**Root cause:** Blocking operations (shell command, file I/O, network call) run on the main thread or block the Tokio runtime's thread pool. With enough blocked threads, no async task can make progress.

**Refs:** canopy [#15](https://github.com/The-Banana-Standard/canopy/issues/15), [#18](https://github.com/The-Banana-Standard/canopy/issues/18), [#26](https://github.com/The-Banana-Standard/canopy/issues/26), zed [#37280](https://github.com/zed-industries/zed/issues/37280), [#50151](https://github.com/zed-industries/zed/issues/50151), wezterm [#1860](https://github.com/wezterm/wezterm/issues/1860), [#7661](https://github.com/wezterm/wezterm/issues/7661)

**Fix applied:** Move all blocking I/O off the main thread. Use `std::thread::spawn` for blocking reads (not Tokio `spawn_blocking` if it would starve the pool). Timeouts on all external operations.

**Yord relevance:** Handled. M1 spec: "`std::thread::spawn` per PTY reader (not Tokio -- avoids thread pool starvation)." Dedicated `std::thread` for SQLite writes. Per-PTY actor tasks on Tokio runtime for non-blocking command processing.

---

## Summary of Gaps

Items where M1 spec does **not** fully address the pitfall:

| # | Pitfall | Action needed |
|---|---------|---------------|
| 1.4 | Shell not found | Add fallback chain: `$SHELL` -> `/etc/passwd` -> `/bin/sh` |
| 1.6 | ConPTY artifacts | Document specific ConPTY sequences to filter |
| 1.8 | Windows PATH corruption | Test Windows PTY PATH inheritance; expand `%VARS%` |
| 3.6 | Wayland geometry restore | Accept position restore as no-op on Wayland |
| 4.2 | DB version downgrade | Check DB version > app version on startup, show error |
| 4.3 | Constraint error spam | Use `ON CONFLICT` in component upserts |
| 5.1 | WebGL context loss | Handle `webglcontextlost` event, fall back to canvas |
| 6.3 | Input data dropped | Ensure write path never applies backpressure to user input |
| 6.4 | TERM variable | Set `TERM=xterm-256color` explicitly |
