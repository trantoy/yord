# Refs Research Structure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reorganize reference project research into a self-contained `docs/refs/` directory with consistent per-project files and documented rules.

**Architecture:** Move the survey report from `docs/research/` to `docs/refs/`. Write a README with rules and index. Create 9 per-project files from a template, pre-filling Overview and Key Files from the existing report.

**Spec:** `docs/superpowers/specs/2026-03-21-refs-research-structure-design.md`

---

### Task 1: Move survey report

**Files:**
- Move: `docs/research/refs-report.md` → `docs/refs/report.md`

- [ ] **Step 1: Move the file**

```bash
mv docs/research/refs-report.md docs/refs/report.md
```

- [ ] **Step 2: Verify**

```bash
ls docs/refs/report.md && ! ls docs/research/refs-report.md 2>/dev/null && echo "OK"
```

- [ ] **Step 3: Commit**

```bash
git add docs/research/refs-report.md docs/refs/report.md
git commit -m "docs: move refs report to docs/refs/"
```

---

### Task 2: Write README.md

**Files:**
- Overwrite: `docs/refs/README.md` (currently empty)

- [ ] **Step 1: Write README with rules, license table, index, and research status**

Content:

```markdown
# Reference Projects

> Refs are for inspiration. All code is written fresh.

## Rules

1. **All code must be generated or written fresh.** No code is copied from any ref, even MIT or Apache-2.0 licensed projects.
2. **Refs are for inspiration and understanding.** Study architecture, patterns, bugs, solutions. Then write your own implementation.
3. **Inspiration can come from anywhere.** GPL, MPL, unlicensed — doesn't matter for inspiration. The license distinction is for legal awareness only.
4. **License table is informational.** It documents what each project's license permits. It does not grant permission to copy code.

## License Table

| Project | License | Permits Code Use | Status |
|---------|---------|-----------------|--------|
| [waveterm](https://github.com/wavetermdev/waveterm) | Apache-2.0 | Yes (with notice) | Surveyed |
| [canopy](https://github.com/The-Banana-Standard/canopy) | MIT | Yes | Surveyed |
| [Clif-Code](https://github.com/DLhugly/Clif-Code) | MIT | Yes | Surveyed |
| [wezterm](https://github.com/wezterm/wezterm) | MIT | Yes | Surveyed |
| [zellij](https://github.com/zellij-org/zellij) | MIT | Yes | Surveyed |
| [tauri-plugin-pty](https://github.com/Tnze/tauri-plugin-pty) | MIT | Yes | Surveyed |
| [zed](https://github.com/zed-industries/zed) | GPL-3.0 | No | Surveyed |
| [helix](https://github.com/helix-editor/helix) | MPL-2.0 | No | Surveyed |
| [tauri-terminal](https://github.com/marc2332/tauri-terminal) | None (unlicensed) | No | Surveyed |

"Permits Code Use" is informational — see Rule 1.

## Index

- [list.md](list.md) — raw links with license groups
- [report.md](report.md) — survey report across all refs
- [canopy.md](canopy.md)
- [clif-code.md](clif-code.md)
- [waveterm.md](waveterm.md)
- [wezterm.md](wezterm.md)
- [zellij.md](zellij.md)
- [zed.md](zed.md)
- [helix.md](helix.md)
- [tauri-terminal.md](tauri-terminal.md)
- [tauri-plugin-pty.md](tauri-plugin-pty.md)

## Research Status

| File | Overview | Arch Decisions | Bugs & Fixes | Tech Debt | Lessons | Issues | Key Files |
|------|----------|---------------|-------------|-----------|---------|--------|-----------|
| canopy.md | Done | - | - | - | - | - | Done |
| clif-code.md | Done | - | - | - | - | - | Done |
| waveterm.md | Done | - | - | - | - | - | Done |
| wezterm.md | Done | - | - | - | - | - | Done |
| zellij.md | Done | - | - | - | - | - | Done |
| zed.md | Done | - | - | - | - | - | Done |
| helix.md | Done | - | - | - | - | - | Done |
| tauri-terminal.md | Done | - | - | - | - | - | Done |
| tauri-plugin-pty.md | Done | - | - | - | - | - | Done |
```

- [ ] **Step 2: Verify README renders correctly**

```bash
head -5 docs/refs/README.md
```

- [ ] **Step 3: Commit**

```bash
git add docs/refs/README.md
git commit -m "docs: add refs README with rules, license table, and index"
```

---

### Task 3: Create per-project files — Tauri PTY group

**Files:**
- Create: `docs/refs/tauri-plugin-pty.md`
- Create: `docs/refs/tauri-terminal.md`

- [ ] **Step 1: Create tauri-plugin-pty.md**

Pre-fill Overview and Key Files from report. Content:

```markdown
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
```

- [ ] **Step 2: Create tauri-terminal.md**

```markdown
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
```

- [ ] **Step 3: Commit**

```bash
git add docs/refs/tauri-plugin-pty.md docs/refs/tauri-terminal.md
git commit -m "docs: add per-project research files for tauri-plugin-pty and tauri-terminal"
```

---

### Task 4: Create per-project files — canopy and Clif-Code

**Files:**
- Create: `docs/refs/canopy.md`
- Create: `docs/refs/clif-code.md`

- [ ] **Step 1: Create canopy.md**

```markdown
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
```

- [ ] **Step 2: Create clif-code.md**

```markdown
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
```

- [ ] **Step 3: Commit**

