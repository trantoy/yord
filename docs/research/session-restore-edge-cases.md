# Session Restore Edge Cases — Research from Zellij, Waveterm, Canopy

Research for Yord's progressive 4-phase restore system (design.md section 6.2).

---

## Zellij

**Source files:**
- `zellij-utils/src/session_serialization.rs`
- `zellij-server/src/session_layout_metadata.rs`
- `zellij-utils/src/sessions.rs`

### What data is saved for restore?

Zellij serializes the entire session layout to a KDL file (`session_layout_cache_file_name`). Per pane, it saves:
- **Geometry** (PaneGeom: x, y, rows, cols, stacked status, pinned)
- **Run type** — one of: Command (binary path + args), EditFile (path + line number), Plugin (location + config), Cwd
- **CWD** per terminal pane (read from `/proc/<pid>/cwd` at serialization time)
- **Pane title** (user-set or derived from running command)
- **Focus state** (which pane/tab is focused, per client)
- **Borderless flag, foreground/background colors**
- **Pane contents** (terminal scrollback saved as separate files, referenced via `contents_file`)
- **Tab structure** (tab name, focused, floating pane visibility)
- **Global CWD** (common path prefix extracted from all terminal CWDs)
- **Default shell** (so it can distinguish "running default shell" from "running a command")

The `update_terminal_cwds` method computes a common ancestor path across all terminals and stores it as `global_cwd`, then makes per-pane CWDs relative. This is a key architectural choice: CWDs are portable if a project moves.

### What validation happens before respawning?

