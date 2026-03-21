# PTY Output Buffering Research

> Based on analysis of reference projects. All code is pseudocode/sketches — see [docs/refs/README.md](../../refs/README.md) for rules.

Research into how wezterm and waveterm handle PTY output buffering,
coalescing, and delivery to the terminal renderer.

## Wezterm: Two-Thread Coalescing Pipeline

Source: `refs/wezterm/mux/src/lib.rs`

### Buffer Size and Structure

- **Read buffer**: 1 MB (`BUFSIZE = 1024 * 1024`) for the PTY reader thread
- **Socket buffer**: 1 MB each for send and receive sides of an internal socketpair
- **Parser buffer**: 128 KB (`default_mux_output_parser_buffer_size`) for the parser thread
- No ring buffer; uses a linear `Vec<u8>` reallocated each loop iteration from config

### Data Flow: PTY fd -> buffer -> consumer

Two dedicated threads per pane, connected by an OS socketpair.
**Thread 1** does only blocking reads from the PTY fd (into a 1 MB buffer) and
writes raw bytes into one end of the socketpair. **Thread 2** reads from the
other end of the socketpair (into a 128 KB buffer), parses escape sequences
into structured action values, applies coalescing (poll with a deadline), and
then dispatches the batched actions to the mux, which performs them on the pane
and emits a pane-output notification.

### Coalescing Strategy

The parser thread implements a **deadline-based coalescing** scheme:

1. After each `read()`, parse bytes into actions
2. If accumulated `action_size < buf.len()` (128 KB) and not in hold mode:
   - Set a **deadline** of `now + coalesce_delay` (default **3 ms**)
   - `poll()` the socketpair rx fd with that remaining delay as timeout
   - If data arrives before deadline: loop back and accumulate more
   - If poll times out: flush the accumulated actions
3. If `action_size >= buf.len()`: flush immediately (buffer full, no delay)
4. **SynchronizedOutput** (DEC private mode 2026): sets `hold = true`, which
   suppresses all flushing until the reset sequence arrives. This batches an
   entire "frame" from well-behaved TUI apps

Key: the coalesce delay is deliberately short (3 ms default). It exists to
combine fragmented writes from unoptimized TUI programs into a single
"frame" without adding perceptible latency.

### Backpressure

- Thread 1 writes to a socketpair with a 1 MB send buffer. If Thread 2 (the
  parser) falls behind, `write_all()` on Thread 1 will block when the socket
  buffer fills. This means PTY reads slow down implicitly.
- The OS kernel PTY buffer (typically 4-64 KB) provides a second layer of
  backpressure: if Thread 1 stops reading, the program writing to the PTY
  will eventually block on its own writes.
- There is no explicit drop/overwrite policy. Data is never discarded.

### Thread Safety

- The `dead` flag is a shared `Arc<AtomicBool>` between the two threads
- `pane` is passed as `Weak<dyn Pane>` -- if the pane is removed from the
  mux, the Weak ref fails to upgrade and `dead` is set
- `send_actions_to_mux` dispatches to the main thread via
  `spawn_into_main_thread` for non-main-thread callers
- All Mux state behind `parking_lot::RwLock` / `Mutex`


## Waveterm: Circular BlockFile + Event Bus

Sources:
- `refs/waveterm/pkg/filestore/blockstore.go`
- `refs/waveterm/pkg/filestore/blockstore_cache.go`
- `refs/waveterm/pkg/blockcontroller/shellcontroller.go`
- `refs/waveterm/pkg/blockcontroller/blockcontroller.go`

### Buffer Size and Structure

- **PTY read buffer**: 4 KB (`buf := make([]byte, 4096)`) in the shell controller
- **Circular blockfile**: 2 MB (`DefaultTermMaxFileSize = 2 * 1024 * 1024`),
  created with `Circular: true` option
- **Part size**: 64 KB (`DefaultPartDataSize = 64 * 1024`), so 32 parts for a 2 MB file
- The circular file is a **logical ring**: `Size` grows monotonically (acts as
  a write cursor), and `partIdxAtOffset` uses `partIdx % maxPart` to wrap
  around. Old data is implicitly overwritten.

### Data Flow: PTY fd -> buffer -> consumer

Single goroutine per shell, no coalescing. The goroutine reads from the PTY
in 4 KB chunks, then for each chunk it does two things: (1) appends the data
to an in-memory circular cache (computing a part index from the offset modulo
the max part count and copying data into the corresponding cache entry, while
monotonically updating the file size), and (2) publishes a block-file event
through a broker, with the raw terminal data base64-encoded inline in the
event payload.

The frontend (xterm.js) subscribes to these block-file events and receives
the terminal data via the event bus as base64-encoded payloads. There is no
separate "pull" mechanism for streaming data -- it is pushed every single read.

### Background Flusher (Cache -> SQLite)

A separate goroutine runs `runFlusher()`:
- Sleeps for `DefaultFlushTime = 5 seconds`
- Calls `FlushCache()` which iterates dirty cache entries and writes them to SQLite
- The flush is for **persistence**, not for delivery to xterm.js (that happens
  inline via the event bus)
