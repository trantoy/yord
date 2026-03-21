# Organizing your repo for AI coding agents

**AI coding agents perform dramatically better when repositories provide structured context files, strong type contracts, and automated quality gates.** The ecosystem has rapidly converged around a handful of file formats — CLAUDE.md, AGENTS.md, `.cursor/rules/`, and `.github/copilot-instructions.md` — each read by different tools but sharing ~90% identical content. The most effective repositories treat these files as first-class project artifacts: concise (under 200 lines), specific (exact commands, not vague principles), and layered (root-level defaults with subdirectory overrides). This report covers every major dimension of AI-agent-friendly repository organization, with templates, real-world examples, and tool-specific guidance.

---

## The instruction file landscape: which tools read what

Every major AI coding tool now reads a project-level instruction file, but no single file works everywhere. Understanding the compatibility matrix is essential for multi-tool teams.

**CLAUDE.md** is Anthropic's format for Claude Code. Files load hierarchically: `~/.claude/CLAUDE.md` (global), `./CLAUDE.md` or `./.claude/CLAUDE.md` (project root), and subdirectory CLAUDE.md files load on demand when Claude reads files in those directories. A `CLAUDE.local.md` variant stays out of version control for personal preferences. Contents are injected as a user message (not system prompt), consuming context window tokens. The `/init` command auto-generates a starter file by analyzing your codebase. Claude Code also supports scoped rules in `.claude/rules/*.md` files with glob-based activation — useful for large projects where different subsystems need different instructions.

**AGENTS.md** is the cross-tool open standard, now stewarded by the **Agentic AI Foundation under the Linux Foundation**. It is natively supported by **20+ tools**: OpenAI Codex, GitHub Copilot, Cursor, Windsurf, Gemini CLI, Jules, Amp (Sourcegraph), Aider (via config), Devin, Zed, Warp, and others. The notable exception is Claude Code, which does not natively read AGENTS.md — a symlink bridges the gap. The official recommendation is to keep AGENTS.md under **150 lines**, and more-deeply-nested files take precedence over root-level ones.

**`.cursor/rules/*.mdc`** is Cursor's current system, replacing the legacy `.cursorrules` file. Each rule file uses YAML frontmatter with four activation modes: **Always** (`alwaysApply: true`), **Auto Attached** (activated by `globs` patterns), **Agent Requested** (model reads `description` and decides), and **Manual** (only when `@mentioned`). Cursor also natively reads AGENTS.md, so shared rules can live there while Cursor-specific behaviors go in `.cursor/rules/`.

**`.github/copilot-instructions.md`** is GitHub Copilot's repository-level instruction file. It applies to Copilot Chat, Code Review, and the Copilot Coding Agent — but **not** inline autocomplete suggestions. Copilot also supports path-specific instructions via `.github/instructions/*.instructions.md` with `applyTo` glob frontmatter. Importantly, GitHub Copilot now also reads `AGENTS.md`, `CLAUDE.md`, and `GEMINI.md` at the repo root, merging all found instructions.

**Windsurf** uses `.windsurf/rules/*.md` files with YAML frontmatter (`trigger: always_on | model_decision | glob | manual`) and hard character limits: **6,000 characters** for global rules, **12,000 characters** per workspace rule file. **Aider** has no magic filename — load any conventions file via `--read CONVENTIONS.md` or configure it in `.aider.conf.yml`.

| File | Primary Tool | Also Read By | Placement | Limit |
|------|-------------|-------------|-----------|-------|
| `CLAUDE.md` | Claude Code | Copilot, Amp | Root, subdirs, `~/.claude/` | Context window (keep concise) |
| `AGENTS.md` | Codex, Jules | Copilot, Cursor, Windsurf, 15+ more | Root + subdirs | ≤150 lines recommended |
| `.cursor/rules/*.mdc` | Cursor | Cursor only | `.cursor/rules/` | ~500 lines/file |
| `.github/copilot-instructions.md` | GitHub Copilot | Copilot only | `.github/` | Keep concise |
| `.windsurf/rules/*.md` | Windsurf | Windsurf only | `.windsurf/rules/` | 12,000 chars/file |
| `CONVENTIONS.md` | Aider (via config) | — | Any (loaded explicitly) | No formal limit |

### Recommended content structure for any instruction file

The most effective instruction files across thousands of repositories share six core sections. Here is a universal template that works for CLAUDE.md, AGENTS.md, or any tool:

