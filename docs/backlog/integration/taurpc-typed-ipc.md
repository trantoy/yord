# taurpc for Typed IPC

## Context

M1 uses plain `#[tauri::command]` with manual TypeScript types. taurpc + specta would auto-generate TS types from Rust traits, eliminating type drift.

## Why deferred

taurpc doesn't support Tauri 2's `Channel<T>` — Yord's most critical IPC mechanism (PTY streaming). Using taurpc for commands while using raw channels for streaming creates two IPC patterns.

## When to revisit

M2 (Layout Engine). The IPC surface will grow significantly (layout commands, workspace switching, drag-and-drop). At 20+ commands, manual type maintenance becomes painful.

## Options

1. **taurpc adds Channel support** — check taurpc releases periodically
2. **Hybrid** — taurpc for CRUD/lifecycle commands, raw channels for PTY streaming
3. **`specta` only** — use specta for type generation without taurpc's proc macros
4. **Stay manual** — 11 M1 commands are manageable; keep it simple

## Decision

Deferred to M2. Monitor taurpc's Channel<T> support.
