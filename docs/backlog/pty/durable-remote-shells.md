# Durable Remote Shells

SSH sessions that survive disconnects. Waveterm's pattern: a "job manager" process runs on the remote host via setsid + daemonize, owns the PTY, and reconnects when the client comes back. Output buffered on remote side during disconnect.

## Priority

Future. Not in any current milestone. Requires remote project support first.

## Referenced from

- `docs/refs/projects/waveterm.md` — durable remote shell pattern
