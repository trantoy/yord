# SQLite Single-Writer + Reader Pool — Reference Research

> Based on analysis of reference projects. All code is pseudocode/sketches — see [docs/refs/README.md](../../refs/README.md) for rules.

Research into how Zed (Rust) and Waveterm (Go) implement SQLite with single-writer architecture. Goal: inform Yord's `rusqlite` setup for Tauri 2.

---

## Zed — `sqlez` crate

Source: `crates/sqlez/src/` + `crates/db/src/db.rs`

### Connection count and management

- **Readers**: One `Connection` per thread, stored in `Arc<ThreadLocal<Connection>>`. When a thread first dereferences `ThreadSafeConnection`, a new SQLite connection is opened and cached in thread-local storage. Every subsequent read on that thread reuses the same connection. This means the reader "pool" grows organically — one connection per thread that touches the DB.

- **Writer**: One dedicated write connection per database file, owned by a background thread. The write connection is never shared; it lives entirely within the background thread's closure.

- **Global registry**: A global `static QUEUES: LazyLock<RwLock<HashMap<Arc<str>, WriteQueue>>>` maps database file URIs to write queues. Multiple `ThreadSafeConnection` instances pointing to the same file share a single write queue (and thus a single writer thread). This prevents accidental double-writer scenarios even when different parts of the codebase independently open the same DB.

### Writer serialization

**Dedicated background thread + `std::sync::mpsc` channel.** Not a mutex, not an async task.

The `background_thread_queue()` function (the default) does:

1. Creates a `std::sync::mpsc::channel::<Box<dyn FnOnce()>>`.
2. Spawns a `std::thread::Builder` named `"sqlezWorker"`.
3. The thread loops on `receiver.recv()`, executing each closure synchronously.
4. The sender is wrapped in `UnboundedSyncSender` and stored in the global queue map.

The `write()` method:
1. Looks up the write channel from the global `QUEUES` map.
2. Creates a `futures::channel::oneshot` for the result.
3. Clones `self`, boxes a closure that: dereferences to get/create a connection, calls `connection.with_write(callback)`, sends the result on the oneshot.
4. Sends the boxed closure to the background thread via the mpsc channel.
5. Returns the oneshot receiver as a `Future` — callers can `.await` it.

The `with_write()` method on `Connection` temporarily sets a `write: RefCell<bool>` flag to `true`, runs the callback, then sets it back to `false`. Reader connections have `write` permanently set to `false` after creation. This is a runtime guard — if code accidentally tries to write on a reader connection, it can be detected.

There is also a `locking_queue()` alternative used in tests: uses a `parking_lot::Mutex` instead of a background thread, executing writes inline under the lock. Simpler but blocks the calling thread.

### PRAGMAs

Two categories, set separately:

**DB initialization query** (run once on the writer, changes the DB file): sets `journal_mode=WAL`, `busy_timeout=500`, `case_sensitive_like=TRUE`, and `synchronous=NORMAL`.

**Connection initialization query** (run on every new connection including readers): sets `foreign_keys=TRUE`.

The separation matters because `journal_mode=WAL` only needs to be set once (it persists in the file) while `foreign_keys` must be set per-connection (it defaults to OFF each time).

Zed opens connections with `SQLITE_OPEN_NOMUTEX` — multi-threaded mode, no SQLite-level mutexing. Safe because each connection is only accessed from one thread.

### Migration system

- **Domain trait**: Each logical domain (workspace DB, key-value store, etc.) implements `Domain` with a `NAME` constant and a `MIGRATIONS` constant (a `&[&str]` slice of SQL strings).
- **Migrator trait**: Composes multiple `Domain`s. Tuple implementations exist for up to 5 domains, so `(WorkspaceDomain, KeyValueDomain)` automatically runs both.
- **Storage**: A `migrations` table with columns `(domain TEXT, step INTEGER, migration TEXT)`. Each migration's SQL text is stored verbatim (after `sqlformat` normalization).
- **Execution**: Migrations run eagerly (via `sqlite3_exec` directly, not prepared statements). This allows multi-statement DDL in a single migration string. Wrapped in `SAVEPOINT`/`RELEASE`.
- **Change detection**: On startup, stored migration text is compared to the current code's migration text (both reformatted with `sqlformat`). If they differ, `should_allow_migration_change()` is called — by default it returns `false` and the app bails. This prevents silent schema drift.
- **Retry logic**: Migrations are retried up to 10 times to handle concurrent processes migrating the same file.
- **Foreign keys**: Disabled during migration, re-enabled after, then `PRAGMA foreign_key_check` is run. Orphaned rows from migration side-effects are cleaned up.

