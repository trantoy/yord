# helix

> Post-modern TUI text editor with built-in LSP and tree-sitter. MPL-2.0 (inspiration only). [GitHub](https://github.com/helix-editor/helix)

## Overview

TUI-based modal editor. 7 crates: helix-core (rope, selection, syntax), helix-view (editor state, document mgmt, view tree), helix-term (TUI, app loop, compositor), helix-event (async events with debouncing), helix-lsp, helix-dap, helix-tui.

Relevant to Yord: SlotMap-backed split tree (cleanest layout tree implementation). Separate DocumentId/ViewId (multiple views → same document, confirms Yord's entity/layout-leaf separation). Biased tokio::select! priority ordering in event loop.

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
| `refs/helix/helix-view/src/tree.rs` | SlotMap-backed split tree: parent pointers, recalculate |
| `refs/helix/helix-view/src/editor.rs` | Central Editor state: tree + documents BTreeMap |
| `refs/helix/helix-event/src/lib.rs` | Hook-based event system with async debouncing |
| `refs/helix/helix-term/src/application.rs` | Main loop: biased tokio::select! with priority ordering |
| `refs/helix/helix-term/src/job.rs` | Job system: wait_futures, callback dispatch |
