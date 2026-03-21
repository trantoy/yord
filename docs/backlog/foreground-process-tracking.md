# Foreground Process Tracking

## Context

M1 shows static labels in the tree ("terminal", "zsh"). With foreground process tracking, labels update dynamically: "zsh" → "npm run dev" → "node server.js".

## How it works

- `libc::tcgetpgrp(pty_fd)` returns the foreground process group PID
- `sysinfo` crate provides process name, CWD, and command args for that PID
- Cache with 300ms TTL (wezterm pattern) to avoid per-call overhead

## Platform support

- Linux: `/proc/{pid}/stat`, `/proc/{pid}/exe`, `/proc/{pid}/cwd`, `/proc/{pid}/cmdline`
- macOS: `proc_listallpids()` + `proc_pidinfo(PROC_PIDTBSDINFO)`, `proc_pidpath()`
- Windows: `Toolhelp32Snapshot` + `OpenProcess` + `NtQueryInformationProcess`

Note: wezterm's `procinfo` crate is not on crates.io. Yord must vendor or reimplement.

## When to revisit

M1 stretch goal. Low-hanging fruit for better UX — skip if time-constrained.
