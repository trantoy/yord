# Agents

Instructions for AI coding agents working on this repository.

## Project

Yord is a project-scoped workspace manager built with Tauri 2 (Rust backend) + Svelte 5 (frontend) + SQLite (storage).

## Architecture

Hybrid ECS + Tree. Entities and components are stored flat in SQLite. The UI builds a tree view from flat data. See `docs/architecture.md` for the full design.

Key concepts:
- **Entity** — a unique ID with a set of components
- **Component** — pure data (Label, BelongsTo, Pty, Running, etc.)
- **System** — Rust logic that queries entities by component and processes them

## Key Documents

- `docs/design.md` — full product design, data models, milestones
- `docs/architecture.md` — hybrid ECS + tree architecture rationale

## Stack

- **Backend**: Rust (Tauri 2 commands, process management, PTY, SQLite)
- **Frontend**: Svelte 5 (reactive UI, xterm.js for terminals, webviews for browsers/editors)
- **IPC**: Tauri commands (TS → Rust) + Tauri events (Rust → TS)
- **Storage**: SQLite with `entities` + `components` tables (EAV pattern)

## Conventions

- Rust: `cargo fmt`, `cargo clippy` clean
- Frontend: ESLint + Prettier
- Commits: imperative mood, concise
- New node types: add as new components, not new enum variants where possible
- Layout: components on entities, not a separate tree structure

## Current Milestone

M1 — Tree + Shell. Focus: Tauri scaffold, process tree, SQLite persistence, tree panel UI, terminal nodes with PTY, kill propagation, session restore.
