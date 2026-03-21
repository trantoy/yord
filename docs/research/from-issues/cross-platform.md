# Cross-Platform Bug Catalog

> Based on analysis of reference project issue trackers. See [docs/refs/README.md](../../refs/README.md) for rules.

Focus: bugs affecting Yord M1 subsystems (PTY, terminal rendering, process management, session restore). Editor-specific platform bugs excluded.

---

## Windows

### ConPTY

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| W1 | wezterm [#244](https://github.com/wezterm/wezterm/issues/244) | Bold/bright foreground fixed to white | ConPTY strips SGR bold attribute and substitutes white foreground | Terminal must re-apply bold styling post-ConPTY; cannot rely on passthrough | M1: xterm.js handles this natively. Low risk. |
| W2 | wezterm [#3531](https://github.com/wezterm/wezterm/issues/3531) | Last line of output disappears with shell integration | ConPTY's internal cursor tracking conflicts with shell integration prompt marks (OSC 133) | Disable prompt marks or account for ConPTY's synthetic cursor movements | M1: if Yord adds shell integration later, must test with ConPTY. Not M1. |
| W3 | wezterm [#3624](https://github.com/wezterm/wezterm/issues/3624) | Images rendered by imgcat cut off by cmd.exe/WSL prompts | ConPTY re-renders prompt over image region when next prompt appears | Fixed in nightly; no sixel/image support needed for M1 | M1: not relevant (no image protocol). |
| W4 | wezterm [#4783](https://github.com/wezterm/wezterm/issues/4783) | Undercurl not working in shell vs tmux/neovim | ConPTY strips DECCARA/undercurl escape sequences; only curly underline passthrough in newer Windows builds | Requires Windows 11 22H2+ for undercurl passthrough or ConPTY passthrough mode | M1: cosmetic only. Document minimum Windows version. |
| W5 | wezterm [#4784](https://github.com/wezterm/wezterm/issues/4784) | Using portable_pty clears terminal when streaming to stdout | ConPTY allocates a hidden console window; spawning with inherited handles causes the visible console to be cleared | Set `STARTF_USESTDHANDLES` with all three handles set to `INVALID_HANDLE_VALUE` so child does not inherit parent's console | **M1 critical.** Yord uses portable-pty. Must set stdio handles correctly on spawn. Already documented in wezterm research. |
| W6 | wezterm [#6670](https://github.com/wezterm/wezterm/issues/6670) | tmux -CC not working on Windows ConPTY | ConPTY injects synthetic escape sequences (window title, cursor position) into the output stream; tmux CC mode cannot parse them | Filter or expect unsolicited ConPTY sequences in output processing | M1: Yord spec already notes "ConPTY injects unsolicited escape sequences." xterm.js handles this. |
| W7 | wezterm [#7025](https://github.com/wezterm/wezterm/issues/7025) | portable-pty fails to launch commands on Windows | `CommandBuilder` path resolution fails when `PATHEXT` contains unusual entries or command not found | Validate command exists before spawn; handle `PATHEXT` edge cases | **M1 relevant.** Must validate `Pty.cmd` on Windows before spawn. |
| W8 | wezterm [#6499](https://github.com/wezterm/wezterm/issues/6499) | Panic if `PATHEXT` has empty entry (`;;`) | `which`-style resolution splits on `;` and panics on empty string when building path candidates | Guard against empty entries in `PATHEXT` split | **M1 relevant.** Defensive parsing of Windows env vars during command resolution. |
| W9 | wezterm [#6783](https://github.com/wezterm/wezterm/issues/6783) | portable-pty 0.9.0 does not work on Windows | API breaking change in 0.9.0; ConPTY initialization path changed | Pin to working portable-pty version or track upstream fixes | **M1 relevant.** Pin portable-pty version. Test ConPTY path on CI. |
| W10 | wezterm [#5107](https://github.com/wezterm/wezterm/issues/5107) | `clone_killer` gives invalid handle on Windows | `WinChild::clone_killer()` duplicates a process handle that is already invalid after child exit | Check handle validity before dup; use `OpenProcess` with the stored PID as fallback | **M1 relevant.** Yord's kill propagation uses ChildKiller. Must handle invalid handles on Windows. |
| W11 | wezterm [#1927](https://github.com/wezterm/wezterm/issues/1927) | ConPTY version mismatch with system | System ConPTY may be outdated; bundling via NuGet package provides newer version | Bundle `conpty.dll` or dynamically load from system with version check | M1: document minimum Windows 10 1809+ requirement. Spec already does this. |
| W12 | waveterm [#2787](https://github.com/wavetermdev/waveterm/issues/2787) | TUI animations scroll instead of animating in-place | Missing Synchronized Output (DEC mode 2026) support; ConPTY does not pass through BSU/ESU markers | Implement DECSM 2026 handling in terminal frontend or accept visual glitch | M1: xterm.js supports mode 2026. ConPTY may still strip it. Cosmetic. |
| W13 | wezterm (research) | `WinChild::do_kill` inverted error logic | `TerminateProcess` returns nonzero on success but code checks `if res != 0` as error | Invert the check: nonzero = success | **M1 critical.** If Yord wraps or references this pattern, the kill logic is silently broken. Write fresh with correct semantics. |
| W14 | wezterm (research) | Pipe reader stays blocked after child death | Windows named pipe read does not return EOF when ConPTY child exits on some systems | Dedicated waiter thread calling `process.wait()` that signals the reader to stop | **M1 relevant.** Yord's PTY reader thread must have a shutdown signal independent of EOF. |
| W15 | wezterm (research) | Thread handle leak from `CreateProcessW` | `PROCESS_INFORMATION` contains both process and thread handles; thread handle not closed | Wrap thread handle in `OwnedHandle` immediately after `CreateProcessW` returns | **M1 relevant.** Resource leak if Yord spawns via Win32 API directly. portable-pty should handle this, but verify. |
| W16 | zed [#42166](https://github.com/zed-industries/zed/issues/42166) | Crash when opening terminal on Windows | ConPTY initialization fails on certain Windows configurations | Graceful error handling on ConPTY init failure instead of panic | **M1 relevant.** Yord must handle ConPTY init failure gracefully (show error in UI, not crash). |
| W17 | zed [#42345](https://github.com/zed-industries/zed/issues/42345) | Terminal overwrites old lines after switching apps | ConPTY re-renders its internal buffer when focus returns, overwriting xterm.js state | Accept ConPTY re-render or force full redraw on focus return | M1: cosmetic. May need focus-return handler to scroll to bottom. |
| W18 | zed [#46514](https://github.com/zed-industries/zed/issues/46514) | pwsh.exe prompt moves to top on panel resize | ConPTY resize reflows content differently than Unix PTY; prompt jumps to top of resized region | Debounce resize; accept ConPTY reflow behavior | M1: spec already includes resize debounce (50-100ms). ConPTY reflow is a known limitation. |
| W19 | zed [#46650](https://github.com/zed-industries/zed/issues/46650) | Custom shell settings broken (os error 123) | Windows path with invalid characters (colon in wrong position, forward slashes) passed to `CreateProcessW` | Normalize shell path: backslashes, no trailing spaces, valid drive letter | **M1 relevant.** Yord `Pty.cmd` on Windows must be normalized before spawn. |

### Job Objects / Process Groups

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| W20 | zed [#36934](https://github.com/zed-industries/zed/issues/36934) | Zombie processes accumulate after closing/reopening Zed | Child processes not assigned to a Job Object; survive parent exit | Use `CreateJobObject` + `AssignProcessToJobObject` with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` | **M1 critical.** Spec already mandates Job Objects on Windows. Must implement. |
| W21 | zed [#48722](https://github.com/zed-industries/zed/issues/48722) | Zombie node.exe processes persist after Zed exit (regression) | Process group cleanup regressed; ACP agent processes not in Job Object | Ensure all spawned processes (including agent subprocesses) are in the Job Object | **M1 relevant.** All PTY children must be in Job Object, not just the shell. |
| W22 | zed [#47455](https://github.com/zed-industries/zed/issues/47455) | Agent processes lock files after Zed close on Windows | No Job Object for agent child processes; survive parent crash | Job Object with kill-on-close flag | M1: agents are post-M1, but the pattern (Job Object for all children) must be established in M1. |
| W23 | canopy (research) | Subprocesses orphaned on tab close | `child.kill()` only kills direct child (the shell); grandchildren survive | Use Job Objects to group all descendants | **M1 critical.** Already in spec: "On Windows, use Job Objects." |
| W24 | clif-code (research) | No SIGTERM before SIGKILL; ports remain bound | `child.kill()` sends `TerminateProcess` (equivalent to SIGKILL); no graceful shutdown | Send `CTRL_C_EVENT` or `CTRL_BREAK_EVENT` to process group first, then `TerminateProcess` after timeout | **M1 relevant.** Windows graceful shutdown requires `GenerateConsoleCtrlEvent` before `TerminateProcess`. |

### Path Handling

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| W25 | wezterm [#4205](https://github.com/wezterm/wezterm/issues/4205) | portable-pty removes custom paths from PATH on Windows | `CommandBuilder` reconstructs PATH from system defaults, dropping user additions | Preserve existing PATH and append/prepend rather than replace | **M1 relevant.** Yord must inherit current PATH when spawning PTY, not reconstruct it. |
| W26 | wezterm [#2980](https://github.com/wezterm/wezterm/issues/2980) | Environment variables not expanded on Windows | `%USERPROFILE%` and similar Windows env var syntax not expanded in config | Expand `%VAR%` syntax in `Pty.env` values on Windows before passing to child | M1: minor. Expand env vars in `Pty.cwd` and `Pty.cmd` on Windows. |
| W27 | zed [#42805](https://github.com/zed-industries/zed/issues/42805) | "Failed to load environment variables" after update | Shell profile loading fails when default shell path changes across updates | Detect shell from registry/`COMSPEC` rather than hardcoding; handle missing shell gracefully | **M1 relevant.** Yord must detect default shell on Windows robustly. |
| W28 | zed [#47823](https://github.com/zed-industries/zed/issues/47823) | Fails when `'` in project path | Single quote in Windows path breaks shell invocation | Properly escape or quote paths passed to shell `-c` commands | **M1 relevant.** `Pty.cwd` with special characters must be handled. |
| W29 | zed [#46307](https://github.com/zed-industries/zed/issues/46307) | VHD-mounted directories cause limited terminal functionality | Path canonicalization resolves VHD mount points to UNC paths that ConPTY/shells cannot handle | Skip canonicalization for VHD paths or use original mount point | M1: edge case. Document limitation. |

### Other

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| W30 | zed [#43252](https://github.com/zed-industries/zed/issues/43252) | Running a task loads PowerShell profile (slow startup) | Terminal spawns `powershell.exe -Command` which loads profile by default | Use `-NoProfile` flag for non-interactive commands; only load profile for interactive shells | M1: Yord spawns interactive shells, so profile loading is expected. But startup commands should use `-NoProfile`. |
| W31 | zed [#49442](https://github.com/zed-industries/zed/issues/49442) | Slow startup on Windows | Windows Defender real-time scanning on Tauri/Electron binary and spawned processes | Sign binary; add to Defender exclusions in docs; consider `smartscreen` metadata | M1: documentation concern. May affect PTY spawn latency. |
| W32 | zed [#49687](https://github.com/zed-industries/zed/issues/49687) | Auto-update crash with permission denied | Update replaces running binary; open handles prevent rename | Close all child processes before update; use Windows Restart Manager API | Post-M1: auto-update concern. |
| W33 | zed [#33520](https://github.com/zed-industries/zed/issues/33520) | Crash after locking/unlocking computer | GPU context loss on lock/unlock; WebView2/WKWebView does not recover | Handle `visibilitychange` event; reinitialize rendering on unlock | M1: potential issue for xterm.js WebGL renderer. Use Canvas fallback. |

---

## macOS

### WKWebView

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| M1 | waveterm [#2895](https://github.com/wavetermdev/waveterm/issues/2895) | Terminals freeze for several minutes after macOS sleep | WKWebView suspends JavaScript execution during sleep; WebGL context may be lost | Handle `visibilitychange` / `pageshow` events; reinitialize WebGL on wake; buffer PTY output in Rust during sleep | **M1 critical.** Yord spec already includes Rust ring buffer (256KB per PTY). Must handle WKWebView jettison: buffer output in Rust, re-feed xterm.js on wake. |
| M2 | waveterm (research) | WebGL context loss on GPU driver reset | Resource pressure or display sleep causes WebGL context loss in WKWebView | `WebglAddon.onContextLoss` triggers fallback to Canvas renderer | **M1 relevant.** Spec already includes renderer fallback chain: WebGL -> Canvas -> DOM. Implement the fallback. |
| M3 | zed [#48692](https://github.com/zed-industries/zed/issues/48692) | Crash on launch on macOS Monterey (x86_64) due to Mach port guard violation | SIGKILL from macOS kernel due to Mach port misuse in older macOS versions | Ensure minimum macOS version requirement; handle Mach port APIs correctly | M1: document minimum macOS version. Tauri 2 handles most of this. |

### PATH / GUI App Environment

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| M4 | canopy (research) | PTY child processes get minimal PATH | macOS GUI apps launched from Dock/Spotlight inherit a minimal environment (`/usr/bin:/bin:/usr/sbin:/sbin`), not the user's shell profile PATH | `ensure_full_path()`: augment PATH with `~/.local/bin`, `~/.cargo/bin`, `/usr/local/bin`, `/opt/homebrew/{bin,sbin}`, `~/.nvm/versions/node/*/bin`, `~/.fnm/aliases/default/bin` | **M1 critical.** Yord will hit this. Must augment PATH before PTY spawn. Spec notes this in research section. Needs explicit implementation. |
| M5 | zed [#35759](https://github.com/zed-industries/zed/issues/35759) | "Failed to load environment variables" with Bash | Shell profile sourcing fails when launched from GUI; `bash -l -c env` hangs or errors | Use `login -fp $USER` on macOS to get proper environment, or parse shell profile output with timeout | **M1 relevant.** Yord should source user environment at startup with a timeout. If it fails, use augmented PATH as fallback. |
| M6 | zed [#43121](https://github.com/zed-industries/zed/issues/43121) | Always shows "failed to load environment variables" | Shell profile contains interactive-only commands that fail in non-interactive mode | Run env loading in login shell mode; handle non-zero exit gracefully | **M1 relevant.** Same pattern as M5. Must not treat env loading failure as fatal. |

### Cocoa fd Leaks

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| M7 | wezterm (research) | Child processes inherit Cocoa framework file descriptors | macOS Cocoa allocates fds for Mach ports, IOKit, GPU handles that are not `FD_CLOEXEC`; these leak across `fork()` | `close_random_fds()` / `close_fds::close_open_fds(3, &[])` after fork, before exec -- enumerate `/dev/fd` and close everything > 2 | **M1 critical.** Spec already notes "Use `close_fds` crate for closing inherited file descriptors after fork (zellij pattern)." Must implement. |
| M8 | zellij (research) | Child panes can write to other panes' PTY master fds | Server holds multiple PTY master fds; without closing them in `pre_exec`, a child inherits all of them | `login_tty(pid_secondary)` + `close_open_fds(3, &[])` in `pre_exec()` | **M1 critical.** Same issue as M7 but with PTY-specific consequences. Yord must close all fds > 2 in child pre_exec. |
| M9 | zed [#1672](https://github.com/zed-industries/zed/issues/1672) | `CGEvent` and `CGEventSource` objects leaked | `chars_for_modified_key` creates Core Graphics objects without release | Call `CFRelease` on CGEvent/CGEventSource after use | M1: unlikely to affect Yord directly (Tauri handles keyboard events), but illustrates macOS resource leak pattern. |

### Other

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| M10 | wezterm (research) | `ttyname_r()` returns ERANGE with large buffers | macOS kernel bug: `ttyname_r` fails even with 64KB buffer on some PTY paths | Cap retry loop; return `None` if buffer exceeds 64KB | M1: minor. Only affects PTY name lookup for display purposes. |
| M11 | wezterm (research) | Short-lived process EOF interleaves with output | macOS kernel delivers EOF and final output data in non-deterministic order for very fast processes | 20ms `thread::sleep` before dropping PTY writer; non-deterministic | **M1 relevant.** If Yord spawns short-lived commands via PTY (e.g., env probes), use pipes instead. Spec already notes this. |
| M12 | waveterm (research) | Paste deduplication needed with xterm.js | xterm.js `paste()` triggers internal `onData`, doubling pasted text on macOS | Track `lastPasteData`/`lastPasteTime` with 30ms suppression | **M1 relevant.** Yord uses xterm.js. Must handle paste deduplication. |
| M13 | zed [#31721](https://github.com/zed-industries/zed/issues/31721) | Window restore on macOS does not reload state properly | macOS native window restoration conflicts with app-level session restore | Disable macOS native window restore (`NSQuitAlwaysKeepsWindows = false`); use own restore logic | M1: Yord has its own session restore. Must disable macOS native restore to avoid conflicts. |
| M14 | zed [#43067](https://github.com/zed-industries/zed/issues/43067) | Restart clumps all instances onto one Desktop | macOS Spaces assignment lost during restart; all windows placed on current Space | Store Space assignment and restore via `NSWindow.collectionBehavior` | Post-M1: multi-window concern. |
| M15 | zed [#50151](https://github.com/zed-industries/zed/issues/50151) | UI deadlock in `window_did_change_key_status` | macOS `windowDidBecomeKey`/`windowDidResignKey` callbacks re-enter event handling, causing deadlock | Defer state updates from window activation callbacks; use `DispatchQueue.main.async` | M1: single-window app unlikely to hit this, but guard against re-entrant Cocoa callbacks. |
| M16 | wezterm [#3416](https://github.com/wezterm/wezterm/issues/3416) | OSC 52 clipboard text losing newlines on macOS | macOS pasteboard normalizes line endings; `\n` stripped or converted | Use `NSPasteboard` with explicit UTF-8 encoding preserving newlines | Post-M1: clipboard integration concern. |

---

## Linux

### WebKitGTK

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| L1 | design.md (spec) | WebGL renderer fails on older WebKitGTK | WebKitGTK versions before 2.42 have incomplete WebGL2 support; xterm.js WebGL addon fails silently or crashes | Renderer fallback chain: WebGL -> Canvas -> DOM. Detect WebGL support before enabling. | **M1 critical.** Already in spec: "WebGL fails on older Linux WebKitGTK." Must implement fallback and detect WebKitGTK version. |
| L2 | design.md (spec) | `backdrop-filter` CSS broken on WebKitGTK | WebKitGTK does not support `backdrop-filter` CSS property (used for frosted glass effects) | Avoid `backdrop-filter` entirely; use solid backgrounds or alternative styling | **M1 relevant.** Already in spec. Must not use this CSS property. |
| L3 | zed [#37280](https://github.com/zed-industries/zed/issues/37280) | App hangs on startup with Hyprland/Wayland (futex deadlock) | NVIDIA driver + Wayland compositor interaction causes GPU initialization deadlock | Detect Wayland + NVIDIA and set fallback rendering mode; `GDK_BACKEND=x11` as workaround | M1: Tauri 2 on Wayland may hit this. Document `GDK_BACKEND=x11` workaround. Not a Yord code issue. |
| L4 | zed [#42015](https://github.com/zed-industries/zed/issues/42015) | Randomly fails to start due to NVIDIA driver deadlock | NVIDIA proprietary driver races with WebKitGTK GPU initialization | Retry startup; use `WEBKIT_DISABLE_COMPOSITING_MODE=1` env var | M1: document for NVIDIA users. |
| L5 | zed [#15959](https://github.com/zed-industries/zed/issues/15959) | Session restore does not work on Wayland | Wayland does not provide window position to applications; `window.position()` returns (0,0) | Skip window position restore on Wayland; only restore size | M1: session restore should handle missing position gracefully. |
| L6 | zed [#43952](https://github.com/zed-industries/zed/issues/43952) | Window position not restored on Wayland | Same as L5: Wayland protocol does not expose window coordinates | Accept Wayland limitation; restore only window size | M1: same as L5. |

### WSL

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| L7 | wezterm [#3080](https://github.com/wezterm/wezterm/issues/3080) | Crashes in WSL2 with WSLg | WSLg (Wayland compositor in WSL2) has incomplete GPU support; WebGL initialization crashes | Detect WSLg and force Canvas renderer; set `LIBGL_ALWAYS_SOFTWARE=1` | M1: if Yord runs natively in WSL2 (unlikely for M1), must detect WSLg. More likely Yord runs on Windows with ConPTY. |
| L8 | zed [#44448](https://github.com/zed-industries/zed/issues/44448) | Connection fails with "os error 206" (filename too long) in WSL + direnv | WSL path translation produces paths exceeding `MAX_PATH`; direnv + NixOS creates deeply nested paths | Use extended-length path prefix or avoid deep nesting | M1: edge case for WSL users. |
| L9 | zed [#50910](https://github.com/zed-industries/zed/issues/50910) | cwd path forward slashes converted to backslashes in WSL | Windows-side path normalization incorrectly converts WSL Unix paths to Windows-style backslashes | Detect WSL context and preserve Unix path separators | M1: if Yord stores paths in SQLite, must handle WSL path format for restore. |
| L10 | wezterm (research) | WSL env passthrough incomplete | `WSLENV` variable not forwarded correctly between Windows and WSL contexts | Implement proper `WSLENV` variable handling per Microsoft spec | Post-M1: only relevant if Yord bridges Windows<->WSL contexts. |
| L11 | zed [#48754](https://github.com/zed-industries/zed/issues/48754) | No external agents work in WSL | WSL2 networking does not expose localhost services to Windows host by default | Configure WSL2 networking mode or use `localhost` forwarding | Post-M1: agent concern, not terminal. |

### Terminal / Terminfo

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| L12 | tauri-terminal (research) | `spawn_command()` returns EPERM on Linux | `TIOCSCTTY` fails when process already has a controlling terminal, or `setsid()` not called first | Call `setsid()` before `TIOCSCTTY`; handle EPERM gracefully (shell works anyway) | **M1 relevant.** Yord must call `setsid()` in `pre_exec` before `TIOCSCTTY`. Spec already includes this. |
| L13 | tauri-terminal (research) | Wrong `TERM` variable causes rendering issues | Setting `TERM=xterm-256color` when terminfo entry missing; some programs fall back to vt100 | Set `TERM=xterm-256color`; ship terminfo entry or use `TERM=xterm` as safe fallback | **M1 relevant.** Yord must set `TERM` correctly. Consider shipping a bundled terminfo definition. |
| L14 | wezterm (research) | `FD_CLOEXEC` not set on D-Bus/GPU handles | Linux desktop environments (GNOME/mutter) leak fds through D-Bus connections and GPU handles that lack `FD_CLOEXEC` | `close_random_fds()` after fork -- same pattern as macOS Cocoa leak | **M1 critical.** Same fix as M7/M8. `close_fds::close_open_fds(3, &[])` in `pre_exec`. |
| L15 | wezterm (research) | Flatpak TIOCSCTTY failure | `TIOCSCTTY` fails across container namespace boundaries (Flatpak sandboxing) | `CommandBuilder::set_controlling_tty(false)` for Flatpak environments | M1: Yord is not distributed as Flatpak for M1. Document for future packaging. |
| L16 | zellij [#518](https://github.com/zellij-org/zellij/issues/518) | Zombie child processes not reaped on pane/tab exit | `waitpid()` not called for all children; zombies accumulate | Implement proper `SIGCHLD` handler or periodic `waitpid(-1, WNOHANG)` sweep | **M1 relevant.** Yord must reap children. Use `child.wait()` in the waiter thread, and sweep for orphans on startup (PID check from SQLite). |
| L17 | zellij (research) | Signal shutdown hangs when shell traps SIGTERM | Shells that trap SIGTERM (fish, custom bashrc) delay exit indefinitely | 3-attempt graceful shutdown: SIGTERM x3 with 10ms polling, then SIGKILL | **M1 relevant.** Spec already includes SIGHUP -> 250ms -> SIGKILL pattern. |
| L18 | zellij [#4496](https://github.com/zellij-org/zellij/issues/4496) | Soft limit for open files bumped in child panes | `sysinfo` crate increases `RLIMIT_NOFILE` soft limit as side effect; inherited by children | Call `setrlimit` in `pre_exec` to restore original limits, or avoid `sysinfo` in the spawn path | M1: if Yord uses `sysinfo` crate, test that child processes inherit expected rlimits. |
| L19 | zellij [#1722](https://github.com/zellij-org/zellij/issues/1722) | "Could not find the SHELL variable: NotPresent" | `$SHELL` not set in minimal environments (containers, CI, systemd services) | Fall back to `/bin/sh` or parse `/etc/passwd` for user's shell | **M1 relevant.** Yord must handle missing `$SHELL`. Spec says "Respect `$SHELL` and launch as interactive shell by default" but needs fallback. |
| L20 | zed [#51790](https://github.com/zed-industries/zed/issues/51790) | Terminal leaks `ALACRITTY_WINDOW_ID` env var | Embedded terminal sets Alacritty-specific env var; child processes misdetect terminal capabilities | Filter out internal env vars before PTY spawn; set only standard vars (`TERM`, `COLORTERM`, `TERM_PROGRAM`) | **M1 relevant.** Yord should set `TERM_PROGRAM=yord` and filter out any stale terminal-specific env vars. |
| L21 | zed [#48355](https://github.com/zed-industries/zed/issues/48355) | Raw ANSI escape sequences printed for arrow keys in devcontainer | PTY not properly initialized as controlling terminal; `TERM` not propagated to container | Ensure `TERM` is set in the spawned environment; verify PTY is controlling terminal | M1: container support is post-M1, but `TERM` propagation is M1. |

### Other

| # | Ref | Symptom | Root Cause | Fix/Workaround | Yord Impact |
|---|-----|---------|-----------|----------------|-------------|
| L22 | zed [#23288](https://github.com/zed-industries/zed/issues/23288) | GPU device loss not recoverable | Linux GPU driver crash (NVIDIA/AMD) leaves WebKitGTK in broken state; no recovery mechanism | Implement graceful GPU recovery: detect device loss, reinitialize rendering, or restart WebView | M1: rare but catastrophic. xterm.js Canvas fallback helps. Document for users with unstable GPU drivers. |
| L23 | zed [#25095](https://github.com/zed-industries/zed/issues/25095) | Unable to open or save files after initial edit | Linux kernel inotify limit exhausted; file watcher stops working | Increase `fs.inotify.max_user_watches` in sysctl; document requirement | Post-M1: file watching concern. |
| L24 | zellij [#2613](https://github.com/zellij-org/zellij/issues/2613) | Exits immediately on SLURM compute nodes | SLURM sets unusual `$TERM`, cgroup limits, or missing `/dev/ptmx` | Check PTY device availability before spawn; handle cgroup-limited environments | M1: edge case. Handle PTY open failure gracefully. |
| L25 | zed [#49776](https://github.com/zed-industries/zed/issues/49776) | 450k+ open `/proc/[PID]/stat` file descriptors after 21 hours | Process info polling (`sysinfo` crate) opens `/proc` fds without closing them in a loop | Use scoped fd handling; call `sysinfo` sparingly with proper cleanup | M1: if Yord uses process tree tracking (post-M1 for foreground process display), must manage fd lifetime. |
| L26 | design.md (spec) | `PR_SET_PDEATHSIG` crash safety | Linux-specific: if Yord crashes, child processes should die automatically | Set `PR_SET_PDEATHSIG(SIGTERM)` in `pre_exec` | **M1 critical.** Already in spec. Must implement. |

---

## Cross-Platform Patterns

Issues that manifest on all platforms but with platform-specific root causes or fixes.

### Process Group / Kill Propagation

| # | Ref | Symptom | Platforms | Root Cause | Yord Impact |
|---|-----|---------|-----------|-----------|-------------|
| X1 | zellij [#518](https://github.com/zellij-org/zellij/issues/518), [#1286](https://github.com/zellij-org/zellij/issues/1286), [#3024](https://github.com/zellij-org/zellij/issues/3024) | Zombie processes after pane close | All | Only direct child PID killed; no process group signal | **M1 critical.** Unix: `setsid()` + `kill(-pgid, sig)`. Windows: Job Objects. Already in spec. |
| X2 | zed [#47412](https://github.com/zed-industries/zed/issues/47412), [#34932](https://github.com/zed-industries/zed/issues/34932) | Child processes not terminated when terminal closes | All | No `SIGHUP`/`SIGTERM` sent to process group before close | **M1 critical.** Spec defines 3-step kill: SIGHUP -> poll 250ms -> SIGKILL. Must signal group, not PID. |
| X3 | canopy [#23](https://github.com/The-Banana-Standard/canopy/issues/23) | `close_terminal` relies on Drop impl | All | No explicit kill; relies on PTY master `Drop` which only closes fd, does not signal children | **M1 critical.** Yord must explicitly kill process group before dropping PTY. |
| X4 | zed [#46474](https://github.com/zed-industries/zed/issues/46474), [#43864](https://github.com/zed-industries/zed/issues/43864) | Node.js/LSP processes persist after app exit | All | Spawned processes not in a process group; no cleanup on app exit | **M1 relevant.** Yord's `RunEvent::ExitRequested` handler must walk all Running entities and kill groups. |

### PTY Reader / EOF

| # | Ref | Symptom | Platforms | Root Cause | Yord Impact |
|---|-----|---------|-----------|-----------|-------------|
| X5 | wezterm (research) | EIO vs BrokenPipe on child exit | macOS vs Linux | macOS returns `ErrorKind::Other` for EIO, Linux returns `BrokenPipe`; both mean "child exited" | **M1 critical.** Yord's PTY reader must treat both EIO and BrokenPipe as normal EOF. |
| X6 | wezterm [#4718](https://github.com/wezterm/wezterm/issues/4718) | portable-pty does not report EOF on process exit | Windows | Windows named pipe read blocks indefinitely after ConPTY child exits | **M1 critical.** Yord must have a waiter thread that signals the reader to stop when `child.wait()` returns. |
| X7 | wezterm (research) | Writer must be taken even if unused | All | If `take_writer()` never called, no EOF sent on master drop; child blocks on stdin forever | **M1 critical.** Yord must call `take_writer()` after spawn, even for read-only terminals. |
| X8 | wezterm (research) | Slave must drop before master | All | If master drops first (or slave outlives master), child may hang or receive spurious EIO | **M1 critical.** `drop(pair.slave)` immediately after `spawn_command()`. Spec notes this in research. |

### fd / Resource Leaks

| # | Ref | Symptom | Platforms | Root Cause | Yord Impact |
|---|-----|---------|-----------|-----------|-------------|
| X9 | wezterm (research), zellij (research) | Child inherits GUI toolkit fds | macOS, Linux | Cocoa (Mach ports, IOKit) and GNOME (D-Bus, GPU) fds leak across fork; `FD_CLOEXEC` not universally set | **M1 critical.** `close_fds::close_open_fds(3, &[])` in `pre_exec`. Already in spec. |
| X10 | zellij [#796](https://github.com/zellij-org/zellij/issues/796) | File descriptors leaked to shell processes | Linux | Server PTY master fds not closed in child pre_exec | **M1 critical.** Same fix as X9. |
| X11 | wezterm (research) | Windows thread handle leak | Windows | `CreateProcessW` returns thread handle in `PROCESS_INFORMATION`; not closed | Wrap in `OwnedHandle`. portable-pty should handle this but verify. |

### Resize

| # | Ref | Symptom | Platforms | Root Cause | Yord Impact |
|---|-----|---------|-----------|-----------|-------------|
| X12 | zed [#42365](https://github.com/zed-industries/zed/issues/42365) | Terminal resize does not propagate | All (remote) | Resize event not forwarded through remote connection to PTY | M1: local only, but resize must propagate to PTY via `pty.resize()`. |
| X13 | canopy (research) | No resize debouncing | All | `ResizeObserver` fires on every pixel; floods PTY with `SIGWINCH` | **M1 relevant.** Spec includes 50-100ms resize debounce. Must implement. |

---

## Summary: M1-Critical Platform Bugs

Bugs that must be addressed before M1 ships, grouped by subsystem.

**PTY Spawning:**
- W5: Set stdio handles to `INVALID_HANDLE_VALUE` on Windows spawn
- W7/W8: Validate command and handle `PATHEXT` edge cases on Windows
- M4: Augment PATH for macOS GUI app environment
- M7/M8/L14/X9/X10: Close inherited fds in `pre_exec` (`close_fds` crate)
- L12: Call `setsid()` before `TIOCSCTTY`
- L19: Handle missing `$SHELL` with fallback
- L26: Set `PR_SET_PDEATHSIG(SIGTERM)` on Linux
- X7/X8: `take_writer()` and `drop(slave)` immediately after spawn

**Process Management / Kill Propagation:**
- W20/W23/X1: Job Objects on Windows; `setsid()` + `kill(-pgid)` on Unix
- W13: Correct `TerminateProcess` return value semantics
- W24: Use `GenerateConsoleCtrlEvent` before `TerminateProcess`
- X2/X3/X4: Signal process group, not individual PID

**PTY I/O:**
- W14/X6: Waiter thread to signal reader on child exit (Windows EOF)
- X5: Treat both EIO and BrokenPipe as normal EOF
- W6: Account for ConPTY-injected escape sequences

**Terminal Rendering:**
- L1: WebGL -> Canvas -> DOM fallback for WebKitGTK
- M1 (WKWebView): Buffer PTY output during macOS sleep; re-feed on wake

**Session Restore:**
- W16: Handle ConPTY init failure gracefully (show error, don't crash)
- W19: Normalize Windows paths before spawn
