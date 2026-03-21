# tauri-terminal

> Minimal Tauri v1 terminal demo with single PTY and xterm.js. No license. [GitHub](https://github.com/marc2332/tauri-terminal)

## Overview

Proof-of-concept single-PTY terminal in Tauri v1. Uses portable-pty 0.8, requestAnimationFrame polling for PTY output. Entire backend is 125 lines, frontend is 62 lines. App exits when the shell exits.

Relevant to Yord: minimal reference for portable-pty + xterm.js integration. Tauri v1 (not v2), so IPC patterns don't directly apply.

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
| `refs/tauri-terminal/src-tauri/src/main.rs` | Entire Rust backend (125 lines) |
| `refs/tauri-terminal/src/main.ts` | Entire frontend (62 lines) |