```markdown
# Project Name

## Build & test commands
npm run build          # Full build
npm run dev            # Development server
npm run lint           # ESLint (--max-warnings=0)
npm run typecheck      # tsc --noEmit
npm run test           # Vitest (all tests)
npm run test -- path/to/file.test.ts  # Single test

## Project structure
- src/           — Application source (Svelte frontend)
- src-tauri/     — Rust backend (Tauri commands)
- docs/          — Architecture docs and ADRs
- tests/         — Integration tests

## Code style & conventions
- TypeScript: strict mode, no `any` types, prefer explicit return types
- Rust: run `cargo clippy -- -D warnings` before committing
- Svelte: use runes ($state, $derived), not legacy reactive syntax
- Imports: use absolute paths from `$lib/`, not relative `../../`

## Architecture decisions
See `docs/decisions/` for ADRs. Read relevant ADRs before modifying:
- 0001-use-tauri-v2.md — Why Tauri 2, not Electron
- 0002-sqlite-over-indexeddb.md — Local-first data strategy
- 0003-xterm-integration.md — Terminal emulation approach

## Git workflow
- Branch naming: feature/*, fix/*, refactor/*
- Run lint + typecheck + tests before committing
- Never amend commits or force push

## Boundaries — do NOT
- Never bypass pre-commit hooks (no --no-verify)
- Never add dependencies without checking for vulnerabilities
- Never hardcode secrets or API keys
- Never modify generated files in src-tauri/bindings/
- Do not use `console.log` — use the structured logger at `$lib/logger.ts`
```

---

## Multi-agent strategy: one source of truth

When a team uses Claude Code, Cursor, and Copilot simultaneously, maintaining separate instruction files creates **instruction drift** — files going out of sync as conventions change. The files themselves don't conflict (each tool reads only its own), but divergent content causes inconsistent AI behavior.

**The recommended approach is AGENTS.md as the canonical source** with tool-specific files referencing it. Three practical strategies work:

**Symlink strategy** (simplest): `ln -s AGENTS.md CLAUDE.md` ensures both files always have identical content. Add `.github/copilot-instructions.md` as another symlink or a one-line pointer: "Follow the rules in ./AGENTS.md".

**Shared base with tool-specific overrides** (most flexible): Write 80% of common rules in AGENTS.md. Use CLAUDE.md only for Claude-specific features (MCP server config, auto-memory preferences, `/` command references). Use `.cursor/rules/` only for Cursor's glob-based activation and MDC frontmatter. Use `.github/copilot-instructions.md` for Copilot-specific review policies.

**Pre-commit sync** (most robust): Maintain a single `docs/ai-rules-base.md` and use a pre-commit hook to copy it into all tool-specific locations. This guarantees synchronization and works even when symlinks cause issues on Windows.

---

## Repository structure that helps agents navigate

**Monorepos significantly outperform polyrepos for AI agents.** When the full codebase is visible, agents can make atomic cross-service changes, read real implementations instead of relying on documentation, and recognize consistent patterns. Airbnb reportedly compressed an 18-month migration to 6 weeks using agents in a monorepo. For polyrepo setups, tools like Nx Polygraph can create a synthetic dependency graph across repos.

Key structural principles that improve agent performance:

**Use descriptive, semantic file names** that signal purpose. `UserAuthService.ts` beats `service1.ts`. Specify exact file locations in your instruction file so agents don't waste tokens exploring: "For charts, copy `app/components/Charts/Bar.tsx`."

**Document your directory structure explicitly** in the instruction file with purpose annotations. AI agents cannot intuit organizational intent from folder names alone. OpenAI's own Codex repository contains **88 AGENTS.md files** across different directories, each scoped to its subdirectory context — a pattern worth emulating in large codebases.

**Use progressive disclosure** for documentation. Keep the root instruction file concise and link to detailed docs: AGENTS.md references `docs/ARCHITECTURE.md`, which references `docs/API.md`, which references inline doc comments. This creates a discoverable resource tree without bloating the primary context.

---

## API contracts that prevent hallucination

**Every gap in documentation is an invitation for the AI to hallucinate.** If your docs describe an endpoint but don't document rate limits, the AI will invent plausible-sounding limits. The most effective defense is **contract-first development** with machine-readable schemas as the single source of truth.

