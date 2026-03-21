# IDE & Terminal Tool Pain Points — Research Summary

Research from VS Code, Neovim, tmux/zellij, and Tauri+xterm.js ecosystems.
Compiled to inform Yord's M1 implementation decisions.

---

## P0 — Must get right in M1

### Process group kills (not single PID)
**Source:** VS Code #1 complaint, Neovim #1 complaint, tmux users
- `npm run dev` spawns `node server.js`. Killing the shell PID leaves node running.
- **Fix:** `setsid()` on spawn to create new process group. Kill with `kill(-pgid, SIGTERM)`. Store PGID in `Running` component.
- On Windows: use Job Objects (only reliable way to kill process trees).
- On Linux: set `PR_SET_PDEATHSIG(SIGTERM)` so children die if Yord crashes.

### PTY output batching
**Source:** VS Code terminal freezes, Neovim terminal flooding, Tauri IPC overhead
- `cat /dev/urandom` or verbose builds can produce MB/s of output.
- Sending per-read events overwhelms Tauri IPC and xterm.js rendering.
- **Fix:** Rust-side ring buffer per PTY. Flush every 16ms (60fps) or at 4KB threshold. Use Tauri 2 `Channel<T>` instead of `emit()`.

### All terminals in one webview
**Source:** Tauri multi-webview performance
- Each Tauri `<webview>` is a separate OS process. 10+ webviews = massive memory.
- **Fix:** Render ALL xterm.js instances in the main Svelte app webview. Reserve Tauri webviews for browser/editor nodes (M3).

### Keyboard shortcut safety
**Source:** VS Code, Neovim, every terminal app
- NEVER intercept `Ctrl+C`, `Ctrl+D`, `Ctrl+Z`, `Ctrl+L`, `Ctrl+R` at app level.
- Use `Cmd`/`Super`-based shortcuts for app actions (already in design).
- Check `event.isComposing` before acting on key events (IME safety).

### Focus management
**Source:** Neovim terminal mode confusion, Tauri webview focus bugs
- Only one xterm.js instance gets `.focus()` at a time.
- Visual focus indicator (border color) so user always knows which pane receives input.
- On Linux/WebKitGTK, `focus`/`blur` events can be unreliable — track focus in Svelte store.

---

## P1 — Important for quality

### Rust-side scrollback buffer
**Source:** VS Code loses scrollback on restart, macOS WKWebView jettisoning
- macOS can kill WKWebView under memory pressure. All xterm.js state lost.
- **Fix:** Keep a ring buffer (e.g., 256KB) per PTY in Rust. Re-feed to xterm.js on reconnect or restore. Enables optional scrollback persistence later.

### PTY resize debouncing
**Source:** VS Code, Neovim resize races
- Resize during pane divider drag sends hundreds of events.
- **Fix:** Debounce 50-100ms. Send fresh resize when hidden terminal becomes visible.

### xterm.js renderer fallback
**Source:** Linux WebKitGTK WebGL issues
- WebGL may fail on older Linux distros (Ubuntu 22.04, WebKitGTK 2.38).
- **Fix:** Try WebGL → Canvas → DOM. Catch errors at each step.

### Font bundling
**Source:** Cross-platform font rendering differences
- Different webview engines render fonts differently. CJK, ligatures, emoji diverge.
- **Fix:** Ship a monospace font (JetBrains Mono or similar). System monospace as fallback.

### Session restore validation
**Source:** Neovim session plugins, tmux-resurrect
- `cwd` may no longer exist. `cmd` may no longer be on PATH.
- **Fix:** Validate before respawn. Show clear error in tree if restore fails. Don't silently swallow.

### SQLite WAL mode
**Source:** Neovim shada corruption from concurrent writes
- **Fix:** `PRAGMA journal_mode=WAL`. Use transactions for atomic persist. Single writer.

### Stale PID cleanup
**Source:** VS Code orphan processes
- On startup, check each `Running` component's PID — is it still alive?
- If alive and matches, try reattach. If dead, transition to Crashed/Stopped.
- Store PIDs in SQLite so even after crash, next launch can clean up.

---

## P2 — Good defaults

### Scrollback limits
- Default 5,000-10,000 lines in xterm.js. Unlimited kills memory (~2-5MB per instance).
- Configurable per entity via `Pty` component.

### Lazy xterm.js instantiation
- Only create xterm.js instance when terminal is visible in layout.
- Hidden terminals: buffer PTY output in Rust, don't render.

### Kill propagation timeouts
- SIGTERM → wait 5s → SIGKILL. Configurable per component type (Docker needs more time).

### Cross-platform CSS
- Avoid `backdrop-filter` (broken on WebKitGTK).
- Don't rely on custom scrollbar CSS (inconsistent).
- Use xterm.js built-in scrollbar.

### xterm.js search addon
- Integrate `@xterm/addon-search`. Bind `Cmd+F` within terminal panes.

### Startup speed
- Show UI skeleton first (<500ms), then spawn PTYs asynchronously.
- Tree renders immediately with "starting..." indicators.

---

## Yord's architectural advantages over VS Code / Neovim

| Problem | VS Code / Neovim | Yord |
|---------|-----------------|------|
| Orphan processes | Single PID kill, no process groups | Process group kill + PR_SET_PDEATHSIG |
| Terminal confined to panel | Terminal is second-class UI citizen | All entities are equal in tiling layout |
| Workspace switch kills everything | Full reload, processes killed | Layout swap only, processes keep running |
| No cross-project awareness | Each window is an island | ECS query across all entities |
| Terminal scrollback lost on restart | Not persisted | Rust ring buffer, optional persist |
| Memory grows unboundedly | Electron + extension host bloat | Tauri native webview, no extensions, lazy render |
| Process visibility | No way to see child processes | Tree panel with status per entity |
| Session restore is fragile | Serializes impossible state | Honest restore: respawn cmd in cwd |

---

## Platform-specific notes

### Windows
- Requires Windows 10 1809+ for ConPTY
- Use Job Objects for process tree management
- WebView2 (Chromium-based) — most capable, fewest issues

### macOS
- WKWebView can be jettisoned under memory pressure — need Rust-side scrollback
- Dead keys / IME can double characters — test accent input
- Generally the most polished experience

### Linux
- WebKitGTK version varies by distro — document minimum (2.42+ recommended)
- Default to Canvas renderer, not WebGL
- Test on Ubuntu 22.04 LTS as baseline
- `PR_SET_PDEATHSIG` available for crash safety

---

*Research date: 2026-03-21*
*Sources: VS Code GitHub issues, Neovim community, tmux/zellij ecosystems, Tauri community benchmarks*
