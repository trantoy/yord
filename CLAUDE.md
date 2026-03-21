# Yord

Project-scoped workspace manager. Tauri 2 (Rust) + Svelte 5 + SQLite.

## Architecture

Hybrid ECS + Tree — flat entity/component storage, tree rendered as UI view.
See `docs/architecture.md` for full rationale.
Design doc: `docs/design.md`

## Stack

- Backend: Rust (Tauri 2 commands, PTY via portable-pty, process management)
- Frontend: Svelte 5 + TypeScript + xterm.js
- Storage: SQLite via rusqlite (single-writer + reader pool, WAL mode)
- IPC: `#[tauri::command]` + manual TS types (commands), `Channel<T>` (PTY streaming)
- Logging: `tracing` crate
- CSS: Plain CSS + CSS variables (scoped Svelte styles)
- Package manager: Bun

## Conventions

- Rust: `cargo fmt`, `cargo clippy` clean
- Frontend: ESLint + Prettier
- New node types: add as components, not enum variants
- Docs are in `docs/`, English language
- `docs/notes/` — personal notes in Russian, exception to English-only rule
- Dual licensed: MIT + Apache-2.0

## Refs Rules

Reference projects cloned to `/refs/` (gitignored). All code must be written fresh — refs are for inspiration only, regardless of license. See `docs/refs/README.md`.

## Docs Structure

```
docs/
├── architecture.md          — hybrid ECS + tree rationale
├── design.md                — full design document
├── refs/
│   ├── README.md            — rules, license table, index
│   ├── LIST.md              — project list
│   ├── report.md            — survey report
│   └── projects/            — per-project research (9 files)
├── research/
│   ├── pty/                 — actor patterns, output buffering, escape filtering
│   ├── storage/             — SQLite single-writer, session restore
│   ├── frontend/            — flat-to-tree (buildTree)
│   ├── landscape/           — IDE pain points, Tauri pitfalls
│   └── from-issues/         — 32k issue analysis (pitfalls, cross-platform, spec validation)
├── backlog/                 — deferred features
├── notes/                   — personal notes (Russian)
└── superpowers/
    ├── specs/               — design specs
    └── plans/               — implementation plans
```

## Current Status

Pre-implementation. M1 spec complete at `docs/superpowers/specs/2026-03-21-m1-tree-shell-design.md` (v3.1).

M1 scope: Tauri 2 + Svelte 5 scaffold, ECS storage, tree UI, terminal PTY, kill propagation, session restore.

M1 plan at `docs/superpowers/plans/2026-03-21-m1-tree-shell.md` (21 tasks, 4 phases).