**OpenAPI specifications** should live at the repository root (or a well-known path like `api/openapi.yaml`). Define the contract first, generate server stubs and client libraries, validate implementations against the spec in CI, and use Prism for mock servers during development. AI agents can read OpenAPI YAML directly and will produce accurate API calls when a spec exists.

**TypeScript strict mode is the single most impactful typing investment.** Anders Hejlsberg noted: "If you ask AI to translate half a million lines of code, it might hallucinate. But if types check the result, you get a reliable outcome." TypeScript overtook JavaScript and Python as the most-used language on GitHub in 2025, largely because of its synergy with AI tooling. Enable `strict: true` in `tsconfig.json`, define explicit interfaces for all API boundaries, and use **Zod schemas** for runtime validation alongside compile-time types. For Tauri specifically, the `tauri-specta` crate can generate TypeScript bindings from Rust command signatures, eliminating an entire class of hallucination.

**Rust's trait definitions and doc comments** serve as strong contracts. Use `///` doc comments on all public APIs with `@param`, `@returns`, and `@example` sections. The compiler's ownership model and strict type system catch hallucinated implementations before runtime. For IPC between Tauri's Rust backend and Svelte frontend, maintain a shared types directory that both sides reference.

**Write doc comments as if explaining to a new developer who has never seen your codebase.** Include parameter types, return types, edge cases, error conditions, and realistic examples. One real code snippet beats three paragraphs of description. An IBM study found that well-structured documentation improved AI response accuracy by up to **47%**.

---

## Architecture Decision Records keep agents aligned

ADRs are natural-language markdown files — ideal for LLM consumption. They provide structured context explaining **why** architectural decisions were made, preventing AI agents from unknowingly contradicting past choices. As Chris Swan wrote: "It's such an obviously good way to provide context to a coding assistant — enough structure to ensure key points are addressed, but in natural language, which is perfect for LLMs."

### Two practical ADR formats

**Michael Nygard's original format** (2011) is the simplest and most widely adopted. Each ADR has four sections — Status, Context, Decision, Consequences — and should fit on 1-2 pages. Store them in `docs/decisions/` with sequential numbering (`0001-use-postgresql.md`). ADRs are immutable: if a decision changes, create a new ADR that supersedes the old one.

```markdown
# ADR 0003: Use SQLite via Tauri's SQL plugin

## Status
Accepted

## Context
We need local-first data persistence for the terminal application.
IndexedDB is browser-only and inaccessible from Rust. A full
PostgreSQL server adds deployment complexity inappropriate for
a desktop app.

## Decision
We will use SQLite via Tauri's official SQL plugin, with migrations
managed in src-tauri/migrations/.

## Consequences
- Positive: Zero-config persistence, ACID compliance, single-file database
- Positive: Direct access from both Rust commands and frontend via plugin
- Negative: No concurrent write access from multiple processes
- Negative: Schema migrations require careful handling during auto-updates
```

**MADR 4.0.0** (Markdown Architectural Decision Records) extends Nygard's format with explicit options analysis, decision drivers, and pros/cons for each considered option. Use MADR when decisions involve evaluating multiple alternatives and you want to document the reasoning for future reference. The minimal MADR template requires only Title, Context/Problem Statement, Considered Options, and Decision Outcome.

### The ADR-agent integration pattern

The most effective pattern, from Elliot Nelson's "Claude ADR Pattern," keeps your instruction file small while providing deep architectural context on demand:

1. Create a `docs/decisions/` directory with numbered ADRs
2. List ADR titles in CLAUDE.md/AGENTS.md with one-line summaries
3. Instruct the agent: "Before significant modifications, review relevant ADRs from the list below"
4. Create a custom slash command (`.claude/commands/adr.md`) that forces ADR review before task planning

This approach dropped one practitioner's CLAUDE.md from a bloated file to under 2KB while actually improving Claude's architectural awareness. The agent selectively loads only the ADRs relevant to the current task.

**Tooling**: `adr-tools` (Bash CLI) handles creation, numbering, and cross-linking. **Log4brains** generates a searchable static site from your ADRs. **Archgate** (new, March 2026) turns ADRs into executable rules with companion `.rules.ts` files, validated in CI and accessible to AI agents via MCP.

---

## Writing effective negative constraints