- **Default shell detection**: `is_default_shell()` checks if a command matches the configured default shell with no args. If so, `run` is set to `None` (just spawn a new shell, don't try to re-run the exact binary path).
- **Editor detection**: `detect_editor_panes()` identifies panes running vim/nvim/emacs/nano/kak/helix and upgrades them from `Run::Command` to `Run::EditFile`, so on restore they open the file in whatever editor is currently configured (not the exact binary from last time).
- **Cross-matching**: vi/vim/nvim are treated as interchangeable. hx/helix are interchangeable. But vim is NOT cross-matched with emacs.
- **Session socket liveness**: `assert_socket()` connects to the IPC socket and sends `ConnStatus` to verify the server is alive. On connection refused, it cleans up the stale socket file.
- **Dirty check**: `is_dirty()` determines if the current layout differs from the base layout (different pane count, or non-default-shell commands running). Only dirty sessions need to be serialized.

### What fails and how they handle it?

- **Stale socket (dead server)**: On Unix, `assert_socket` tries IPC connect. On `ConnectionRefused`, it deletes the socket file and returns false. On Windows, reads the PID from a marker file and checks with `OpenProcess`; if dead, removes marker.
- **Corrupted layout file**: `resurrection_layout()` tries `Layout::from_kdl()`. If parsing fails, logs the error and returns `Err` with a message. The session is listed as resurrectable but attach fails with an error message.
- **Missing CWD**: No explicit validation. The serialized CWD is used as-is. If it no longer exists, the shell spawns in the default directory (behavior depends on OS/shell).
- **Missing binary**: Commands are serialized with `start_suspended: true`. On resurrection, panes with commands appear with a "press Enter to run" prompt rather than auto-starting. This is the safety mechanism — the user confirms before the command runs.
- **Plugin IDs**: `remove_plugin_from_layout()` strips plugins that are no longer valid.

### Order of restoration

1. Read layout cache file from `ZELLIJ_SESSION_INFO_CACHE_DIR/<session_name>/`
2. Parse KDL layout
3. Recreate tabs in order (focused tab tracked)
4. Recreate panes per tab (tiled first, then floating)
5. Commands start in `start_suspended` mode — user must press Enter
6. Plugins are started automatically
7. Focus is restored (tab focus + pane focus per client)

### User feedback during restore

- Session list shows `(EXITED - attach to resurrect)` for dead sessions with color formatting
- Session list shows `[Created <duration> ago]` timestamps
- Session names are auto-generated (adjective + noun) if not specified
- `suggest` crate provides fuzzy matching for mistyped session names
- No progress bar or restore status indicator during resurrection itself

---

## Waveterm

**Source files:**
- `pkg/wcore/wcore.go` (EnsureInitialData, startup)
- `pkg/wcore/window.go` (CheckAndFixWindow)
- `pkg/blockcontroller/blockcontroller.go` (ResyncController)
- `pkg/blockcontroller/shellcontroller.go` (Start, run, setupAndStartShellProcess)
- `frontend/app/view/term/termwrap.ts` (loadInitialTerminalData)

### What data is saved for restore?

Waveterm stores everything in SQLite via a wave object store (`wstore`). Per block (terminal):
- **Block metadata** (MetaMapType): controller type, connection name, CWD, command, cmd flags (runOnce, runOnStart, clearOnStart, noWsh)
- **Terminal output**: circular blockfile (2MB max, `BlockFile_Term`), stored via `filestore.WFS`
- **Terminal cache**: serialized xterm.js state (`cache:term:full`) with metadata including `ptyoffset` and `termsize`
- **Runtime info**: shell integration status, last command, shell state
- **Window/workspace/tab hierarchy**: window -> workspace -> tabs -> blocks (each is a waveobj with OID)
- **Client data**: window IDs list, install ID, temp OID

### What validation happens before respawning?

**ResyncController** is the key function. It performs a cascade of checks:

1. **Validate IDs**: rejects empty `tabId` or `blockId`
2. **Load block data from DB**: `wstore.DBMustGet` — fails if block doesn't exist
3. **Read controller type and connection name** from block metadata
4. **Connection change detection**: if existing controller's connection differs from block metadata, destroy and recreate
5. **No controller needed**: if `controllerName == ""`, destroy existing controller
6. **Controller type morphing**: checks if existing controller type matches what's needed (ShellController vs DurableShellController vs TsunamiController). Destroys and recreates on mismatch.
7. **Force restart**: destroys existing controller if `force == true`
8. **Done status cleanup**: destroys controllers in `Status_Done` before restarting
9. **Connection status check** (for non-local connections): `CheckConnStatus` verifies SSH/WSL connection is alive before spawning

**CheckAndFixWindow** validates window state on startup:
- If window's workspace can't be loaded, closes the entire window
- If workspace has zero tabs, creates a new tab (self-healing)

### What fails and how they handle it?

- **Missing block in DB**: `DBMustGet` returns error, ResyncController propagates it up. The block is not respawned.
- **Connection not available** (SSH/WSL): `CheckConnStatus` returns error `"not connected: <status>"`. Controller stays in `Status_Init`, not started. User sees the block but shell isn't running.
- **WSL/SSH wsh agent failure**: If starting shell with wsh integration fails, it falls back to `StartWslShellProcNoWsh` / `StartRemoteShellProcNoWsh` (graceful degradation — shell works, just without wave shell integration).
- **Blockfile creation failure**: `filestore.WFS.MakeFile` error stops the shell from starting. If file already exists (`fs.ErrExist`), it resets terminal state instead.
- **Controller already running**: `run()` checks `LockRunLock()` — if already executing, returns silently. Prevents double-start race conditions.
- **Stale blockfile**: On truncate, it also deletes the cache file. Handles `fs.ErrNotExist` gracefully.
- **Terminal size mismatch on restore**: `loadInitialTerminalData` detects if the cache was saved at a different terminal size than the current one. It temporarily resizes the terminal to the cached size, writes the cached data, then resizes back. This prevents garbled output.
- **Missing workspace for tab**: `CheckAndFixWindow` auto-creates a tab if the workspace has none.

### Order of restoration

1. **EnsureInitialData**: Load or create client singleton from SQLite. Validate `TempOID` and `InstallId`.
2. If single window: `CheckAndFixWindow` (validate window -> workspace -> tabs exist, fix if broken)
3. If no windows: create window (and starter workspace on first launch)
4. Frontend loads, xterm.js initializes
5. **loadInitialTerminalData** (frontend):
   a. Fetch cache file (`cache:term:full`) — serialized xterm.js state + ptyoffset
   b. If cache exists, check terminal size. If different, temp-resize to cached size, write cache, resize back.
   c. Fetch main term file from ptyoffset onward (incremental data since cache)
   d. Write remaining data to terminal
6. **resyncController** triggered by first resize event
7. `ResyncController` creates/restarts the shell process
8. Shell output streams via `mainFileSubject` (pub/sub on blockfile appends)

### User feedback during restore

- Console logging: `"terminal loaded cachefile:<N> main:<N> bytes, <N>ms"`
- Console logging: `"terminal restore size mismatch, temp resize"` when cache size differs
- Console logging: `"error controller resync"` on failure
- No explicit UI progress indicator visible in this code — restoration is designed to look instantaneous because the cached terminal state is shown immediately while the actual process starts in the background
- Blocks that fail to start remain visible but show an init/error state

---

## Canopy

**Source file:**
- `src/hooks/useTerminal.ts`

### What data is saved for restore?

Canopy uses **localStorage** (browser storage, not SQLite for tab state). Per tab it saves:
- `id` (UUID)
- `label` (tab name)
- `isClaudeSession` (boolean)
- `isProjectOverview` (boolean flag for non-terminal tabs)
- `projectPath` (filesystem path to project)
- `projectName` (derived from path)
- `isWorkspaceAgent` (boolean)
- `sessionId` (optional, for Claude session resume)

Notably absent: terminal scrollback, CWD, environment, command string, PID, exit status.

### What validation happens before respawning?

- **JSON parse failure**: `loadSavedSessions` wraps everything in try/catch. On any error, returns empty tabs + HOME_TAB_ID. The app starts clean rather than crashing.
- **No process validation**: `terminalId` is explicitly set to `null` on load — all restored tabs are "dead" (no running PTY). They must be relaunched by the user.
- **No CWD or binary validation**: `projectPath` is stored but not checked for existence at restore time.
- **Status mapping**: Project overview tabs get status `"starting"`, all other restored tabs get `"done-success"`. This means terminal tabs show as completed, not as running.

### What fails and how they handle it?

- **Corrupted localStorage**: Caught by try/catch in `loadSavedSessions()`. Returns empty state. Total loss of tab state, but app starts.
- **Missing project path**: No validation. If the project directory was deleted, the tab will be restored pointing to a nonexistent path. Failure occurs only when user tries to relaunch.
- **Stale session IDs**: Claude session IDs are stored for resume but no validation that the session still exists. The relaunch uses the stored `sessionId` and failures are handled at the Claude CLI level.
- **Tab cleanup**: `closeAllDeadTabs()` removes all tabs with done/error status. `relaunchTab()` creates a new tab with the same project settings and removes the dead one.

### Order of restoration

1. Read JSON from `localStorage` key `canopy-tab-sessions`
2. Parse into `SavedSession[]`
3. Map to `TerminalTab[]` with `terminalId: null` (no running PTY)
4. Restore `activeTabId` (or default to HOME_TAB_ID)
5. No processes are spawned automatically — user must interact to start them

### User feedback during restore

- Tabs appear immediately in the tab bar (from localStorage)
- All restored terminal tabs show as `"done-success"` — indicating they need relaunch
- No restore progress indicator
- No error messages for restore failures (silently falls back to empty state)

---

## Compiled Edge Cases for Yord's Session Restore

Based on the patterns and failures observed across all three projects, organized by restore phase:

### Phase 1: Load entities from SQLite (0-200ms)

| Edge Case | Observed In | Recommended Handling |
|---|---|---|
| **Corrupted SQLite database** | Canopy (localStorage equivalent) | Run `PRAGMA integrity_check` on startup. If failed, copy corrupt DB to `.bak`, start fresh, log error. Show user notification with path to backup. |
| **Schema migration needed** | Waveterm (wstore versioning) | Run migrations before loading entities. If migration fails, refuse to start with clear error message showing the version mismatch. |
| **Empty database (first launch)** | Waveterm (EnsureInitialData) | Detect first launch, create default workspace with welcome content. Skip all restore phases. |
| **Window geometry exceeds current display** | None explicitly handled | Clamp saved window position/size to current monitor bounds. Handle monitor count changes (second monitor removed). |

### Phase 2: Stale PID cleanup (200ms-1s)

| Edge Case | Observed In | Recommended Handling |
|---|---|---|
| **PID recycled by OS** | Zellij (assert_socket checks PID liveness) | `kill(pid, 0)` only tells you a process exists, not that it's YOUR process. Store `(pid, start_time)` and compare against `/proc/<pid>/stat` start time on Linux, or `CreateToolhelp32Snapshot` on Windows. If mismatch, treat as dead. |
| **Zombie/orphan processes from previous session** | Waveterm (StopAllBlockControllersForShutdown) | On unclean shutdown, stored PIDs might be zombies. Check process state, not just existence. If zombie, remove `Running` component but DON'T add `Crashed` (we don't know the exit code). Add a `StaleProcess` marker instead. |
| **Process group left behind** | None | Store PGID alongside PID. On stale PID detection, also kill the entire process group (`killpg`) to clean up child processes (dev servers spawning workers, etc.). |
| **Running component exists but PID field is 0/invalid** | Waveterm (CheckAndFixWindow self-healing) | Treat as crashed. Remove `Running`, add `Crashed` with a synthetic "lost process reference" error. |
| **Multiple entities claim same PID** | None | Should not happen in normal operation. If detected, treat all as stale. Log a warning. |

