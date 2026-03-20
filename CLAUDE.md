# Yord

Project-scoped workspace manager. Tauri 2 (Rust) + Svelte 5 + SQLite.

## Architecture

Hybrid ECS + Tree — flat entity/component storage, tree rendered as UI view.
See `docs/architecture.md` for full rationale.
Design doc: `docs/design.md`

## Stack

- Backend: Rust (Tauri 2 commands, PTY, process management)
- Frontend: Svelte 5 + xterm.js + Tauri webviews
- Storage: SQLite (entities + components tables, EAV pattern)
- IPC: Tauri commands (TS→Rust) + events (Rust→TS)

## Conventions

- Rust: `cargo fmt`, `cargo clippy` clean
- Frontend: ESLint + Prettier
- New node types: add as components, not enum variants
- Docs are in `docs/`, English language
- Dual licensed: MIT + Apache-2.0

## Current Status

Pre-implementation. M1 target: tree + shell (scaffold, process tree, SQLite, terminal PTY, kill propagation, session restore).
