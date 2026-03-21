# M1 Spec Validation Against Issue Findings

> Based on analysis of reference project issue trackers. See [docs/refs/README.md](../../refs/README.md) for rules.

Cross-references 592 relevant issues from `raw-findings.md` against the 8 major design decisions in the [M1 spec](../../superpowers/specs/2026-03-21-m1-tree-shell-design.md).

---

### 1. Per-PTY Actor Pattern

**Spec reference:** Section 5.4

**Validations** (issues confirming our approach is correct):
- wezterm [#7661](https://github.com/wezterm/wezterm/issues/7661) — tmux CC domain detach deadlocks entire GUI when last window is closed. Demonstrates exactly the global-mutex failure mode our per-PTY actor pattern avoids: one stuck domain blocks the entire application.
- zed [#211](https://github.com/zed-industries/zed/issues/211) — Deadlock when splitting panes with a shared buffer. Shared mutable state across panes causes deadlocks; per-actor isolation prevents this class of bug.
- wezterm [#1860](https://github.com/wezterm/wezterm/issues/1860) — Deadlock on ssh_domain connect when stdout is printed from remote `.bashrc`. A blocking read on one connection froze the entire connection system. Per-actor pattern isolates blocking reads to a single task.
- zed [#50151](https://github.com/zed-industries/zed/issues/50151) — UI deadlock in `window_did_change_key_status` during window activation (macOS, multi-window). Shared lock contention in UI thread causes full freeze; validates the "no global mutex" rationale.
- helix [#12316](https://github.com/helix-editor/helix/issues/12316) — Possible deadlock in concurrent write test. Concurrent writes through shared state deadlock; single-owner actor pattern eliminates this.
- zed [#17](https://github.com/zed-industries/zed/issues/17) — Investigate using `scoped_pool` for text layout. Thread pool overhead and starvation concerns led Zed to move away from naive pool patterns — validates our decision to use dedicated tasks rather than a shared thread pool.

**Challenges** (issues suggesting we need to change something):
- None found. The per-PTY actor pattern is well-validated by the deadlock issues above. No issues suggest a global actor would be better.

**Additions** (edge cases the spec doesn't cover):
- zellij [#1949](https://github.com/zellij-org/zellij/issues/1949) — Killing client crashes server. When the controlling client dies unexpectedly, the actor must handle the broken command channel gracefully rather than panicking. The spec mentions `catch_unwind` for the reader thread but should also specify: if the command channel (`cmd_tx`) is dropped (e.g., frontend reload), the actor should clean up and exit rather than panic.
- zed [#42015](https://github.com/zed-industries/zed/issues/42015), [#42019](https://github.com/zed-industries/zed/issues/42019) — NVIDIA driver deadlock on Linux startup. GPU driver deadlocks can freeze Tokio threads. Actor tasks should avoid any GPU interaction; terminal rendering must stay on the frontend side only. Already the case in our design, but worth documenting as a constraint.

---

### 2. Ring Buffer 256KB + 4ms Flush

**Spec reference:** Section 5.3

**Validations** (issues confirming our approach is correct):
- zellij [#525](https://github.com/zellij-org/zellij/issues/525) — High memory usage and latency when a program produces output too quickly. Zellij's unbounded buffering caused OOM. Our 256KB ring buffer with overwrite-oldest semantics directly prevents this — xterm.js has its own scrollback, so dropped intermediate frames are acceptable.
- zellij [#803](https://github.com/zellij-org/zellij/issues/803) — Zellij killed by OOM. Another case of unbounded memory growth from fast output. Ring buffer caps memory at 256KB per PTY regardless of output rate.
- zed [#23008](https://github.com/zed-industries/zed/issues/23008) — Memory leak in Zed's built-in terminal when running `yes $(echo -en "\a")`. Rapid bell character output causes unbounded memory growth. Our ring buffer would overwrite rather than accumulate.
- zellij [#1789](https://github.com/zellij-org/zellij/issues/1789), [#1625](https://github.com/zellij-org/zellij/issues/1625) — Memory leak reports in zellij from long-running sessions. Bounded ring buffer prevents slow memory creep from accumulated output data.

**Challenges** (issues suggesting we need to change something):
- None found. The bounded ring buffer approach is strongly validated. No issues suggest that overflow/overwrite causes user-visible problems — xterm.js scrollback handles the user's need to scroll back.

**Additions** (edge cases the spec doesn't cover):
- waveterm [#2787](https://github.com/wavetermdev/waveterm/issues/2787) — TUI animations scroll instead of animating in-place (missing xterm.js Synchronized Output support). When batching output at 4ms intervals, partial frames from TUI apps may cause visual tearing. Consider: support synchronized output (DEC mode 2026) — hold output until the matching end marker arrives, then flush as one batch. This could be a post-M1 enhancement but is worth noting.
- zellij [#2828](https://github.com/zellij-org/zellij/issues/2828) — Flickering due to segmented rendering with apps that refresh constantly (e.g., `top`, `pw-top`). Related to the above — the 4ms timer-based flush can split a single application frame across two IPC messages. The spec's "flush when >75% full" adaptive threshold partially addresses this but synchronized output support would fully solve it.
- zed [#35797](https://github.com/zed-industries/zed/issues/35797) — Terminal hanging on `terraform init`. Long-running commands producing large bursts can interact badly with backpressure. The spec's 10MB pending-bytes cap (section 5.4) handles write-side backpressure, but the read side should also have a watchdog: if the ring buffer has been full for >N seconds with no consumer draining it, log a warning.

---

### 3. Escape Filtering (Two-Layer: Rust VTE + xterm.js)

**Spec reference:** Section 7.2

**Validations** (issues confirming our approach is correct):
- waveterm [#2845](https://github.com/wavetermdev/waveterm/issues/2845) — OSC 52 clipboard copying fails when window/block not focused. Directly validates our OSC 52 focus gating design: clipboard writes should only succeed when the terminal is focused, exactly as the spec states.
- zellij [#1709](https://github.com/zellij-org/zellij/issues/1709) — OSC 52 events not forwarded when content >= 1024 bytes. Validates our spec's explicit size limits (75KB decoded, 128KB raw) — an arbitrary small limit breaks legitimate use. Our limits are generous enough for real clipboard content while still preventing abuse.
- wezterm [#812](https://github.com/wezterm/wezterm/issues/812) — Crashing due to unknown escape sequence. Validates the need for Rust-side filtering: malformed or unexpected escape sequences should be stripped before reaching the renderer, not crash the application.
- zed [#9742](https://github.com/zed-industries/zed/issues/9742) — Copilot CLI error with unexpected escape sequence from terminal. Programs may emit escape sequences that confuse host applications. Two-layer defense catches issues that either layer alone would miss.
- zed [#48355](https://github.com/zed-industries/zed/issues/48355) — Devcontainer terminal prints raw ANSI escape sequences for arrow keys. Escape sequences leaking to the user as raw text indicates incomplete parsing; our Rust VTE layer handles this before xterm.js.
- helix [#15284](https://github.com/helix-editor/helix/issues/15284) — Terminal escape codes leak to stdout during startup. Validates that escape filtering must be active from the very first byte of output, not after some initialization period.

**Challenges** (issues suggesting we need to change something):
- wezterm [#4791](https://github.com/wezterm/wezterm/issues/4791) — OSC 52 stops working after a while. Time-based or state-based corruption can break OSC 52 silently. Consider: add a diagnostic event/log when OSC 52 is blocked (focus gate, size limit, or filter) so users can debug clipboard issues. The spec should add a `pty:osc52-blocked` event with the reason.
- zellij [#3951](https://github.com/zellij-org/zellij/issues/3951) — Zellij prevents Neovim from detecting OSC 52 clipboard support. Our Rust VTE layer blocks OSC 52 *queries* but must still allow the *capability response* path so programs can detect support. The spec says "Block OSC 52 clipboard queries" — clarify: block *read* queries (`?` parameter), but allow *write* operations and respond to capability queries (XTGETTCAP) with OSC 52 support indicated.
- zellij [#4320](https://github.com/zellij-org/zellij/issues/4320) — Support XTGETTCAP. Programs like Neovim query terminal capabilities via XTGETTCAP to detect OSC 52 support. If we strip DCS sequences (as the spec states), we break capability detection. Recommended spec change: allow XTGETTCAP DCS requests through and respond with appropriate capabilities, or handle them in the Rust layer and respond directly.

**Additions** (edge cases the spec doesn't cover):
- waveterm [#2895](https://github.com/wavetermdev/waveterm/issues/2895) — Terminals freeze for several minutes after macOS screen lock/sleep. WebGL context loss after sleep can freeze rendering. The spec mentions "WebGL -> Canvas -> DOM" fallback but should also specify: detect WebGL context loss events and automatically fall back to Canvas renderer rather than freezing. Re-attempt WebGL after a delay.
- zellij [#3512](https://github.com/zellij-org/zellij/issues/3512) — Consider using OSC 7 to get CWD. The spec already includes OSC 7 CWD tracking in Layer 2, validating this approach. But add: when OSC 7 reports a CWD change, update the entity's `Pty.cwd` component so session restore uses the latest directory, not the original spawn directory.
- wezterm [#3416](https://github.com/wezterm/wezterm/issues/3416) — OSC 52 clipboard text losing newlines on macOS. Platform-specific clipboard handling can mangle content. The spec should note: preserve raw bytes through OSC 52 decode without platform-specific newline normalization.

---

### 4. Channel\<T\> for PTY Streaming

**Spec reference:** Section 6.2

**Validations** (issues confirming our approach is correct):
- tauri-plugin-pty [#5](https://github.com/Tnze/tauri-plugin-pty/issues/5) — Calling `kill()` does not terminate the indefinite read loop. Using events rather than channels for PTY data creates lifecycle coupling problems. Our per-entity `Channel<Vec<u8>>` with explicit lifecycle management (check `channel.send()` result) avoids this.
- The spec's note about checking `channel.send()` and stopping on broken channel is validated by the general pattern of frontend reload issues across all projects — dead listeners that keep producing data.

**Challenges** (issues suggesting we need to change something):
- None found. The `Channel<T>` approach is a Tauri 2-specific mechanism with limited issue coverage in the reference projects (most use different IPC). The design is sound based on Tauri documentation and the tauri-plugin-pty issues.

**Additions** (edge cases the spec doesn't cover):
- The spec mentions cleaning up `channel.onmessage` explicitly (Tauri memory leak #13133) and deleting it on component destroy. Add: also handle the case where the frontend component is destroyed *during* a PTY output burst — the channel callback may fire after the Svelte component's `onDestroy` has run. Use a `destroyed` flag checked in the callback, or use Svelte 5's `$effect` cleanup which handles this automatically.
- wezterm [#2466](https://github.com/wezterm/wezterm/issues/2466) — "too many open files" on long-running session. Each `Channel<T>` presumably holds a file descriptor or IPC handle. If terminals are created and destroyed frequently, ensure channels are fully closed and resources released. The spec should note: verify with `lsof` or `/proc/self/fd` that channel cleanup doesn't leak handles over terminal create/destroy cycles.

---

### 5. Single-Writer SQLite + WAL

**Spec reference:** Section 3.1

**Validations** (issues confirming our approach is correct):
- zed [#49241](https://github.com/zed-industries/zed/issues/49241) — "Failed to load the database file" on startup, with WAL keyword. WAL mode databases can fail on startup if the WAL file is corrupted or left from a crash. Our spec's `PRAGMA integrity_check` on Phase 1 restore and WAL checkpoint on clean shutdown directly address this.
- zed [#49585](https://github.com/zed-industries/zed/issues/49585) — Malformed SQLite DB broke Recent Projects and Workspace Reload after update. Validates our spec's approach of copying corrupt DB to `.bak` and starting fresh (Phase 1 restore) rather than crashing.
- zed [#47543](https://github.com/zed-industries/zed/issues/47543) — Crash and process hang when opening a database migrated by a newer Zed version. Validates our simple append-only migration system — forward-only migrations with a version table prevent downgrade confusion.
- zed [#36839](https://github.com/zed-industries/zed/issues/36839) — Failed to load the database file. General database load failures validate the need for defensive startup: `busy_timeout`, integrity checks, and graceful degradation.
- zed [#47109](https://github.com/zed-industries/zed/issues/47109) — Zed logs flooded with database constraint errors. Validates our single-writer architecture: concurrent writes from multiple threads can violate constraints. A dedicated write thread with serialized access prevents this.

**Challenges** (issues suggesting we need to change something):
- None found. The single-writer + WAL pattern with integrity checking and backup-on-corruption is well-validated by the Zed SQLite issues.

**Additions** (edge cases the spec doesn't cover):
- zed [#45696](https://github.com/zed-industries/zed/issues/45696) — Memory leak consuming 71GB+ RAM causing potential filesystem corruption. If the app crashes hard (OOM kill, SIGKILL), the WAL file may not be checkpointed. The spec handles this via `PRAGMA wal_checkpoint(TRUNCATE)` on shutdown, but add: on startup, if the WAL file exists and is large (>1MB), run checkpoint before `integrity_check` to recover data first, then check integrity.
- zed [#49162](https://github.com/zed-industries/zed/issues/49162) — Stale buffer content after terminal commands modify files on disk. Not directly a SQLite issue, but relevant to our in-memory cache: if the spec uses in-memory caching of entity data, ensure the cache is invalidated when components are updated via the write thread. The single-writer channel approach naturally serializes this, but document the invariant.

---

### 6. Process Group Kill with SIGHUP-First

**Spec reference:** Section 5.5

**Validations** (issues confirming our approach is correct):
- canopy [#23](https://github.com/The-Banana-Standard/canopy/issues/23) — `close_terminal` should explicitly kill child process before removing. Relying on `Drop` to kill processes is unreliable. Our explicit two-step kill sequence (SIGHUP foreground group, then shell group, then SIGKILL) is the correct approach.
- zed [#9482](https://github.com/zed-industries/zed/issues/9482) — Orphan Node process after Zed quit (sometimes 100+% CPU). Node.js child processes not killed on exit. Our process group kill (`killpg`) addresses this by killing the entire process tree, not just the shell.
- zed [#34932](https://github.com/zed-industries/zed/issues/34932) — Opening another project while running a task doesn't kill process. Validates the need for explicit kill propagation when removing entities, not relying on implicit cleanup.
- zed [#46474](https://github.com/zed-industries/zed/issues/46474) — Zed does not kill Node.js processes upon exit, creating zombie processes. Directly validates process group kill: individual process kill misses child processes.
- zed [#47412](https://github.com/zed-industries/zed/issues/47412) — Child processes not terminated when integrated terminal closes. Keywords include `orphan, sigterm, sighup` — validates our SIGHUP-first approach for terminal processes.
- zellij [#518](https://github.com/zellij-org/zellij/issues/518) — Zombie child processes not reaped on pane/tab exit (macOS and Linux). Validates that process reaping must be explicit, not left to the OS.
- zellij [#1286](https://github.com/zellij-org/zellij/issues/1286) — Zombie process on tab/pane close. Persistent zombie processes from improper kill handling.
- zellij [#3024](https://github.com/zellij-org/zellij/issues/3024) — Zombie process after closing a pane command. Keywords: `zombie, sigterm, sighup`. Validates the signal sequence matters.
- zellij [#4054](https://github.com/zellij-org/zellij/issues/4054) — Copy command leaves lots of zombie processes. Even short-lived commands can leave zombies if not properly reaped.
- zed [#37458](https://github.com/zed-industries/zed/issues/37458), [#51155](https://github.com/zed-industries/zed/issues/51155) — Orphaned agent/CLI processes persist after Zed quit. Validates that all child process types (not just shells) need process group kill.
- helix [#4068](https://github.com/helix-editor/helix/issues/4068) — `hx` creates zombie process on exit. Language server processes not killed properly validates depth-first kill by component type.
- wezterm [#5101](https://github.com/wezterm/wezterm/issues/5101) — WezTerm killing tmux window when closed. SIGHUP to the PTY kills tmux — validates that SIGHUP is the right signal for terminal processes (it's what terminals normally send on close).

**Challenges** (issues suggesting we need to change something):
- zed [#36934](https://github.com/zed-industries/zed/issues/36934) — Windows: Unable to terminate lingering zombie processes after closing and reopening Zed. Windows process group kill behaves differently. The spec mentions Job Objects for Windows but should specify: create a Job Object per PTY, assign the child process to it, configure `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` so all children die when the job handle is closed. This is the Windows equivalent of `killpg`.
- zed [#47455](https://github.com/zed-industries/zed/issues/47455), [#48722](https://github.com/zed-industries/zed/issues/48722) — Claude agent zombie `node.exe` processes persist on Windows. Reinforces that Windows Job Objects must be properly configured, not just created. Add: verify Job Object assignment with `IsProcessInJob()` after spawn.
- zellij [#1747](https://github.com/zellij-org/zellij/issues/1747) — Zellij doesn't detach on SIGHUP. The app itself must handle SIGHUP gracefully (detach or clean shutdown), not just send it to children. Add to spec: install a SIGHUP handler on the Yord process that triggers clean shutdown (kill propagation + WAL checkpoint).

**Additions** (edge cases the spec doesn't cover):
- helix [#13853](https://github.com/helix-editor/helix/issues/13853) — Helix does not run `setsid` on LSPs. Without `setsid`, child processes share the parent's process group, so signals intended for one process hit all of them. The spec already specifies `setsid()` for PTY processes — validate this is actually called and verify with `ps -o pgid`.
- helix [#5424](https://github.com/helix-editor/helix/issues/5424) — Clipboard contents lost when used within zellij due to process group/setsid issues. Improper process group setup can cause signal leakage. Validates our `setsid()` approach.
- zed [#43864](https://github.com/zed-industries/zed/issues/43864) — Language servers not properly shutting down when closing Zed. Post-M1 relevance: when Agent/Editor components are added, they need the same kill propagation discipline. The spec's depth-first-by-component-type ordering is forward-compatible.
- zellij [#3918](https://github.com/zellij-org/zellij/issues/3918) — Zombie sessions refuse to die. When the parent process itself becomes a zombie, children are orphaned to init. Add: the spec's stale PID cleanup on startup (Phase 2) should also check for orphaned process groups from previous Yord instances, not just individual PIDs.
- zed [#45211](https://github.com/zed-industries/zed/issues/45211) — Gemini ACP extension leaks zombie processes causing memory exhaustion. Zombie processes that aren't reaped accumulate. Add: after `killpg` + SIGKILL, call `waitpid` (or equivalent) to reap the zombie. The spec's `try_wait()` polling handles this during the grace period, but after SIGKILL, do a final blocking `waitpid` with a short timeout.

---

### 7. Progressive Session Restore

**Spec reference:** Section 5.7

**Validations** (issues confirming our approach is correct):
- zellij [#3374](https://github.com/zellij-org/zellij/issues/3374) — Tabs/panes lost their working directories on session restore after reboot. Validates our spec's CWD validation (Phase 3): check that `Pty.cwd` exists, fallback to parent project CWD or `$HOME`.
- zed [#25022](https://github.com/zed-industries/zed/issues/25022) — Zed stopped restoring sessions after an update. Update-related restore failures validate our defensive approach: integrity check, `.bak` copy on corruption, and graceful degradation.
- zed [#40666](https://github.com/zed-industries/zed/issues/40666), [#43952](https://github.com/zed-industries/zed/issues/43952) — Window location and size not restored properly. Validates our use of `tauri-plugin-window-state` for geometry restore (Phase 1).
- zed [#40314](https://github.com/zed-industries/zed/issues/40314) — Session restore not working when settings UI is open. Validates the need for per-entity restore isolation — one entity's restore failure shouldn't block others.
- zed [#7371](https://github.com/zed-industries/zed/issues/7371) — Restarts should be non-destructive on workspace restore/reload. Validates progressive restore: show the tree immediately (Phase 1), spawn processes later (Phase 3), so the app is usable during restore.
- zellij [#1524](https://github.com/zellij-org/zellij/issues/1524) — Save and auto-restore all layout after restart. Users expect tmux-resurrect-like behavior. Our 4-phase progressive restore with `RestoreOnStart` component delivers this.
- zellij [#3280](https://github.com/zellij-org/zellij/issues/3280) — Cannot resurrect after upgrading to 0.40.0. Version upgrades can break serialization. Our ECS model (JSON components in SQLite) is inherently forward-compatible — unknown component types are preserved but ignored.
- zellij [#4413](https://github.com/zellij-org/zellij/issues/4413) — Session resurrection completely non-functional. When resurrection breaks entirely, users lose all state. Our integrity check + `.bak` + start fresh approach is the right fallback.
- zed [#52040](https://github.com/zed-industries/zed/issues/52040) — "Failed to restore 1 workspace" message without useful log info. Validates our Phase 4 toast: "5 terminals restored, 1 failed (missing directory)" — actionable error messages.
- waveterm [#1662](https://github.com/wavetermdev/waveterm/issues/1662) — Terminal folder gets forgotten when reloading app. Validates persisting CWD as a component and restoring it.

**Challenges** (issues suggesting we need to change something):
- zellij [#1255](https://github.com/zellij-org/zellij/issues/1255) — Sessions appear lost when zellij is upgraded. Binary upgrade changes the socket/IPC path, making old sessions invisible. Not directly applicable (we use SQLite, not sockets), but add: store a `yord_version` field in the DB. On startup, if the stored version is newer than the running binary, warn the user rather than silently proceeding with potentially incompatible data.
- zellij [#3861](https://github.com/zellij-org/zellij/issues/3861) — Improve session resurrection: user sources env vars from a shell file before starting. Our spec filters session-specific env vars on restore but doesn't address user-specific env setup. Consider: allow an optional `on_restore` hook or command per entity that runs before the main command, for environment initialization.
- zed [#50910](https://github.com/zed-industries/zed/issues/50910) — Fatal error when restoring historical session via WSL Remote: CWD path forward slashes converted to backslashes. Path separator normalization on restore can corrupt paths. Add to CWD validation in Phase 3: preserve path separators as-is from the stored value; do not normalize. On Windows, check both `\` and `/` forms.

**Additions** (edge cases the spec doesn't cover):
- zellij [#3151](https://github.com/zellij-org/zellij/issues/3151) — Attaching resurrection session crashes. Resurrection itself can crash the app. The spec should wrap Phase 3 entity spawning in per-entity error handling (it mentions per-entity mutex and crash loop prevention, but add: `catch_unwind` around each entity's restore to prevent one bad entity from crashing the entire restore).
- zellij [#4873](https://github.com/zellij-org/zellij/issues/4873) — Session resurrection captures wrong command when parent has multiple child processes. When a shell spawns child processes, `Pty.cmd` may reflect the child (e.g., `npm run dev`) rather than the shell. On restore, spawning the child command without its shell context fails. The spec's default shell detection partially addresses this, but add: if `Pty.cmd` is not a shell and not found on PATH, fall back to spawning the default shell in the same CWD rather than showing `Crashed`.
- zed [#15959](https://github.com/zed-industries/zed/issues/15959) — Linux Wayland: restore_on_startup doesn't restore session. Wayland-specific window state issues. The spec uses `tauri-plugin-window-state`, which may have Wayland limitations. Add a note: test window geometry restore on Wayland specifically; if it fails, fall back to default window size rather than showing nothing.
- zed [#44042](https://github.com/zed-industries/zed/issues/44042) — Tabs of windows aren't working well upon restart. Tab ordering corruption on restore. Our `Order` component in the ECS model handles this, but add: if `Order.index` values have gaps or duplicates after restore, re-normalize them (compact sequential assignment) during Phase 1.
- zellij [#4281](https://github.com/zellij-org/zellij/issues/4281) — Restore `puma` rails server. Long-running server processes may not be safely restartable (e.g., they bind ports, hold locks). The spec's `RestoreOnStart` component is opt-in, which is correct. But add: on restore, if a process exits within <2s (crash loop prevention), also check if the port it was using is already bound — provide a more specific error message: "Port 3000 already in use."

---

### 8. `$state.raw()` for Tree Data

**Spec reference:** Section 7.1

**Validations** (issues confirming our approach is correct):
- The 592 reference issues are from Rust-native (Zed, Helix, WezTerm, Zellij) and Electron (WaveTerm) projects, none of which use Svelte. No direct `$state.raw()` or Svelte proxy performance issues appear in the dataset. However, the design decision is validated by:
- zed [#25843](https://github.com/zed-industries/zed/issues/25843) — Zed uses 100% CPU when files outside the editor are changed. Reactivity systems that re-render on every external change can cause CPU spikes. Our `$state.raw()` + explicit `buildTree()` recalculation avoids continuous reactive updates — the tree only rebuilds when an entity event fires.
- zellij [#2435](https://github.com/zellij-org/zellij/issues/2435) — Reasonably slow text selection in zellij. UI performance issues from unnecessary redraws. Using `$state.raw()` prevents Svelte from creating deep proxies on the tree data structure, avoiding the "5,000x perf cliff" noted in the spec.
- zellij [#4389](https://github.com/zellij-org/zellij/issues/4389) — Zellij session stuck at 100% CPU on pane resize. Resize events triggering excessive re-computation. Our `queueMicrotask()` debounce for rapid events prevents this class of issue.

**Challenges** (issues suggesting we need to change something):
- None found. The reference projects don't use Svelte, so there are no direct challenges. The `$state.raw()` decision is based on Svelte 5 documentation and internal benchmarks rather than issue evidence.

**Additions** (edge cases the spec doesn't cover):
- zed [#48968](https://github.com/zed-industries/zed/issues/48968) — Memory leak from rapid FS events in watched directory. Rapid external events can overwhelm any reactivity system. The spec's `queueMicrotask()` debounce handles same-tick coalescing, but consider: if entity events arrive faster than the tree can rebuild (sustained >100 events/second), use a rate limiter (e.g., `requestAnimationFrame`-gated rebuild) rather than processing every microtask batch.
- waveterm [#2731](https://github.com/wavetermdev/waveterm/issues/2731) — Electron memory leak. While not Svelte-specific, highlights that frontend frameworks can leak when components are created/destroyed rapidly. Ensure that `buildTree()` returns fresh arrays (no references to old tree nodes) so garbage collection can reclaim previous trees. The spec's "returns fresh `TreeNode[]` for `$state.raw()` reassignment" already handles this correctly.

---

## Summary

| Decision | Validations | Challenges | Additions |
|----------|------------|------------|-----------|
| 1. Per-PTY actor pattern | 6 | 0 | 2 |
| 2. Ring buffer 256KB + 4ms flush | 4 | 0 | 3 |
| 3. Escape filtering (two-layer) | 6 | 3 | 3 |
| 4. Channel\<T\> for PTY streaming | 2 | 0 | 2 |
| 5. Single-writer SQLite + WAL | 5 | 0 | 2 |
| 6. Process group kill with SIGHUP-first | 12 | 3 | 5 |
| 7. Progressive session restore | 10 | 3 | 5 |
| 8. `$state.raw()` for tree data | 3 | 0 | 2 |

**Overall assessment:** All 8 design decisions are well-validated by reference project issues. No decision needs fundamental rethinking. The strongest signal comes from process group kill (6) and session restore (7), where dozens of issues across multiple projects confirm the exact failure modes our design addresses.

**Highest-priority spec additions:**
1. XTGETTCAP / DCS passthrough for capability detection (decision 3) — without this, programs like Neovim can't detect OSC 52 support
2. Windows Job Object verification after spawn (decision 6) — Windows zombie process issues are extremely common in Zed
3. SIGHUP handler on the Yord process itself (decision 6) — handle external signals gracefully
4. Per-entity `catch_unwind` during restore Phase 3 (decision 7) — one bad entity shouldn't crash the entire restore
5. Synchronized output / DEC mode 2026 awareness for TUI apps (decision 2) — prevents visual tearing with batched output
