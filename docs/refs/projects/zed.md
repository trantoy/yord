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
