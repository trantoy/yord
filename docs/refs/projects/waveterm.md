# waveterm

> AI-integrated terminal with drag-and-drop block layouts and durable SSH sessions. Apache-2.0. [GitHub](https://github.com/wavetermdev/waveterm)

## Overview

Open-source terminal emulator with AI chat widgets, file previews, and drag-and-drop block layouts. Go backend + Electron + React 19 + SQLite + xterm.js v5.5. Hierarchy: Client → Window → Workspace → Tab → Block. Blocks differentiated by `Meta.view` (term, preview, codeeditor, waveai, web) — similar to Yord's component-based identity.

Relevant to Yord: most sophisticated scrollback persistence (circular buffer + xterm.js serialize addon snapshots). SQLite with JSON blobs per table. Graceful kill: SIGHUP → 400ms → SIGKILL. Controller registry pattern (blockId → Controller interface). Durable remote shells via job manager.

## Architecture Decisions

**Go instead of Rust**
Goroutines map naturally to per-shell concurrency (PTY read, input, wait loops) with channels for coordination. Go's implicit interface satisfaction makes the `ConnInterface` abstraction (local PTY, SSH, WSL) low-friction. Go's SSH library is battle-tested vs Rust's less mature ecosystem. Trade-off: no memory safety guarantees; codebase has `sync.Mutex`/`sync.Once` everywhere and several `time.Sleep(100ms)` timing hacks suggesting concurrency bugs.

**Electron instead of Tauri**
Go backend runs as a separate server process; Electron connects via WebSocket. Multi-window support is critical (Window → Workspace mapping). Stderr pipe used for server-to-Electron events (`WAVESRV-EVENT:...` prefix) — creative but brittle. Trade-off: large binary, high memory, WebSocket+JSON IPC adds latency vs Tauri's direct IPC.

**JSON blobs per table instead of normalized schema**
Each object type gets a table `db_{otype}` with columns `(oid TEXT, version INT, data BLOB)`. The `Meta` field is `map[string]any` with 100+ possible keys across entity types — normalizing would require a column per key plus migrations on every change. Generic CRUD (`DBGet[T]`, `DBUpdate`, etc.) works across all types. `json_extract()` used for queries when needed. Trade-off: no foreign keys, no referential integrity, no indexes on JSON fields. Finding a tab for a block requires walking up to 5 parent levels.

**Separate filestore DB for terminal output**
Two SQLite databases: `waveterm.db` (workspace objects, low-write) and `filestore.db` (terminal output, high-write). Prevents PTY byte appends from blocking metadata reads. Filestore has a full write cache with background flusher every 5 seconds; wstore does direct writes. File data split into 64KB parts for partial reads and circular buffer support. Trade-off: two connections, two migration systems, potential cross-DB consistency issues.

**Circular buffer for scrollback**
Terminal output files created with `Circular: true, MaxSize: 2MB`. Logical `Size` grows monotonically but data wraps via `partIdx = (offset / partDataSize) % maxPart`. Reads before `DataStartIdx()` are truncated; writes before circular start are dropped. Bounded disk per terminal regardless of session length. Trade-off: lose old scrollback (~40-50K lines at 2MB). No per-terminal size configuration.

**Pub/sub event broker (wps)**
Scoped pub/sub with event types and scope filters (e.g., `block:{uuid}`, wildcard `*`/`**`). Connects backend terminal writes → `Event_BlockFile` with base64 data → WebSocket → xterm.js. Decouples producers from consumers; persistent events allow late subscribers to catch up. Trade-off: single mutex-protected broker could bottleneck under heavy output; every PTY byte gets base64-encoded and JSON-serialized.

**WebSocket for IPC**
One WebSocket per browser window with `stableId`. Custom application-level ping/pong (JSON, not WebSocket protocol pings). RPC and file data (base64-encoded terminal output) multiplexed on same connection. Trade-off: 33% inflation from base64; `wsMaxMessageSize = 10MB` and `WebSocketChannelSize = 128` suggest throughput issues were hit.

**Controller registry pattern**
Global `map[string]Controller` maps blockId to controller instance. Interface: `Start`, `Stop`, `GetRuntimeStatus`, `SendInput`. Three implementations: `ShellController`, `DurableShellController`, `TsunamiController`. `ResyncController()` is the single idempotent entry point. Atomic replacement: stop old controller before inserting new. Trade-off: uses `time.Sleep(100ms)` in 5 places after stopping controllers — timing hack to avoid cleanup races.

**Block differentiation via Meta.view / Meta.controller**
Blocks are generic containers; `meta.view` selects the frontend component, `meta.controller` selects the backend controller. Adding new types requires zero schema changes. Blocks can change type at runtime. Trade-off: no type safety (a block with `view: "term"` but no controller silently does nothing), 100+ field god-struct for MetaTSType, querying by view requires `json_extract` full table scan, no validation of view/controller combinations.

## Bugs Encountered & How Solved

- **IME double-input** (`termwrap.ts`): xterm.js fires `onData` during IME composition, duplicating CJK characters. Fixed by tracking composition state (`isComposing`, `composingData`, etc.) and suppressing `onData` events during composition unless within 20ms of `compositionend`. Blur during composition resets stale state.
- **Paste deduplication** (`termwrap.ts`): xterm.js `paste()` triggers an internal `onData`, doubling pasted text. Fixed by tracking `lastPasteData`/`lastPasteTime` with a `pasteActive` flag and 30ms suppression timeout.
- **WebGL context loss** (`termwrap.ts`): GPU driver resets or resource pressure lose the WebGL context. `WebglAddon.onContextLoss` triggers automatic fallback to Canvas renderer. WebGL support detected upfront.
- **PTY read loop + process wait ordering** (`shellcontroller.go`): Three goroutines per shell (PTY read, input, wait). Read loop cleanup sets `ShellInputCh = nil`, calls `Wait()`, then sleeps 100ms before closing input channel — workaround for race with wait loop also calling `Wait()`. `sync.Once` on `CmdWrap.Wait()` prevents double-wait panics.
- **Graceful kill** (`conninterface.go`): Local shells get SIGHUP, non-shell commands get SIGTERM, then SIGKILL after 400ms timeout. SSH: immediate Kill (signals unreliable over SSH). WSL: `os.Interrupt` then force Kill. Controller `Stop()` releases lock during wait to prevent deadlocks.
- **Terminal state reset on reconnect** (`shellcontroller.go`): Appends `\033c` reset sequence plus newline to blockfile on shell exit or restart, preventing xterm.js from inheriting stale colors, cursor position, or alt-screen state.
- **Scroll position preservation during resize** (`termwrap.ts`): Tracks `lastScrollAtBottom` with a 1-second grace period. After resize, scrolls back to bottom with 20ms delay if user was recently at bottom.
- **Repaint transaction scroll management** (`termwrap.ts`): Programs like `htop`/`vim` use synchronized update mode (DEC mode 2026) with clear-scrollback (CSI 3J). Intercepted CSI sequences track repaint transactions; scrolls to bottom after transaction completes.

## Current Bugs & Tech Debt

- **Timing hacks in controller lifecycle** (`blockcontroller.go`): `time.Sleep(100ms)` in 5 places during `ResyncController()`. Comment: `// TODO better sync here (don't let two starts happen at the same times)`. Classic "sleep and hope" — no synchronization primitive to know when cleanup is complete.
- **Input channel close race** (`shellcontroller.go`): 100ms sleep before closing `shellInputCh`. Proper fix would use a context or done channel.
- **Missing connection ready wait** (`wslconn.go`, `conncontroller.go`): 300ms sleep with `// TODO remove this sleep`. Connserver has no readiness signal.
- **No quoting validation for remote commands** (`shellexec.go`): `// TODO check quoting of cmdStr` — user commands passed to remote shells via `-c` aren't shell-escaped.
- **Flush-in-progress handling on shutdown** (`main-server.go`): `// TODO deal with flush in progress` — data could be lost if flusher is mid-write at exit.
- **Private API reliance in fit addon** (`fitaddon.ts`): `// TODO: Remove reliance on private API` — accesses private xterm.js internals for dimension calculation.
- **Favicon cache not using blockstore** (`faviconcache.go`): `// TODO store in blockstore`.
- **FlushErrors data loss** (`blockstore_cache.go`): After 3 flush failures, cache entry is cleared entirely — terminal data silently dropped. Error count never resets on success.
- **No timeout on frontend RPC calls** (`wshclient.ts`): `// TODO implement a timeout` — hung backend calls wait forever.
- **Stale circular buffer reads** (`termutil.ts`): Defensive clamping to valid buffer range; known xterm.js buffer issue.
- **Physical vs logical line mismatch** (`term-wsh.tsx`): Line numbers off when wrapping; noted as not worth fixing.
- **v0.12 cleanup debt** (`wslconn.go`, `conncontroller.go`): Backward-compat code that should have been removed.

## What To Learn

- **Circular buffer + serialize addon for scrollback restore**: Storage layer uses circular file (2MB max, 64KB parts in SQLite). After 100KB processed and 5s idle, `serializeAddon.serialize()` captures full terminal state as a `cache:term:full` file with `ptyoffset` metadata. Restore: load cache snapshot (temporarily resize xterm.js if needed), then replay only data written after the snapshot. Avoids replaying entire PTY history.
- **Graceful kill pattern**: SIGHUP for shells (notifies children), SIGTERM for commands. 400ms timeout then SIGKILL. SSH: immediate kill (signals unreliable). WSL: `os.Interrupt` then kill. For Yord: use `Setpgid: true` + signal negative PID to kill entire process group.
- **Controller registry (idempotent resync)**: `ResyncController()` is single entry point — same input = same result regardless of current state. Atomic replacement stops old controller before inserting new. Per-block mutex prevents concurrent resyncs. Versioned status updates (monotonic `VersionTs`) prevent out-of-order frontend event processing. For Yord: frontend calls "ensure this block has a controller" rather than explicit create/destroy.
- **SQLite setup**: WAL mode + single connection + 5s busy timeout. WAL allows concurrent reads with one writer. Single connection serializes writes (no benefit from pooling with SQLite's single-writer limitation). Migrations via embedded SQL files. For Yord: consider adding `PRAGMA synchronous=NORMAL` (safe with WAL) and `PRAGMA mmap_size` for faster reads.
- **xterm.js addon stack**: FitAddon (custom fork), SerializeAddon (session restore), SearchAddon, WebLinksAddon, WebglAddon (with Canvas fallback on context loss), CanvasAddon. Custom OSC handlers: OSC 7 (cwd), OSC 52 (clipboard), OSC 16162 (waveterm protocol). Custom CSI handlers for repaint transaction tracking (mode 2026, CSI 3J). IME handling via compositionstart/update/end events with 20ms post-composition window.
- **Durable remote shell pattern**: `DurableShellController` holds a `JobId` instead of a `ShellProc`. Job runs remotely via job manager. `Start()` reconnects to existing job if alive. `Stop()` without `destroy` is a no-op (job keeps running). Input delivery uses session ID (random UUID) + monotonic sequence number for exactly-once semantics across reconnections. Key insight: process lifecycle separated from UI lifecycle.

## Issues

- **Terminal output throughput through WebSocket**: Every PTY byte goes through Go read → filestore cache write → base64 encode → JSON serialize → WebSocket → JS deserialize → JSON parse → base64 decode → xterm.js write. 33% overhead from base64 on every chunk. Yord should use binary WebSocket frames or Tauri's direct IPC.
- **Timing hacks indicate missing synchronization**: 5x `time.Sleep(100ms)` in `ResyncController()` plus 1x in PTY read loop cleanup. For Yord: use done channel/future from `Stop()` that callers await before creating a new controller.
- **No process group management**: `KillGraceful()` signals `Cmd.Process.Pid` directly, not a process group. Children can become orphans. For Yord: use `Setpgid: true` in `SysProcAttr`, signal `-pid` to kill entire group.
- **Session restore lossy on terminal size change**: Temporary resize to old dimensions, write cache, resize to new dimensions. Reflow can corrupt wide chars, alt-screen state, or column-dependent programs.
- **Circular buffer max size not configurable per terminal**: `DefaultTermMaxFileSize = 2MB` is hardcoded. `term:scrollback` meta key only controls xterm.js in-memory scrollback, not persistent buffer size.
- **Write cache flusher can drop data silently**: 5-second flush interval means up to 5s of output lost on crash. After 3 flush errors per entry, all buffered data cleared.
- **Frontend-driven lifecycle creates cold-start latency**: Shell process only starts after frontend renders block, creates xterm.js, and calls `resyncController()` RPC. Visible delay before shell prompt. For Yord: consider eager process start on session restore.
- **JSON blob storage prevents efficient bulk queries**: No indexes on JSON fields. Queries like `DBGetBlockViewCounts()` require full table scan with `json_extract()`. Yord's ECS model with separate indexed component tables avoids this.

## Key Files

| File | Why |
|------|-----|
| `refs/waveterm/pkg/shellexec/shellexec.go` | PTY spawning for local/SSH/WSL |
| `refs/waveterm/pkg/shellexec/conninterface.go` | ConnInterface with graceful kill |
| `refs/waveterm/pkg/blockcontroller/shellcontroller.go` | Output/input loops, process lifecycle |
| `refs/waveterm/pkg/blockcontroller/blockcontroller.go` | Controller registry, block close, shutdown |
| `refs/waveterm/pkg/waveobj/wtype.go` | All object types (Client, Window, Workspace, Tab, Block, Job) |
| `refs/waveterm/pkg/wstore/wstore_dbsetup.go` | SQLite setup (WAL mode, single conn) |
| `refs/waveterm/pkg/wstore/wstore_dbops.go` | Generic CRUD for wave objects |
| `refs/waveterm/pkg/filestore/blockstore.go` | Circular buffer for terminal scrollback |
| `refs/waveterm/frontend/app/view/term/termwrap.ts` | xterm.js wrapper with addons, IME, paste handling |
