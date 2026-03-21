# ConPTY Artifact Filtering

Windows ConPTY injects unsolicited escape sequences (cursor position reports, scrollback manipulation). These need a platform-specific filter catalog so they don't corrupt terminal output or trigger false positive in escape sequence sanitization.

## Priority

M1 (Windows support) or M2. Depends on when Windows testing starts.

## Referenced from

- `docs/research/from-issues/cross-platform.md` — 19 ConPTY bugs documented
- `docs/research/pty/escape-filtering.md` — notes ConPTY artifact handling needed
