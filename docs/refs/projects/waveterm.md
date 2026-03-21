# waveterm

> AI-integrated terminal with drag-and-drop block layouts and durable SSH sessions. Apache-2.0. [GitHub](https://github.com/wavetermdev/waveterm)

## Overview

Open-source terminal emulator with AI chat widgets, file previews, and drag-and-drop block layouts. Go backend + Electron + React 19 + SQLite + xterm.js v5.5. Hierarchy: Client → Window → Workspace → Tab → Block. Blocks differentiated by `Meta.view` (term, preview, codeeditor, waveai, web) — similar to Yord's component-based identity.

Relevant to Yord: most sophisticated scrollback persistence (circular buffer + xterm.js serialize addon snapshots). SQLite with JSON blobs per table. Graceful kill: SIGHUP → 400ms → SIGKILL. Controller registry pattern (blockId → Controller interface). Durable remote shells via job manager.

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
| `refs/waveterm/pkg/shellexec/shellexec.go` | PTY spawning for local/SSH/WSL |
| `refs/waveterm/pkg/shellexec/conninterface.go` | ConnInterface with graceful kill |
| `refs/waveterm/pkg/blockcontroller/shellcontroller.go` | Output/input loops, process lifecycle |
| `refs/waveterm/pkg/blockcontroller/blockcontroller.go` | Controller registry, block close, shutdown |
| `refs/waveterm/pkg/waveobj/wtype.go` | All object types (Client, Window, Workspace, Tab, Block, Job) |
| `refs/waveterm/pkg/wstore/wstore_dbsetup.go` | SQLite setup (WAL mode, single conn) |
| `refs/waveterm/pkg/wstore/wstore_dbops.go` | Generic CRUD for wave objects |
| `refs/waveterm/pkg/filestore/blockstore.go` | Circular buffer for terminal scrollback |
| `refs/waveterm/frontend/app/view/term/termwrap.ts` | xterm.js wrapper with addons, IME, paste handling |
