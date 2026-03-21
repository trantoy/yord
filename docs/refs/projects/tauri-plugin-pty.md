# tauri-plugin-pty

> Tauri v2 plugin exposing PTY functionality with node-pty-compatible API. MIT. [GitHub](https://github.com/Tnze/tauri-plugin-pty)

## Overview

Tauri v2 plugin providing multi-session PTY management. Uses portable-pty 0.9, exposes spawn/read/write/resize/kill commands via Tauri plugin system. TypeScript API mirrors node-pty's `IPty` interface. Sessions tracked in `BTreeMap<u32, Arc<Session>>` behind `RwLock`.

Relevant to Yord: closest existing Tauri v2 PTY plugin. Multi-session pattern directly applicable.

## Architecture Decisions
<!-- Their high-level choices, trade-offs, why X over Y.
     Including decisions Yord should NOT follow. -->

## Bugs Encountered & How Solved
<!-- What broke, why, how they fixed it. From issues, commits, code comments. -->

## Current Bugs & Tech Debt
<!-- Open issues, TODOs in code, known limitations. -->

## What To Learn
<!-- Patterns and approaches worth drawing inspiration from for Yord. -->

## Issues
<!-- Notable issue tracker findings relevant to Yord's M1. -->

## Key Files

| File | Why |
|------|-----|
| `refs/tauri-plugin-pty/src/lib.rs` | Entire Rust plugin (216 lines): session management, spawn/read/write/resize/kill/exitstatus commands |
| `refs/tauri-plugin-pty/api/index.ts` | TypeScript API (309 lines): TauriPty class with node-pty-compatible interface |
| `refs/tauri-plugin-pty/api/eventEmitter2.ts` | Minimal event emitter (48 lines) |
| `refs/tauri-plugin-pty/build.rs` | Tauri v2 plugin build script |
| `refs/tauri-plugin-pty/examples/vanilla/src/index.js` | Example app showing xterm.js integration |
| `refs/tauri-plugin-pty/examples/vanilla/src-tauri/capabilities/default.json` | Tauri v2 permissions config |
