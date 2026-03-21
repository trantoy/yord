# M1 — Tree + Shell: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A desktop app where you create projects, add terminal nodes, and manage everything in a tree with clean process management and session restore.

**Architecture:** Tauri 2 (Rust) + Svelte 5 (TS) with ECS storage (SQLite, entities + components tables). Per-PTY Tokio actor pattern with ring buffer output streaming. Two-layer escape sequence filtering. Progressive 4-phase session restore.

**Tech Stack:** Rust, Svelte 5, TypeScript, Tauri 2, rusqlite, portable-pty 0.9+, xterm.js, vte, tracing, thiserror, uuid v7, close-fds, sysinfo

**Spec:** `docs/superpowers/specs/2026-03-21-m1-tree-shell-design.md`

**Research:** `docs/research/` (pty/, storage/, frontend/, landscape/, from-issues/)

**Reference projects:** `docs/refs/` (canopy, wezterm, zellij, zed, waveterm, helix — for inspiration only, all code written fresh)

---

## Phase 1: Scaffold + ECS Storage (Tasks 1–7)

Produces: A Tauri 2 app with blank window, SQLite database, entity CRUD via IPC, and frontend type wrappers. No UI yet.

---

### Task 1: Tauri 2 + Svelte 5 Scaffold

**Files:**
- Create: entire project scaffold via `bun create tauri-app`
- Modify: `src-tauri/Cargo.toml` (add dependencies)
- Modify: `package.json` (add dependencies)
- Modify: `src-tauri/tauri.conf.json` (CSP, window config)
- Modify: `src-tauri/capabilities/default.json` (permissions)

- [ ] **Step 1: Create Tauri app**

```bash
cd /home/fox/projects/yord
bun create tauri-app . --template svelte-ts --manager bun
```

If the directory isn't empty (docs exist), move docs out, scaffold, move back:
```bash
mv docs /tmp/yord-docs && mv CLAUDE.md AGENTS.md CHANGELOG.md CONTRIBUTING.md LICENSE-MIT LICENSE-APACHE README.md .editorconfig .gitattributes .gitignore /tmp/yord-docs/
bun create tauri-app . --template svelte-ts --manager bun
mv /tmp/yord-docs/docs . && mv /tmp/yord-docs/CLAUDE.md /tmp/yord-docs/AGENTS.md /tmp/yord-docs/CHANGELOG.md /tmp/yord-docs/CONTRIBUTING.md /tmp/yord-docs/LICENSE-MIT /tmp/yord-docs/LICENSE-APACHE /tmp/yord-docs/README.md /tmp/yord-docs/.editorconfig /tmp/yord-docs/.gitattributes /tmp/yord-docs/.gitignore .
```

- [ ] **Step 2: Add Rust dependencies to `src-tauri/Cargo.toml`**

```toml
[dependencies]
tauri = { version = "2", features = ["devtools"] }
tauri-plugin-window-state = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rusqlite = { version = "0.32", features = ["bundled"] }
uuid = { version = "1", features = ["v7"] }
thiserror = "2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
tokio = { version = "1", features = ["full"] }
thread_local = "1"
```

- [ ] **Step 3: Add frontend dependencies**

```bash
cd /home/fox/projects/yord
bun add @xterm/xterm @xterm/addon-fit @xterm/addon-search @xterm/addon-canvas @xterm/addon-webgl
```

- [ ] **Step 4: Configure CSP in `src-tauri/tauri.conf.json`**

Add to the `security` section:
```json
"security": {
  "csp": "default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self' data:; connect-src 'self' ipc: http://ipc.localhost"
}
```

Set window config:
```json
"windows": [{
  "title": "Yord",
  "width": 1200,
  "height": 800,
  "minWidth": 600,
  "minHeight": 400
}]
```

- [ ] **Step 5: Set up tracing in `src-tauri/src/main.rs`**

```rust
fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "yord=debug,tauri=info".into()),
        )
        .init();

    tauri::Builder::default()
        .plugin(tauri_plugin_window_state::Builder::new().build())
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 6: Verify the app builds and launches**

```bash
cd /home/fox/projects/yord
cargo build --manifest-path src-tauri/Cargo.toml
bun run tauri dev
```

Expected: blank Tauri window appears.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: scaffold Tauri 2 + Svelte 5 app with dependencies"
```

---

### Task 2: Error Types + ECS Type Definitions

**Files:**
- Create: `src-tauri/src/error.rs`
- Create: `src-tauri/src/ecs/mod.rs`
- Create: `src-tauri/src/ecs/entity.rs`
- Create: `src-tauri/src/ecs/component.rs`
- Modify: `src-tauri/src/main.rs` (add modules)

- [ ] **Step 1: Create `src-tauri/src/error.rs`**

```rust
use serde::Serialize;

#[derive(thiserror::Error, Debug)]
pub enum YordError {
    #[error("Entity not found: {0}")]
    EntityNotFound(String),

    #[error("Component not found: {entity_id}/{component_type}")]
    ComponentNotFound {
        entity_id: String,
        component_type: String,
    },

    #[error("PTY error: {0}")]
    Pty(String),

    #[error("Database error: {0}")]
    Db(#[from] rusqlite::Error),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}

// Tauri requires Serialize for command return errors
impl Serialize for YordError {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        serializer.serialize_str(&self.to_string())
    }
}
```

- [ ] **Step 2: Create `src-tauri/src/ecs/entity.rs`**

```rust
use serde::{Deserialize, Serialize};

pub type EntityId = String;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Entity {
    pub id: EntityId,
    pub created_at: i64,
    pub updated_at: i64,
}

impl Entity {
    pub fn new() -> Self {
        let now = chrono_now();
        Self {
            id: uuid::Uuid::now_v7().to_string(),
            created_at: now,
            updated_at: now,
        }
    }
}

fn chrono_now() -> i64 {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs() as i64
}
```

- [ ] **Step 3: Create `src-tauri/src/ecs/component.rs`**

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

/// All component types Yord supports.
/// Identity comes from which components an entity has, not from a type enum.
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(tag = "type", content = "data")]
pub enum ComponentData {
    Label(LabelData),
    BelongsTo(BelongsToData),
    Order(OrderData),
    Project(ProjectData),
    Pty(PtyData),
    Running(RunningData),
    Crashed(CrashedData),
    RestoreOnStart(EmptyData),
    Pinned(EmptyData),
}

