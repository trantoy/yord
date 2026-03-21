# Issue Analysis Pipeline — Design Spec

> Mine 32,297 GitHub issues from 9 reference projects for bugs, pitfalls, and design validation relevant to Yord's M1.

---

## 1. Goal

Extract actionable knowledge from real-world issue trackers to:
- Preempt bugs we'll hit during M1 implementation
- Validate or challenge M1 spec design decisions
- Build reference documents for implementation
- Prioritize testing effort by bug frequency

## 2. Input

32,297 issues downloaded to `docs/refs/issues/` (gitignored, 200MB):

| Repo | Issues | Tier |
|------|--------|------|
| canopy | 15 | 1 (read all) |
| clif-code | 0 | 1 |
| tauri-plugin-pty | 3 | 1 (read all) |
| tauri-terminal | 3 | 1 (read all) |
| waveterm | 700 | 2 (labels + keywords) |
| wezterm | 4,275 | 2 (labels + keywords) |
| zellij | 2,549 | 2 (labels + keywords) |
| zed | 18,992 | 2 (labels + keywords) |
| helix | 5,760 | 2 (labels + keywords) |

## 3. Filtering Strategy

### Tier 1 — Read all (small repos, 21 issues)

canopy (15), tauri-plugin-pty (3), tauri-terminal (3). Every issue read and categorized. These are the closest to Yord's stack.

### Tier 2 — Labels + keywords (big repos, 32k+ issues)

**Step 1: Label discovery.** Extract all labels per repo, identify bug/crash/regression/security labels.

**Step 2: Label filter.** Pull all issues with bug/crash/regression labels.

**Step 3: Keyword search.** Search title + body across ALL issues for M1-relevant terms:

PTY/process keywords:
`pty`, `terminal`, `kill`, `process`, `orphan`, `zombie`, `sigterm`, `sigkill`, `sighup`, `process group`, `pgid`, `setsid`

Session/restore keywords:
`restore`, `session`, `persist`, `resurrect`, `startup`, `stale pid`, `cleanup`

Terminal rendering keywords:
`xterm`, `escape`, `sequence`, `osc 52`, `clipboard`, `resize`, `focus`, `rendering`, `webgl`, `canvas`

Platform keywords:
`ConPTY`, `WebKitGTK`, `WKWebView`, `wsl`, `macos`, `linux`, `windows`

Storage keywords:
`sqlite`, `wal`, `database`, `corrupt`, `migration`

Design-specific keywords:
`actor`, `channel`, `ring buffer`, `backpressure`, `deadlock`, `mutex`, `thread pool`, `leak`, `ipc`

**Step 4: Deduplicate.** Label hits and keyword hits will overlap. Deduplicate by issue number per repo.

## 4. Extraction Goals

### 4.1 PTY/Terminal Bug Mine
Search for real bugs users hit with PTY spawning, terminal rendering, process management. Extract: what broke, root cause, fix applied, whether it's relevant to Yord.

### 4.2 Known Pitfalls Document
Categorize all findings by M1 area:
- PTY spawning pitfalls
- Kill propagation edge cases
- Session restore failures
- SQLite issues
- xterm.js rendering bugs
- Cross-platform quirks

### 4.3 Spec Validation
Search for issues related to Yord's specific design choices:
- Actor pattern problems (deadlock, backpressure, ordering)
- Ring buffer sizing issues
- Escape filtering gaps (sequences that got through)
- Channel/streaming issues
- Single-writer SQLite problems
- Process group kill failures

### 4.4 Cross-Platform Bug Catalog
Filter for platform-specific issues:
- Windows: ConPTY bugs, Job Objects issues, path handling
- macOS: WKWebView jettison, PATH issues, Cocoa fd leaks
- Linux: WebKitGTK rendering, missing terminfo, WSL quirks

### 4.5 Label Analysis
Count bug categories across all repos to identify:
- Most common bug types (where to invest testing)
- Regression frequency (what breaks repeatedly)
- Security issues (what's been exploited)

## 5. Output

```
docs/research/from-issues/
├── known-pitfalls.md        — pitfalls by M1 area
├── cross-platform.md        — platform-specific bugs
├── spec-validation.md       — issues affecting design decisions
├── label-analysis.md        — bug frequency, testing priorities
└── raw-findings.md          — all relevant issues with links
```

All files follow the research rules: no copied code, ideas and analysis only. Each file gets the standard header linking to refs rules.

After analysis, update M1 spec with any findings that change design decisions.

## 6. Process

1. **Label discovery** — extract labels per repo, identify relevant ones
2. **Keyword search** — search title + body, collect matching issues
3. **Label filter** — pull bug/crash/regression labeled issues from big repos
4. **Deduplicate** — merge label + keyword results per repo
5. **Dispatch analysis agents** — one per repo (or per topic for big repos), categorize findings
6. **Write output docs** — compile into the 5 output files
7. **Update M1 spec** — apply findings that affect design decisions
8. **Commit**

## 7. Scope Boundary

- Only issues relevant to Yord's M1 scope (PTY, tree, kill, restore, SQLite, xterm.js)
- Skip feature requests, UI polish, editor-specific issues (irrelevant to Yord)
- Skip closed issues with no resolution/fix (nothing to learn from)
- Focus on bugs with clear root cause + fix (actionable knowledge)

---

*Spec date: 2026-03-21*