Research on thousands of AI instruction files reveals that **negative constraints are among the highest-value instructions** — but only when written correctly. An ETH Zurich study confirmed that rules the agent can already infer from configuration files actually **degrade** performance by consuming context tokens. Every constraint should earn its place by solving a real, observed problem.

**Provide context for prohibitions.** Anthropic's own best practices show that explaining *why* dramatically improves compliance. "NEVER use ellipses" is less effective than "Never use ellipses — our text-to-speech engine cannot pronounce them." This gives the model reasoning to anchor on, not just a rule to follow.

**Pair every negative with a positive alternative.** "Do NOT use `any` type" is weaker than "Do NOT use `any` type — use explicit types or generics instead." "Never import from internal modules directly — use the public API from `lib/index.ts`" gives the agent a concrete path forward.

**Use the three-tier permission model** for safety-critical boundaries. This pattern from Addy Osmani and others clearly separates what agents can do freely, what requires confirmation, and what is absolutely prohibited:

```markdown
## Safety and permissions
Allowed without asking: Read files, list files, run tsc/prettier/eslint on single files
Ask first: Installing packages, deleting files, git push
Never: Modifying production configs, committing secrets, running git commit --no-verify
```

**Enforce via tooling wherever possible.** If a constraint can be a linter rule, a config flag, or a CI check, it should not be an AI instruction. "Don't use `var`" belongs in ESLint's `no-var` rule. "Use strict mode" belongs in `tsconfig.json`. Reserve instruction file space for constraints that are genuinely non-inferable — team preferences, architectural boundaries, and workflow rules that no tooling can enforce.

**Be aware of the instruction ceiling.** Research suggests frontier LLMs can follow approximately **150-200 instructions** with reasonable consistency. Overloading with hundreds of rules (including negatives) causes the model to follow none well. Curate ruthlessly.

---

## CONTRIBUTING.md and PR templates for AI workflows

GitHub's official documentation now recommends using CONTRIBUTING.md to "document your expectations for AI-generated source code and content." The key additions for AI-assisted workflows focus on disclosure, accountability, and quality standards.

**AI disclosure in PR templates** has become standard practice. MicroPython's February 2026 template requires every contributor to select whether AI tools were used and whether generated code was reviewed by a human. A practical PR template for AI-assisted projects should include:

```markdown
## AI Disclosure
- [ ] No AI tools were used in creating this PR
- [ ] AI tools were used (specify: _____________)
  - [ ] All generated code reviewed and understood by submitter
  - [ ] Generated code tested locally

## Checklist
- [ ] Code compiles without errors (`npm run typecheck`)
- [ ] All tests pass (`npm run test`)
- [ ] Lint passes with zero warnings (`npm run lint`)
- [ ] No hallucinated APIs or non-existent packages
- [ ] Documentation updated if project structure changed
- [ ] Relevant ADRs consulted for architectural changes
```

**Code review guidelines for AI-generated code** should address AI-specific risks. Research shows AI-authored PRs contain **1.7× more issues** and are **2.74× more prone to XSS vulnerabilities** than human-written code. Reviewers should verify that unfamiliar method names exist in official documentation, check for "slopsquatting" (malicious packages resembling legitimate ones that AI tools suggest), and confirm all dependencies are real and actively maintained.

**Automated review tools** complement human review effectively. CodeRabbit (configurable via `.coderabbit.yaml`) provides codebase-aware AI review with path-specific instructions. GitHub Copilot's own code review feature can be added to `CODEOWNERS` as a default reviewer. PR-Agent (Qodo) offers a self-hosted option for sensitive codebases.

---

## CI/CD pipelines as the authoritative safety net

AI agents routinely forget to run manual quality checks and will sometimes disable linting rules or lower thresholds to make code pass. **Server-side CI is the authoritative enforcement layer that cannot be bypassed** — it must be strict enough to catch every category of AI error.

### The multi-layer defense model

Quality enforcement operates at four layers, each catching what the previous layer missed:

