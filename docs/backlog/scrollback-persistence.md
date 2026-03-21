# Scrollback Persistence

## Context

M1 restore respawns the command but does not preserve terminal scrollback. Users lose all previous output on restart.

## Proven pattern (waveterm)

1. PTY output appended to circular blockfile (2MB cap) in SQLite filestore
2. Periodically, `@xterm/addon-serialize` creates a terminal state snapshot → `cache:term:full`
3. On restore: load snapshot, replay only data written after the snapshot's `ptyoffset`
4. Result: fast restore without replaying entire history

## Yord approach

- Rust ring buffer (256KB) already exists per PTY for output batching
- Extend to persist: on PTY exit or periodic flush, write ring buffer to SQLite blob
- On restore: pre-populate xterm.js with persisted buffer before new shell starts
- Display a visual separator between old and new session output

## Considerations

- `@xterm/addon-serialize` for full terminal state (cursor position, colors, alt-screen)
- Terminal size mismatch on restore: waveterm temp-resizes to cached size, writes, resizes back
- Storage: ~2MB per terminal × N terminals. Cap total storage or use LRU eviction.

## When to revisit

Post-M1, pre-M2. Low-hanging fruit once the ring buffer is in place.
