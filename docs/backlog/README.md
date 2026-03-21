# Backlog

Deferred features and ideas, organized by topic. Each item has a target milestone.

## By Milestone

| Milestone | Items |
|-----------|-------|
| M1 stretch | Foreground process tracking |
| M2 | Layout engine, workspace switcher, scrollback persistence, taurpc, IME, multi-parent nodes |
| M3 | Web nodes (browser, editor, file manager), port pool |
| M4 | Agent nodes, alacritty_terminal for agent context |
| M5 | Quick nav, theming, onboarding, virtual scrolling |
| Future | Browser platform, multi-window, remote/SSH, WASM plugins, durable remote shells, CRDT/collab, TUI file manager IPC, domain migrations, bevy_ecs, ConPTY filtering |

## By Topic

### [pty/](pty/)
- [foreground-process-tracking.md](pty/foreground-process-tracking.md) — dynamic tree labels via tcgetpgrp (M1 stretch)
- [conpty-artifact-filtering.md](pty/conpty-artifact-filtering.md) — Windows ConPTY escape sequence catalog (Future)
- [durable-remote-shells.md](pty/durable-remote-shells.md) — SSH sessions surviving disconnects (Future)

### [storage/](storage/)
- [scrollback-persistence.md](storage/scrollback-persistence.md) — persist terminal scrollback across sessions (M2)
- [domain-based-migrations.md](storage/domain-based-migrations.md) — per-crate SQLite migration domains (Future)
- [bevy-ecs-runtime.md](storage/bevy-ecs-runtime.md) — swap SQLite ECS for bevy_ecs (Future)

### [frontend/](frontend/)
- [layout-engine.md](frontend/layout-engine.md) — tiling split/tabbed renderer (M2)
- [workspace-switcher.md](frontend/workspace-switcher.md) — Cmd+K project switcher (M2)
- [quick-nav.md](frontend/quick-nav.md) — Cmd+P fuzzy search (M5)
- [theming.md](frontend/theming.md) — per-project accent colors, dark/light (M5)
- [virtual-scrolling.md](frontend/virtual-scrolling.md) — large tree performance (M5)
- [onboarding.md](frontend/onboarding.md) — first-run experience (M5)

### [platform/](platform/)
- [browser-platform.md](platform/browser-platform.md) — browser platform support (M3+)
- [multi-window.md](platform/multi-window.md) — detach projects to OS windows (Future)
- [remote-projects.md](platform/remote-projects.md) — SSH remote development (Future)
- [web-nodes.md](platform/web-nodes.md) — browser, editor, file manager nodes (M3)

### [agents/](agents/)
- [agent-nodes.md](agents/agent-nodes.md) — first-class agent entities (M4)
- [alacritty-terminal-library.md](agents/alacritty-terminal-library.md) — server-side terminal state for agent context (M4)

### [integration/](integration/)
- [taurpc-typed-ipc.md](integration/taurpc-typed-ipc.md) — auto-gen TS types from Rust (M2)
- [wasm-plugins.md](integration/wasm-plugins.md) — WASM plugin system (Future)
- [ime-handling.md](integration/ime-handling.md) — advanced CJK input (M2)
- [tui-filemanager-ipc.md](integration/tui-filemanager-ipc.md) — yazi plugin API integration (Future)

### [infrastructure/](infrastructure/)
- [multi-parent-nodes.md](infrastructure/multi-parent-nodes.md) — BelongsTo with parent_ids[] (M2+)
- [crdt-collaboration.md](infrastructure/crdt-collaboration.md) — shared workspace via CRDT (Future)