- Flush errors are tolerated up to 3 times, then the cache entry is cleared

### Coalescing Strategy

**None.** Every PTY read (up to 4 KB) is immediately:
1. Written to the in-memory circular cache
2. Published as a websocket event with the raw data base64-encoded

There is no timer-based batching, no frame coalescing, no poll-based
accumulation. The 4 KB read buffer size inherently limits event frequency
(a 100 MB/s output stream would produce ~25,000 events/sec).

### Backpressure

- **PTY -> cache**: `AppendData` acquires a per-entry mutex. If the lock is
  contended (flush in progress), the PTY read goroutine blocks.
- **Cache -> frontend**: The `wps.Broker.Publish()` call is fire-and-forget.
  If the websocket/event consumer is slow, events may queue in the broker.
  There is no explicit backpressure from the frontend to the PTY reader.
- The circular file acts as a **bounded buffer** for persistence (2 MB max),
  but not as a flow-control mechanism.

### Thread Safety

- `FileStore` has a global `sync.Mutex` for cache key management
- Each `CacheEntry` has its own `sync.Mutex` (two-level locking via `withLock`)
- `ShellController` has a `sync.Mutex` for its mutable state
- The `IsFlushing` flag prevents concurrent flushes


## Comparison Summary

| Aspect | Wezterm | Waveterm |
|--------|---------|----------|
| Architecture | 2 threads + socketpair | 1 goroutine, direct write |
| Read buffer | 1 MB | 4 KB |
| Parser buffer | 128 KB | N/A (no parsing) |
| Storage buffer | N/A (in terminal model) | 2 MB circular blockfile |
| Coalesce delay | 3 ms (configurable) | None |
| Coalesce trigger | poll() timeout or buffer full | N/A |
| Sync output | DEC 2026 hold/release | N/A |
| Backpressure | Socketpair + kernel PTY buf | Mutex contention only |
| Data to frontend | Parsed actions -> render | Raw bytes via event bus |
| Persistence | Terminal model in memory | SQLite (5s flush interval) |


## Recommendation for Yord

Yord's architecture differs from both: it sends raw bytes via Tauri `Channel<T>`
to xterm.js in the webview (no terminal model on the Rust side, no parsed
actions). This is closer to waveterm's model but with a Tauri channel instead of
a websocket event bus.

### Proposed Design: Ring Buffer + Timer-Based Flush

A single dedicated thread per PTY with an in-process ring buffer and a
timer-driven flush loop. No socketpair, no escape sequence parsing on the Rust
side (xterm.js handles that).

**Why a ring buffer instead of a growable Vec:**
- Bounded memory (256 KB per PTY, predictable)
- Old data is irrelevant once xterm.js has parsed it -- if the consumer falls
  behind, dropping old bytes is acceptable since xterm.js maintains its own
  screen buffer
- No allocations in the hot path

**Why timer-based coalescing:**
- Tauri Channel messages have per-message overhead (serialization, IPC boundary
  crossing, JS event dispatch). Sending 4 KB at a time like waveterm would be
  wasteful at high throughput.
- A 4-8 ms flush interval limits messages to 125-250/sec regardless of output
  rate, reducing IPC overhead while staying under the 16 ms frame budget.

**Why NOT the wezterm two-thread model:**
- Wezterm parses escape sequences on the Rust side and sends structured
  `Action` values. Yord sends raw bytes to xterm.js, so there is nothing to
  parse, and the socketpair is unnecessary overhead.

### Concrete Design (Proposed for Yord)

