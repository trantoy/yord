# PTY Actor Patterns — Research: Canopy, Zellij, Waveterm

Research for Yord's PTY management layer. Each project solves the same
fundamental problem: managing N concurrent PTY sessions from a single process.

---

## 1. Canopy (Tauri 2 + Rust)

**Source:** `refs/canopy/src-tauri/src/state.rs`, `commands/terminal.rs`

### 1.1 State storage

Single `Arc<Mutex<HashMap<String, TerminalInstance>>>` on `AppState`.
Each `TerminalInstance` holds the PTY master, a `Box<dyn Write + Send>` writer,
and the child process handle. All fields are behind the same mutex.

### 1.2 Command dispatch

Direct Tauri command handlers. `write_to_terminal`, `resize_terminal`, and
`close_terminal` each lock the global `terminals` mutex, look up the ID, and
call the method directly. No message passing, no queuing.

### 1.3 Backpressure

None. `write_to_terminal` calls `writer.write_all()` synchronously while
holding the mutex. If the kernel buffer is full, the Tauri command handler
thread blocks, which blocks the IPC thread for the entire app. Other terminals
cannot be written to until the lock is released.

### 1.4 Reader thread coordination

One `std::thread::spawn` per terminal. The reader loop calls
`reader.read(&mut buf)` in a blocking loop and sends output via a Tauri
`Channel<TerminalEvent>`. When read returns 0 or error, the reader thread:

1. Locks the global mutex and removes the `TerminalInstance`.
2. Calls `child.wait()` outside the lock.
3. Sends an `Exit` event with the exit code.

The reader thread is the cleanup owner. There is no separate waiter.

### 1.5 Death/cleanup

Triggered by one of:
- Reader thread detects EOF/error (normal exit).
- `close_terminal` command: locks mutex, removes entry, calls `child.kill()`.

There is no process group kill, no signal propagation beyond `child.kill()`.

### 1.6 Assessment

Simple and correct for a small number of terminals. The global mutex is a
bottleneck: every write, resize, and close contends on the same lock. No
backpressure means a stuck PTY can block the entire app.

---

## 2. Zellij (Rust terminal multiplexer)

**Source:** `refs/zellij/zellij-server/src/pty.rs`, `thread_bus.rs`,
`pty_writer.rs`

### 2.1 State storage

The `Pty` struct holds several `HashMap<u32, _>` maps:

- `id_to_child_pid` — terminal ID to child PID
- `task_handles` — terminal ID to tokio `JoinHandle` (for the reader task)
- `active_panes` — client ID to active pane
- `originating_plugins` — terminal ID to the plugin that spawned it

There is no `TerminalInstance` struct. The PTY file descriptors are held by the
OS input layer (`ServerOsApi`), not by the Pty struct directly.

### 2.2 Command dispatch

**Thread-per-concern with crossbeam channels.** `ThreadSenders` holds typed
senders to every subsystem:

- `to_pty` — `SenderWithContext<PtyInstruction>`
- `to_screen` — `SenderWithContext<ScreenInstruction>`
- `to_pty_writer` — `SenderWithContext<PtyWriteInstruction>`
- `to_plugin`, `to_server`, `to_background_jobs`

`PtyInstruction` is a large enum (25+ variants): `SpawnTerminal`, `ClosePane`,
`CloseTab`, `ReRunCommandInPane`, `SendSigintToPaneId`, `Exit`, etc.

The main `pty_thread_main()` function is a blocking `loop { match bus.recv() }`
that dispatches each instruction variant. This is effectively the actor pattern
implemented manually: one thread, one mailbox (the channel), message-driven
processing.

### 2.3 Backpressure (the PtyWriter)

**This is Zellij's most important design decision.** Writes do NOT go through
the Pty thread. There is a dedicated `pty_writer` thread with its own channel
and its own instruction enum: `PtyWriteInstruction::Write(bytes, terminal_id,
completion)`.

The writer maintains a `HashMap<u32, VecDeque<PendingWrite>>` — a per-terminal
queue of pending writes. It handles backpressure explicitly:

1. If the kernel accepts fewer bytes than sent (`write` returns < len), the
   remaining bytes stay in the queue with an offset.
2. If `write` returns 0 (EAGAIN), it moves on to the next terminal.
3. If queued bytes exceed `MAX_PENDING_BYTES` (10 MB), the entire queue for
   that terminal is dropped — a deliberate data loss choice to prevent OOM.
