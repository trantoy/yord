# Building an IDE from scratch with Tauri, Svelte, and Vite

**The single most important thing to understand about IDE architecture is that every failed IDE project died from the same disease: wrong boundaries.** Xi Editor split the editor core from the frontend across processes and spent months implementing scrolling. Atom gave extensions full access to everything and died of a thousand performance cuts. Eclipse over-engineered its plugin module system into dependency hell. VS Code succeeded because it drew exactly one critical boundary — isolating extensions from the UI — and kept everything else in-process. Your Tauri IDE must learn from these corpses. This guide covers every architectural decision you'll face, in the order you'll face them, with specific opinions grounded in what actually shipped and what actually failed.

The Tauri + Svelte + Vite stack gives you a unique position: a Rust backend for performance-critical work (file I/O, process management, parsing) with a web frontend for rapid UI iteration. This is architecturally similar to Lapce's proxy model, but with a browser-grade rendering engine instead of a custom GPU toolkit. The critical challenge is deciding what crosses the IPC boundary and what doesn't.

---

## How VS Code, Zed, and Lapce actually work inside

Understanding the three dominant architectures gives you a map of what's possible and what tradeoffs each approach makes.

**VS Code runs five cooperating OS processes.** The main Electron process (`src/vs/code/electron-main/app.ts`) handles window lifecycle and OS integration. Each window gets a Chromium renderer process that draws the entire UI via the Monaco editor and the `Workbench` class. Extensions run in a dedicated Node.js Extension Host process, communicating with the renderer via a typed JSON-RPC proxy system defined in `extHost.protocol.ts`. Method names follow a `$` prefix convention (`$registerHoverProvider`, `$provideHover`). A hidden Shared Process handles background tasks like extension gallery queries, and a separate Pty Host process owns all terminal shells so a crashing shell can't take down the editor. The codebase enforces strict layering via a build-time checker at `build/checker/layersChecker.ts` — lower layers cannot import from higher ones. This architecture enabled VS Code for the Web: strip the main process, run extension hosts as Web Workers, and everything still works.

**Zed takes the opposite approach — everything in one Rust process.** Built by the original Atom team (Nathan Sobo et al.), Zed uses GPUI, a custom hybrid immediate/retained-mode GPU framework. Rendering uses signed distance functions for primitives with platform-native text shaping (CoreText on macOS, DirectWrite on Windows). The entity system replaces traditional OOP: `Entity<T>` is a reference-counted handle in GPUI's registry, state changes go through `cx.update()`, and reactivity flows through `cx.notify()` and `cx.observe()`. Actions are Rust structs with `#[derive(Action)]` that get dispatched up the focus tree. Zed doesn't use tokio — on macOS it wraps Grand Central Dispatch directly, with `foreground_executor` for main-thread work and `background_executor` for thread pool dispatch. The monorepo contains **~220 crates**, with key ones being `gpui` (framework), `editor` (text editing), `workspace` (pane management), `project` (file/language services), and `language` (syntax/LSP).

