# zellij

> Terminal workspace and multiplexer with plugin system (WASM). MIT. [GitHub](https://github.com/zellij-org/zellij)

## Overview

Terminal multiplexer with tiling/floating panes, tabs, sessions, and WASM plugin system. Client-server architecture over Unix domain sockets. 5 main crates: zellij-server (PTY, screen, plugins), zellij-client (input, IPC), zellij-utils (types, layout, serialization), zellij-tile (plugin API), zellij-tile-utils.

Relevant to Yord: session serialization/restore to KDL files. Raw `nix::pty::openpty()` with tokio `AsyncFd`. Layout tree structure (TiledPaneLayout recursive tree, but flat BTreeMap for runtime pane storage — same hybrid as Yord). Signal escalation: SIGHUP then SIGKILL. Per-thread architecture with crossbeam channels.

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
| `refs/zellij/zellij-server/src/os_input_output_unix.rs` | PTY spawn, kill, resize, signal handling |
| `refs/zellij/zellij-server/src/terminal_bytes.rs` | Async read loop: PTY bytes → ScreenInstruction::PtyBytes |
| `refs/zellij/zellij-server/src/pty.rs` | Pty struct, PtyInstruction enum, spawn/close orchestration |
| `refs/zellij/zellij-server/src/lib.rs` | Server startup, thread spawning, SessionMetaData::drop cleanup |
| `refs/zellij/zellij-utils/src/session_serialization.rs` | Session → KDL layout serialization |
| `refs/zellij/zellij-server/src/session_layout_metadata.rs` | Runtime metadata collection for persistence |
| `refs/zellij/zellij-utils/src/input/layout.rs` | Layout, TiledPaneLayout, SplitDirection |
| `refs/zellij/zellij-utils/src/pane_size.rs` | PaneGeom, Dimension, Constraint |