### Async runtime integration

Zed uses `smol`/GPUI's executor, not Tokio. The `write()` method returns a `Future` (oneshot receiver), so callers `.await` the result. The actual SQLite work is never on the async executor — it is on the dedicated `std::thread`. Reads happen synchronously on whatever thread calls `.deref()`. There is no `spawn_blocking`.

---

## Waveterm — `wstore` package

Source: `pkg/wstore/wstore_dbsetup.go`, `pkg/wstore/wstore_dbops.go`, `pkg/util/migrateutil/migrateutil.go`

### Connection count and management

**Single connection only.** `SetMaxOpenConns(1)` is called explicitly on the `sqlx.DB` handle. Go's `database/sql` internally pools connections, but with max-open-conns set to 1 it effectively serializes everything. No reader pool. Readers and writers contend for the same connection.

The `globalDB` variable is a package-level singleton — `InitWStore()` creates it once at startup.

### Writer serialization

**Implicit via `MaxOpenConns(1)`.** Go's `database/sql` package queues connection requests internally. Since only one connection is allowed, all queries (reads and writes) are serialized. There is no explicit mutex, channel, or background goroutine for write serialization.

Transactions are wrapped via `txwrap.WithTx` and `txwrap.WithTxRtn` helpers, which handle begin/commit/rollback. These are synchronous, blocking calls.

### PRAGMAs

