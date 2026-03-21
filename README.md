# Yord

AI-native workspace manager — projects own their tools, not the other way around.

## What is this?

Modern development spans multiple tools: editor, terminal, browser, file manager, AI agent. These tools are unaware of each other. The developer is the glue.

Yord inverts this. The project is the root. Tools are children. Kill a project, kill its children. Restore a session, everything comes back exactly as you left it.

## Core Ideas

- **Project is the primitive** — not the tool, not the window
- **Everything is a node** — terminals, editors, browsers, agents, dev servers all live in a tree
- **Process tree != Layout tree** — what's running is separate from how it's arranged on screen
- **Session restore** — close and reopen, pick up where you left off
- **Bring your own tools** — Yord doesn't force an editor or terminal

## Architecture

Hybrid ECS + Tree. Data stored flat (entities + components), rendered as a tree in the UI. See [docs/architecture.md](docs/architecture.md) for details.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| UI | Svelte 5 + TypeScript |
| Desktop shell | Tauri 2 |
| Backend | Rust |
| Terminal | xterm.js |
| PTY | portable-pty |
| Storage | SQLite (rusqlite, single-writer + WAL) |
| IPC | `#[tauri::command]` + `Channel<T>` |

## Status

Pre-implementation. M1 target: tree + shell.

- [Design doc](docs/design.md)
- [Architecture](docs/architecture.md)
- [M1 spec](docs/superpowers/specs/2026-03-21-m1-tree-shell-design.md)
- [M1 plan](docs/superpowers/plans/2026-03-21-m1-tree-shell.md)

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
- MIT License ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
