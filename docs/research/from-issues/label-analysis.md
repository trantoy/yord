# Label Analysis — Reference Project Issues

> Based on analysis of 32,297 issues across 9 reference project issue trackers (canopy, clif-code, helix, tauri-plugin-pty, tauri-terminal, waveterm, wezterm, zed, zellij). See [docs/refs/README.md](../../refs/README.md) for rules.

## Summary

Across all reference projects, **bugs dominate issue trackers**: 34.8% of all issues carry a bug label, compared to 19.5% for feature/enhancement requests. The ratio is even more skewed in pure-terminal projects — wezterm has 67% bugs, helix 43%, waveterm 40%.

The most common bug areas (drawn primarily from zed and helix, which have detailed area labels) are:

1. **Terminal/PTY behavior** — rendering, input handling, escape sequences, compatibility (zed: 662 terminal issues, 240 terminal bugs; helix: 391 helix-term bugs; wezterm: 114 keyboard bugs, 166 Wayland bugs)
2. **Cross-platform issues** — Windows, Wayland, macOS differences (zed: 571 Linux + 211 Windows + 124 macOS bugs; wezterm: 175 Windows + 166 Wayland + 134 macOS bugs; zellij: 81 compatibility co-labeled bugs)
3. **Editor/workspace state** — crashes in editor core, workspace serialization, settings (zed: 629 editor bugs, 446 workspace bugs, 234 settings bugs)
4. **Language server / integration** — LSP crashes and failures (zed: 381 language-server bugs, 147 server-failure bugs; helix: 106 language-server bugs)
5. **Performance & memory** — leaks, freezes, render latency (zed: 438 performance issues, 75 memory leaks, 88 freezes; wezterm: 6 render-latency, 3 model-latency)
6. **Crashes** — 379 crash-labeled issues in zed alone, 216 regressions
7. **Input/keyboard** — keybinding conflicts, IME, mouse handling (zed: 883 keybind issues, 225 mouse issues; wezterm: 136 keyboard bugs)

**Testing priority recommendation**: Terminal I/O (PTY read/write, resize, escape sequence handling), cross-platform behavior (especially Wayland and Windows), process lifecycle (spawn/kill/orphan), and session state persistence are where terminal-oriented projects break most. Yord should invest testing effort there first.

## Per-Repo Label Frequency

### canopy (15 issues, 8 unique labels)

| Label | Count |
|-------|------:|
| enhancement | 14 |
| performance | 5 |
| code quality | 3 |
| good first issue | 3 |
| bug | 1 |
| cross-platform | 1 |
| testing | 1 |
| accessibility | 1 |

### clif-code (0 issues)

No issues in tracker.

### helix (5,760 issues, 41 unique labels)

| Label | Count |
|-------|------:|
| C-bug | 2,478 |
| C-enhancement | 2,299 |
| A-helix-term | 937 |
| R-duplicate | 342 |
| E-easy | 188 |
| E-good-first-issue | 171 |
| A-language-server | 171 |
| A-language-support | 143 |
| A-tree-sitter | 119 |
| upstream | 95 |
| A-documentation | 85 |
| A-keymap | 82 |
| A-theme | 79 |
| A-packaging | 73 |
| E-has-instructions | 59 |
| A-command | 58 |
| A-core | 53 |
| E-help-wanted | 49 |
| A-gui | 42 |
| C-discussion | 40 |
| O-windows | 37 |
| A-plugin | 33 |
| E-medium | 28 |
| S-waiting-on-author | 26 |
| A-debug-adapter | 26 |
| E-hard | 23 |
| R-wontfix | 22 |
| A-vcs | 19 |
| A-indent | 16 |
| S-needs-discussion | 13 |
| S-waiting-on-review | 11 |
| O-macos | 9 |
| A-dependencies | 7 |
| C-perf | 4 |
| rust | 2 |
| O-linux | 2 |
| hacktoberfest | 2 |
| github_actions | 1 |
| S-needs-testing | 1 |
| R-breaking-change | 1 |
| S-inactive | 1 |

### tauri-plugin-pty (3 issues, 0 unique labels)

No labels used. All issues unlabeled.