1. **Agent-level hooks** (Claude Code's PreToolUse/PostToolUse) intercept commands before they execute
2. **Pre-commit hooks** (Husky + lint-staged) enforce formatting, linting, and type-checking locally in under 5 seconds
3. **Pre-push hooks** run the full test suite before code leaves the developer's machine
4. **CI pipeline** (GitHub Actions) provides authoritative, unskippable enforcement of all quality gates

### Essential CI workflow for AI-generated code

```yaml
name: CI Quality Gates
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npx eslint "**/*.{ts,tsx}" --max-warnings=0
      - run: npx prettier --check .
      - run: npx tsc --noEmit

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npx vitest run --coverage
      - uses: codecov/codecov-action@v4

  rust:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo clippy -- -D warnings
      - run: cargo test
      - run: cargo fmt --check

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=moderate
      - uses: semgrep/semgrep-action@v1
        with: { config: 'p/default' }
      - uses: gitleaks/gitleaks-action@v2
```

**Test-Driven Development is the single most effective practice for AI-generated code quality.** Without TDD, agents write implementation code that appears correct and then generate tests that verify what they just wrote — passing by construction but not verifying requirements. The TDD workflow forces agents to write a failing test first, confirm it fails, then write minimum implementation to make it pass. Multiple engineering teams report this produces dramatically better results. Always allow agents to run build and test commands autonomously so they can iterate on automated feedback.

**Explicitly prohibit agent "cheating" in your instruction file.** Include: "Never run `git commit --no-verify` or set `HUSKY=0`. Never disable or weaken ESLint rules to make code pass. Never lower coverage thresholds without human approval. When blocked by CI or hooks, stop and report the failing command and error output."

---

## Real-world examples worth studying

Several open-source repositories demonstrate excellent AI instruction file practices:

**Trail of Bits' `claude-code-config`** (701 stars) is the gold standard for CLAUDE.md configuration. It provides a comprehensive global template covering development philosophy ("No speculative features," "No premature abstraction," "Replace don't deprecate"), hard limits on function length and complexity, language-specific toolchains for Python/TypeScript/Rust/Bash, and a hooks system that blocks dangerous operations like `rm -rf` or pushes to main. The two-tier hierarchy (global `~/.claude/CLAUDE.md` + project-level overrides) is particularly well-designed.

**OpenAI's Codex repository** (66.1k stars) contains a 195-line AGENTS.md with module-specific instructions organized by crate. It includes concrete coding conventions (inline format args, argument comment lint using `/*param_name*/` before opaque literals), sandbox-aware boundaries ("Never modify code related to `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR`"), and module-specific testing patterns. The repository contains **88 AGENTS.md files** across different directories.

**Compiler Explorer** (18.6k stars) has a 137-line CLAUDE.md that is remarkably practical: build commands, a strict 5-step commit process, and emphatic guardrails ("⚠️ NEVER BYPASS PRE-COMMIT HOOKS!", "⚠️ NEVER amend commits or force push"). Style rules are specific (4-space indent, 120 char width, single quotes) rather than vague.

**GitHub's `awesome-copilot`** is the official collection of Copilot customizations, defining the canonical structure for instructions (`*.instructions.md` with `applyTo` frontmatter), prompts (`*.prompt.md`), agents (`*.agent.md`), and skills (directories with `SKILL.md`). It covers C++, Dart/Flutter, .NET, GitHub Actions, and secure coding.

**PatrickJS/awesome-cursorrules** remains the largest curated collection of Cursor rules, organized by technology stack with dozens of ready-to-use configurations for React, Python, Unity, and more.

The collection **josix/awesome-claude-md** catalogs 104+ exemplary CLAUDE.md files with a 60+ point quality criteria, categorized by technology and purpose — an excellent starting point for finding patterns relevant to your specific stack.

---

## Conclusion

The most impactful actions for AI-agent-friendly repositories follow a clear priority order. **Start with an AGENTS.md file** (or CLAUDE.md if Claude Code is your primary tool) covering build commands, project structure, coding conventions, and explicit boundaries — keep it under 150 lines and symlink it for cross-tool compatibility. **Enable TypeScript strict mode and Rust clippy with `-D warnings`** — strong type systems prevent more hallucinations than any instruction file can. **Adopt contract-first API design** with OpenAPI specs or auto-generated TypeScript bindings for your Tauri IPC boundary. **Implement ADRs** in `docs/decisions/` and reference them from your instruction file so agents check architectural context before making changes. **Make CI strict and unskippable** with `--max-warnings=0`, coverage thresholds, and security scanning — then tell agents they cannot bypass these gates. **Use TDD** as your default workflow with AI agents, since it is the single practice with the strongest evidence of improving AI-generated code quality. Finally, curate your negative constraints ruthlessly: every rule must earn its place by solving a real observed problem, and anything enforceable by tooling should be a linter rule rather than an AI instruction.
