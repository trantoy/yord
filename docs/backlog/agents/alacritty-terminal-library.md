# alacritty_terminal as Server-Side Terminal State

## Context

Yord currently sends raw PTY bytes to xterm.js and has no server-side terminal state. For agent nodes (M4), agents need to "read" terminal output — query what's currently on screen, parse command output, etc.

## Zed's approach

Zed uses `alacritty_terminal` as a library — `Term<ZedListener>` provides a full terminal emulator (VTE parsing, grid, scrolling, selection, search) on the Rust side. Events bridged through channels to the UI.

## Use case for Yord

- Agent context wiring: "agent, look at what's in terminal X" requires knowing terminal content
- Search across terminal output server-side
- Command output parsing for structured data extraction

## Trade-offs

- Adds complexity: maintaining two terminal states (Rust + xterm.js)
- Coupling to alacritty's internal APIs (they change between versions, `unsafe` workarounds needed)
- Memory: full grid per terminal on the Rust side

## When to revisit

M4 (Agent nodes). Only needed when agents need to read terminal content. The ring buffer provides raw byte history; `alacritty_terminal` provides parsed/structured state.
