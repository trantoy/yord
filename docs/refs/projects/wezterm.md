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