### tauri-terminal (3 issues, 0 unique labels)

No labels used. All issues unlabeled.

### waveterm (700 issues, 11 unique labels)

| Label | Count |
|-------|------:|
| triage | 297 |
| enhancement | 279 |
| bug | 278 |
| legacy | 222 |
| maintainer-interest | 15 |
| duplicate | 7 |
| question | 7 |
| need more info | 5 |
| documentation | 5 |
| wontfix | 4 |
| invalid | 2 |

### wezterm (4,275 issues, 38 unique labels)

| Label | Count |
|-------|------:|
| bug | 2,843 |
| enhancement | 1,064 |
| fixed-in-nightly | 704 |
| needs:triage | 468 |
| Windows | 199 |
| Wayland | 174 |
| waiting-on-op | 162 |
| macOS | 155 |
| keyboard | 136 |
| docs | 127 |
| multiplexer | 88 |
| X11 | 46 |
| PR-welcome | 38 |
| cant-reproduce | 24 |
| Stale | 22 |
| Linux | 18 |
| termwiz | 17 |
| wontfix | 10 |
| conpty | 10 |
| frontend:webgpu | 10 |
| lua-api | 9 |
| needs:decision | 8 |
| render-latency | 6 |
| tmux -CC | 5 |
| nvidia | 5 |
| ssh | 5 |
| fonts | 3 |
| upstream:libssh | 3 |
| model-latency | 3 |
| needs:design | 2 |
| performance | 2 |
| freebsd | 2 |
| flatpak | 2 |
| duplicate | 2 |
| downstream:arch | 1 |
| upstream:libssh2 | 1 |
| upstream:mutter | 1 |
| donotreap | 1 |

### zed (18,992 issues, 215 unique labels)

Top 40 labels (of 215 total):

| Label | Count |
|-------|------:|
| bug | 4,763 |
| feature | 2,503 |
| state:needs repro | 1,973 |
| frequency:common | 1,710 |
| area:editor | 1,703 |
| area:ai | 1,587 |
| priority:P2 | 1,519 |
| priority:P3 | 1,466 |
| stale | 1,332 |
| state:reproducible | 1,266 |
| area:languages | 1,212 |
| platform:linux | 1,056 |
| platform:windows | 1,016 |
| frequency:uncommon | 1,013 |
| area:workspace | 999 |
| area:controls/keybinds | 883 |
| area:settings | 874 |
| area:parity/vim | 813 |
| area:language server | 723 |
| meta:duplicate | 710 |
| area:integrations/terminal | 662 |
| area:integrations/git | 640 |
| support | 541 |
| frequency:always | 482 |
| area:project panel | 465 |
| area:performance | 438 |
| crash | 379 |
| area:languages/python | 377 |
| area:ai/assistant | 369 |
| meta:easy repro steps | 360 |
| platform:macOS | 354 |
| area:search | 343 |
| area:popovers | 310 |
| area:languages/rust | 278 |
| area:languages/typescript | 270 |
| area:ai/agent thread | 268 |
| state:needs info | 265 |
| area:extensions/infrastructure | 260 |
| never stale | 258 |
| state:needs triage | 257 |

### zellij (2,549 issues, 30 unique labels)

| Label | Count |
|-------|------:|
| suspected bug | 879 |
| enhancement | 132 |
| help wanted | 114 |
| compatibility | 87 |
| good first issue | 62 |
| action | 58 |
| config | 49 |
| plugin system | 44 |
| duplicate | 29 |
| layout | 29 |
| plugin | 27 |
| stability | 22 |
| hacktoberfest | 21 |
| discussion | 20 |
| darwin | 17 |
| mouse | 16 |
| ease of use | 16 |
| theme | 14 |
| documentation | 13 |
| build | 13 |
| input | 7 |
| packaging | 7 |
| defaults | 6 |
| active development report | 4 |
| windows | 3 |
| Needs Reproduction | 3 |
| Plugin Idea | 2 |
| bsd | 2 |
| dependencies | 2 |
| mentored | 1 |

## Bug/Crash/Regression Labels by Repo

This mapping identifies which labels indicate bugs in each project's labeling scheme.