Set via the DSN connection string with query parameters: `mode=rwc` (read-write-create), `_journal_mode=WAL`, and `_busy_timeout=5000` (5 second busy timeout, more aggressive than Zed's 500ms).

No `synchronous`, `foreign_keys`, `temp_store`, `mmap_size`, or `cache_size` PRAGMAs are set. This is a minimal configuration.

### Migration system

Uses the standard Go `golang-migrate/migrate` library:

- Migration files are embedded in the binary via `io/fs` (`embed.FS`).
- Files live in a `migrations-wstore/` directory with standard up/down naming.
- `migrate.Up()` runs all pending migrations.
- Dirty state is checked — if the DB is dirty, startup fails with an error (no auto-fix).
- Version tracking is handled by the migrate library's internal `schema_migrations` table.

Simple, off-the-shelf, no custom logic.

### Async runtime integration

Waveterm is synchronous Go. No async runtime. All DB calls are blocking. Goroutines provide concurrency but `MaxOpenConns(1)` means only one goroutine touches the DB at a time.

---

## Comparison Summary

| Aspect | Zed | Waveterm |
|--------|-----|---------|
| Language | Rust | Go |
| Connections | ThreadLocal readers + 1 writer thread | 1 connection total |
| Writer serialization | Background thread + mpsc channel | Implicit (MaxOpenConns=1) |
| Read concurrency | Full (one conn per thread, WAL allows parallel reads) | None (single conn) |
| PRAGMAs | WAL, synchronous=NORMAL, busy_timeout=500, foreign_keys, case_sensitive_like | WAL, busy_timeout=5000 |
| Open flags | SQLITE_OPEN_NOMUTEX | Default (via Go driver) |
| Migrations | Custom (domain/step tracking, text comparison, retry) | golang-migrate library |
| Migration storage | In-DB `migrations` table | In-DB `schema_migrations` table |
| Async integration | Future via oneshot channel | Synchronous (goroutines) |

**Key takeaway**: Zed's architecture is the one to follow. Waveterm's single-connection approach does not scale to concurrent reads, which matter when Tauri commands arrive simultaneously from the UI thread. Zed's ThreadLocal reader pattern is elegant and avoids contention entirely for reads.

---

## Recommended Implementation for Yord

### Architecture (proposed sketch)

```
                  Tauri async command
                         |
            +-----------+-----------+
            |                       |
        [read path]           [write path]
            |                       |
    ThreadLocal<Connection>   tokio::sync::mpsc
    (one per OS thread)       channel to writer
            |                       |
        direct call           spawn_blocking
        (sync, fast)          (dedicated thread)
            |                       |
         rusqlite                rusqlite
         Connection              Connection
         (read-only)            (read-write)
```

### Implementation sketch (proposed for Yord)

```rust
use std::path::{Path, PathBuf};
use std::sync::Arc;
use rusqlite::{Connection, OpenFlags, params};
use thread_local::ThreadLocal;
use tokio::sync::{mpsc, oneshot};

// ---------------------------------------------------------------------------
// Connection setup — shared between reader and writer
// ---------------------------------------------------------------------------

/// PRAGMAs run on every connection (readers and writer).
const CONNECTION_PRAGMAS: &str = "\
    PRAGMA foreign_keys = ON;\
    PRAGMA busy_timeout = 5000;\
    PRAGMA temp_store = MEMORY;\
    PRAGMA case_sensitive_like = TRUE;\
";

/// PRAGMAs run once on the writer to configure the DB file.
const DB_INIT_PRAGMAS: &str = "\
    PRAGMA journal_mode = WAL;\
    PRAGMA synchronous = NORMAL;\
    PRAGMA mmap_size = 268435456;\
    PRAGMA cache_size = -16000;\
";

fn open_connection(path: &Path, readonly: bool) -> rusqlite::Result<Connection> {
    let flags = if readonly {
        OpenFlags::SQLITE_OPEN_READ_ONLY
            | OpenFlags::SQLITE_OPEN_NO_MUTEX   // we guarantee single-thread access
            | OpenFlags::SQLITE_OPEN_URI
    } else {
        OpenFlags::SQLITE_OPEN_READ_WRITE
            | OpenFlags::SQLITE_OPEN_CREATE
            | OpenFlags::SQLITE_OPEN_NO_MUTEX
            | OpenFlags::SQLITE_OPEN_URI
    };

    let conn = Connection::open_with_flags(path, flags)?;
    conn.execute_batch(CONNECTION_PRAGMAS)?;
    Ok(conn)
}

fn open_writer(path: &Path) -> rusqlite::Result<Connection> {
    let conn = open_connection(path, false)?;
    conn.execute_batch(DB_INIT_PRAGMAS)?;
    Ok(conn)
}

// ---------------------------------------------------------------------------
// Database handle — the main API
// ---------------------------------------------------------------------------

type WriteFn = Box<dyn FnOnce(&Connection) + Send + 'static>;

/// Thread-safe database handle. Clone freely, share across Tauri state.
#[derive(Clone)]
pub struct Database {
    path: Arc<PathBuf>,
    readers: Arc<ThreadLocal<Connection>>,
    write_tx: mpsc::UnboundedSender<WriteFn>,
}

impl Database {
    /// Create the database, run migrations, return the handle.
    pub async fn open(path: impl Into<PathBuf>) -> anyhow::Result<Self> {
        let path = Arc::new(path.into());

        // --- Writer thread ---
        let (write_tx, mut write_rx) = mpsc::unbounded_channel::<WriteFn>();

        let writer_path = path.clone();
        std::thread::Builder::new()
            .name("yord-db-writer".into())
            .spawn(move || {
                let conn = open_writer(&writer_path)
                    .expect("Failed to open writer connection");

                // Run migrations on the writer before accepting work
                run_migrations(&conn).expect("Migrations failed");

                while let Some(f) = write_rx.blocking_recv() {
                    f(&conn);
                }
                // Channel closed — app is shutting down.
            })?;

        // --- Reader pool (ThreadLocal) ---
        let readers = Arc::new(ThreadLocal::new());

        Ok(Self { path, readers, write_tx })
    }

    // -- Read path ----------------------------------------------------------

    /// Get a reference to this thread's reader connection.
    /// Call from sync context (inside spawn_blocking).
    pub fn reader(&self) -> &Connection {
        self.readers.get_or(|| {
            open_connection(&self.path, true)
                .expect("Failed to open reader connection")
        })
    }

    /// Run a read query from an async Tauri command.
    /// Moves the closure to a blocking thread, returns the result.
    pub async fn read<F, T>(&self, f: F) -> T
    where
        F: FnOnce(&Connection) -> T + Send + 'static,
        T: Send + 'static,
    {
        let db = self.clone();
        tokio::task::spawn_blocking(move || f(db.reader()))
            .await
            .expect("Read task panicked")
    }

    // -- Write path ---------------------------------------------------------

    /// Queue a write and await its result.
    /// The closure runs on the dedicated writer thread.
    pub async fn write<F, T>(&self, f: F) -> T
    where
        F: FnOnce(&Connection) -> T + Send + 'static,
        T: Send + 'static,
    {
        let (result_tx, result_rx) = oneshot::channel();

        self.write_tx
            .send(Box::new(move |conn| {
                let result = f(conn);
                let _ = result_tx.send(result);
            }))
            .expect("Writer thread has shut down");

        result_rx.await.expect("Writer dropped result")
    }

    /// Fire-and-forget write (no result needed).
    pub fn write_fire(&self, f: impl FnOnce(&Connection) + Send + 'static) {
        self.write_tx
            .send(Box::new(f))
            .expect("Writer thread has shut down");
    }
}

// ---------------------------------------------------------------------------
// Migrations
// ---------------------------------------------------------------------------

/// Simple, append-only migrations. Each entry is a SQL string.
/// Never modify or delete entries — only append.
const MIGRATIONS: &[&str] = &[
    // 0: initial schema
    "CREATE TABLE IF NOT EXISTS entities (
        id         TEXT PRIMARY KEY,
        created_at INTEGER NOT NULL,
        updated_at INTEGER NOT NULL
    );
    CREATE TABLE IF NOT EXISTS components (
        entity_id      TEXT NOT NULL REFERENCES entities(id) ON DELETE CASCADE,
        component_type TEXT NOT NULL,
        data           TEXT NOT NULL,
        PRIMARY KEY (entity_id, component_type)
    );
    CREATE INDEX IF NOT EXISTS idx_component_type ON components(component_type);
    CREATE INDEX IF NOT EXISTS idx_belongs_to
        ON components(component_type, json_extract(data, '$.parent_id'))
        WHERE component_type = 'BelongsTo';",

    // 1: next migration goes here
];

fn run_migrations(conn: &Connection) -> anyhow::Result<()> {
    // Create migration tracking table
    conn.execute_batch(
        "CREATE TABLE IF NOT EXISTS _migrations (
            version INTEGER PRIMARY KEY,
            applied_at INTEGER NOT NULL
        );"
    )?;

    let current_version: i64 = conn
        .query_row(
            "SELECT COALESCE(MAX(version), -1) FROM _migrations",
            [],
            |row| row.get(0),
        )?;

    // Foreign keys OFF during migrations (Zed pattern)
    conn.execute_batch("PRAGMA foreign_keys = OFF;")?;

    let tx = conn.transaction()?;
    for (i, sql) in MIGRATIONS.iter().enumerate() {
        let version = i as i64;
        if version > current_version {
            tx.execute_batch(sql)?;
            tx.execute(
                "INSERT INTO _migrations (version, applied_at) VALUES (?1, unixepoch())",
                params![version],
            )?;
        }
    }
    tx.commit()?;

    conn.execute_batch("PRAGMA foreign_keys = ON;")?;
    conn.execute_batch("PRAGMA foreign_key_check;")?;

    Ok(())
}

// ---------------------------------------------------------------------------
// Tauri integration
// ---------------------------------------------------------------------------

// In main.rs / lib.rs:
//
//   use tauri::Manager;
//
//   #[tokio::main]  // Tauri 2 uses tokio
//   async fn main() {
//       let db = Database::open("yord.db").await.unwrap();
//
//       tauri::Builder::default()
//           .manage(db)  // inject as Tauri state
//           .invoke_handler(tauri::generate_handler![get_entities, create_entity])
//           .run(tauri::generate_context!())
//           .unwrap();
//   }
//
//   #[tauri::command]
//   async fn get_entities(db: tauri::State<'_, Database>) -> Result<Vec<Entity>, String> {
//       db.read(|conn| {
//           // rusqlite query here
//           query_all_entities(conn)
//       })
//       .await
//       .map_err(|e| e.to_string())
//   }
//
//   #[tauri::command]
//   async fn create_entity(
//       db: tauri::State<'_, Database>,
//       components: Vec<ComponentData>,
//   ) -> Result<String, String> {
//       db.write(|conn| {
//           // rusqlite insert here, inside a transaction
//           insert_entity(conn, &components)
//       })
//       .await
//       .map_err(|e| e.to_string())
//   }
```

### Design decisions explained

**Why a background thread for writes, not `spawn_blocking`?**
`spawn_blocking` uses Tokio's blocking thread pool, which has multiple threads. Two concurrent `spawn_blocking` calls could run writes in parallel, defeating single-writer. The dedicated thread guarantees serial execution without any mutex. This is exactly what Zed does.

**Why `ThreadLocal` for readers, not a fixed-size pool?**
Tokio's blocking thread pool (where `spawn_blocking` runs) reuses threads. `ThreadLocal` means each blocking thread gets one connection, and it stays alive for the thread's lifetime. No checkout/return overhead. Under typical Tauri load (UI thread + a few blocking threads), this creates 2-4 reader connections total — minimal overhead. This is exactly what Zed does.

**Why `mpsc::unbounded_channel` instead of bounded?**
Write operations in a workspace manager are infrequent (user actions, not streaming data). Backpressure is unnecessary. If the writer is slow, queued writes accumulate — but with sub-millisecond SQLite writes, this never happens in practice. Zed also uses an unbounded channel.

**Why `SQLITE_OPEN_NO_MUTEX` (multi-threaded mode)?**
Each connection is guaranteed to be accessed from exactly one thread (ThreadLocal for readers, dedicated thread for writer). SQLite's internal mutex is pure overhead in this case. Zed uses the equivalent `SQLITE_OPEN_NOMUTEX`.

**Why `SQLITE_OPEN_READ_ONLY` for reader connections?**
Defense in depth. Even if application code accidentally issues a write on a reader, SQLite will reject it. Zed achieves this differently (runtime `write` flag) because their connection type wraps raw `libsqlite3-sys`. With `rusqlite`, the open flag is simpler.

**PRAGMA rationale:**

| PRAGMA | Value | Why |
|--------|-------|-----|
| `journal_mode = WAL` | Enables concurrent reads during writes. Readers never block the writer and vice versa. Essential for the single-writer + reader pool pattern. |
| `synchronous = NORMAL` | In WAL mode, NORMAL is safe against corruption (only loses the last transaction on OS crash, not power loss). FULL would fsync on every commit — unnecessary for a workspace manager. |
| `busy_timeout = 5000` | 5 seconds. Handles contention when WAL checkpoint is running. Waveterm uses 5000ms. Zed uses 500ms (they have fewer long-running transactions). 5000ms is safer for startup migrations. |
| `foreign_keys = ON` | Required for `ON DELETE CASCADE` in the components table. Must be set per-connection (SQLite default is OFF). |
| `temp_store = MEMORY` | Temp tables and indexes in RAM instead of disk. Faster for `json_extract` operations on the components table. |
| `mmap_size = 268435456` | 256MB memory-mapped I/O. Faster reads by avoiding `read()` syscalls. Safe because Yord's DB is small (KBs to low MBs). |
| `cache_size = -16000` | 16MB page cache (negative = KB). Larger cache reduces disk I/O for repeated queries. Default is 2MB. |
| `case_sensitive_like = TRUE` | Not included in Yord's spec (Zed uses it). Only needed if LIKE queries must be case-sensitive. Optional. |

**Migration approach:**
Simpler than Zed (no domain separation needed — Yord has one schema), simpler than Waveterm (no external library). Append-only `&[&str]` slice with a version counter. Foreign keys disabled during migration (Zed pattern) to allow table recreation without cascade issues. `foreign_key_check` after migration catches any orphaned references.

### What to watch for

1. **`spawn_blocking` for reads**: Tauri 2 async commands run on the Tokio runtime. `rusqlite` is synchronous. Every read must go through `spawn_blocking` (or the `read()` helper above). Never call `reader()` directly from an async context — it blocks the executor.

2. **Writer thread lifetime**: The writer thread dies when the `mpsc` sender is dropped. Since `Database` is in Tauri state, it lives for the app's lifetime. On shutdown, Tauri drops state, the sender drops, the writer thread exits cleanly after processing remaining items.

3. **WAL checkpoint**: SQLite auto-checkpoints when the WAL reaches 1000 pages. In WAL mode with multiple connections, the last connection to close triggers a checkpoint. Since the reader connections are ThreadLocal and may not close cleanly, consider running `PRAGMA wal_checkpoint(TRUNCATE)` on the writer during shutdown.

4. **SQLite version**: Verify the bundled SQLite is >= 3.51.3 (WAL-reset data race fix, per design.md). `rusqlite` bundles SQLite via the `bundled` feature — check the version it pulls.

5. **`PRAGMA optimize`**: Run on the writer connection during shutdown or periodically (design.md mentions Phase 4 at 5s+ after startup). This lets SQLite update internal statistics for the query planner.
