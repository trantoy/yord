# Refs Research Structure — Design Spec

> Organize reference project research into a self-contained `docs/refs/` directory with consistent per-project files, clear rules, and a template for incremental research.

---

## 1. Problem

The refs research is scattered: raw links in `docs/refs/list.md`, the survey report in `docs/research/refs-report.md`, and no structure for tracking per-project findings. There are no documented rules about how refs should be used (inspiration vs code copying). Future implementation plans need a solid ground of "what problems did others hit and how did they solve them" — this doesn't exist yet.

## 2. Goals

- Self-contained `docs/refs/` directory for all ref-related research
- Clear rules: all code is written fresh, refs are for inspiration only regardless of license
- Consistent per-project files that can be filled incrementally over multiple sessions
- Per-project files structured around four research categories: bugs & solutions, tech debt, lessons, and issues
- Clean separation: `docs/refs/` for ref-specific research, `docs/research/` for broader research

## 3. Rules for Refs

These rules apply to all reference projects regardless of license:

1. **All code must be generated or written fresh.** No code is copied from any ref, even MIT or Apache-2.0 licensed projects.
2. **Refs are for inspiration and understanding.** Study their architecture, patterns, bugs, and solutions. Then write your own implementation.
3. **Inspiration can come from anywhere.** GPL, MPL, unlicensed — doesn't matter for inspiration. The license distinction exists for legal awareness, not for changing the "write your own code" rule.
4. **License table is informational.** It documents what each project's license permits, so contributors understand the legal landscape. It does not grant permission to copy code.

## 4. Directory Structure

### After reorganization

```
docs/refs/
├── README.md              — rules, index, license table, research status
├── list.md                — raw links with license groups (exists, unchanged)
├── report.md              — survey report (moved from docs/research/refs-report.md)
├── canopy.md              — per-project research
├── clif-code.md
├── waveterm.md
├── wezterm.md
├── zellij.md
├── zed.md
├── helix.md
├── tauri-terminal.md
└── tauri-plugin-pty.md
```

### Unchanged

```
docs/research/
├── ide-pain-points.md     — broader research, not ref-specific
├── ide-path.md
└── tauri-pitfalls.md
```

## 5. Per-Project File Template

Each per-project file follows this structure:

```markdown
# {Project Name}

> One-line description. License. Link.

## Overview
<!-- What it is, tech stack, relation to Yord. -->

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
<!-- Specific files worth studying, with one-line reason. -->
```

**Section purposes:**

| Section | What goes here | Sources |
|---------|---------------|---------|
| Overview | What it is, stack, Yord relevance | README, Cargo.toml, package.json |
| Architecture Decisions | High-level choices and trade-offs | Source structure, docs, design docs |
| Bugs Encountered & How Solved | Past bugs and fixes | Git history, closed issues, code comments |
| Current Bugs & Tech Debt | Known limitations now | Open issues, TODOs, code smells |
| What To Learn | Patterns for Yord inspiration | Source code analysis |
| Issues | Issue tracker findings for M1 | GitHub issues |
| Key Files | Files worth studying | Source exploration |

## 6. README.md Structure

```markdown
# Reference Projects

> Refs are for inspiration. All code is written fresh.

## Rules
{rules from section 3}

## License Table
{project / license / can-take-code? / status}

## Index
{links to all files in docs/refs/}

## Research Status
{which per-project files are filled vs empty template}
```

## 7. License Corrections

During spec review, a discrepancy was found: `tauri-plugin-pty` was listed as "Undefined" in `list.md` but confirmed MIT in both `Cargo.toml` and `package.json`. Fixed: moved to MIT group in `list.md`. `tauri-terminal` remains the only unlicensed ref.

## 8. Implementation Steps

1. Move `docs/research/refs-report.md` to `docs/refs/report.md`
2. Write `docs/refs/README.md` with rules, license table, index, status
3. Create 9 per-project files from template (Overview and Key Files pre-filled from report's per-project sections and Key Files Index, other sections empty)
4. Update `docs/research/` if needed (remove dangling refs)

---

*Spec date: 2026-03-21*