| Repo | Bug labels | Crash labels | Regression labels |
|------|-----------|-------------|-------------------|
| canopy | `bug` | — | — |
| clif-code | (no issues) | — | — |
| helix | `C-bug` | — | — |
| tauri-plugin-pty | (no labels used) | — | — |
| tauri-terminal | (no labels used) | — | — |
| waveterm | `bug` | — | — |
| wezterm | `bug` | — | — |
| zed | `bug` | `crash` | `meta:regression` |
| zellij | `suspected bug` | — | — |

### Bug sub-areas (co-labels on bug issues)

**Helix** — bugs by area:

| Area | Bug count |
|------|----------:|
| A-helix-term (terminal) | 391 |
| A-language-server | 106 |
| A-tree-sitter | 58 |
| A-language-support | 46 |
| A-packaging | 26 |
| A-command | 23 |
| A-documentation | 23 |
| A-core | 19 |
| A-theme | 18 |
| A-debug-adapter | 11 |
| A-keymap | 10 |
| A-indent | 8 |
| A-gui | 8 |

**Wezterm** — bug co-labels:

| Co-label | Bug count |
|----------|----------:|
| fixed-in-nightly | 538 |
| needs:triage | 360 |
| Windows | 175 |
| Wayland | 166 |
| waiting-on-op | 148 |
| macOS | 134 |
| keyboard | 114 |
| multiplexer | 71 |
| X11 | 42 |
| cant-reproduce | 24 |
| conpty | 9 |
| nvidia | 5 |
| render-latency | 5 |

**Zed** — bug issues by area (top 20):

| Area | Bug count |
|------|----------:|
| area:languages | 703 |
| area:editor | 629 |
| area:workspace | 446 |
| area:language server | 381 |
| area:parity/vim | 307 |
| area:controls/keybinds | 279 |
| area:integrations/terminal | 240 |
| area:settings | 234 |
| area:ai | 210 |
| area:performance | 201 |
| area:ai/assistant | 175 |
| area:project panel | 171 |
| area:language server/server failure | 147 |
| area:languages/typescript | 144 |
| area:popovers | 141 |
| area:languages/rust | 121 |
| area:languages/python | 107 |
| area:internationalization | 106 |
| area:search | 104 |
| area:integrations/git | 102 |

**Zed** — crash issues by area (top 10):

| Area | Crash count |
|------|------------:|
| area:editor | 27 |
| area:workspace | 26 |
| area:languages | 17 |
| area:gpui | 13 |
| area:ai | 11 |
| area:ai/assistant | 11 |
| area:performance | 10 |
| area:settings | 10 |
| area:collab | 9 |
| area:search | 8 |

**Zed** — bugs by platform:

| Platform | Bug+crash count |
|----------|----------------:|
| platform:linux | 571 |
| platform:windows | 211 |
| platform:macOS | 124 |
| platform:linux/wayland | 101 |
| platform:linux/x11 | 97 |
| platform:remote | 77 |

**Zellij** — bug co-labels:

| Co-label | Bug count |
|----------|----------:|
| compatibility | 81 |
| help wanted | 23 |
| stability | 17 |
| darwin | 13 |
| good first issue | 9 |
| duplicate | 8 |
| plugin | 8 |
| plugin system | 5 |
| layout | 4 |
| config | 4 |
| mouse | 3 |
| theme | 3 |

## Bug Category Distribution

Across all 32,297 issues (from repos that use labels):

| Category | Issues | % of total |
|----------|-------:|-----------:|
| Bug (all types) | 11,242 | 34.8% |
| Feature / enhancement | 6,291 | 19.5% |
| Crash | 379 | 1.2% |
| Regression | 216 | 0.7% |
| (Unlabeled / other) | ~14,369 | 44.5% |

### Bug rates by project type

| Project | Type | Issues | Bug % | Feature % |
|---------|------|-------:|------:|----------:|
| wezterm | terminal emulator | 4,275 | 67% | 25% |
| helix | terminal editor | 5,760 | 43% | 40% |
| waveterm | terminal+workspace | 700 | 40% | 40% |
| zellij | terminal multiplexer | 2,549 | 34% | 5% |
| zed | GUI editor | 18,992 | 25% | 13% |
| canopy | workspace manager | 15 | 7% | 93% |
| tauri-plugin-pty | PTY library | 3 | 0%* | 0%* |
| tauri-terminal | terminal component | 3 | 0%* | 0%* |