```bash
git add docs/refs/canopy.md docs/refs/clif-code.md
git commit -m "docs: add per-project research files for canopy and Clif-Code"
```

---

### Task 5: Create per-project files — wezterm and zellij

**Files:**
- Create: `docs/refs/wezterm.md`
- Create: `docs/refs/zellij.md`

- [ ] **Step 1: Create wezterm.md**

```markdown
# wezterm

> GPU-accelerated terminal emulator and multiplexer in Rust. Origin of portable-pty crate. MIT. [GitHub](https://github.com/wezterm/wezterm)

## Overview

Full-featured terminal emulator with multiplexing, tabs, panes, ligatures, and GPU rendering. ~220-crate Rust workspace. Key sub-crate: `portable-pty` (published on crates.io) which Yord plans to use as a dependency.

Relevant to Yord: portable-pty API and internals (PtySystem, MasterPty, Child, ChildKiller traits). Process tree tracking via /proc. Two-thread PTY reader pipeline with coalescing. Kill pattern: SIGHUP → 250ms grace → SIGKILL. No session persistence (mux server keeps processes alive instead).

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
| `refs/wezterm/pty/src/lib.rs` | portable-pty API: all traits (PtySystem, MasterPty, Child, ChildKiller) |
| `refs/wezterm/pty/src/unix.rs` | Unix PTY: openpty, setsid, signal reset, fd cleanup |
| `refs/wezterm/pty/src/win/conpty.rs` | Windows ConPTY implementation |
| `refs/wezterm/pty/examples/bash.rs` | Minimal portable-pty usage example |
| `refs/wezterm/mux/src/lib.rs` | read_from_pane_pty: two-thread reader pipeline with coalescing |
| `refs/wezterm/procinfo/src/linux.rs` | Full process tree from /proc |
```

- [ ] **Step 2: Create zellij.md**

```markdown
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
```

- [ ] **Step 3: Commit**

```bash
git add docs/refs/wezterm.md docs/refs/zellij.md
git commit -m "docs: add per-project research files for wezterm and zellij"
```

---

### Task 6: Create per-project files — waveterm

**Files:**
- Create: `docs/refs/waveterm.md`

- [ ] **Step 1: Create waveterm.md**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add docs/refs/waveterm.md
git commit -m "docs: add per-project research file for waveterm"
```

---

### Task 7: Create per-project files — zed and helix

**Files:**
- Create: `docs/refs/zed.md`
- Create: `docs/refs/helix.md`

- [ ] **Step 1: Create zed.md**

```markdown
# zed

> High-performance code editor built in Rust with GPUI framework. GPL-3.0 (inspiration only). [GitHub](https://github.com/zed-industries/zed)

## Overview

224-crate Rust monorepo. Custom GPUI framework with built-in entity system and GPU rendering. Uses `alacritty_terminal` as a library for integrated terminal. Custom `sqlez` SQLite wrapper with domain-based migrations.

Relevant to Yord: GPUI entity model (typed entity store with reactive observation — EntityMap via SlotMap, Entity<T> handles, read(cx)/update(cx) access, observe/subscribe reactivity). Two-step kill: `killpg()` foreground process group then shell. Foreground process tracking via `tcgetpgrp()` + sysinfo. Domain-based SQLite migrations. Pane group as recursive enum.

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
| `refs/zed/crates/gpui/src/app/entity_map.rs` | Entity store: SlotMap + ref counting + typed handles |
| `refs/zed/crates/gpui/src/app/context.rs` | Context methods: observe, subscribe, notify, emit |
| `refs/zed/crates/gpui/src/_ownership_and_data_flow.rs` | Reactive system documentation |
| `refs/zed/crates/terminal/src/terminal.rs` | PTY integration via alacritty_terminal |
| `refs/zed/crates/terminal/src/pty_info.rs` | Foreground process tracking via tcgetpgrp + sysinfo |
| `refs/zed/crates/workspace/src/pane_group.rs` | Recursive enum: Member::Pane / Member::Axis |
| `refs/zed/crates/workspace/src/persistence.rs` | Workspace → SQLite serialization |
| `refs/zed/crates/sqlez/src/domain.rs` | Domain-based migration pattern |
```

- [ ] **Step 2: Create helix.md**

```markdown
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
```

- [ ] **Step 3: Commit**

```bash
git add docs/refs/zed.md docs/refs/helix.md
git commit -m "docs: add per-project research files for zed and helix"
```

---

### Task 8: Verify final structure

- [ ] **Step 1: List all files in docs/refs/**

```bash
ls -1 docs/refs/
```

Expected output:
```
README.md
canopy.md
clif-code.md
helix.md
list.md
report.md
tauri-plugin-pty.md
tauri-terminal.md
waveterm.md
wezterm.md
zed.md
zellij.md
```

- [ ] **Step 2: Verify docs/research/ no longer has refs-report.md**

```bash
ls -1 docs/research/
```

Expected output:
```
ide-pain-points.md
ide-path.md
tauri-pitfalls.md
```

- [ ] **Step 3: Verify all per-project files have the template sections**

```bash
for f in docs/refs/canopy.md docs/refs/clif-code.md docs/refs/waveterm.md docs/refs/wezterm.md docs/refs/zellij.md docs/refs/zed.md docs/refs/helix.md docs/refs/tauri-terminal.md docs/refs/tauri-plugin-pty.md; do
  echo "=== $(basename $f) ==="
  grep -c "^## " "$f"
done
```

Expected: each file shows `7` (7 h2 sections).
