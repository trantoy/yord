# helix

> Post-modern TUI text editor with built-in LSP and tree-sitter. MPL-2.0 (inspiration only). [GitHub](https://github.com/helix-editor/helix)

## Overview

TUI-based modal editor. 7 crates: helix-core (rope, selection, syntax), helix-view (editor state, document mgmt, view tree), helix-term (TUI, app loop, compositor), helix-event (async events with debouncing), helix-lsp, helix-dap, helix-tui.

Relevant to Yord: SlotMap-backed split tree (cleanest layout tree implementation). Separate DocumentId/ViewId (multiple views → same document, confirms Yord's entity/layout-leaf separation). Biased tokio::select! priority ordering in event loop.

## Architecture Decisions

**SlotMap for the view tree (not Vec, HashMap, or arena)**
`ViewId` uses `slotmap::new_key_type!`. The tree stores `nodes: SlotMap<ViewId, Node>` (`tree.rs:14`). SlotMap gives O(1) insert/remove/lookup with stable generational keys — a `ViewId` remains valid after other nodes are removed (generation catches stale references). Unlike HashMap, iteration is dense and cache-friendly. Unlike typed-arena, removals are first-class and free slots are reused. The generational key prevents use-after-free: generation mismatch causes a clean panic instead of silent corruption.
- *For Yord*: SlotMap is the right choice for any tree with dynamic insert/remove and IDs referenced from multiple places.

**Explicit parent pointers in nodes**
Each `Node` stores `parent: ViewId` (`tree.rs:22`). The root's parent is itself (`tree.rs:94`). Enables O(1) parent lookup during split, remove, and directional navigation. `find_split_in_direction` walks up via `parent` to find the right container. The self-referencing root sentinel eliminates `Option<ViewId>` and simplifies base cases: `if parent == id { return None; }`.
- *For Yord*: use parent pointers. The self-referencing root sentinel is clean and avoids Option overhead.

**Separate Document and View identity**
`DocumentId` is a manually-incremented `NonZeroUsize` (enables `Option<DocumentId>` niche optimization). `ViewId` is a SlotMap key. Documents live in `BTreeMap<DocumentId, Document>` on the Editor; Views live in the Tree's SlotMap. A `View` holds `doc: DocumentId`. Multiple views can display the same document (splits), each with its own cursor position, scroll offset, jump list, and access history. The document stores per-view selections in a HashMap keyed by ViewId.
- *For Yord*: maps directly to the ECS model. Entity = node ID, Document data = component, View/layout data = component.

**Biased `tokio::select!` in the event loop**
Both `event_loop_until_idle` (`application.rs:325`) and `wait_event` (`editor.rs:2276`) use `biased` select. Application-level priority: Signals > Input > Callbacks > Status > Futures > Editor events. Editor-level priority: Saves > Config > LSP > Debug > Redraw request > Redraw timer > Idle timer. Biased select ensures deterministic priority — signals always handled first, terminal input next, background work last. Without bias, tokio's randomized polling could starve input handling during LSP notification bursts.
- *For Yord*: use biased select with the same ordering. Put process/PTY lifecycle events (kill signals) first, then user input, then background work.

**Hook-based event system with debouncing**
`helix-event` provides two mechanisms:
1. *Synchronous hooks*: Registered via `register_hook!`, dispatched via `dispatch()`. Each hook is `Fn(&mut Event) -> Result<()>`. Runs inline, can mutate editor state immediately. Uses a custom vtable (not `dyn Trait`) because double-indirection is too slow for the hot path.
2. *AsyncHook* (`debounce.rs`): Receives events through a channel, manages debounce timers. `handle_event` can consume immediately or set a debounce deadline; `finish_debounce` fires when the deadline expires. Used for completions, signature help, auto-save, diagnostics, document colors/links, word indexing.

`send_blocking` avoids `block_on` overhead by trying `try_send` first, with a 10ms timeout fallback to prevent freezes.
- *For Yord*: synchronous hooks for tree mutations and focus changes; async hooks with debouncing for process output, file watchers, and expensive operations.

**BTreeMap for documents**
`documents: BTreeMap<DocumentId, Document>` (`editor.rs:1191`). Since `DocumentId` is an incrementing integer, BTreeMap provides ordered iteration (useful for buffer lists) and deterministic behavior. O(log n) lookup is fine — typically fewer than 100 open documents.
- *For Yord*: BTreeMap or HashMap both work. BTreeMap is slightly better for deterministic ordering in workspace/node listings.

**Compositor pattern for UI rendering**
`Compositor` holds a `Vec<Box<dyn Component>>` as layers. Events bubble front-to-back; rendering draws bottom-to-top. Each layer implements `Component` with `handle_event`, `render`, `cursor`, `required_size`. `EventResult::Consumed` stops propagation; `EventResult::Ignored` lets it bubble. Layers identified by `type_name()`. This is a flat layer stack — no nesting — which works because terminal UIs have simple z-ordering.
- *For Yord*: the compositor pattern itself is less relevant (Tauri provides native layering via HTML/CSS), but the consumed/ignored event bubbling pattern is worth adopting for the Svelte component tree.

## Bugs Encountered & How Solved

- **Tree collapse on remove** (`tree.rs:250-270`): When a view is removed and its parent has only one child left, the container is collapsed — the remaining child replaces the container in the grandparent. Prevents degenerate single-child nesting that causes unequal space distribution after split/close cycles. Verified by `all_vertical_views_have_same_width` test.
- **Focus management on remove** (`tree.rs:251-253`): When removing the focused view, focus shifts to `self.prev()` before removal, preventing a dangling focus ID. `prev()` wraps around to the last view if at the beginning.
- **prev/next traversal is "very dumb"** (`tree.rs:549, 569`): Self-acknowledged inefficiency. Uses full tree traversal because visual (in-order) ordering differs from structural ordering. Fine in practice — "unlikely you'll open enough splits to notice."
- **Split layout mismatch** (`tree.rs:159-206`): Two paths — same-direction split inserts as sibling (no new container); cross-direction split creates an intermediate container. Avoids unnecessary nesting.
- **Directional navigation edge case** (`tree.rs:480-491`): When movement within a container fails, recurses upward to the parent instead of giving up, because the container may be embedded in a larger layout where further movement is possible.
- **Swap split across different parents** (`tree.rs:629-666`): Two code paths — same-parent (swap positions and areas) vs. cross-parent (needs `get_disjoint_mut` for four simultaneous mutable borrows, re-parents nodes). Cross-parent was likely a later fix.
- **View position after recalculate** (`tree.rs:355-361`): When the tree is empty, `recalculate` resets focus to root and returns early, guarding the stack-based layout calculation.
- **Rounding in vertical layout gaps** (`tree.rs:412-414`): Subtracts `inner_gap * (n-2)` using `saturating_sub(2)` — arguably should be `n-1`. Verified by tests, suggesting test-driven debugging.
- **Document close lifecycle** (`editor.rs:1965-2041`): `close_document` collects all actions into a Vec before executing, avoiding iterator invalidation. If no views remain after closing, creates a new view with an existing document or fresh scratch buffer — preventing a viewless state.

## Current Bugs & Tech Debt

**Tree**
- `TODO: call f()` (`tree.rs:377`): Recalculate sets view areas directly but has a placeholder for a layout-change-notification callback.
- `TODO: reuse the one we use on update` (`tree.rs:677`): `Traverse` iterator allocates its own `Vec<ViewId>` stack instead of reusing the tree's existing `stack` field. Minor allocation waste.

**Editor**
- `TODO` for `WhitespaceRenderValue::Selection` (`editor.rs:879-880`): Planned but unimplemented rendering mode.
- Race condition with LSP init (`editor.rs:1702`): `text_document_did_open` can race with `on_init` if initialization happens too quickly. Known timing bug, no fix.

**Application**
- `TODO: need to recalculate view tree if necessary` (`application.rs:288`): Render function does not check whether the view tree needs recalculation before rendering.
- `TODO: show multiple status messages at once` (`application.rs:347`): Messages overwrite each other; concurrent messages from fast LSP events are lost.
- `TODO: fix being overwritten by lsp` (`application.rs:646`): "File written" status clobbered by LSP progress messages.

**View**
- HACK: per-view diagnostics handler (`view.rs:152-159`): Each view has its own `DiagnosticsHandler` when there should be one global handler. Comment explicitly says this requires "refactoring View and Document into entity component like structure" — directly validates Yord's ECS approach.
- `TODO: use actual sel` (`view.rs:178`): JumpList initialized with `Selection::point(0)` instead of the actual selection.

**Handlers**
- `TODO: check edits before applying` (`handlers/lsp.rs:55`): LSP workspace edits applied without pre-validation.
- `TODO: make this transactional` (`handlers/lsp.rs:134`): Workspace edits lack transactional semantics; partial application can leave inconsistent state.

**Other**
- Shell command timeout (`expansion.rs:155`): No protection against long-running shell commands.
- DAP offset handling (`handlers/dap.rs:31`): Offsets are "probably utf-16" — uncertain encoding.
- Clipboard performance (`clipboard.rs:155`): `FIXME: check performance of is_exit_success`.
- Multiple LSP signature help (`handlers/signature_help.rs:109, 299`): Only uses the first language server that supports signature help; doesn't merge results from multiple servers.

## What To Learn

**SlotMap tree: split logic** (`tree.rs:144-215`)
1. *Same-direction split*: If the parent container's layout matches the requested direction, insert the new view as a sibling at `focus_index + 1`. No new containers needed.
2. *Cross-direction split*: If layouts differ, create an intermediate container with the desired layout, move the focused view and new view into it, replace the focused view with the container in the parent.

Produces minimal tree depth. Apply to Yord for workspace pane splitting.

**SlotMap tree: remove with collapse** (`tree.rs:250-270`)
1. Move focus to `prev()` if removing the focused view
2. Remove the node from its parent's children list
3. If parent now has exactly one child and is not root, collapse: replace parent with its sole child in the grandparent

The collapse step is critical for maintaining equal spacing.

**SlotMap tree: recalculate** (`tree.rs:355-441`)
Iterative stack-based approach (not recursion):
1. Push `(root, full_area)` onto stack
2. Pop `(node, area)`. If View, assign area. If Container, divide area among children based on layout, push each child with its sub-area.
3. Last child gets remaining space to handle integer division rounding.

Horizontal layout divides height equally. Vertical layout divides width equally with 1px inner gaps. No weighted/resizable splits.

**Document/View separation with multiple views per document**
`Document` stores per-view state in HashMaps keyed by `ViewId` (selections, view offsets, history revision tracking). `View` stores per-document state (doc_revisions, docs_access_history, jump list). When switching documents in a view, `sync_changes` lazily applies pending edits — avoids updating all views on every keystroke.

**Biased select! priority ordering**
- Application level (`application.rs:325-365`): Signals > Input > Callbacks > Status > Futures > Editor events
- Editor level (`editor.rs:2276-2310`): Saves > Config > LSP > Debug > Redraw request > Redraw timer > Idle timer
- Redraw timer (`editor.rs:2296-2298`): 33ms deadline (~30fps). Multiple redraw requests within the window are coalesced into a single render.

**AsyncHook with debouncing** (`debounce.rs:15-37`)
1. Spawn a tokio task with a channel receiver
2. Task loops: if a debounce deadline exists, use `timeout_at` to receive next event or fire the debounce
3. `handle_event` returns `Option<Instant>` — `Some` sets/resets the debounce timer, `None` clears it
4. `finish_debounce` runs when the timer expires (e.g., send the LSP completion request)

Clean, reusable debounce primitive. Each handler implements this trait with its own event enum.

**Job system** (`job.rs:48-53`)
Two categories:
1. *Fire-and-forget*: Spawned as tokio tasks, results sent back via `Callback` channel, errors reported via status messages.
2. *Wait-before-exit*: Pushed into `FuturesUnordered`, editor waits for these during shutdown.

Callbacks come in two flavors: `EditorCompositorCallback` (has `&mut Editor` + `&mut Compositor`) and `EditorCallback` (only `&mut Editor`). `dispatch_blocking` enables synchronous code to queue work for the main loop.

## Issues

**Patterns relevant to Yord's layout tree**
- *Equal-weight-only splits*: Helix doesn't support resizable splits (no drag-to-resize). Each child gets `width / n` or `height / n`. For Yord, consider storing weights per child if resizable panes are needed.
- *No tab/stacking layout*: `Layout` is only `Horizontal | Vertical` with a comment `// could explore stacked/tabbed` (`tree.rs:53`). Yord's workspace model may need tabbed groups.
- *Directional navigation coupled to visual position*: `find_split_in_direction` uses absolute screen coordinates (`area.left()`, `area.top()`) to find the visually closest split. Correct but couples navigation to rendered layout. Won't work if Yord supports virtual/off-screen panes.
- *Tree serialization absent*: No serialization for tree structure. Session restore reconstructs from open files, not saved tree state. Yord needs to serialize the tree to SQLite for session restore.

**Patterns relevant to Yord's event handling**
- *Diagnostics handler HACK is an explicit call for ECS*: The comment at `view.rs:152-159` says the per-view handler "can only happen by refactoring View and Document into entity component like structure." Yord's ECS design directly solves this.
- *Lazy view sync avoids N*M updates*: With N views and M edits, eager sync updates all views. Helix's `doc_revisions` tracking means each view catches up only when active. Essential for Yord if supporting many concurrent pane views.
- *Status message clobbering is a real UX problem*: Multiple TODOs about messages being overwritten. Yord should implement a message queue or toast system from the start.
- *LSP init race condition*: The race at `editor.rs:1702` is an ordering problem. Yord's event system should enforce sequencing for lifecycle events (process started -> ready -> can receive input).
- *Compositor as flat layer stack*: Works for terminal UI, won't scale to Yord. Svelte's component tree with slot-based composition is the better model. However, `EventResult::Consumed/Ignored` bubbling is worth adopting.
- *Frame rate limiting via redraw timer*: 33ms coalescing in `wait_event` prevents render storms. Yord should implement similar coalescing for terminal output and UI updates, likely via `requestAnimationFrame` on the frontend side.

## Key Files

| File | Why |
|------|-----|
| `refs/helix/helix-view/src/tree.rs` | SlotMap-backed split tree: parent pointers, recalculate |
| `refs/helix/helix-view/src/editor.rs` | Central Editor state: tree + documents BTreeMap |
| `refs/helix/helix-event/src/lib.rs` | Hook-based event system with async debouncing |
| `refs/helix/helix-term/src/application.rs` | Main loop: biased tokio::select! with priority ordering |
| `refs/helix/helix-term/src/job.rs` | Job system: wait_futures, callback dispatch |