*No labels used on these small projects.

### What bugs look like by domain

**Terminal I/O & rendering** (highest volume):
- Wezterm: 2,843 bugs total, with keyboard (114), Wayland (166), Windows (175) dominating
- Helix: 391 of 2,478 bugs are in `A-helix-term` (terminal layer) — 16% of all bugs
- Zed: 240 terminal-area bugs, 662 terminal-area issues overall
- Zellij: 81 of 879 bugs tagged `compatibility` (terminal escape sequence / app compat)

**Cross-platform** (second highest):
- Wezterm: Windows (175), Wayland (166), macOS (134) — 475 platform bugs = 17% of all bugs
- Zed: Linux (571), Windows (211), macOS (124) — 906 platform bugs
- Zellij: darwin (13), windows (2), bsd (1)

**Keyboard/input**:
- Wezterm: 114 keyboard-labeled bugs
- Zed: 279 keybind-area bugs, 225 mouse bugs, 17 IME issues
- Zellij: 3 mouse + 1 input bugs

**Crashes & stability**:
- Zed: 379 crash-labeled issues, 88 freeze-labeled issues, 75 memory leaks
- Zellij: 22 stability-labeled issues

## Testing Priority Recommendation

Based on where reference projects break most, Yord should prioritize testing in these areas (ordered by impact):

### 1. Terminal / PTY layer (highest priority)
- **What breaks**: PTY read/write, resize events, escape sequence handling, shell prompt detection, process spawning
- **Evidence**: Wezterm's bug tracker is 67% bugs; helix-term is the single largest bug area in helix (391 bugs); zed has 240 terminal bugs; zellij has 81 compatibility bugs
- **Testing approach**: Property-based tests for PTY I/O, fuzzing escape sequences, integration tests for spawn/resize/kill lifecycle

### 2. Cross-platform behavior
- **What breaks**: Windows ConPTY, Wayland vs X11 rendering, macOS-specific behavior
- **Evidence**: Wezterm has 475 platform-specific bugs (17% of total); zed has 906 platform bugs; every terminal project shows platform as a top bug co-label
- **Testing approach**: CI matrix across OS targets, platform-specific PTY integration tests, Wayland protocol testing

### 3. Process lifecycle management
- **What breaks**: Orphaned processes, kill propagation failures, zombie processes, multiple terminal instances causing deadlocks
- **Evidence**: canopy issue #23 (close_terminal orphan leak), tauri-plugin-pty issue #2 (deadlock at 4+ PTY instances), zed workspace crashes (26 crash issues)
- **Testing approach**: Stress tests for concurrent terminal spawn/kill, process tree verification after close, session restore after crash

### 4. Keyboard and input handling
- **What breaks**: Keybinding conflicts, IME support, mouse events, modifier keys, non-US keyboard layouts
- **Evidence**: Wezterm has 114 keyboard bugs; zed has 279 keybind bugs + 225 mouse bugs; tauri-terminal issue #5 (characters missed when typing)
- **Testing approach**: Input event simulation, IME integration tests, modifier key matrix testing

### 5. Performance and memory
- **What breaks**: Memory leaks from long-running terminals, render latency, UI freezes from blocking I/O
- **Evidence**: Zed has 438 performance issues, 75 memory leaks, 88 freezes; canopy issues #15 (blocking sleep), #18 (no timeout on gh CLI), #26 (blocking I/O)
- **Testing approach**: Long-running soak tests, memory profiling in CI, async I/O audit, timeout enforcement

### 6. State persistence and serialization
- **What breaks**: Session restore, workspace state, settings deserialization
- **Evidence**: Zed has 143 serialization issues, 234 settings bugs; waveterm has 222 legacy issues (migration/compat); zed has 216 regressions often in settings/workspace
- **Testing approach**: Roundtrip serialization tests, schema migration tests, corrupt-state recovery tests
