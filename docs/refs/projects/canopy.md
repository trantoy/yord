# canopy

> Tauri v2 workspace manager for Claude Code sessions with terminal multiplexing. MIT. [GitHub](https://github.com/The-Banana-Standard/canopy)

## Overview

Desktop app for managing Claude Code CLI sessions and plain terminals across project workspaces. Tauri v2 + React 19 + TypeScript + portable-pty 0.8 + SQLite (via tauri-plugin-sql) + xterm.js v6. Includes daily planner, GitHub dashboard, skills manager, session persistence.

Relevant to Yord: closest stack match (Tauri 2 + PTY + SQLite + xterm.js). Uses `Channel<TerminalEvent>` for PTY streaming (same approach Yord plans). Tab session persistence via localStorage. PATH augmentation for macOS GUI apps.

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
| `refs/canopy/src-tauri/src/commands/terminal.rs` | PTY lifecycle: spawn, reader thread, Channel streaming, cleanup |
| `refs/canopy/src-tauri/src/state.rs` | AppState with terminal instance map |
| `refs/canopy/src-tauri/src/lib.rs` | Tauri app setup, plugin registration, PTY cleanup on window destroy |
| `refs/canopy/src/hooks/useTerminal.ts` | Tab management, session persistence, status tracking |
| `refs/canopy/src/components/Terminal/TerminalView.tsx` | xterm.js integration, attention detection, session restore |
| `refs/canopy/src/services/terminal-service.ts` | IPC wrappers for PTY commands using Tauri Channel |
| `refs/canopy/src/services/database-service.ts` | SQLite schema and CRUD operations |
| `refs/canopy/src-tauri/src/commands/projects.rs` | Workspace scanning, tech stack detection |
