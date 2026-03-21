# Clif-Code

> Monorepo: ClifPad (Tauri 2 IDE with Monaco + terminal) and ClifCode (TUI AI coding agent). MIT. [GitHub](https://github.com/DLhugly/Clif-Code)

## Overview

Two products: ClifPad is a ~20MB native code editor/IDE (Tauri 2 + SolidJS + Monaco + portable-pty 0.8 + xterm.js v6). ClifCode is a standalone TUI AI coding agent (pure Rust, no runtime). ClifPad supports multi-window, file watching (notify 7), and AI agent integration.

Relevant to Yord: alternative PTY pattern with kill flag + window-scoped PTY tracking. 32KB read buffer (vs canopy's 4KB). Uses `emit_to()` events instead of Channel. File tree component with drag-and-drop.

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
| `refs/Clif-Code/clif-pad-ide/src-tauri/src/commands/pty.rs` | PTY with kill flag, window scoping, 32KB buffer |
| `refs/Clif-Code/clif-pad-ide/src-tauri/src/lib.rs` | Tauri setup with multi-window, menu building, dual state |
| `refs/Clif-Code/clif-pad-ide/src/components/terminal/TerminalPanel.tsx` | Terminal component with tab management, auto-restart |
| `refs/Clif-Code/clif-pad-ide/src/stores/terminalStore.ts` | SolidJS store for terminal tab state |
| `refs/Clif-Code/clif-pad-ide/src/lib/tauri.ts` | Complete IPC wrapper layer |
| `refs/Clif-Code/clif-code-tui/src/session.rs` | Session persistence and context compaction |