**Lapce introduces the proxy pattern most relevant to your Tauri IDE.** Written in pure Rust with Floem (a reactive native UI framework evolved from Xi Editor's Druid), Lapce separates the frontend from all I/O through `lapce-proxy`. The proxy handles file reads/writes, LSP communication, and plugin interaction. The frontend stores file content locally but delegates all mutations through the proxy. This architecture directly enables remote development — run `lapce-proxy` on a remote host and connect from a local frontend with no code changes. Plugins compile to **WASI-compatible WebAssembly**, providing sandboxed extensibility.

The common thread across all three: **the editor core lives in the same process as the UI.** Every project that tried to split these across a process boundary (Xi, xray) failed to ship.

---

## Architectural decisions that have killed real IDE projects

The Xi Editor post-mortem by Raph Levien (June 2020) is the single most valuable document in IDE architecture history. Xi attempted a Rust core communicating with platform-native frontends via JSON-RPC over stdio. The process separation between frontend and core was, in Levien's words, "not a good idea."

**The async IPC split caused cascading failures.** Live window resizing triggered continuous word-wrap recalculation. With async communication between frontend and core, tearing artifacts were inevitable and race conditions between edits and wraps were unsolvable. Scrolling took months to implement because fetching visible text outside the frontend's cache required sophisticated cache coherence protocols. Levien's conclusion: "If we just had the text available as an in-process data structure for the UI to query, it would have been quite straightforward." The CRDT approach (designed for Android's concurrent input method) added massive complexity without benefit for a single-user editor. And the modular architecture paradoxically killed community contribution — everyone was blocked on architectural rework, and features like Vi keybindings were never merged because they required changes across module boundaries.

**Atom died from the "just let everything access everything" anti-pattern.** Extensions ran in the same Chromium renderer with full DOM access and no isolation. Any extension could monkey-patch private methods, block the UI thread, or break other extensions silently. VS Code's Extension Host solved this entirely — a misbehaving extension cannot crash the editor. Atom's "everything is a package" philosophy meant ~130 bundled packages each adding to startup time, with no lazy activation. When Microsoft acquired GitHub, maintaining two Electron editors was redundant, and the one with better architecture won.

**Light Table coupled too tightly to a niche technology.** Written in ClojureScript, it got stuck on a pre-1.x version that couldn't be upgraded without a monolithic coordinated branch across the entire codebase. Its novel BOT (Behavior-Object-Tag) reactive dispatch system was unfamiliar to most developers, creating a steep learning curve that strangled contributions. Eclipse's OSGi plugin system provides the complementary lesson: over-engineering modularity creates accidental complexity that overwhelms the benefits. Dependency resolution across hundreds of bundles with version ranges and classloader isolation produced `NoClassDefFoundError` at runtime that never appeared at compile time.

**The pattern is clear.** Don't split editor core from UI across processes. Don't give extensions unrestricted access. Don't couple to niche technologies. Don't over-engineer the plugin boundary. Do isolate extensions. Do keep the editor core in-process with the UI. Do use proven text buffer implementations.

---

## CodeMirror 6 is the only serious choice for your editor core

For a Tauri + Svelte IDE, there are three editor core options. CodeMirror 6 is the right one. Here's why, and why the others aren't.

**CodeMirror 6** follows a functional, Redux-inspired architecture where `EditorState` is strictly immutable. Changes flow unidirectionally: user input → transaction → new state → view update. The extension system is built on **facets** (typed, composable slots that multiple extensions contribute to), **StateFields** (custom state that updates as pure functions of transactions), and **StateEffects** (typed values for inter-extension communication). This architecture was designed specifically for IDE-scale extensibility over ~13 years of iteration by Marijn Haverbeke. It uses viewport virtualization (only visible lines are in the DOM), handles million-line documents, and ships with Lezer, an incremental LR parser for syntax highlighting. Bundle size is modular — **~150-200KB minified** for a basic setup. MIT licensed. The recently released `@codemirror/lsp-client` (by Haverbeke, sponsored by zoo.dev) provides official LSP integration with autocompletion, hover, go-to-definition, and rename support.

**Monaco Editor** is extracted from VS Code's codebase. You get multicursor, minimap, and code folding out of the box, but it's too opinionated for a custom IDE. You cannot apply different themes to separate editor instances (single global theme). The built-in Monarch tokenizer is weaker than tree-sitter or TextMate, and hacking in TextMate grammars requires compiling Oniguruma to WASM and forking Monaco internals. It spawns Web Workers for language services that are hard to customize. Bundle size is **2-4MB+**. Designed as a drop-in widget, not a composable foundation.

**ProseMirror** is designed for rich text editing (Google Docs, not VS Code). It operates on a schema-based document model with block and inline nodes — fundamentally different from flat-text-with-decorations needed for code. Use it only if you need a markdown WYSIWYG panel alongside your code editor.

**Building your own is almost certainly wrong.** VS Code's text buffer journey illustrates the iceberg. They started with an array of lines — a 35MB file with 13.7M lines consumed **600MB** because each `ModelLine` object needed ~40-60 bytes. They switched to a piece table backed by a red-black tree, where each node caches left-subtree length and line-feed count for O(log n) lookups. But the real complexity isn't the data structure. It's selections and multicursor merging, IME (input method editor) handling for CJK languages, bidirectional text with RTL/LTR mixing, accessibility via ARIA live regions, viewport virtualization with scroll anchoring during content changes, and copy-paste with rectangular selection distribution. As Peng Lyu documented: "The hottest method was `getLineContent`. Every time I found that my assumptions about which methods would be hot did not match reality." CodeMirror 6 has solved all of these problems.

**Integration pattern for Svelte:** Create a custom wrapper component rather than using a third-party binding. Mount `EditorView` in `onMount`, override `dispatch` to bridge transactions to your external state system, and use `EditorView.updateListener.of()` to react to state changes. Feed LSP diagnostics via `setDiagnostics()` from `@codemirror/lint`, completions via `autocompletion()` override sources, and hover via `hoverTooltip()` returning promises that resolve from your LSP client.

---

## Tree-sitter belongs in Rust, not in your webview

Tree-sitter provides incremental parsing that produces a full concrete syntax tree in sub-millisecond time after edits. The decision is where to run it in a Tauri architecture.

**Run tree-sitter natively in the Rust backend.** The `tree-sitter` Rust crate wraps the C library with zero overhead. Incremental parsing (`parser.parse(new_source, Some(&old_tree))` after `old_tree.edit(input_edit)`) only re-parses changed regions. The alternative — `web-tree-sitter` compiled to WASM via Emscripten — runs **1.5-3x slower** than native and blocks the JS main thread unless offloaded to a Web Worker. The Rust crate also has a `wasm` feature flag that uses Wasmtime to load grammar `.wasm` files dynamically from native code, giving you both native speed and dynamic grammar loading.

**Use tree-sitter for instant, local intelligence and LSP for semantic depth.** Tree-sitter provides syntactic highlighting with perfect error recovery in under 1ms. LSP semantic tokens (`textDocument/semanticTokens`) require a server round-trip of **10-100ms+** but provide richer information — rust-analyzer can distinguish mutable from immutable variable references, for instance. The optimal strategy is what Zed does: tree-sitter as the base highlighting layer with LSP semantic tokens optionally overlaid on top.

Zed demonstrates the deepest tree-sitter integration in any editor. Beyond highlighting via `highlights.scm` queries, they use `outline.scm` for symbol outlines (deliberately avoiding LSP `documentSymbol` for more control over presentation), `indents.scm` for auto-indentation, `brackets.scm` for matching and rainbow coloring, `runnables.scm` for identifying test functions, and full language injection support for multi-language documents. Neovim similarly has built-in tree-sitter with its C library linked directly into the editor, powering highlighting, folding via `vim.treesitter.foldexpr()`, and structural text objects.

**The IPC cost is negligible.** Sending highlight token ranges (offset pairs + type identifiers) over Tauri IPC takes ~0.1-1ms. Run parsing on a dedicated Rust thread to keep both the main thread and UI thread free. Store syntax trees in the Rust backend where they can be shared with other systems (code analysis, symbol indexing, fold computation).

---

## Implementing an LSP client from the Rust backend

LSP uses JSON-RPC 2.0 over stdio with `Content-Length` header framing. There are three message types: requests (with `id`, expecting response), responses (paired by `id`), and notifications (fire-and-forget). The lifecycle is strict: `initialize` request → `initialized` notification → normal operation → `shutdown` request → `exit` notification.

**There is no production-ready LSP client crate in the Rust ecosystem.** The `tower-lsp` crate is server-only. The `lsp-types` crate (0.97+) provides all type definitions but no transport. Both Zed and Lapce implement their own LSP clients from scratch. You must do the same.

**Model it after Zed's `LanguageServer` struct** (in `crates/lsp/src/lsp.rs`). The pattern is: spawn the language server process with `std::process::Command` capturing stdin/stdout. A background tokio task continuously reads stdout, parses `Content-Length` headers, and deserializes JSON-RPC messages. Responses (messages with `id`) are routed to waiting `tokio::sync::oneshot` channels stored in a `HashMap<RequestId, oneshot::Sender<Result>>`. Notifications (no `id`) are dispatched by method name to registered handlers. Writing to the server goes through `tokio::sync::mpsc` to serialize access to stdin.

**The Tauri integration pattern maps naturally to LSP's model.** Use Tauri commands (request-response) for LSP features the user triggers: completion, hover, go-to-definition, rename. Use Tauri events (fire-and-forget, Rust → frontend) for server-pushed data: diagnostics via `textDocument/publishDiagnostics`, progress via `$/progress`. A single `LspManager` in Tauri managed state holds a `HashMap<LanguageServerId, LanguageServer>` since different file types need different servers. Zed uses a **50ms debounce** for most LSP requests and **250ms for code actions** — adopt similar thresholds.

**Incremental document sync is essential.** Track document versions in Rust with monotonically increasing integers. When the frontend reports an edit, Rust updates its internal buffer and forwards `textDocument/didChange` with only the changed range (`TextDocumentSyncKind::Incremental`) to the server. Full sync wastes bandwidth on large files. The key crates are `lsp-types` for all LSP structs, `serde_json` for serialization, `tokio` for async I/O, and `tokio::sync::{oneshot, mpsc}` for the request routing machinery.

---

## The Tauri process boundary: Rust owns all I/O, Svelte owns all pixels

The fundamental rule: **if it touches the operating system, it lives in Rust. If it touches the screen, it lives in Svelte.** This isn't a suggestion — Tauri's security model enforces it. The webview has zero direct filesystem access; all FS operations go through Rust commands with capability-based permission scopes.

**Rust backend responsibilities:** File reading/writing (via VFS trait), file watching (`notify` crate), directory traversal (`walkdir` + `ignore`), process spawning (language servers via `std::process::Command`, terminal shells via `portable-pty`), LSP client (JSON-RPC framing, request routing, document sync), tree-sitter parsing (native, on background threads), git operations (`git2` for reads, CLI for writes), search (`ripgrep` integration), and workspace state management.

**Svelte frontend responsibilities:** CodeMirror 6 rendering and input handling, completion/hover/diagnostic UI rendering, file tree component, terminal display via xterm.js, panel layout and splitting, theme application, keybinding capture and command dispatch, and all visual state (scroll positions, selections, panel sizes).

**Tauri v2 IPC is fast enough for IDE workloads.** Commands use a custom protocol (HTTP-like fetch that doesn't hit the network) plus `postMessage()`. A 150MB file transfer takes under 60ms in Tauri v2 versus ~50 seconds previously. For typical LSP responses and file contents, overhead is sub-millisecond. For high-frequency streaming (terminal output), use Tauri events with batching — collect output for **16ms frames** before emitting to avoid overwhelming the webview. Use the `taurpc` crate with `specta` for fully typed IPC that auto-generates TypeScript types from Rust trait definitions.

---

## Plugin architecture: WASM sandboxing or regret

There are four approaches to plugin systems. Only two are viable, and only one is recommended as the primary system.

**WASM-based plugins via Extism or Wasmtime in the Rust backend** is the recommended primary approach. Extism provides plugin lifecycle management, host function injection, fuel-based execution limits, and WASI support for sandboxed filesystem access. Plugins compile to `wasm32-wasip1` from any supported language (Rust, Go, JS, C#, Zig). Host functions enable a **capability-based permission model** — plugins declare what they need (`filesystem:read`, `network:http`, `editor:decorations`), and the host grants only what's declared. This is exactly what Zed does, using Wasmtime with WIT (WebAssembly Interface Types) for rich type sharing across the WASM boundary.

**WebWorker-based UI plugins** provide a secondary path for frontend extensions — custom panels, themes, editor decorations. Workers can't block the UI thread and communicate via `postMessage`. This is suitable for lightweight visual plugins but not for heavy backend operations.

**IPC-based separate processes** (the VS Code Extension Host model) provide maximum isolation but maximum complexity. This is overkill unless you're building VS Code's scale of ecosystem. **"Just import JS modules"** is the Atom anti-pattern — no isolation, no versioning, no security boundary, no resource limits. It feels powerful initially and kills the platform within two years.

**The critical early decision is defining extension points before you have plugins.** Your own built-in features should register through the same systems that future plugins will use. When you build the file tree, use `registerTreeProvider()`. When you add a language, use `registerLanguage()`. This guarantees the plugin API works because you're its first consumer. Define a plugin manifest (`plugin.toml`) with metadata, version, required capabilities, and activation events from day one, even if the plugin loader doesn't exist yet.

---

## State management at IDE scale demands command-first architecture

Naive Svelte stores (`$store` pattern) will collapse under IDE-scale state for specific, predictable reasons. When hundreds of components subscribe to stores tracking thousands of entities, every store update triggers all subscribers. Derived stores don't batch updates across multiple dependency changes — changing two dependencies triggers the mapper callback after the first change. There's no transactional update mechanism (renaming a file needs atomic updates to file tree, open editors, tabs, and git status). There's no built-in undo/redo, no serialization story for session persistence, and the store-per-entity anti-pattern (one writable store per open file, per diagnostic) creates thousands of stores with multiplicative subscription overhead.

**The solution is a command-first architecture with the Rust backend as source of truth.** All state mutations flow through a central command registry: user action → command object → Rust backend processes → emits state change events → frontend stores update as reactive views. This gives you natural undo/redo (move a history index backward/forward), audit trails, time-travel debugging, and a guaranteed interception point for future plugins.

**Partition state by domain and location.** File contents, LSP state, git state, workspace model, and command history live in Rust — they're large, performance-critical, and need native I/O. UI layout, scroll positions, selections, panel visibility, and theme live in Svelte. Open file lists and diagnostics are shared via Tauri events. In Svelte, use collection stores (`writable(Map<FileId, FileState>)`) with derived stores for filtered views rather than one store per entity. Use Svelte's context API for state scoped to component subtrees (per-editor-panel state).

**Design for CRDTs now, implement them later.** Separate your document model from your view model. Express text editing as operations (insert at position, delete range) rather than full-document replacement — this maps directly to CRDT operations. Use an abstract buffer interface in Rust (`ropey` crate today, swappable for `yrs` or `automerge` tomorrow). The `yrs` crate is the Rust port of Yjs; `automerge` is Rust-native. Both can run in your Tauri backend with frontend sync via IPC.

---

## File system abstraction and terminal integration

**The VFS trait is an irreversible decision — get it right in week one.** Every subsystem (editor, search, git, tree-sitter) touches files. If you hardcode `std::fs` calls, you can never support remote files, in-memory buffers, or archives. Define a Rust trait with `read()`, `write()`, `stat()`, `read_dir()`, and `watch()` methods, all addressed by URI with scheme (`file://`, `memory://`, `git://`). Implement `LocalFileSystem` first. VS Code's `FileSystemProvider` interface is the reference design.

**File watching uses the `notify` crate** (v7.x), which powers Zed, rust-analyzer, cargo-watch, and Deno. It uses inotify on Linux, FSEvents on macOS, and ReadDirectoryChangesW on Windows. **Debouncing is mandatory** — a single file save generates 3-5 raw events. Use `notify-debouncer-full` which tracks file renames via inode/volume IDs. Filter events through the `ignore` crate (by the ripgrep author) to skip `node_modules/`, `.git/objects/`, and `target/`. Always have a polling fallback for network-mounted filesystems that don't support native watchers.

**For large files, use memory-mapped I/O** via `memmap2` — the OS handles paging, and only accessed pages load into physical RAM. Combine with viewport-based loading (render only visible lines) and sparse line indexing (checkpoint every N lines for files >10MB). For search, bundle ripgrep or use the `grep-searcher`/`grep-regex` crates from the ripgrep workspace. VS Code has shipped ripgrep as its search backend since 2017 — writing a competitive search engine from scratch is months of work for an inferior result.

**Terminal integration follows the bridge pattern.** The Rust backend spawns a PTY via the `portable-pty` crate (from WezTerm, **4.6M downloads**, supports POSIX PTY on Unix and ConPTY on Windows 10+). A background tokio task reads PTY stdout and emits batched output to the frontend via Tauri events at ~60Hz. The Svelte frontend renders via xterm.js with the WebGL addon for **10-100x faster rendering** than the default canvas renderer. User keystrokes go back to Rust via `invoke("pty_write", data)`, which writes to the PTY's stdin. Resize events flow similarly. The `tauri-plugin-pty` crate provides a simpler API for prototyping, but `portable-pty` directly gives maximum control.

---

## What to learn from each open-source IDE

**Zed** (Rust/GPUI) is the reference for deep tree-sitter integration, action dispatch design, and WASM-based extensions. Study its `crates/lsp/src/lsp.rs` for the gold-standard Rust LSP client pattern. Study `highlights.scm`, `outline.scm`, `indents.scm`, and `runnables.scm` for tree-sitter query design. Its entity system (`Entity<T>` with `cx.observe()` reactivity) demonstrates how to manage complex state without a traditional reactive framework.

**VS Code** (TypeScript/Electron) teaches extension isolation (the Extension Host architecture), API governance (proposed API process with debt-week reviews), the piece-tree text buffer, and the command/keybinding pipeline. Its source organization with enforced layering (`build/checker/layersChecker.ts`) shows how to keep a million-line codebase maintainable. The `extHost.protocol.ts` proxy system is the template for any cross-process typed RPC.

**Lapce** (Rust/Floem) demonstrates the proxy architecture most directly applicable to your Tauri IDE. Its separation of `lapce-app` (frontend) and `lapce-proxy` (I/O) maps almost exactly to your Tauri frontend/backend split. Its WASI-based plugin system shows practical WASM extensibility. Study how `floem-editor-core` separates editor logic from rendering.

**Xi Editor** (archived) is a cautionary tale, but its surviving artifacts are valuable: the `xi-rope` crate has excellent worst-case performance, and the `syntect` library (TextMate-compatible syntax highlighting) remains the fastest open-source implementation. Read Raph Levien's full retrospective for the deepest analysis of what goes wrong when you over-architect an editor.

**Neovim** provides the template for tree-sitter integration in a mature editor. Its built-in `vim.treesitter` API with `InspectTree` and `EditQuery` commands for live tree/query inspection is the debugging workflow you'll want. The `nvim-treesitter-textobjects` plugin demonstrates structural editing that works uniformly across all languages.

**Helix** (Rust, terminal-based) shows a minimal but effective architecture with built-in tree-sitter and LSP. Its selection-first editing model and tight integration of tree-sitter text objects demonstrate how far you can push structural editing without plugins.

---

## Build this in four phases, irreversible decisions first

Five architectural decisions are irreversible and must be right before you write feature code. **The IPC contract** (Tauri command/event API shapes) — once the frontend depends on specific signatures, changing them cascades everywhere; use `taurpc` for typed contracts from day one. **The VFS trait** — every subsystem touches files through this interface. **The command/action system** — all user actions route through a central registry; this is the backbone for keybindings, command palette, macros, and plugins. **The document model** — how text buffers work internally (use `ropey` for rope-based buffers). **The panel/layout system** — the docking/splitting model determines how editor tabs, terminals, sidebars, and panels compose.

**Phase 1 (weeks 1-8): Minimal viable foundation.** Tauri v2 + Svelte 5 + Vite app shell with resizable panels. Type-safe IPC via `taurpc`. VFS trait with `LocalFileSystem`. File tree via `walkdir` + `ignore` with lazy loading. CodeMirror 6 editor with custom Svelte wrapper. `ropey`-based document buffers in Rust. File watching with `notify` + `notify-debouncer-full`. Command registry with command palette (fuzzy search via `nucleo` or `fuzzy-matcher`). Data-driven keybinding system with user overrides. Settings hierarchy (default → user → workspace) via TOML/JSON. **Goal: open, edit, and save a file with proper architecture.**

**Phase 2 (weeks 9-16): Intelligence layer.** Tree-sitter integration for syntax highlighting (native Rust, dedicated background thread). LSP client following Zed's pattern — start with one language (rust-analyzer). Autocomplete, diagnostics, and go-to-definition wired through the IPC bridge. Terminal via `portable-pty` + xterm.js + WebGL addon. Workspace search via bundled ripgrep. Basic git integration (`git2` for status, branch display, inline modification markers). **Goal: the editor becomes useful for real coding work.**

**Phase 3 (weeks 17-28): Power features.** WASM plugin system via Extism or Wasmtime with capability-based permissions and plugin manifest format. Advanced git (inline blame, diff viewer, staging hunks). Multi-terminal with splits. DAP (Debug Adapter Protocol) integration. Large file handling with `memmap2` + sparse indexing. Document outline and breadcrumbs from tree-sitter. **Goal: extensibility and professional-grade tooling.**

**Phase 4 (weeks 29+): Scale and polish.** CRDT-based collaboration via `yrs` or `automerge`. Remote development (VFS provider over SSH). Extension marketplace. AI integration (inline completions, chat panel). Auto-update via Tauri's updater plugin. Performance profiling and startup optimization. **Goal: production-ready for a broad user base.**

Everything else — specific UI styling, which LSP servers to support, theme details, search result presentation, keybinding defaults — can be refactored freely at any point. The five irreversible decisions and the phase ordering are what protect you from the failure modes that killed Xi, Atom, and Light Table.

---

## Conclusion

The path to a viable IDE is narrower than it appears. **Use CodeMirror 6** — don't build an editor core, don't use Monaco's opinionated internals. **Run tree-sitter natively in Rust** with LSP semantic tokens as an overlay. **Implement your own LSP client** following Zed's `LanguageServer` pattern — no off-the-shelf Rust crate exists. **Keep all I/O in Rust** and make the Svelte frontend a thin reactive view layer. **Use WASM for plugins** via Extism or Wasmtime, and eat your own dog food by building features through the same extension points plugins will use. **Design the command system, VFS trait, and IPC contracts in week one** — these are the decisions you cannot undo.

The counterintuitive lesson from every failed IDE project is that monolithic architectures, where the editor core and UI share a process, are both simpler and more successful than elegant process-separated designs. Xi's beautiful async architecture couldn't implement scrolling; VS Code's pragmatic "extensions in a separate process, everything else in-process" ships to millions. Zed's "everything in one Rust process" delivers sub-16ms latency. Complexity at the wrong boundary is the universal IDE killer. Put the boundary where VS Code proved it belongs: between your core editor and third-party extensions, and nowhere else.
