# wezterm

> GPU-accelerated terminal emulator and multiplexer in Rust. Origin of portable-pty crate. MIT. [GitHub](https://github.com/wezterm/wezterm)

## Overview

Full-featured terminal emulator with multiplexing, tabs, panes, ligatures, and GPU rendering. ~220-crate Rust workspace. Key sub-crate: `portable-pty` (published on crates.io) which Yord plans to use as a dependency.

Relevant to Yord: portable-pty API and internals (PtySystem, MasterPty, Child, ChildKiller traits). Process tree tracking via /proc. Two-thread PTY reader pipeline with coalescing. Kill pattern: SIGHUP → 250ms grace → SIGKILL. No session persistence (mux server keeps processes alive instead).

## Architecture Decisions

**Trait-based PTY abstraction (PtySystem, MasterPty, SlavePty)**
Multiple backends must be selectable at runtime: Unix (`openpty(3)`), Windows ConPTY (`CreatePseudoConsole` via dynamic `kernel32.dll` load), and serial ports. The `Downcast` supertrait allows consumers to reach platform-specific methods (`process_group_leader()`, `get_termios()`, `as_raw_fd()`). `PtyPair` enforces drop order -- slave listed first so it drops first (RFC 1857), preventing hangs on platforms where slave outliving master is fatal.

**Separate ChildKiller from Child**
`Child::wait()` is blocking `&mut self`. To kill from another thread while blocked in `wait()`, `clone_killer()` returns `Box<dyn ChildKiller + Send + Sync>`. The mux layer uses `split_child()` to separate into: (1) background thread blocking on `wait()` sending result through bounded channel, (2) a `ChildKiller` stored in pane state, (3) the pid.

**try_clone_reader() via dup'd fd**
Returns `Box<dyn Read + Send>` by `dup()`-ing the fd. The reader moves to a dedicated blocking-read thread. Master retains its own fd for `resize()`, `get_size()`, `process_group_leader()`, `get_termios()`. One reader per pane in practice.

**take_writer() -- single call enforced**
Uses `RefCell<bool>` to enforce one-call semantics. On Unix, `UnixMasterWriter::Drop` sends `\n` + `VEOF`. Multiple writers would cause premature EOF. On Windows, it is literally `Option<FileDescriptor>::take()`.

**\n + VEOF on writer drop**
In canonical mode, VEOF is only recognized at line start. The `\n` flushes any partial line buffer first, then VEOF on the empty line signals true EOF. Gracefully terminates shells without closing the master fd (which would cause EIO on slave).

**EIO → EOF conversion in read**
When the slave side closes, master reads return `EIO`, not `Ok(0)`. Without conversion, standard I/O combinators would error instead of terminating gracefully. Well-known Unix PTY behavior every consumer must handle.

**setsid() + TIOCSCTTY in pre_exec**
- `setsid()` -- new session, child becomes session/process group leader. Required for job control and to isolate child from parent's signal delivery.
- `TIOCSCTTY` -- sets PTY slave as controlling terminal. Without it: no SIGWINCH on resize, no `tcgetpgrp()`/`tcsetpgrp()`, no `/dev/tty`. Can be disabled for Flatpak/container scenarios.
- Also clears signal dispositions (`SIG_DFL` for SIGCHLD, SIGHUP, SIGINT, SIGQUIT, SIGTERM, SIGALRM) and unblocks all signals via `sigprocmask`.

**close_random_fds() after fork**
Enumerates `/dev/fd` and `close(2)`s every fd > 2. Needed because macOS Cocoa and Linux gnome/mutter leak fds to child processes. Even with `FD_CLOEXEC` on PTY fds, GUI toolkit / D-Bus / GPU handles are not guaranteed to be.

**Two-thread reader pipeline with coalescing**
- Thread 1 (reader): blocking reads from PTY fd, writes raw bytes into a socketpair.
- Thread 2 (parser): reads socketpair, parses escape sequences via `termwiz::Parser`, coalesces actions before flushing to mux/UI.
- Coalescing: if accumulated buffer < 1MB, `poll()` socketpair with configurable delay. More data = accumulate. Deadline expires or buffer full = flush. Synchronized output mode (DECSM 2026) overrides heuristic.
- Socketpair chosen over pipe/channel because it supports `poll()` and explicit buffer sizing (`SO_SNDBUF`/`SO_RCVBUF` = 1MB).
- Why it matters: unoptimized TUI programs send many small escape sequences per frame; batching reduces lock contention and render passes.

**Kill strategy: SIGHUP first, not SIGTERM**
SIGHUP ("your terminal is gone") is the traditional terminal-close signal. Shells specifically handle it for cleanup. Full sequence in `ChildKiller for std::process::Child`: SIGHUP → poll `try_wait()` 5x at 50ms (250ms grace) → `SIGKILL`. The `ProcessSignaller` (from `clone_killer()`) only sends SIGHUP with no SIGKILL fallback -- deliberate gentler approach for user-initiated kills. `LocalPane::drop()` also invokes the signaller to prevent zombies (issue #558).

## Bugs Encountered & How Solved

**macOS ttyname_r ERANGE bug**
`ttyname_r()` returns `ERANGE` even with very large buffers. Fix: cap retry loop at 64KB, return `None` beyond that.

**macOS EOF-interleaving race condition**
For very short-lived processes, EOF data interleaves with reader data at the kernel level. Workaround: 20ms `thread::sleep` before dropping the writer. Non-deterministic; no better solution found.

**Windows ConPTY stdio handle inheritance**
Spawned processes inherit redirected stdio handles from daemonized `wezterm-mux-server` (e.g., writing to a log file instead of the PTY). Fix: set `STARTF_USESTDHANDLES` with all three handles pointing to `INVALID_HANDLE_VALUE`.

**Windows ConPTY synthesized escape sequences**
ConPTY injects its own escape sequences (window title changes) into the output stream. Consumers must expect sequences they did not originate.

**Windows thread handle leak**
`CreateProcessW` returns both process and thread handles in `PROCESS_INFORMATION`. Fix: wrap thread handle in `OwnedHandle::from_raw_handle()` so it is closed on drop.

**Linux/macOS fd leaks from desktop environments**
The entire `close_random_fds()` function exists to clean up fds leaked by macOS Cocoa and Linux gnome/mutter after fork.

**Flatpak TIOCSCTTY failure**
`TIOCSCTTY` fails across container namespace boundaries. Fix: `CommandBuilder::set_controlling_tty(false)`. Refs: flatpak/flatpak#3697, flatpak/flatpak#3285.

**Zombie process prevention**
`LocalPane::drop()` sends kill signal to avoid lingering zombies (wezterm#558).

**Pipe reader blocked after child death (Windows)**
On some systems, the pipe reader stays blocked after child exits. Fix: dedicated waiter thread blocking on `process.wait()` that triggers `prune_dead_windows()`. Without this, `exit` in `cmd.exe` keeps the pane alive.

**tcgetpgrp performance (700us per call)**
With 10 tabs, hovering causes 7ms of blocking. Fix: `CachedLeaderInfo` with 300ms TTL serving stale data while background thread refreshes.

**Serial port blocking read divergence**
Unix serial crate uses non-blocking mode (needs explicit `poll()`), Windows blocks for 50ms timeout (needs retry loop). Same trait impl, different loop logic per platform.

## Current Bugs & Tech Debt

- **`WinChild::do_kill` inverted error logic** -- checks `if res != 0` for error, but `TerminateProcess` returns nonzero on *success*. Masked by `kill()` calling `.ok()` which discards the result.
- **`PSEUDOCONSOLE_PASSTHROUGH_MODE` defined but unused** -- `#[allow(dead_code)]` ConPTY flag for direct VT passthrough mode. Never actually used.
- **`ProcessSignaller` has no SIGKILL fallback** -- `clone_killer()` pathway only sends SIGHUP. If child ignores it, no force-kill.
- **Process tree enumeration is O(total_procs)** -- Linux reads all of `/proc`, macOS calls `proc_listallpids()` + per-pid info, Windows takes `Toolhelp32Snapshot` + `ReadProcessMemory`. All on every 300ms cache miss.
- **TMux tab listing TODO** -- `localpane.rs:879`: "do we need to proactively list available tabs here?"
- **WSL env passthrough incomplete** -- `domain.rs:308`: `WLSENV` variable passthrough not implemented.
- **Pane/Tab ID disambiguation** -- `lib.rs:1117, 1189`: two methods with unresolved "TODO: disambiguate with TabId".
- **Clipboard FIXMEs** -- `lib.rs:1233, 1240, 1389`: clipboard and split pane pixel dimensions unresolved.
- **TermWizTermTab disconnected writer** -- `termwiztermtab.rs:113`: `Box::new(Vec::new())` with "FIXME: connect to something?"
- **TerminalWaker panic** -- `termwiztermtab.rs:441-443`: panics if called because it assumes SystemTerminal.
- **SSH mux FIXME** -- `ssh.rs:266-268`: "this isn't useful without a way to talk to the remote mux."

## What To Learn

**portable-pty API surface**
Deliberately small: `openpty(size) → PtyPair`, `spawn_command(cmd) → Child`, `try_clone_reader()`, `take_writer()` (call once), `resize()`, `wait()`/`try_wait()`, `kill()`/`clone_killer()`. Key insight: slave is only for spawning, then immediately dropped. All I/O goes through master.

**Cross-platform abstraction strategy**
Conditional compilation (`#[cfg(unix)]`, `#[cfg(windows)]`) for platform modules, platform-neutral traits. Platform-specific methods gated by `#[cfg]` on the trait itself. Windows ConPTY dynamically loaded at runtime with fallback to sideloaded `conpty.dll`. Directly reusable by Yord via Tauri 2.

**Process tree tracking via /proc**
- Linux: `/proc/{pid}/stat` (parse around comm parentheses), `/proc/{pid}/exe`, `/proc/{pid}/cwd`, `/proc/{pid}/cmdline` (NUL-separated). Build tree from ppid.
- macOS: `proc_listallpids()` + `proc_pidinfo(PROC_PIDTBSDINFO)`, `proc_pidpath()`, `sysctl(KERN_PROCARGS2)`.
- Windows: `Toolhelp32Snapshot` + `OpenProcess` + `NtQueryInformationProcess` + `ReadProcessMemory` (handles 32/64-bit).
- The `procinfo` crate is not on crates.io. Yord must vendor or reimplement.

**Output coalescing for xterm.js**
Parse → accumulate → poll for more data up to deadline → flush all at once. Respect SynchronizedOutput (DECSM 2026). Directly applicable to Yord: batch PTY output before writing to xterm.js to reduce DOM updates with programs like `cat large_file.txt` or `make -j16`.

**ChildKiller / split_child pattern**
`split_child(child) → (Receiver<ExitStatus>, Box<dyn ChildKiller>, Option<u32>)`. Store signaller in pane/session state. Use from UI threads, process manager, or session teardown. Background thread blocks on `wait()` and notifies via bounded channel. Yord should adopt this directly.

## Issues

- **Drop order matters** -- slave must drop before master. Examples explicitly `drop(pair.slave)` after spawning, `drop(pair.master)` only after child exits. Yord must manage PTY lifetimes carefully during session teardown.
- **Writer must be taken even if unused** -- if `take_writer()` is never called, no EOF is sent on master drop, child may block forever on stdin.
- **Blocking reads are unavoidable** -- serial ports, ConPTY pipes, and some Unix PTY configs do not support non-blocking mode reliably. Dedicated read thread is the only portable approach.
- **ConPTY injects unsolicited output** -- xterm.js handles this natively, but any Yord output filtering/logging must account for ConPTY injections.
- **ConPTY minimum version** -- `CreatePseudoConsole` requires Windows 10 October 2018+. `load_conpty()` panics if absent. Yord should handle gracefully or document the requirement.
- **Process enumeration cost** -- every cache miss enumerates ALL system processes. Yord should consider lighter alternatives (just `tcgetpgrp()` + `/proc/{pid}/exe` for foreground process).
- **No process group kill on Unix** -- `ChildKiller` sends SIGHUP to child PID, not process group (`-pid`). Subprocesses become orphans. Yord's kill propagation will need `kill(-pgid, sig)` or tree walking.
- **`WinChild::do_kill` inverted error check** -- success/error branches swapped, masked by `.ok()`. Be aware if wrapping this code.
- **macOS short-lived process race** -- the 20ms sleep workaround is non-deterministic. If Yord spawns short-lived commands via PTY (e.g., shell integration probes), consider using pipes instead.
- **Serial port trait semantics diverge** -- `ChildKiller::kill()` is a no-op for serial; child never exits. Illustrates that trait implementations have very different semantics.

## Key Files

| File | Why |
|------|-----|
| `refs/wezterm/pty/src/lib.rs` | portable-pty API: all traits (PtySystem, MasterPty, Child, ChildKiller) |
| `refs/wezterm/pty/src/unix.rs` | Unix PTY: openpty, setsid, signal reset, fd cleanup |
| `refs/wezterm/pty/src/win/conpty.rs` | Windows ConPTY implementation |
| `refs/wezterm/pty/examples/bash.rs` | Minimal portable-pty usage example |
| `refs/wezterm/mux/src/lib.rs` | read_from_pane_pty: two-thread reader pipeline with coalescing |
| `refs/wezterm/procinfo/src/linux.rs` | Full process tree from /proc |