impl ComponentData {
    pub fn component_type(&self) -> &'static str {
        match self {
            ComponentData::Label(_) => "Label",
            ComponentData::BelongsTo(_) => "BelongsTo",
            ComponentData::Order(_) => "Order",
            ComponentData::Project(_) => "Project",
            ComponentData::Pty(_) => "Pty",
            ComponentData::Running(_) => "Running",
            ComponentData::Crashed(_) => "Crashed",
            ComponentData::RestoreOnStart(_) => "RestoreOnStart",
            ComponentData::Pinned(_) => "Pinned",
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct LabelData {
    pub text: String,
    pub icon: Option<String>,
    pub color: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct BelongsToData {
    pub parent_id: String,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct OrderData {
    pub index: i32,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct ProjectData {
    pub cwd: String,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct PtyData {
    pub cmd: Vec<String>,
    pub cwd: String,
    pub env: Option<HashMap<String, String>>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct RunningData {
    pub pid: u32,
    pub pgid: u32,
    pub started_at: i64,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct CrashedData {
    pub exit_code: i32,
    pub stderr_tail: String,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct EmptyData {}

/// An entity with all its components attached.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EntityWithComponents {
    pub id: String,
    pub created_at: i64,
    pub updated_at: i64,
    pub components: HashMap<String, serde_json::Value>,
}
```

- [ ] **Step 4: Create `src-tauri/src/ecs/mod.rs`**

```rust
pub mod component;
pub mod entity;

pub use component::*;
pub use entity::*;
```

- [ ] **Step 5: Wire modules in `src-tauri/src/main.rs`**

Add `mod error; mod ecs;` at top.

- [ ] **Step 6: Verify it compiles**

```bash
cargo build --manifest-path src-tauri/Cargo.toml
```

- [ ] **Step 7: Commit**

```bash
git add src-tauri/src/error.rs src-tauri/src/ecs/ src-tauri/src/main.rs
git commit -m "feat: add YordError, Entity, ComponentData types"
```

---

### Task 3: Database Module (Single-Writer + Migrations)

**Files:**
- Create: `src-tauri/src/db/mod.rs`
- Create: `src-tauri/src/db/migrations.rs`
- Create: `src-tauri/tests/db_test.rs` (integration test)
- Modify: `src-tauri/src/main.rs` (add module)

Reference: `docs/research/storage/sqlite-single-writer.md` for full design.

- [ ] **Step 1: Write failing test for Database open + migration**

Create `src-tauri/tests/db_test.rs`:
```rust
use yord_lib::db::Database;
use tempfile::NamedTempFile;

#[tokio::test]
async fn test_database_opens_and_migrates() {
    let tmp = NamedTempFile::new().unwrap();
    let db = Database::open(tmp.path()).await.unwrap();

    // Verify tables exist by querying them
    let count: i64 = db.read(|conn| {
        conn.query_row("SELECT COUNT(*) FROM entities", [], |row| row.get(0))
            .unwrap()
    }).await;

    assert_eq!(count, 0);
}

#[tokio::test]
async fn test_database_write_and_read() {
    let tmp = NamedTempFile::new().unwrap();
    let db = Database::open(tmp.path()).await.unwrap();

    db.write(|conn| {
        conn.execute(
            "INSERT INTO entities (id, created_at, updated_at) VALUES (?1, ?2, ?3)",
            rusqlite::params!["test-1", 1000, 1000],
        ).unwrap();
    }).await;

    let count: i64 = db.read(|conn| {
        conn.query_row("SELECT COUNT(*) FROM entities", [], |row| row.get(0))
            .unwrap()
    }).await;

    assert_eq!(count, 1);
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cargo test --manifest-path src-tauri/Cargo.toml --test db_test
```

Expected: FAIL — module `db` not found.

- [ ] **Step 3: Implement `src-tauri/src/db/migrations.rs`**

```rust
use rusqlite::{Connection, params};
use tracing::info;

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
        WHERE component_type = 'BelongsTo';
    CREATE TABLE IF NOT EXISTS _yord_meta (
        key   TEXT PRIMARY KEY,
        value TEXT NOT NULL
    );",
];

pub fn run_migrations(conn: &mut Connection) -> anyhow::Result<()> {
    conn.execute_batch(
        "CREATE TABLE IF NOT EXISTS _migrations (
            version    INTEGER PRIMARY KEY,
            applied_at INTEGER NOT NULL
        );"
    )?;

    let current_version: i64 = conn
        .query_row("SELECT COALESCE(MAX(version), -1) FROM _migrations", [], |row| row.get(0))?;

    conn.execute_batch("PRAGMA foreign_keys = OFF;")?;

    let tx = conn.transaction()?;
    for (i, sql) in MIGRATIONS.iter().enumerate() {
        let version = i as i64;
        if version > current_version {
            info!(version, "Running migration");
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
```

- [ ] **Step 4: Implement `src-tauri/src/db/mod.rs`**

Follow the design from `docs/research/storage/sqlite-single-writer.md`:

```rust
pub mod migrations;

use std::path::{Path, PathBuf};
use std::sync::Arc;
use rusqlite::{Connection, OpenFlags};
use thread_local::ThreadLocal;
use tokio::sync::{mpsc, oneshot};
use tracing::{error, info};

const CONNECTION_PRAGMAS: &str = "\
    PRAGMA foreign_keys = ON;\
    PRAGMA busy_timeout = 5000;\
    PRAGMA temp_store = MEMORY;\
";

const DB_INIT_PRAGMAS: &str = "\
    PRAGMA journal_mode = WAL;\
    PRAGMA synchronous = NORMAL;\
    PRAGMA mmap_size = 268435456;\
    PRAGMA cache_size = -16000;\
";

fn open_connection(path: &Path, readonly: bool) -> rusqlite::Result<Connection> {
    let flags = if readonly {
        OpenFlags::SQLITE_OPEN_READ_ONLY
            | OpenFlags::SQLITE_OPEN_NO_MUTEX
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

type WriteFn = Box<dyn FnOnce(&mut Connection) + Send + 'static>;

#[derive(Clone)]
pub struct Database {
    path: Arc<PathBuf>,
    readers: Arc<ThreadLocal<Connection>>,
    write_tx: mpsc::UnboundedSender<WriteFn>,
}

impl Database {
    pub async fn open(path: impl Into<PathBuf>) -> anyhow::Result<Self> {
        let path = Arc::new(path.into());
        let (write_tx, mut write_rx) = mpsc::unbounded_channel::<WriteFn>();

        let writer_path = path.clone();
        std::thread::Builder::new()
            .name("yord-db-writer".into())
            .spawn(move || {
                let mut conn = open_connection(&writer_path, false)
                    .expect("Failed to open writer connection");
                conn.execute_batch(DB_INIT_PRAGMAS)
                    .expect("Failed to set DB init pragmas");

                migrations::run_migrations(&mut conn)
                    .expect("Migrations failed");

                info!("Database writer thread started");

                while let Some(f) = write_rx.blocking_recv() {
                    f(&mut conn);
                }

                // Shutdown: checkpoint WAL
                let _ = conn.execute_batch("PRAGMA wal_checkpoint(TRUNCATE);");
                let _ = conn.execute_batch("PRAGMA optimize;");
                info!("Database writer thread stopped");
            })?;

        let readers = Arc::new(ThreadLocal::new());
        Ok(Self { path, readers, write_tx })
    }

    pub fn reader(&self) -> &Connection {
        self.readers.get_or(|| {
            open_connection(&self.path, true)
                .expect("Failed to open reader connection")
        })
    }

    pub async fn read<F, T>(&self, f: F) -> T
    where
        F: FnOnce(&mut Connection) -> T + Send + 'static,
        T: Send + 'static,
    {
        let db = self.clone();
        tokio::task::spawn_blocking(move || f(db.reader()))
            .await
            .expect("Read task panicked")
    }

    pub async fn write<F, T>(&self, f: F) -> T
    where
        F: FnOnce(&mut Connection) -> T + Send + 'static,
        T: Send + 'static,
    {
        let (result_tx, result_rx) = oneshot::channel();
        self.write_tx
            .send(Box::new(move |conn: &mut Connection| {
                let result = f(conn);
                let _ = result_tx.send(result);
            }))
            .expect("Writer thread has shut down");
        result_rx.await.expect("Writer dropped result")
    }
}
```

- [ ] **Step 5: Add `anyhow` and `tempfile` to Cargo.toml**

```toml
[dependencies]
anyhow = "1"

[dev-dependencies]
tempfile = "3"
```

- [ ] **Step 6: Wire `db` module in `main.rs` and create `src-tauri/src/lib.rs` for test access**

Create `src-tauri/src/lib.rs`:
```rust
pub mod db;
pub mod ecs;
pub mod error;
```

- [ ] **Step 7: Run tests**

```bash
cargo test --manifest-path src-tauri/Cargo.toml --test db_test
```

Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add src-tauri/src/db/ src-tauri/src/lib.rs src-tauri/tests/ src-tauri/Cargo.toml
git commit -m "feat: add Database with single-writer, ThreadLocal readers, migrations"
```

---

### Task 4: EntityStore Trait + SqliteEntityStore

**Files:**
- Create: `src-tauri/src/ecs/traits.rs`
- Create: `src-tauri/src/ecs/store.rs`
- Create: `src-tauri/tests/store_test.rs`
- Modify: `src-tauri/src/ecs/mod.rs`

- [ ] **Step 1: Write failing tests for EntityStore**

Create `src-tauri/tests/store_test.rs`:
```rust
use yord_lib::db::Database;
use yord_lib::ecs::component::*;
use yord_lib::ecs::store::SqliteEntityStore;
use yord_lib::ecs::traits::EntityStore;
use tempfile::NamedTempFile;

async fn setup() -> SqliteEntityStore {
    let tmp = NamedTempFile::new().unwrap();
    let db = Database::open(tmp.path()).await.unwrap();
    SqliteEntityStore::new(db)
}

#[tokio::test]
async fn test_create_and_get_entity() {
    let store = setup().await;
    let components = vec![
        ComponentData::Label(LabelData {
            text: "my project".into(),
            icon: None,
            color: None,
        }),
        ComponentData::Project(ProjectData {
            cwd: "/home/user/project".into(),
        }),
    ];

    let id = store.create(components).await.unwrap();
    let entity = store.get(&id).await.unwrap();

    assert_eq!(entity.id, id);
    assert!(entity.components.contains_key("Label"));
    assert!(entity.components.contains_key("Project"));
}

#[tokio::test]
async fn test_delete_entity() {
    let store = setup().await;
    let id = store.create(vec![
        ComponentData::Label(LabelData { text: "tmp".into(), icon: None, color: None }),
    ]).await.unwrap();

    store.delete(&id).await.unwrap();
    let result = store.get(&id).await;
    assert!(result.is_err());
}

#[tokio::test]
async fn test_set_and_remove_component() {
    let store = setup().await;
    let id = store.create(vec![
        ComponentData::Label(LabelData { text: "test".into(), icon: None, color: None }),
    ]).await.unwrap();

    store.set_component(&id, ComponentData::Pinned(EmptyData {})).await.unwrap();
    let entity = store.get(&id).await.unwrap();
    assert!(entity.components.contains_key("Pinned"));

    store.remove_component(&id, "Pinned").await.unwrap();
    let entity = store.get(&id).await.unwrap();
    assert!(!entity.components.contains_key("Pinned"));
}

#[tokio::test]
async fn test_get_all() {
    let store = setup().await;
    store.create(vec![
        ComponentData::Label(LabelData { text: "a".into(), icon: None, color: None }),
    ]).await.unwrap();
    store.create(vec![
        ComponentData::Label(LabelData { text: "b".into(), icon: None, color: None }),
    ]).await.unwrap();

    let all = store.get_all().await.unwrap();
    assert_eq!(all.len(), 2);
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cargo test --manifest-path src-tauri/Cargo.toml --test store_test
```

- [ ] **Step 3: Implement `src-tauri/src/ecs/traits.rs`**

```rust
use crate::ecs::component::{ComponentData, EntityWithComponents};
use crate::ecs::entity::EntityId;
use crate::error::YordError;

pub trait EntityStore: Send + Sync {
    fn create(&self, components: Vec<ComponentData>) -> impl std::future::Future<Output = Result<EntityId, YordError>> + Send;
    fn delete(&self, id: &str) -> impl std::future::Future<Output = Result<(), YordError>> + Send;
    fn get(&self, id: &str) -> impl std::future::Future<Output = Result<EntityWithComponents, YordError>> + Send;
    fn get_all(&self) -> impl std::future::Future<Output = Result<Vec<EntityWithComponents>, YordError>> + Send;
    fn set_component(&self, id: &str, component: ComponentData) -> impl std::future::Future<Output = Result<(), YordError>> + Send;
    fn remove_component(&self, id: &str, comp_type: &str) -> impl std::future::Future<Output = Result<(), YordError>> + Send;
    fn query_by_component(&self, comp_type: &str) -> impl std::future::Future<Output = Result<Vec<EntityWithComponents>, YordError>> + Send;
    fn query_children(&self, parent_id: &str) -> impl std::future::Future<Output = Result<Vec<EntityWithComponents>, YordError>> + Send;
}
```

Uses native `async fn` in traits (stable since Rust 1.75). No `async-trait` dependency needed. Includes `query_by_component` and `query_children` (needed by kill propagation and restore systems).

- [ ] **Step 4: Implement `src-tauri/src/ecs/store.rs`**

```rust
use crate::db::Database;
use crate::ecs::component::*;
use crate::ecs::entity::*;
use crate::ecs::traits::EntityStore;
use crate::error::YordError;
use rusqlite::params;
use std::collections::HashMap;
use tracing::debug;

pub struct SqliteEntityStore {
    db: Database,
}

impl SqliteEntityStore {
    pub fn new(db: Database) -> Self {
        Self { db }
    }
}

impl EntityStore for SqliteEntityStore {
    async fn create(&self, components: Vec<ComponentData>) -> Result<EntityId, YordError> {
        let entity = Entity::new();
        let id = entity.id.clone();

        self.db.write(move |conn: &mut Connection| {
            let tx = conn.transaction()?;
            tx.execute(
                "INSERT INTO entities (id, created_at, updated_at) VALUES (?1, ?2, ?3)",
                params![entity.id, entity.created_at, entity.updated_at],
            )?;
            for component in &components {
                let data = serde_json::to_string(component)
                    .map_err(|e| rusqlite::Error::ToSqlConversionFailure(Box::new(e)))?;
                // Extract just the data portion for storage
                let data_value: serde_json::Value = serde_json::from_str(&data).unwrap();
                let inner_data = data_value.get("data").cloned().unwrap_or(serde_json::Value::Object(Default::default()));
                tx.execute(
                    "INSERT OR REPLACE INTO components (entity_id, component_type, data)
                     VALUES (?1, ?2, ?3)",
                    params![entity.id, component.component_type(), serde_json::to_string(&inner_data).unwrap()],
                )?;
            }
            tx.commit()?;
            debug!(entity_id = %entity.id, "Entity created");
            Ok::<_, rusqlite::Error>(())
        }).await?;

        Ok(id)
    }

    async fn delete(&self, id: &str) -> Result<(), YordError> {
        let id_owned = id.to_string();
        let id_for_error = id_owned.clone();
        let rows = self.db.write(move |conn| {
            conn.execute("DELETE FROM entities WHERE id = ?1", params![id_owned])
        }).await?;
        if rows == 0 {
            return Err(YordError::EntityNotFound(id_for_error));
        }
        Ok(())
    }

    async fn get(&self, id: &str) -> Result<EntityWithComponents, YordError> {
        let id = id.to_string();
        self.db.read(move |conn| {
            let mut stmt = conn.prepare(
                "SELECT id, created_at, updated_at FROM entities WHERE id = ?1"
            )?;
            let entity = stmt.query_row(params![id], |row| {
                Ok((
                    row.get::<_, String>(0)?,
                    row.get::<_, i64>(1)?,
                    row.get::<_, i64>(2)?,
                ))
            }).map_err(|_| YordError::EntityNotFound(id.clone()))?;

            let mut comp_stmt = conn.prepare(
                "SELECT component_type, data FROM components WHERE entity_id = ?1"
            )?;
            let components: HashMap<String, serde_json::Value> = comp_stmt
                .query_map(params![id], |row| {
                    Ok((
                        row.get::<_, String>(0)?,
                        row.get::<_, String>(1)?,
                    ))
                })?
                .filter_map(|r| r.ok())
                .map(|(ct, data)| (ct, serde_json::from_str(&data).unwrap_or_default()))
                .collect();

            Ok(EntityWithComponents {
                id: entity.0,
                created_at: entity.1,
                updated_at: entity.2,
                components,
            })
        }).await
    }

    async fn get_all(&self) -> Result<Vec<EntityWithComponents>, YordError> {
        self.db.read(|conn| {
            let mut stmt = conn.prepare("SELECT id, created_at, updated_at FROM entities")?;
            let entities: Vec<(String, i64, i64)> = stmt
                .query_map([], |row| {
                    Ok((row.get(0)?, row.get(1)?, row.get(2)?))
                })?
                .filter_map(|r| r.ok())
                .collect();

            let mut result = Vec::with_capacity(entities.len());
            let mut comp_stmt = conn.prepare(
                "SELECT component_type, data FROM components WHERE entity_id = ?1"
            )?;

            for (id, created_at, updated_at) in entities {
                let components: HashMap<String, serde_json::Value> = comp_stmt
                    .query_map(params![id], |row| {
                        Ok((row.get::<_, String>(0)?, row.get::<_, String>(1)?))
                    })?
                    .filter_map(|r| r.ok())
                    .map(|(ct, data)| (ct, serde_json::from_str(&data).unwrap_or_default()))
                    .collect();

                result.push(EntityWithComponents { id, created_at, updated_at, components });
            }

            Ok(result)
        }).await
    }

    async fn set_component(&self, id: &str, component: ComponentData) -> Result<(), YordError> {
        let id = id.to_string();
        let comp_type = component.component_type().to_string();
        let data = serde_json::to_value(&component).unwrap();
        let inner_data = data.get("data").cloned().unwrap_or_default();

        self.db.write(move |conn| {
            conn.execute(
                "INSERT OR REPLACE INTO components (entity_id, component_type, data)
                 VALUES (?1, ?2, ?3)",
                params![id, comp_type, serde_json::to_string(&inner_data).unwrap()],
            )
        }).await?;
        Ok(())
    }

    async fn remove_component(&self, id: &str, comp_type: &str) -> Result<(), YordError> {
        let id = id.to_string();
        let comp_type = comp_type.to_string();
        self.db.write(move |conn| {
            conn.execute(
                "DELETE FROM components WHERE entity_id = ?1 AND component_type = ?2",
                params![id, comp_type],
            )
        }).await?;
        Ok(())
    }

    async fn query_by_component(&self, comp_type: &str) -> Result<Vec<EntityWithComponents>, YordError> {
        let comp_type = comp_type.to_string();
        self.db.read(move |conn| {
            let mut stmt = conn.prepare(
                "SELECT DISTINCT e.id, e.created_at, e.updated_at
                 FROM entities e
                 JOIN components c ON e.id = c.entity_id
                 WHERE c.component_type = ?1"
            )?;
            let entity_ids: Vec<(String, i64, i64)> = stmt
                .query_map(params![comp_type], |row| Ok((row.get(0)?, row.get(1)?, row.get(2)?)))?
                .filter_map(|r| r.ok())
                .collect();

            let mut comp_stmt = conn.prepare(
                "SELECT component_type, data FROM components WHERE entity_id = ?1"
            )?;
            let mut result = Vec::with_capacity(entity_ids.len());
            for (id, created_at, updated_at) in entity_ids {
                let components: HashMap<String, serde_json::Value> = comp_stmt
                    .query_map(params![id], |row| Ok((row.get::<_, String>(0)?, row.get::<_, String>(1)?)))?
                    .filter_map(|r| r.ok())
                    .map(|(ct, data)| (ct, serde_json::from_str(&data).unwrap_or_default()))
                    .collect();
                result.push(EntityWithComponents { id, created_at, updated_at, components });
            }
            Ok(result)
        }).await
    }

    async fn query_children(&self, parent_id: &str) -> Result<Vec<EntityWithComponents>, YordError> {
        let parent_id = parent_id.to_string();
        self.db.read(move |conn| {
            let mut stmt = conn.prepare(
                "SELECT e.id, e.created_at, e.updated_at
                 FROM entities e
                 JOIN components c ON e.id = c.entity_id
                 WHERE c.component_type = 'BelongsTo'
                   AND json_extract(c.data, '$.parent_id') = ?1"
            )?;
            let entity_ids: Vec<(String, i64, i64)> = stmt
                .query_map(params![parent_id], |row| Ok((row.get(0)?, row.get(1)?, row.get(2)?)))?
                .filter_map(|r| r.ok())
                .collect();

            let mut comp_stmt = conn.prepare(
                "SELECT component_type, data FROM components WHERE entity_id = ?1"
            )?;
            let mut result = Vec::with_capacity(entity_ids.len());
            for (id, created_at, updated_at) in entity_ids {
                let components: HashMap<String, serde_json::Value> = comp_stmt
                    .query_map(params![id], |row| Ok((row.get::<_, String>(0)?, row.get::<_, String>(1)?)))?
                    .filter_map(|r| r.ok())
                    .map(|(ct, data)| (ct, serde_json::from_str(&data).unwrap_or_default()))
                    .collect();
                result.push(EntityWithComponents { id, created_at, updated_at, components });
            }
            Ok(result)
        }).await
    }
}
```

- [ ] **Step 5: Update `src-tauri/src/ecs/mod.rs`**

```rust
pub mod component;
pub mod entity;
pub mod store;
pub mod traits;

pub use component::*;
pub use entity::*;
```

- [ ] **Step 6: Run tests**

```bash
cargo test --manifest-path src-tauri/Cargo.toml --test store_test
```

Expected: all PASS.

- [ ] **Step 7: Commit**

```bash
git add src-tauri/src/ecs/ src-tauri/tests/store_test.rs src-tauri/Cargo.toml
git commit -m "feat: add EntityStore trait and SqliteEntityStore implementation"
```

---

### Task 5: Tauri Commands for Entity CRUD

**Files:**
- Create: `src-tauri/src/commands/mod.rs`
- Create: `src-tauri/src/commands/entities.rs`
- Modify: `src-tauri/src/main.rs` (register commands, manage state)
- Modify: `src-tauri/src/lib.rs` (add commands module)

- [ ] **Step 1: Create `src-tauri/src/commands/entities.rs`**

```rust
use crate::ecs::component::*;
use crate::ecs::store::SqliteEntityStore;
use crate::ecs::traits::EntityStore;
use crate::error::YordError;
use tauri::State;

#[tauri::command]
pub async fn create_entity(
    store: State<'_, SqliteEntityStore>,
    components: Vec<ComponentData>,
) -> Result<String, YordError> {
    store.create(components).await
}

#[tauri::command]
pub async fn delete_entity(
    store: State<'_, SqliteEntityStore>,
    id: String,
) -> Result<(), YordError> {
    store.delete(&id).await
}

#[tauri::command]
pub async fn get_entities(
    store: State<'_, SqliteEntityStore>,
) -> Result<Vec<EntityWithComponents>, YordError> {
    store.get_all().await
}

#[tauri::command]
pub async fn set_component(
    store: State<'_, SqliteEntityStore>,
    entity_id: String,
    component: ComponentData,
) -> Result<(), YordError> {
    store.set_component(&entity_id, component).await
}

#[tauri::command]
pub async fn remove_component(
    store: State<'_, SqliteEntityStore>,
    entity_id: String,
    component_type: String,
) -> Result<(), YordError> {
    store.remove_component(&entity_id, &component_type).await
}
```

- [ ] **Step 2: Create `src-tauri/src/commands/mod.rs`**

```rust
pub mod entities;
pub use entities::*;
```

- [ ] **Step 3: Update `src-tauri/src/main.rs` — register commands + initialize DB**

```rust
mod commands;
mod db;
mod ecs;
mod error;

use db::Database;
use ecs::store::SqliteEntityStore;
use tracing::info;

fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "yord=debug,tauri=info".into()),
        )
        .init();

    let rt = tokio::runtime::Runtime::new().expect("Failed to create Tokio runtime");

    let db = rt.block_on(async {
        let app_dir = dirs::data_dir()
            .unwrap_or_else(|| std::path::PathBuf::from("."))
            .join("yord");
        std::fs::create_dir_all(&app_dir).ok();
        let db_path = app_dir.join("yord.db");
        info!(?db_path, "Opening database");
        Database::open(db_path).await.expect("Failed to open database")
    });

    let store = SqliteEntityStore::new(db);

    tauri::Builder::default()
        .plugin(tauri_plugin_window_state::Builder::new().build())
        .manage(store)
        .invoke_handler(tauri::generate_handler![
            commands::create_entity,
            commands::delete_entity,
            commands::get_entities,
            commands::set_component,
            commands::remove_component,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

Add `dirs = "5"` to Cargo.toml.

- [ ] **Step 4: Add event emission to commands**

Each mutation command must emit a Tauri event so the frontend store rebuilds the tree:

```rust
// In create_entity, after store.create():
app.emit("entity:component-changed", serde_json::json!({
    "entity_id": id, "component_type": "created", "data": null
})).ok();

// In delete_entity, after store.delete():
app.emit("entity:removed", serde_json::json!({ "entity_id": id })).ok();

// In set_component / remove_component:
app.emit("entity:component-changed", serde_json::json!({
    "entity_id": entity_id, "component_type": comp_type, "data": null
})).ok();
```

Commands need `app: tauri::AppHandle` parameter to emit events.

- [ ] **Step 5: Verify it compiles and launches**

```bash
cargo build --manifest-path src-tauri/Cargo.toml
bun run tauri dev
```

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/commands/ src-tauri/src/main.rs src-tauri/src/lib.rs src-tauri/Cargo.toml
git commit -m "feat: add Tauri commands for entity CRUD with event emission"
```

---

### Task 6: Frontend Types + IPC Wrappers

**Files:**
- Create: `src/lib/types/entity.ts`
- Create: `src/lib/types/tree.ts`
- Create: `src/lib/ipc/commands.ts`
- Create: `src/lib/ipc/events.ts`

- [ ] **Step 1: Create `src/lib/types/entity.ts`**

```typescript
export interface EntityWithComponents {
  id: string;
  created_at: number;
  updated_at: number;
  components: Record<string, unknown>;
}

export type ComponentData =
  | { type: "Label"; data: { text: string; icon?: string; color?: string } }
  | { type: "BelongsTo"; data: { parent_id: string } }
  | { type: "Order"; data: { index: number } }
  | { type: "Project"; data: { cwd: string } }
  | { type: "Pty"; data: { cmd: string[]; cwd: string; env?: Record<string, string> } }
  | { type: "Running"; data: { pid: number; pgid: number; started_at: number } }
  | { type: "Crashed"; data: { exit_code: number; stderr_tail: string } }
  | { type: "RestoreOnStart"; data: Record<string, never> }
  | { type: "Pinned"; data: Record<string, never> };
```

- [ ] **Step 2: Create `src/lib/types/tree.ts`**

```typescript
import type { EntityWithComponents } from "./entity";

export interface TreeNode {
  id: string;
  entity: EntityWithComponents;
  children: TreeNode[];
  depth: number;
}
```

- [ ] **Step 3: Create `src/lib/ipc/commands.ts`**

```typescript
import { invoke } from "@tauri-apps/api/core";
import type { EntityWithComponents, ComponentData } from "../types/entity";

export async function createEntity(components: ComponentData[]): Promise<string> {
  return invoke("create_entity", { components });
}

export async function deleteEntity(id: string): Promise<void> {
  return invoke("delete_entity", { id });
}

export async function getEntities(): Promise<EntityWithComponents[]> {
  return invoke("get_entities");
}

export async function setComponent(entityId: string, component: ComponentData): Promise<void> {
  return invoke("set_component", { entityId, component });
}

export async function removeComponent(entityId: string, componentType: string): Promise<void> {
  return invoke("remove_component", { entityId, componentType });
}
```

- [ ] **Step 4: Create `src/lib/ipc/events.ts`**

```typescript
import { listen, type UnlistenFn } from "@tauri-apps/api/event";

export interface ComponentChangedPayload {
  entity_id: string;
  component_type: string;
  data: unknown;
}

export interface EntityRemovedPayload {
  entity_id: string;
}

export async function onComponentChanged(
  handler: (payload: ComponentChangedPayload) => void
): Promise<UnlistenFn> {
  return listen("entity:component-changed", (event) => {
    handler(event.payload as ComponentChangedPayload);
  });
}

export async function onEntityRemoved(
  handler: (payload: EntityRemovedPayload) => void
): Promise<UnlistenFn> {
  return listen("entity:removed", (event) => {
    handler(event.payload as EntityRemovedPayload);
  });
}
```

- [ ] **Step 5: Commit**

```bash
git add src/lib/
git commit -m "feat: add frontend types and IPC wrappers"
```

---

### Task 7: buildTree() + Tests

**Files:**
- Create: `src/lib/stores/tree.ts`
- Create: `src/lib/stores/tree.test.ts`
- Modify: `package.json` (add vitest)

Reference: `docs/research/frontend/flat-to-tree.md` for full algorithm.

- [ ] **Step 1: Install vitest**

```bash
bun add -D vitest
```

Add to `package.json` scripts: `"test": "vitest run"`

- [ ] **Step 2: Write failing tests in `src/lib/stores/tree.test.ts`**

```typescript
import { describe, it, expect } from "vitest";
import { buildTree } from "./tree";
import type { EntityWithComponents } from "../types/entity";

function entity(id: string, parentId?: string, order?: number, extra?: Record<string, unknown>): EntityWithComponents {
  const components: Record<string, unknown> = {
    Label: { text: id },
    ...(extra ?? {}),
  };
  if (parentId) components.BelongsTo = { parent_id: parentId };
  if (order !== undefined) components.Order = { index: order };
  return { id, created_at: 0, updated_at: 0, components };
}

describe("buildTree", () => {
  it("returns empty array for empty input", () => {
    expect(buildTree([])).toEqual([]);
  });

  it("returns root nodes (no parent)", () => {
    const tree = buildTree([entity("a"), entity("b")]);
    expect(tree).toHaveLength(2);
    expect(tree[0].id).toBe("a");
  });

  it("nests children under parents", () => {
    const tree = buildTree([
      entity("project", undefined, 0, { Project: { cwd: "/" } }),
      entity("term1", "project", 0),
      entity("term2", "project", 1),
    ]);
    expect(tree).toHaveLength(1);
    expect(tree[0].children).toHaveLength(2);
    expect(tree[0].children[0].id).toBe("term1");
    expect(tree[0].children[1].id).toBe("term2");
  });

  it("sorts by Order.index", () => {
    const tree = buildTree([
      entity("project"),
      entity("b", "project", 2),
      entity("a", "project", 1),
      entity("c", "project", 0),
    ]);
    expect(tree[0].children.map(c => c.id)).toEqual(["c", "a", "b"]);
  });

  it("promotes orphans to root", () => {
    const tree = buildTree([entity("orphan", "nonexistent")]);
    expect(tree).toHaveLength(1);
    expect(tree[0].id).toBe("orphan");
  });

  it("handles cycles without infinite recursion", () => {
    const tree = buildTree([
      entity("a", "b"),
      entity("b", "a"),
    ]);
    // Both promoted to root (cycle detected)
    expect(tree.length).toBeGreaterThan(0);
  });

  it("sets depth correctly", () => {
    const tree = buildTree([
      entity("root"),
      entity("child", "root"),
      entity("grandchild", "child"),
    ]);
    expect(tree[0].depth).toBe(0);
    expect(tree[0].children[0].depth).toBe(1);
    expect(tree[0].children[0].children[0].depth).toBe(2);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
bun run test
```

Expected: FAIL — `buildTree` not found.

- [ ] **Step 4: Implement `src/lib/stores/tree.ts`**

Follow `docs/research/frontend/flat-to-tree.md`:

```typescript
import type { EntityWithComponents } from "../types/entity";
import type { TreeNode } from "../types/tree";

export function buildTree(entities: EntityWithComponents[]): TreeNode[] {
  const byId = new Map<string, EntityWithComponents>();
  const childrenOf = new Map<string, EntityWithComponents[]>();

  for (const entity of entities) {
    byId.set(entity.id, entity);
  }

  const hasParent = new Set<string>();

  for (const entity of entities) {
    const belongsTo = entity.components.BelongsTo as { parent_id: string } | undefined;
    if (belongsTo?.parent_id) {
      const parentId = belongsTo.parent_id;
      if (parentId === entity.id) continue;
      if (byId.has(parentId)) {
        let siblings = childrenOf.get(parentId);
        if (!siblings) {
          siblings = [];
          childrenOf.set(parentId, siblings);
        }
        siblings.push(entity);
        hasParent.add(entity.id);
      }
    }
  }

  for (const siblings of childrenOf.values()) {
    siblings.sort((a, b) => {
      const aIdx = (a.components.Order as { index: number } | undefined)?.index ?? Infinity;
      const bIdx = (b.components.Order as { index: number } | undefined)?.index ?? Infinity;
      return aIdx - bIdx;
    });
  }

  const visited = new Set<string>();

  function buildNode(entity: EntityWithComponents, depth: number): TreeNode {
    if (visited.has(entity.id)) {
      return { id: entity.id, entity, children: [], depth };
    }
    visited.add(entity.id);

    const childEntities = childrenOf.get(entity.id) ?? [];
    const children = childEntities.map((child) => buildNode(child, depth + 1));

    return { id: entity.id, entity, children, depth };
  }

  const roots: EntityWithComponents[] = [];
  for (const entity of entities) {
    if (!hasParent.has(entity.id)) {
      roots.push(entity);
    }
  }

  roots.sort((a, b) => {
    const aIdx = (a.components.Order as { index: number } | undefined)?.index ?? Infinity;
    const bIdx = (b.components.Order as { index: number } | undefined)?.index ?? Infinity;
    return aIdx - bIdx;
  });

  return roots.map((root) => buildNode(root, 0));
}
```

- [ ] **Step 5: Run tests**

```bash
bun run test
```

Expected: all PASS.

- [ ] **Step 6: Commit**

```bash
git add src/lib/stores/tree.ts src/lib/stores/tree.test.ts package.json bun.lockb
git commit -m "feat: add buildTree() with O(n) algorithm, orphan/cycle handling"
```

---

## Phase 2: Tree UI (Tasks 8–10)

Produces: A working tree panel showing projects and terminals. CRUD operations work. No terminal rendering yet.

---

### Task 8: Entity Store (Svelte)

**Files:**
- Create: `src/lib/stores/entities.svelte.ts`

- [ ] **Step 1: Create the entity store**

```typescript
import { getEntities } from "../ipc/commands";
import { onComponentChanged, onEntityRemoved } from "../ipc/events";
import { buildTree } from "./tree";
import type { EntityWithComponents } from "../types/entity";
import type { TreeNode } from "../types/tree";
import type { UnlistenFn } from "@tauri-apps/api/event";

// Use $state.raw() — no deep proxy (5,000x perf cliff with $state())
let entities: EntityWithComponents[] = $state.raw([]);
let tree: TreeNode[] = $state.raw([]);
let selectedId = $state<string | null>(null);

let unlisteners: UnlistenFn[] = [];
let rebuildQueued = false;

function rebuild() {
  tree = buildTree(entities);
}

function queueRebuild() {
  if (!rebuildQueued) {
    rebuildQueued = true;
    queueMicrotask(() => {
      rebuildQueued = false;
      rebuild();
    });
  }
}

export async function loadEntities() {
  entities = await getEntities();
  rebuild();
}

export async function initEventListeners() {
  unlisteners.push(
    await onComponentChanged(() => {
      loadEntities(); // re-fetch all, then rebuild
    }),
    await onEntityRemoved(() => {
      loadEntities();
    })
  );
}

export function cleanupEventListeners() {
  unlisteners.forEach((fn) => fn());
  unlisteners = [];
}

export function getTree(): TreeNode[] {
  return tree;
}

export function getEntitiesRaw(): EntityWithComponents[] {
  return entities;
}

export function getSelectedId(): string | null {
  return selectedId;
}

export function setSelectedId(id: string | null) {
  selectedId = id;
}

export function getSelectedEntity(): EntityWithComponents | undefined {
  return entities.find((e) => e.id === selectedId);
}
```

- [ ] **Step 2: Commit**

```bash
git add src/lib/stores/entities.svelte.ts
git commit -m "feat: add Svelte entity store with $state.raw() and event listeners"
```

---

### Task 9: TreePanel + TreeNode Components

**Files:**
- Create: `src/components/tree/TreePanel.svelte`
- Create: `src/components/tree/TreeNode.svelte`
- Modify: `src/App.svelte`

Use the svelte-file-editor agent for Svelte component creation.

- [ ] **Step 1: Create `src/components/tree/TreeNode.svelte`**

A recursive tree node component showing label, status dot, expand/collapse, click to select.

- [ ] **Step 2: Create `src/components/tree/TreePanel.svelte`**

The tree panel with header ("Projects"), add project button, and recursive TreeNode rendering.

- [ ] **Step 3: Update `src/App.svelte`**

Layout: tree panel on left (250px), content area on right. Load entities on mount.

- [ ] **Step 4: Run dev server and verify tree renders**

```bash
bun run tauri dev
```

Create a test entity via Rust console or dev tools to verify rendering.

- [ ] **Step 5: Commit**

```bash
git add src/components/tree/ src/App.svelte
git commit -m "feat: add TreePanel and TreeNode components"
```

---

### Task 10: Context Menu + Entity CRUD UI

**Files:**
- Create: `src/components/tree/ContextMenu.svelte`
- Create: `src/components/ui/ConfirmDialog.svelte`
- Modify: `src/components/tree/TreeNode.svelte` (add right-click)
- Modify: `src/components/tree/TreePanel.svelte` (add create project)

- [ ] **Step 1: Create context menu component**

Right-click menu with: Add Terminal, Rename, Delete, Start, Stop, Restart.

- [ ] **Step 2: Create confirmation dialog**

Modal for destructive actions (delete project, kill project). Shows count of affected children.

- [ ] **Step 3: Wire CRUD operations**

- "Add Project" → `createEntity([Label, Project])` → reload
- "Add Terminal" → `createEntity([Label, Pty, BelongsTo, Order])` → reload
- "Delete" → confirm → `deleteEntity(id)` → reload
- "Rename" → inline edit → `setComponent(id, Label)` → reload

- [ ] **Step 4: Test all CRUD operations in the UI**

- [ ] **Step 5: Commit**

```bash
git add src/components/
git commit -m "feat: add context menu, confirm dialog, entity CRUD UI"
```

---

## Phase 3: PTY Engine (Tasks 11–17)

Produces: Working terminal nodes. Click a terminal in the tree, see xterm.js, type commands, get output.

---

### Task 11: Ring Buffer

**Files:**
- Create: `src-tauri/src/process/ring_buffer.rs`
- Create: `src-tauri/src/process/mod.rs`
- Create: `src-tauri/tests/ring_buffer_test.rs`

Reference: `docs/research/pty/output-buffering.md`

- [ ] **Step 1: Write failing tests**
- [ ] **Step 2: Implement ring buffer** (push, drain, capacity, 256KB default)
- [ ] **Step 3: Run tests — PASS**
- [ ] **Step 4: Commit**

---

### Task 12: Escape Sequence Filter

**Files:**
- Create: `src-tauri/src/process/escape_filter.rs`
- Create: `src-tauri/tests/escape_filter_test.rs`

Reference: `docs/research/pty/escape-filtering.md`. Add `vte = "0.13"` to Cargo.toml.

- [ ] **Step 1: Write failing tests** (strip CSI 20t/21t, allow normal CSI, block OSC 52 query, pass OSC 52 write with size limit, block DCS except XTGETTCAP)
- [ ] **Step 2: Implement `EscapeFilter`** using `vte::Perform`
- [ ] **Step 3: Run tests — PASS**
- [ ] **Step 4: Commit**

---

### Task 13: ProcessManager Trait + Unix/Windows Impl

**Files:**
- Create: `src-tauri/src/process/manager.rs`
- Modify: `src-tauri/src/ecs/traits.rs` (add ProcessManager trait)

- [ ] **Step 1: Define ProcessManager trait**
- [ ] **Step 2: Implement `UnixProcessManager`** (killpg, kill(pid,0), /proc children)
- [ ] **Step 3: Implement `WindowsProcessManager`** (Job Objects, TerminateProcess, OpenProcess)
- [ ] **Step 4: Compile on current platform**
- [ ] **Step 5: Commit**

---

### Task 14: PTY Spawning (PortablePtyBackend)

**Files:**
- Create: `src-tauri/src/process/pty.rs`
- Modify: `src-tauri/Cargo.toml` (add portable-pty, close-fds)

Reference: `docs/refs/projects/canopy.md`, `docs/refs/projects/wezterm.md` for PTY lifecycle.

- [ ] **Step 1: Add `portable-pty = "0.9"` and `close-fds = "0.4"` to Cargo.toml**
- [ ] **Step 2: Implement shell detection** (fallback chain: $SHELL → /etc/passwd → /bin/sh)
- [ ] **Step 3: Implement env preparation** (TERM, TERM_PROGRAM, PATH augmentation, filter vars)
- [ ] **Step 4: Implement `spawn_pty()`** (openpty, spawn_command, drop slave, take_writer, setsid, close_fds, PR_SET_PDEATHSIG)
- [ ] **Step 5: Integration test** — spawn `echo hello`, read output, verify EOF
- [ ] **Step 6: Commit**

---

### Task 15: PTY Actor (Registry + Actor Loop)

**Files:**
- Create: `src-tauri/src/process/actor.rs`

Reference: `docs/research/pty/actor-patterns.md` for full design.

- [ ] **Step 1: Implement `PtyCommand` enum, `PtyHandle`, `PtyRegistry`**
- [ ] **Step 2: Implement `pty_actor_loop()`** — tokio::select! over read, pending writes, commands. Ring buffer + 4ms flush timer. DEC 2026 sync awareness. Escape filter on output. Channel send.
- [ ] **Step 3: Implement `PtyRegistry::spawn()`** — spawn PTY, create actor task, register handle
- [ ] **Step 4: Implement `PtyRegistry::kill_all()`** — shutdown all actors
- [ ] **Step 5: Integration test** — spawn echo, read output via channel, kill, verify cleanup
- [ ] **Step 6: Commit**

---

### Task 16: Tauri PTY Commands

**Files:**
- Create: `src-tauri/src/commands/pty.rs`
- Modify: `src-tauri/src/commands/mod.rs`
- Modify: `src-tauri/src/main.rs` (manage PtyRegistry, register commands)

- [ ] **Step 1: Implement `start_entity` command** — accepts Channel<Vec<u8>>, spawns PTY via registry, updates Running component
- [ ] **Step 2: Implement `stop_entity` command** — kills via registry, removes Running component
- [ ] **Step 3: Implement `restart_entity` command** — stop + start
- [ ] **Step 4: Implement `pty_write` command** — sends Write to actor
- [ ] **Step 5: Implement `pty_resize` command** — sends Resize to actor
- [ ] **Step 6: Register all commands in main.rs, manage PtyRegistry state**
- [ ] **Step 7: Commit**

---

### Task 17: TerminalView Component (xterm.js)

**Files:**
- Create: `src/components/views/TerminalView.svelte`
- Modify: `src/App.svelte` (show TerminalView when terminal selected)

- [ ] **Step 1: Create TerminalView.svelte** — xterm.js instance with FitAddon, SearchAddon, renderer fallback (WebGL → Canvas → DOM), webglcontextlost handler
- [ ] **Step 2: Wire Channel** — on mount, invoke `start_entity` with Channel, write data to xterm.js on `onmessage`
- [ ] **Step 3: Wire input** — `terminal.onData()` → `pty_write`
- [ ] **Step 4: Wire resize** — FitAddon + ResizeObserver (debounced 100ms) → `pty_resize`
- [ ] **Step 5: Wire cleanup** — on destroy, delete `channel.onmessage`, unlisten all handlers
- [ ] **Step 6: Add error boundary wrapper**
- [ ] **Step 7: Add xterm.js parser handlers** — OSC 52 focus gating, OSC 7 CWD tracking, paste de-fanging
- [ ] **Step 8: Test** — create project + terminal in tree, click terminal, type `ls`, see output
- [ ] **Step 9: Commit**

---

## Phase 4: Kill Propagation + Session Restore (Tasks 18–21)

Produces: Clean shutdown, process cleanup, and session restore on app restart.

---

### Task 18: Kill Propagation System

**Files:**
- Create: `src-tauri/src/systems/kill.rs`
- Create: `src-tauri/src/systems/mod.rs`
- Create: `src-tauri/src/commands/lifecycle.rs`
- Modify: `src-tauri/src/commands/mod.rs`

- [ ] **Step 1: Implement `KillProjectSystem`** — query children by BelongsTo, depth-first kill by component type (Pty only for M1), respect Pinned, two-step kill sequence (SIGHUP foreground → poll → SIGKILL)
- [ ] **Step 2: Implement `kill_project` command** — calls KillProjectSystem
- [ ] **Step 3: Test** — create project with 2 terminals, kill project, verify all processes dead
- [ ] **Step 4: Commit**

---

### Task 19: App Shutdown Handlers

**Files:**
- Modify: `src-tauri/src/main.rs` (RunEvent::ExitRequested, on_window_event)

- [ ] **Step 1: Add `RunEvent::ExitRequested` handler** — kill all PTYs via registry, WAL checkpoint
- [ ] **Step 2: Add `on_window_event(Destroyed)` handler** — try_state(), kill PTY children
- [ ] **Step 3: Install SIGHUP handler** (Unix only) — trigger clean shutdown
- [ ] **Step 4: Write `last_clean_shutdown` timestamp to DB on clean exit**
- [ ] **Step 5: Test** — launch app, start terminals, quit, verify no orphaned processes
- [ ] **Step 6: Commit**

---

### Task 20: Session Restore System

**Files:**
- Create: `src-tauri/src/systems/restore.rs`
- Modify: `src-tauri/src/main.rs` (run restore on startup)

Reference: `docs/research/storage/session-restore.md` for edge cases.

- [ ] **Step 1: Implement Phase 1** — load entities, integrity check, migration version check, Order normalization
- [ ] **Step 2: Implement Phase 2** — stale PID cleanup with (pid, started_at) validation, kill stale process groups
- [ ] **Step 3: Implement Phase 3** — validate cwd/cmd, shell fallback, bounded concurrency (semaphore 4), per-entity catch_unwind, crash loop detection (<2s), env var filtering
- [ ] **Step 4: Implement Phase 4** — PRAGMA optimize, restore summary
- [ ] **Step 5: Emit restore events to frontend** — entity:component-changed for each restored entity
- [ ] **Step 6: Test** — create project + terminals, set RestoreOnStart, quit, relaunch, verify terminals respawn
- [ ] **Step 7: Commit**

---

### Task 21: CI + Final Polish

**Files:**
- Create: `.github/workflows/ci.yml`
- Modify: `CLAUDE.md` (update status)
- Modify: `CHANGELOG.md` (add M1 entries)

- [ ] **Step 1: Create GitHub Actions CI**

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: oven-sh/setup-bun@v2
      - name: Install Linux deps
        if: runner.os == 'Linux'
        run: sudo apt-get update && sudo apt-get install -y libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev
      - run: bun install
      - run: cargo test --manifest-path src-tauri/Cargo.toml
      - run: cargo clippy --manifest-path src-tauri/Cargo.toml -- -D warnings
      - run: cargo fmt --manifest-path src-tauri/Cargo.toml -- --check
      - run: cargo audit --manifest-path src-tauri/Cargo.toml
      - run: bun run test
```

- [ ] **Step 2: Update CLAUDE.md** — change status from "Pre-implementation" to "M1 in progress"
- [ ] **Step 3: Update CHANGELOG.md** — add M1 entries
- [ ] **Step 4: Commit and push**
- [ ] **Step 5: Verify CI passes on all platforms**

---

## Task Dependency Graph

```
Phase 1 (scaffold + storage):
  T1 → T2 → T3 → T4 → T5 → T6 → T7

Phase 2 (tree UI):
  T7 → T8 → T9 → T10

Phase 3 (PTY engine):
  T5 → T11 (ring buffer, independent)
  T5 → T12 (escape filter, independent)
  T5 → T13 (process manager, independent)
  T11 + T12 + T13 → T14 (PTY spawn)
  T14 → T15 (actor)
  T15 → T16 (Tauri PTY commands)
  T16 + T10 → T17 (TerminalView)

Phase 4 (lifecycle):
  T16 → T18 (kill propagation)
  T18 → T19 (shutdown handlers)
  T19 → T20 (session restore)
  T20 + T17 → T21 (CI + polish)
```

Tasks T11, T12, T13 can run in parallel (independent). T8 can start as soon as T7 is done, even while Phase 3 tasks are in progress.

---

## Notes for Implementers

- **Reference research, not ref code.** All code is written fresh. `docs/research/` has pseudocode and design rationale. `docs/refs/projects/` has per-project analysis. Never copy code from refs.
- **Test on Linux first** (dev environment is WSL2). Windows and macOS testing via CI.
- **Pin `portable-pty` version.** v0.9.x has had breaking changes on Windows (wezterm #6783). Pin to a known working version.
- **`$state.raw()` everywhere** for entity/tree data. Never `$state()` — 5,000x perf cliff.
- **Check `channel.send()` result.** If it fails, the frontend has reloaded or the component was destroyed. Clean up the actor.
- **`try_state()` in shutdown handlers.** Not `state()` — panics during abnormal shutdown.
- **Use `app.path().app_data_dir()`** for DB path instead of `dirs` crate (Tauri-idiomatic).
- **`$state.raw()`** does not accept generic type params. Use `let entities: T[] = $state.raw([])` not `$state.raw<T>([])`.
- **Emit events from PTY commands too** — `start_entity` should emit `entity:component-changed` when `Running` is set, `stop_entity` when `Running` is removed.
- **`tempfile::TempDir`** in tests instead of `NamedTempFile` (Windows compatibility).
- **DEC 2026 sync output** — xterm.js handles this internally. The ring buffer flush timer in the actor does NOT need to track it. Spec should be updated to clarify this.
- **`PersistSystem` and `LifecycleSystem`** are not separate structs in this plan. Logic is distributed across Tauri commands and the actor loop. This is a valid simplification for M1.
- **Drag-to-reorder and drag-to-reparent** are spec features not covered by this plan. Add as a follow-up task after M1 core is working.
- **Add `sysinfo` crate to Cargo.toml** if implementing foreground process tracking or PID validation via `/proc`.
- **Add `vitest.config.ts`** with Svelte preprocessor for `.svelte.ts` files.
- **Tasks 14, 15, 17, 20 are dense** — implementers should split them further during execution if needed.
