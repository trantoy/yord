# Issue Analysis Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Mine 32,297 GitHub issues from 9 reference repos for bugs, pitfalls, and design validation relevant to Yord's M1.

**Architecture:** Tiered filtering (read all from small repos, labels + keywords from big repos), then categorize findings into 5 output documents.

**Tech Stack:** Python (json, re) for search/filter, agents for analysis, markdown output.

**Spec:** `docs/superpowers/specs/2026-03-21-issue-analysis-design.md`

---

### Task 1: Label discovery

**Files:**
- Create: `docs/research/from-issues/label-analysis.md`

- [ ] **Step 1: Extract all labels from each repo's issues**

```bash
for repo in canopy clif-code helix tauri-plugin-pty tauri-terminal waveterm wezterm zed zellij; do
  echo "=== $repo ==="
  python3 -c "
import json
from collections import Counter
issues = json.load(open('docs/refs/issues/${repo}.json'))
labels = Counter()
for i in issues:
    for l in i.get('labels', []):
        name = l['name'] if isinstance(l, dict) else l
        labels[name] += 1
for name, count in labels.most_common(30):
    print(f'  {count:>5}  {name}')
"
done
```

- [ ] **Step 2: Identify bug/crash/regression/security labels per repo**

From the label output, note which labels indicate bugs relevant to M1. Create a mapping for Task 3.

- [ ] **Step 3: Write label-analysis.md**

Write `docs/research/from-issues/label-analysis.md` with:
- Label frequency table per repo
- Bug category distribution across repos
- Which bug types are most common (testing priority recommendation)
- Header: `> Based on analysis of reference project issue trackers. See [docs/refs/README.md](../../refs/README.md) for rules.`

- [ ] **Step 4: Commit**

```bash
git add docs/research/from-issues/label-analysis.md
git commit -m "docs: add label analysis from ref project issues"
```

---

### Task 2: Keyword search across all issues

**Files:**
- Create: `docs/research/from-issues/raw-findings.md`

- [ ] **Step 1: Run keyword search**

Search title + body of all 32k issues for M1-relevant keywords. Use Python to search and output matching issue numbers + titles + repo.

Keywords by category:

**PTY/process:** `pty`, `terminal`, `kill`, `process`, `orphan`, `zombie`, `sigterm`, `sigkill`, `sighup`, `process group`, `pgid`, `setsid`

**Session/restore:** `restore`, `session`, `persist`, `resurrect`, `startup`, `stale pid`, `cleanup`

**Terminal rendering:** `xterm`, `escape`, `sequence`, `osc 52`, `clipboard`, `resize`, `focus`, `rendering`, `webgl`, `canvas`

**Platform:** `ConPTY`, `WebKitGTK`, `WKWebView`, `wsl`, `macos`, `linux`, `windows`

**Storage:** `sqlite`, `wal`, `database`, `corrupt`, `migration`

**Design-specific:** `actor`, `channel`, `ring buffer`, `backpressure`, `deadlock`, `mutex`, `thread pool`, `leak`, `ipc`

- [ ] **Step 2: Filter to bug-relevant issues**

From keyword matches, filter out:
- Feature requests with no bug content
- Issues about editor-specific features (irrelevant to Yord)
- Duplicates (same issue matched by multiple keywords)

Keep issues that describe:
- A real bug with symptoms
- A crash or data loss
- A workaround or fix
- A platform-specific behavior

- [ ] **Step 3: Write raw-findings.md**

Write `docs/research/from-issues/raw-findings.md` with all relevant issues organized by keyword category. For each issue: repo, number, title, state (open/closed), relevant labels, 1-line summary of the bug.

- [ ] **Step 4: Commit**

```bash
git add docs/research/from-issues/raw-findings.md
git commit -m "docs: add raw keyword findings from ref project issues"
```

---

### Task 3: Tier 1 — Read all small repo issues

**Files:**
- Modify: `docs/research/from-issues/raw-findings.md` (append)

- [ ] **Step 1: Read all canopy issues (15)**

Read every issue from `docs/refs/issues/canopy.json`. For each, note: is it relevant to M1? What category? What's the bug/lesson?

- [ ] **Step 2: Read all tauri-plugin-pty issues (3)**

Same for `docs/refs/issues/tauri-plugin-pty.json`.

- [ ] **Step 3: Read all tauri-terminal issues (3)**

Same for `docs/refs/issues/tauri-terminal.json`.

- [ ] **Step 4: Append findings to raw-findings.md and commit**

```bash
git add docs/research/from-issues/raw-findings.md
git commit -m "docs: add tier 1 small repo issue findings"
```

---

### Task 4: Tier 2 — Analyze big repo bug issues