4. When pending writes exist, the thread polls with a 10ms timeout instead of
   blocking, so it can retry drains.

**Why a separate thread?** The comment in the source is explicit:

> "we separate these instructions to a different thread because some programs
> get deadlocked if you write into their STDIN while reading from their STDOUT
> (I'm looking at you, vim)"

This is a real and subtle problem. If read and write happen on the same thread
and the program's internal buffer fills up, you get a circular deadlock:
the multiplexer blocks on write, the program blocks on its own write (to
stdout), and nobody makes progress.

### 2.4 Reader thread coordination

Each terminal gets a tokio task (`TerminalBytes::listen()`) spawned via
`async_runtime().spawn()`. The reader sends output to the screen thread via
`send_to_screen(ScreenInstruction::PtyBytes(...))`.

When the child process exits, a `quit_cb` closure fires. This closure is
created at spawn time and captures a clone of `ThreadSenders`. It sends either
`ScreenInstruction::ClosePane` or `ScreenInstruction::HoldPane` depending on
configuration. The PTY thread itself is not involved in the exit path — the
callback goes directly to the screen thread.

### 2.5 Death/cleanup

`close_pane()` on the Pty struct:

1. Aborts the reader `JoinHandle`.
2. Removes the child PID from `id_to_child_pid`.
3. Calls `os_input.kill(child_pid)` — kills by PID.
4. Calls `os_input.clear_terminal_id(id)` — releases the fd.

Exit can be triggered by:
- The `quit_cb` firing when the child exits (reader detects EOF).
- An explicit `PtyInstruction::ClosePane` sent by user action.
- `PtyInstruction::Exit` which breaks the main loop.

### 2.6 Assessment

The most mature architecture of the three. Key insights:

- **Separate writer thread prevents read/write deadlocks.**
- **Per-terminal pending queues with backpressure** prevent OOM.
- **Message-driven dispatch** makes the Pty thread testable and debuggable.
- **Cleanup callbacks captured at spawn time** avoid the need for the Pty
  thread to poll for child exits.

---

## 3. Waveterm (Go)

**Source:** `refs/waveterm/pkg/blockcontroller/blockcontroller.go`,
`shellcontroller.go`

### 3.1 State storage

Global `controllerRegistry` — a `map[string]Controller` protected by a
`sync.RWMutex`. Each block (terminal) is identified by a `blockId` string.

The `Controller` interface requires: `Start`, `Stop`, `GetRuntimeStatus`,
`GetConnName`, `SendInput`. `ShellController` implements this interface and
holds its own `sync.Mutex`, a `ShellProc`, status string, and a
`ShellInputCh chan *BlockInputUnion`.

This is a two-level locking scheme: the registry lock protects the map, each
controller's lock protects its own fields. This avoids the global-lock
bottleneck that Canopy has.

### 3.2 Command dispatch

**Interface-based dispatch with a per-controller input channel.** The public
API (`SendInput`, `DestroyBlockController`) looks up the controller in the
registry and calls its methods directly. But `ShellController.SendInput` does
not write to the PTY directly — it pushes a `*BlockInputUnion` onto
`ShellInputCh`, a buffered Go channel (capacity 32).

A dedicated input goroutine (`range shellInputCh`) reads from this channel and
dispatches:
- `InputData` bytes are written to the PTY via `shellProc.Cmd.Write()`.
- `TermSize` changes call `shellProc.Cmd.SetSize()`.

This is the actor pattern: each controller is an actor with its own mailbox
(the channel) and its own processing goroutine.

### 3.3 Backpressure

Partial. The `ShellInputCh` has a buffer of 32, which provides a small amount
of decoupling. But `shellProc.Cmd.Write()` is called synchronously by the
input goroutine — if the kernel buffer is full, that goroutine blocks. Because
it is a dedicated goroutine (not the main thread), this only affects that
specific terminal. Other terminals are unaffected.

There is no explicit pending-bytes queue or OOM guard like Zellij's.

### 3.4 Reader thread coordination

A dedicated output goroutine reads from `shellProc.Cmd.Read(buf)` in a loop.
Output goes to a circular blockfile via `HandleAppendBlockFile`, which also
publishes a `WaveEvent` (with base64-encoded data) to the frontend via a
pub/sub broker.

When the reader goroutine detects EOF/error:
1. It calls `shellProc.Close()`.
2. Sets `bc.ShellInputCh = nil` under the lock (prevents further input sends).
3. Calls `shellProc.Cmd.Wait()` to get the exit code.
4. Closes `shellInputCh` (stops the input goroutine).

A third goroutine waits on `shellProc.Cmd.Wait()` and updates the controller
status to `Status_Done`.

### 3.5 Death/cleanup

Multiple paths:

- **Normal exit:** Reader goroutine detects EOF, closes the shell proc, closes
  the input channel. The wait goroutine updates status.
- **Block close event:** `handleBlockCloseEvent` fires, calls
  `DestroyBlockController(blockId)` in a new goroutine. This calls
  `controller.Stop(graceful=true)`, which calls `shellProc.Close()` and waits
  on `DoneCh`.
- **Connection change:** `ResyncController` detects a connection name change,
  destroys the old controller, creates a new one.
- **Shutdown:** `StopAllBlockControllersForShutdown` iterates all controllers
  and stops them gracefully in parallel goroutines.

`registerController` will stop an existing controller if one exists for the
same blockId, preventing double-registration.

### 3.6 Assessment

The per-controller actor model is clean. The three-goroutine design (reader,
writer, waiter) provides good isolation. The `Controller` interface makes it
easy to add new controller types (ShellController, DurableShellController,
TsunamiController). The global registry with per-controller locks scales well.

Weaker than Zellij on backpressure (no pending queue, no OOM guard), but the
per-terminal goroutine model means a stuck terminal only blocks itself.

---

## Comparison Matrix

| Dimension | Canopy | Zellij | Waveterm |
|---|---|---|---|
| **State storage** | Global `Mutex<HashMap>` | Per-concern HashMaps on Pty struct | Global registry + per-controller lock |
| **Dispatch** | Direct method calls | Channel + enum (`PtyInstruction`) | Interface method + per-controller channel |
| **Writer isolation** | None (same thread) | Dedicated `pty_writer` thread | Per-controller input goroutine |
| **Backpressure** | None | Per-terminal pending queue, 10MB cap, EAGAIN retry | Buffered channel (32), blocks goroutine on full |
| **Reader** | Blocking thread per terminal | Tokio task per terminal | Goroutine per terminal |
| **Cleanup trigger** | Reader EOF or explicit close | Quit callback + ClosePane instruction | Reader EOF + DoneCh + event-driven destroy |
| **Lock granularity** | One global mutex | No user-facing locks (channel-isolated) | Registry RWMutex + per-controller mutex |
| **Scalability** | Poor (global lock) | Good (channel isolation) | Good (per-controller isolation) |

---

## Recommendation for Yord

### The Pattern: Tokio Actor per PTY, Dedicated Writer Task

Yord should combine Zellij's dedicated writer with Waveterm's per-controller
actor model, implemented idiomatically in async Rust with Tokio.

**Why not Canopy's approach?** The global mutex is a known bottleneck. Yord
will manage more terminals than Canopy (which is purpose-built for Claude
sessions). Any stuck PTY would freeze the entire app.

**Why not Zellij's single-thread-per-concern?** Zellij uses one thread for ALL
pty operations. This works for a terminal multiplexer where operations are
fast, but Yord's ECS architecture means PTY operations may involve DB queries
(reading/writing components). A single thread for all PTYs would serialize
those.

**Why not Waveterm's Go pattern directly?** The pattern is right, but Go
channels and goroutines map to Tokio channels and tasks in Rust. The interface
dispatch maps to an enum + match, which is more idiomatic in Rust.

### Architecture

```
                    Tauri IPC
                       │
                       ▼
              ┌─────────────────┐
              │   PtyRegistry   │  HashMap<EntityId, PtyHandle>
              │  (tokio::sync:: │  PtyHandle = { cmd_tx, status }
              │   RwLock)       │
              └────────┬────────┘
                       │ cmd_tx.send(PtyCommand)
                       ▼
              ┌─────────────────┐
              │   Actor Task    │  one per PTY
              │  (tokio::spawn) │  owns: master_fd, child, pending_writes
              │                 │  loop { select! { cmd, read_ready } }
              └─────────────────┘
                     │    │
          output_tx  │    │  write to fd
                     ▼    ▼
              ┌──────────────┐
              │  ECS / Tauri │  update Running component,
              │   events     │  emit TerminalEvent to frontend
              └──────────────┘
```

### Concrete Rust Sketch

```rust
use std::collections::HashMap;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::sync::{mpsc, oneshot, RwLock};

// --- Messages ---

pub enum PtyCommand {
    Write(Vec<u8>),
    Resize { rows: u16, cols: u16 },
    Kill,
    /// Request current status via a oneshot reply.
    GetStatus(oneshot::Sender<PtyStatus>),
}

#[derive(Clone, Debug)]
pub enum PtyEvent {
    Output(Vec<u8>),
    Exited { code: Option<i32> },
}

#[derive(Clone, Debug)]
pub struct PtyStatus {
    pub pid: u32,
    pub running: bool,
}

// --- Handle (what the registry stores) ---

#[derive(Clone)]
pub struct PtyHandle {
    cmd_tx: mpsc::Sender<PtyCommand>,
}

impl PtyHandle {
    pub async fn write(&self, data: Vec<u8>) -> Result<(), PtyError> {
        self.cmd_tx.send(PtyCommand::Write(data)).await
            .map_err(|_| PtyError::ActorDead)
    }

    pub async fn resize(&self, rows: u16, cols: u16) -> Result<(), PtyError> {
        self.cmd_tx.send(PtyCommand::Resize { rows, cols }).await
            .map_err(|_| PtyError::ActorDead)
    }

    pub async fn kill(&self) -> Result<(), PtyError> {
        self.cmd_tx.send(PtyCommand::Kill).await
            .map_err(|_| PtyError::ActorDead)
    }

    pub async fn status(&self) -> Result<PtyStatus, PtyError> {
        let (tx, rx) = oneshot::channel();
        self.cmd_tx.send(PtyCommand::GetStatus(tx)).await
            .map_err(|_| PtyError::ActorDead)?;
        rx.await.map_err(|_| PtyError::ActorDead)
    }
}

// --- Registry (shared state) ---

pub struct PtyRegistry {
    ptys: RwLock<HashMap<String, PtyHandle>>,
}

impl PtyRegistry {
    pub fn new() -> Self {
        Self { ptys: RwLock::new(HashMap::new()) }
    }

    pub async fn spawn(
        &self,
        entity_id: String,
        cmd: Vec<String>,
        cwd: String,
        event_tx: mpsc::UnboundedSender<(String, PtyEvent)>,
    ) -> Result<(), PtyError> {
        let (cmd_tx, cmd_rx) = mpsc::channel::<PtyCommand>(64);

        // Spawn the PTY (platform-specific, synchronous — run in spawn_blocking)
        let pty_pair = tokio::task::spawn_blocking(move || {
            open_pty_and_spawn(&cmd, &cwd)
        }).await??;

        let handle = PtyHandle { cmd_tx };
        self.ptys.write().await.insert(entity_id.clone(), handle.clone());

        // Spawn the actor task
        let entity_id_clone = entity_id.clone();
        let ptys = self.ptys.clone(); // NOT &self — clone the Arc<RwLock>
        tokio::spawn(async move {
            pty_actor_loop(entity_id_clone, pty_pair, cmd_rx, event_tx).await;
            // Self-cleanup: remove from registry when actor exits
            ptys.write().await.remove(&entity_id);
        });

        Ok(())
    }

    pub async fn get(&self, entity_id: &str) -> Option<PtyHandle> {
        self.ptys.read().await.get(entity_id).cloned()
    }

    pub async fn kill_all(&self) {
        let handles: Vec<_> = self.ptys.read().await.values().cloned().collect();
        for handle in handles {
            let _ = handle.kill().await;
        }
    }
}

// --- Actor loop ---

const MAX_PENDING_BYTES: usize = 10 * 1024 * 1024; // 10 MB, same as Zellij

async fn pty_actor_loop(
    entity_id: String,
    mut pty: PtyPair,          // platform-specific: holds master fd + child
    mut cmd_rx: mpsc::Receiver<PtyCommand>,
    event_tx: mpsc::UnboundedSender<(String, PtyEvent)>,
) {
    let mut read_buf = vec![0u8; 4096];
    let mut pending_writes: Vec<u8> = Vec::new();

    loop {
        tokio::select! {
            // --- Read from PTY ---
            result = pty.master_read.read(&mut read_buf) => {
                match result {
                    Ok(0) | Err(_) => break, // EOF or error — PTY died
                    Ok(n) => {
                        let _ = event_tx.send((
                            entity_id.clone(),
                            PtyEvent::Output(read_buf[..n].to_vec()),
                        ));
                    }
                }
            }

            // --- Drain pending writes (only when there are pending bytes) ---
            result = pty.master_write.write(&pending_writes),
                if !pending_writes.is_empty() => {
                match result {
                    Ok(n) => { pending_writes.drain(..n); }
                    Err(_) => break, // write error — PTY dead
                }
            }

            // --- Receive commands ---
            cmd = cmd_rx.recv() => {
                match cmd {
                    None => break, // all senders dropped
                    Some(PtyCommand::Write(data)) => {
                        if pending_writes.len() + data.len() > MAX_PENDING_BYTES {
                            // Backpressure: drop pending to prevent OOM
                            tracing::warn!(
                                entity_id, "dropping {} pending bytes (OOM guard)",
                                pending_writes.len()
                            );
                            pending_writes.clear();
                        }
                        pending_writes.extend(data);
                    }
                    Some(PtyCommand::Resize { rows, cols }) => {
                        let _ = pty.resize(rows, cols);
                    }
                    Some(PtyCommand::Kill) => {
                        let _ = pty.kill_child();
                        break;
                    }
                    Some(PtyCommand::GetStatus(reply)) => {
                        let _ = reply.send(PtyStatus {
                            pid: pty.child_pid(),
                            running: true,
                        });
                    }
                }
            }
        }
    }

    // --- Cleanup ---
    let exit_code = pty.wait_exit().await;
    let _ = event_tx.send((entity_id, PtyEvent::Exited { code: exit_code }));
    // The spawning future removes us from the registry after this returns.
}

// --- Error type ---

#[derive(Debug, thiserror::Error)]
pub enum PtyError {
    #[error("PTY actor is no longer running")]
    ActorDead,
    #[error("PTY spawn failed: {0}")]
    SpawnFailed(String),
}
```

### Why this design

1. **No global mutex for writes.** Each actor owns its PTY fd. Writes go
   through the actor's channel, never through a shared lock. A stuck PTY
   blocks only its own actor task (and Tokio can run thousands of tasks on a
   few threads).

2. **Read and write in the same `select!` loop.** Unlike Zellij's separate
   writer thread, we use `tokio::select!` to multiplex read, write, and
   command processing in a single task. This is safe because we use async I/O
   — `read` and `write` do not block each other at the OS level. The Zellij
   deadlock issue (vim blocking on stdout while we block on stdin) does not
   apply with async I/O because the `select!` loop will always process
   whichever arm is ready first.

3. **Backpressure from Zellij.** The 10 MB pending-bytes cap prevents OOM.
   Unlike Zellij's separate HashMap of queues, we keep pending bytes in a
   simple `Vec<u8>` per actor — no cross-terminal coordination needed.

4. **Self-cleanup from Waveterm.** When the actor loop exits (EOF, error, or
   kill), it removes itself from the registry. The spawning future handles
   this. No separate cleanup thread or polling needed.

5. **ECS integration.** The `event_tx` channel carries `(entity_id, PtyEvent)`
   tuples. A single receiver task can update the ECS store: add/remove
   `Running` and `Crashed` components, update terminal output buffers, and
   emit Tauri events to the frontend. This keeps the actor loop free of DB
   concerns.

6. **Testability.** The actor is driven entirely by its `cmd_rx` channel and
   PTY fd. In tests, you can replace the PTY with a pipe pair and drive the
   actor with synthetic commands. The registry is just a HashMap — no global
   state.

### What to watch out for

- **Async PTY I/O on macOS.** `tokio::io::AsyncFd` wrapping a raw PTY fd works
  on Linux. On macOS, PTY fds may not work with kqueue in edge-triggered mode.
  You may need `tokio::task::spawn_blocking` for the read loop, which makes
  `select!` slightly more complex (read from a `tokio::sync::mpsc` fed by the
  blocking reader, rather than directly from the fd). Test this early.

- **Signal handling.** The sketch uses `pty.kill_child()` which sends SIGKILL.
  For graceful shutdown (Yord's "kill project" feature), send SIGTERM first,
  wait briefly, then SIGKILL. Consider `nix::sys::signal::killpg` to kill the
  entire process group.

- **Session restore.** The actor is ephemeral — it dies with the PTY. For
  restore, the ECS `Pty` component stores the command and cwd. On app restart,
  `RestoreSystem` queries `[Pty, RestoreOnStart]` and calls
  `registry.spawn()` for each.

---

*Research date: 2026-03-21*
*Related to: docs/architecture.md, docs/design.md*