### Phase 3: Validate and spawn (1-5s)

| Edge Case | Observed In | Recommended Handling |
|---|---|---|
| **CWD no longer exists** | Zellij (no validation, shell picks default), Canopy (no validation) | Check `Pty.cwd` with `std::fs::metadata()`. If missing, fall back to parent project's CWD. If that's also gone, fall back to `$HOME`. Show warning badge on the tree node: "Original directory /foo/bar not found, started in /foo". Don't silently skip — Zellij's approach of letting the shell pick a default is confusing. |
| **Command binary not on PATH** | Zellij (start_suspended), Waveterm (error propagation) | Check `which <cmd>` / PATH resolution before spawning. If missing, show the entity as `Crashed` with message "Command not found: <binary>". Don't spawn at all. User can edit the command and retry. |
| **Command binary exists but is not executable** | None | Check file permissions after PATH resolution. Same handling as missing binary. |
| **Shell binary changed** | Zellij (default shell detection, `is_default_shell()`) | If `Pty.cmd` matches the previously-detected default shell, use the CURRENT default shell instead. Don't hardcode `/usr/bin/zsh` if user switched to `/usr/bin/fish`. |
| **SSH/remote connection unavailable** | Waveterm (CheckConnStatus) | Not M1 scope, but design for it: if a Pty has a remote connection component, check connectivity before spawning. If unavailable, show entity as "waiting for connection" rather than crashing. |
| **Environment variable conflicts** | None | Stored env vars from previous session may conflict. Don't restore `TERM_SESSION_ID`, `TMUX`, `STY`, `WINDOWID`, `SHELL_SESSION_ID`, or other session-specific vars. Filter them out like Yord already plans to filter `CLAUDECODE*` vars. |
| **Port already in use** | Waveterm (port reuse for editors) | For dev server entities: the previously-used port may be taken. Detect port conflict, assign new port, update the entity's port component, and show a notification: "Port 3000 was in use, reassigned to 3001". |
| **Terminal size unknown** | Waveterm (getTermSize fallback to 80x25) | If the window hasn't rendered yet when a PTY is spawned, use a sensible default (80x25) and send a resize once the actual dimensions are known. Waveterm does exactly this. |
| **Mass spawn thundering herd** | None explicitly | If 20 entities have `RestoreOnStart`, don't spawn all 20 at once. Use a bounded semaphore (e.g., 4 concurrent spawns). Prioritize: focused/active tab first (Phase 2), then visible tabs, then background tabs. |
| **Spawn timeout** | None | Set a timeout on PTY spawn (e.g., 5 seconds for local, 15 seconds for remote). If spawn doesn't complete, mark as `Crashed` with "Spawn timed out". |
| **Race between restore and user action** | Waveterm (LockRunLock in run()) | User might click "start" on a node while Phase 3 is still spawning. Use per-entity mutex (Waveterm's `blockResyncMutexMap` pattern). Prevent double-spawn. |

### Phase 4: Post-restore (5s+)

| Edge Case | Observed In | Recommended Handling |
|---|---|---|
| **SQLite PRAGMA optimize fails** | None | Non-fatal. Log and continue. |
| **Entity crashes immediately after spawn** | Waveterm (status transitions) | If a process exits within <2 seconds of spawn during restore, don't auto-restart (avoid crash loop). Show as `Crashed` with the exit code. This is different from a crash during normal operation where a restart policy might apply. |

### Cross-cutting concerns

| Edge Case | Observed In | Recommended Handling |
|---|---|---|
| **Restore after unclean shutdown (crash, SIGKILL, power loss)** | Zellij (stale socket cleanup), Waveterm (full DB state) | SQLite with WAL mode handles this well — incomplete transactions are rolled back. The key issue is stale `Running` components (Phase 2). Write a `last_clean_shutdown` timestamp to DB on clean exit. On startup, if missing or stale, run aggressive PID cleanup. |
| **Concurrent Yord instances** | Zellij (session naming, socket locking) | Single-instance enforcement via lock file or IPC socket. If a second instance is launched, activate the existing window instead. Store the lock file path in a well-known location. |
| **Terminal scrollback not preserved** (M1 scope) | Waveterm (full scrollback restore via blockfile + xterm serialize) | M1 explicitly does NOT restore scrollback. Send `\033c` (terminal reset) as the first thing written to each restored PTY. This clears any stale state in xterm.js (cursor position, colors, alternate screen mode). Already planned in design.md. |
| **User expectations mismatch** | Canopy (tabs show as "done"), Zellij ("press Enter to run") | Be explicit. Show a brief toast or tree-node badge: "Session restored: 5 terminals respawned, 1 failed (missing directory)". Don't silently succeed or fail. |
| **Large number of entities** | Waveterm (controlled startup sequence) | If entity count > ~50, show a progress indicator during Phase 3. "Restoring terminals: 12/47". Most users won't hit this but workspace hoarders will. |
| **Restore ordering matters** | Waveterm (focused terminal first) | Some entities may depend on others (dev server must start before the test runner). M1 doesn't handle dependencies, but spawn order should be deterministic: sort by `Order.index` within each parent, process the active/focused entity first. |