**Files:**
- Modify: `docs/research/from-issues/raw-findings.md` (append)

- [ ] **Step 1: Filter big repos by bug labels**

Using the label mapping from Task 1, extract issues with bug/crash/regression labels from wezterm, zellij, waveterm, zed, helix. Intersect with keyword matches from Task 2 to find the highest-value issues.

- [ ] **Step 2: Dispatch analysis agents (one per repo)**

For each big repo, dispatch an agent to read the filtered issues and categorize findings by M1 area. Agent prompt should include:
- The filtered issue list (number, title, body snippet)
- M1 scope reference
- Categorization instructions (PTY spawning, kill propagation, session restore, SQLite, xterm.js, cross-platform)

- [ ] **Step 3: Merge agent findings into raw-findings.md and commit**

```bash
git add docs/research/from-issues/raw-findings.md
git commit -m "docs: add tier 2 big repo bug issue analysis"
```

---

### Task 5: Write known-pitfalls.md

**Files:**
- Create: `docs/research/from-issues/known-pitfalls.md`

- [ ] **Step 1: Categorize all findings by M1 area**

From raw-findings.md, group issues into sections:

```markdown
## PTY Spawning Pitfalls
## Kill Propagation Edge Cases
## Session Restore Failures
## SQLite Issues
## xterm.js Rendering Bugs
## Keyboard / Focus / IME
## Security (Escape Sequences)
```

- [ ] **Step 2: Write known-pitfalls.md**

For each section, list pitfalls with:
- What happens (symptom)
- Root cause
- Which ref project hit it
- Fix/workaround applied
- Yord relevance (does our spec already handle this? if not, what to add)

- [ ] **Step 3: Commit**

```bash
git add docs/research/from-issues/known-pitfalls.md
git commit -m "docs: add known pitfalls from ref project issues"
```

---

### Task 6: Write cross-platform.md

**Files:**
- Create: `docs/research/from-issues/cross-platform.md`

- [ ] **Step 1: Filter findings by platform**

From raw-findings.md, extract all platform-specific issues. Group by:

```markdown
## Windows
### ConPTY
### Job Objects / Process Groups
### Path Handling

## macOS
### WKWebView
### PATH / GUI App Environment
### Cocoa fd Leaks

## Linux
### WebKitGTK
### WSL
### Terminal / Terminfo
```

- [ ] **Step 2: Write cross-platform.md and commit**

```bash
git add docs/research/from-issues/cross-platform.md
git commit -m "docs: add cross-platform bug catalog from ref project issues"
```

---

### Task 7: Write spec-validation.md

**Files:**
- Create: `docs/research/from-issues/spec-validation.md`

- [ ] **Step 1: Match findings against M1 spec design decisions**

Read `docs/superpowers/specs/2026-03-21-m1-tree-shell-design.md`. For each major design decision, check if any issues challenge or validate it:

- Per-PTY actor pattern (any actor/deadlock/ordering issues?)
- Ring buffer 256KB + 4ms flush (any sizing issues?)
- Escape filtering two-layer defense (any sequences that got through?)
- `Channel<T>` for streaming (any channel/streaming bugs?)
- Single-writer SQLite + WAL (any corruption/contention issues?)
- Process group kill with SIGHUP-first (any kill failures?)
- Progressive session restore (any restore edge cases we missed?)

- [ ] **Step 2: Write spec-validation.md**

For each design decision:
- Validation: issues that confirm our approach is correct
- Challenges: issues that suggest we need to change something
- Additions: edge cases the spec doesn't cover yet

- [ ] **Step 3: Commit**

```bash
git add docs/research/from-issues/spec-validation.md
git commit -m "docs: add spec validation from ref project issues"
```

---

### Task 8: Update M1 spec + final commit

**Files:**
- Modify: `docs/superpowers/specs/2026-03-21-m1-tree-shell-design.md`
- Modify: `docs/research/README.md`

- [ ] **Step 1: Apply spec changes from spec-validation.md**

For any "Challenges" or "Additions" found in Task 7, update the M1 spec.

- [ ] **Step 2: Update research README**

Add `from-issues/` to the topic list in `docs/research/README.md`.

- [ ] **Step 3: Final commit**

```bash
git add docs/superpowers/specs/2026-03-21-m1-tree-shell-design.md docs/research/README.md
git commit -m "docs: update M1 spec with findings from issue analysis"
```

---

### Task 9: Verify

- [ ] **Step 1: Verify all output files exist**

```bash
ls -1 docs/research/from-issues/
```

Expected:
```
known-pitfalls.md
cross-platform.md
spec-validation.md
label-analysis.md
raw-findings.md
```

- [ ] **Step 2: Verify research README updated**

```bash
grep "from-issues" docs/research/README.md
```