```rust
// PROPOSED SKETCH — original design for Yord, not from any reference project.
use std::sync::Arc;
use std::time::{Duration, Instant};
use tauri::ipc::Channel;
use tokio::io::AsyncReadExt;
use tokio::sync::Notify;
use tokio::time::interval;

/// Per-PTY ring buffer. Single-producer (PTY reader), single-consumer (flush timer).
/// NOT thread-safe on its own -- access is serialized by the owner task.
struct RingBuffer {
    buf: Box<[u8]>,
    /// Write cursor (next byte goes here). Wraps modulo capacity.
    write: usize,
    /// Number of valid bytes in the buffer (0..=capacity).
    len: usize,
}

impl RingBuffer {
    fn new(capacity: usize) -> Self {
        Self {
            buf: vec![0u8; capacity].into_boxed_slice(),
            write: 0,
            len: 0,
        }
    }

    fn capacity(&self) -> usize {
        self.buf.len()
    }

    /// Append data. If data exceeds remaining capacity, the oldest bytes
    /// are silently overwritten (ring semantics).
    fn push(&mut self, data: &[u8]) {
        let cap = self.capacity();
        // If data is larger than the entire buffer, only keep the tail.
        let data = if data.len() > cap {
            &data[data.len() - cap..]
        } else {
            data
        };

        for chunk in [
            // May need two copies if wrapping around the end of the buffer.
        ] {
            // ... simplified below
        }

        let first_len = cap - self.write;
        if data.len() <= first_len {
            self.buf[self.write..self.write + data.len()].copy_from_slice(data);
        } else {
            self.buf[self.write..].copy_from_slice(&data[..first_len]);
            self.buf[..data.len() - first_len].copy_from_slice(&data[first_len..]);
        }

        self.write = (self.write + data.len()) % cap;
        self.len = (self.len + data.len()).min(cap);
    }

    /// Drain all buffered data as a contiguous Vec. Resets the buffer.
    fn drain(&mut self) -> Vec<u8> {
        if self.len == 0 {
            return Vec::new();
        }
        let cap = self.capacity();
        let read_start = (self.write + cap - self.len) % cap;
        let mut out = Vec::with_capacity(self.len);

        if read_start + self.len <= cap {
            out.extend_from_slice(&self.buf[read_start..read_start + self.len]);
        } else {
            out.extend_from_slice(&self.buf[read_start..]);
            out.extend_from_slice(&self.buf[..self.len - (cap - read_start)]);
        }

        self.len = 0;
        // write cursor stays -- doesn't matter since len is 0
        out
    }

    fn is_empty(&self) -> bool {
        self.len == 0
    }
}

const RING_CAPACITY: usize = 256 * 1024; // 256 KB per PTY
const READ_BUF_SIZE: usize = 32 * 1024;  // 32 KB reads from PTY fd
const FLUSH_INTERVAL: Duration = Duration::from_millis(4);
const FLUSH_INTERVAL_MAX: Duration = Duration::from_millis(8);

/// Spawn a PTY output task. Returns a handle that can be used to abort.
///
/// `pty_read`: the async reader for the PTY master fd
/// `channel`: Tauri Channel that delivers `Vec<u8>` to the JS side
fn spawn_pty_output_task(
    mut pty_read: impl AsyncReadExt + Unpin + Send + 'static,
    channel: Channel<Vec<u8>>,
) -> tokio::task::JoinHandle<()> {
    tokio::spawn(async move {
        let mut ring = RingBuffer::new(RING_CAPACITY);
        let mut read_buf = vec![0u8; READ_BUF_SIZE];
        let mut flush_timer = interval(FLUSH_INTERVAL);
        flush_timer.set_missed_tick_behavior(tokio::time::MissedTickBehavior::Delay);

        loop {
            tokio::select! {
                // Priority: always try to read from PTY first
                biased;

                result = pty_read.read(&mut read_buf) => {
                    match result {
                        Ok(0) | Err(_) => {
                            // EOF or error: flush remaining data and exit
                            if !ring.is_empty() {
                                let _ = channel.send(ring.drain());
                            }
                            break;
                        }
                        Ok(n) => {
                            ring.push(&read_buf[..n]);

                            // If buffer is >75% full, flush immediately
                            // (adaptive: don't wait for timer under load)
                            if ring.len > ring.capacity() * 3 / 4 {
                                let _ = channel.send(ring.drain());
                                flush_timer.reset();
                            }
                        }
                    }
                }

                _ = flush_timer.tick() => {
                    if !ring.is_empty() {
                        let _ = channel.send(ring.drain());
                    }
                }
            }
        }
    })
}
```

### Design Rationale

| Decision | Rationale |
|----------|-----------|
| 256 KB ring | Enough for ~5 screenfuls of dense TUI output. Bounded memory per PTY. Larger than wezterm's 128 KB parser buffer because we batch raw bytes, not parsed actions. |
| 32 KB read buffer | Large enough to drain the kernel PTY buffer (typically 4-64 KB on Linux) in one or two reads |
| 4 ms flush interval | One message per 4 ms = 250 msgs/sec max. Well under JS event loop pressure. Stays within half a 16 ms frame budget so xterm.js can process between frames. |
| 75% fill threshold | Adaptive flush under high throughput: if a `cat largefile` is running, don't wait for the timer -- flush when the ring is nearly full. Prevents data loss from ring wraparound. |
| `biased` select | Prioritize reading from PTY over flushing. Ensures we drain the kernel buffer before sending, maximizing batch size. |
| Single async task | No socketpair or thread coordination. The `tokio::select!` loop handles both reading and flushing. Simpler than wezterm's two-thread design because we don't parse escape sequences. |
| `channel.send()` failure ignored | If the webview is closed, the channel is dead. The task will notice on the next PTY read (the process will be killed). No need to propagate the error. |
| No SynchronizedOutput handling | xterm.js handles DEC 2026 internally. Yord sends raw bytes and lets xterm.js decide when to render. |

### Backpressure Consideration

Unlike wezterm, the ring buffer **drops old data** rather than blocking the PTY
reader. This is acceptable because:
1. xterm.js maintains its own scrollback buffer; dropped bytes from the ring
   are bytes that would have scrolled off the visible terminal anyway
2. Blocking the PTY reader would stall the child process, which is worse UX
   than missing some intermediate output during a `cat /dev/urandom` scenario
3. The 75% adaptive flush threshold makes actual data loss unlikely in practice

If backpressure is desired in the future (e.g., for session recording), the
ring can be replaced with a bounded channel that blocks the reader, similar to
wezterm's socketpair approach.
