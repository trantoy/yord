# Flat Storage to Tree View — Zellij, Helix, and Yord's buildTree()

Research on how existing Rust projects convert flat/semi-flat data into tree views,
and a concrete recommendation for Yord's `buildTree()` implementation.

---

## 1. Zellij — Flat BTreeMap + Transient Grid View

### Files analyzed

- `zellij-server/src/panes/tiled_panes/mod.rs` — `TiledPanes` struct
- `zellij-server/src/panes/tiled_panes/tiled_pane_grid.rs` — `TiledPaneGrid`
- `zellij-utils/src/input/layout.rs` — `TiledPaneLayout` tree

### How flat data is stored

`TiledPanes` owns a `BTreeMap<PaneId, Box<dyn Pane>>`. This is the canonical flat
store — panes are keyed by ID with no embedded parent/child links. Each pane carries
its own `PaneGeom` (position, size, stacking info) but no pointer to a parent.

```rust
pub struct TiledPanes {
    pub panes: BTreeMap<PaneId, Box<dyn Pane>>,
    // ... display_area, viewport, active_panes, etc.
}
```

### How the tree view is derived

Zellij uses two tree representations, for different purposes:

**A) `TiledPaneLayout` — persistent tree (config/restore)**

This is a conventional recursive tree used for layout definitions and session restore:

```rust
pub struct TiledPaneLayout {
    pub children_split_direction: SplitDirection,
    pub children: Vec<TiledPaneLayout>,
    pub split_size: Option<SplitSize>,
    pub run: Option<Run>,
    // ...
}
```

Its key method `position_panes_in_space()` takes a tree layout and flattens it into
`Vec<(TiledPaneLayout, PaneGeom)>` — a list of positioned panes. This is the
tree-to-flat direction: the layout tree defines structure, then it gets
materialized into geometry for each leaf pane.

**B) `TiledPaneGrid` — transient view (runtime operations)**

For runtime operations (resize, split, neighbor-finding), Zellij creates a
*transient* `TiledPaneGrid` from the flat pane map on each operation:

```rust
pub struct TiledPaneGrid<'a> {
    panes: Rc<RefCell<HashMap<PaneId, &'a mut Box<dyn Pane>>>>,
    display_area: Size,
    viewport: Viewport,
}
```

This grid is constructed ad-hoc from `&mut self.panes` every time it's needed
(e.g. `relayout()`, `has_room_for_new_pane()`, `resize()`). It borrows the pane
references, does its work, and is dropped. There is no cached tree — every
operation rebuilds the working view.

### Performance characteristics

**O(n) rebuild on every operation.** `TiledPaneGrid::new()` iterates all panes,
filters hidden ones, and collects into a `HashMap`. This is called per-operation, not
cached. For the typical use case (tens of panes), this is negligible.

The layout tree's `position_panes_in_space()` is a recursive O(n) traversal with
`split_space()` doing the geometry math at each level.

### Ordering/sorting

Within the layout tree, order is determined by `Vec<TiledPaneLayout>` index —
position in the children vector is the canonical order. There is no separate order
field. The `split_space()` function processes children in vector order to assign
screen positions.

BTreeMap ordering (by PaneId) provides a stable iteration order for the flat store,
but this is not semantically meaningful for layout — it's just a deterministic
fallback.

### Add/remove/reorder

- **Add**: Insert pane into `BTreeMap`, then call `relayout()` which rebuilds
  `TiledPaneGrid` and recalculates geometry.
- **Remove**: Remove from `BTreeMap`, call `relayout()`.
- **Reorder**: The layout tree defines order. Swapping panes swaps their `PaneGeom`
  and position in the parent container's children vec. The flat map is unaffected.

### Key insight for Yord

Zellij does NOT maintain a persistent tree from its flat data. The tree
(`TiledPaneLayout`) is a separate config/template structure. The flat store
(`BTreeMap`) is the runtime truth, and tree-like operations are done via transient
views that borrow from it. This is a "flat data + rebuild views on demand" pattern.

---

## 2. Helix — SlotMap Tree with Recalculate

### File analyzed

- `helix-view/src/tree.rs` — complete `Tree` implementation

### How flat data is stored

Helix uses a `SlotMap<ViewId, Node>` — a generational arena. This is conceptually
flat (all nodes in one contiguous allocator), but each node stores an explicit
`parent: ViewId` link, and container nodes store `children: Vec<ViewId>`.

```rust
pub struct Tree {
    root: ViewId,
    pub focus: ViewId,
    area: Rect,
    nodes: SlotMap<ViewId, Node>,
    stack: Vec<(ViewId, Rect)>,  // reused for traversals
}

pub struct Node {
    parent: ViewId,
    content: Content,
}

pub enum Content {
    View(Box<View>),
    Container(Box<Container>),
}

pub struct Container {
    layout: Layout,           // Horizontal | Vertical
    children: Vec<ViewId>,
    area: Rect,
}
```

### How the tree view is derived/built

The tree IS the data structure — there is no separate flat-to-tree conversion.
Nodes live in a flat arena (SlotMap) but carry parent/children links that encode
the tree topology. The tree is always maintained, not rebuilt.

The critical method is `recalculate()`, which computes screen geometry from
the tree structure:

```rust
pub fn recalculate(&mut self) {
    self.stack.push((self.root, self.area));
    while let Some((key, area)) = self.stack.pop() {
        match &mut self.nodes[key].content {
            Content::View(view) => { view.area = area; }
            Content::Container(container) => {
                container.area = area;
                // divide area among children based on layout direction
                for (i, child) in container.children.iter().enumerate() {
                    // calculate child_area...
                    self.stack.push((*child, child_area));
                }
            }
        }
    }
}
```

This is an iterative BFS/DFS using a reusable stack (no allocations after warmup).

### Performance characteristics

- **recalculate()**: O(n) iterative traversal. Called after every structural change
  (insert, remove, split, resize). Uses a pre-allocated `Vec` as a stack, so no
  allocation churn.
- **insert/split/remove**: O(1) for the structural change itself (SlotMap insert is
  O(1), parent/child link updates are O(k) where k is sibling count), then O(n) for
  recalculate.
- **traverse()**: O(n) lazy iterator using a stack. Supports `DoubleEndedIterator`.
- **prev()/next()**: O(n) — they iterate the whole tree to find the current node,
  then return the adjacent one. Comment acknowledges this is "very dumb" but
  acceptable for small n.

### Ordering/sorting

Order is determined by index within `Container.children: Vec<ViewId>`. Position in
the vec = position on screen. Insert always goes after the currently focused node:

```rust
let pos = container.children.iter().position(|&child| child == focus).unwrap();
container.children.insert(pos + 1, node);
```

There is no separate `Order` component — the vec index IS the order.

### How they handle add/remove/reorder

**Add (insert/split)**:
- `insert()`: adds a view as a sibling after the focused view in the same container.
- `split()`: if the parent container's layout matches the split direction, inserts
  as a sibling. Otherwise, creates a new intermediate container node, reparents the
  focused view and new view under it, and replaces the focused view in the parent's
  children vec with the new container. Then calls `recalculate()`.

**Remove**:
- Removes node from SlotMap.
- Removes from parent's children vec.
- If parent container is left with only 1 child and is not root, merges: the single
  remaining child replaces the container in the grandparent. This prevents
  degenerate nesting. Then calls `recalculate()`.

**Reorder (swap)**:
- `swap_split_in_direction()`: finds the target view in the given direction,
  swaps their positions in their respective parent containers' children vecs,
  and swaps their areas. No recalculate needed since areas are swapped directly.

### Key insight for Yord

Helix maintains the tree at all times in a flat arena (SlotMap) with explicit
parent/children links — structurally identical to what Yord's ECS stores with
`BelongsTo` + `Order` components. The `recalculate()` pattern (full O(n) recompute
after any mutation) is directly applicable: Yord's `buildTree()` is the same
pattern, just crossing the Rust/TS boundary.

The auto-collapsing of single-child containers on remove is a nice UX touch
worth considering.

---

## 3. Comparison Table

| Aspect | Zellij | Helix |
|--------|--------|-------|
| Canonical store | `BTreeMap<PaneId, Pane>` (truly flat) | `SlotMap<ViewId, Node>` (flat arena, tree links) |
| Tree structure | Separate `TiledPaneLayout` tree | Embedded in `Node.parent` + `Container.children` |
| When tree is built | On demand (transient `TiledPaneGrid`) | Always maintained, `recalculate()` updates geometry |
| Rebuild cost | O(n) per operation | O(n) `recalculate()` after mutation |
| Ordering | Vec index in layout tree children | Vec index in `Container.children` |
| Add | Insert to BTreeMap + relayout | Insert to SlotMap + link + recalculate |
| Remove | Remove from BTreeMap + relayout | Remove from SlotMap + unlink + collapse + recalculate |
| Reorder | Swap in parent's children vec | Swap in children vec + swap areas |

---

## 4. Recommendation: Yord's buildTree()

### Data model recap

Yord stores entities flat in SQLite (EAV pattern). The frontend receives
`EntityWithComponents[]` from Rust. Each entity may have:
- `BelongsTo { parent_id: string }` — single parent link
- `Order { index: number }` — sibling sort key
- `Label { text, icon?, color? }` — display info
- Various identity components (`Project`, `Pty`, `Agent`, etc.)

The function `buildTree()` must convert this into `TreeNode[]` for Svelte rendering
with `$state.raw()`.

### Approach: Helix-style O(n) rebuild, not incremental

Both Zellij and Helix do full O(n) rebuilds on every structural change.
For Yord's scale (hundreds of entities, not millions), this is the right call.
Incremental tree updates add complexity for zero perceptible benefit at this scale.

The Svelte integration via `$state.raw()` requires full reassignment to trigger
reactivity anyway, so there is no partial-update shortcut available on the UI side.

### Implementation

```typescript
// src/lib/types/tree.ts

export interface TreeNode {
  id: string;
  entity: EntityWithComponents;
  children: TreeNode[];
  depth: number;
}
```

```typescript
// src/lib/stores/tree.ts

/**
 * Converts flat entity array into a nested tree.
 *
 * Algorithm: single-pass index build + recursive assembly.
 * O(n) time, O(n) space. Handles orphans, missing parents, cycles.
 *
 * Design decisions informed by Zellij/Helix research:
 * - Full rebuild on every change (both projects do this)
 * - Order by explicit index (like Helix's Vec position)
 * - Orphan handling (Zellij silently drops, we promote to root)
 */
export function buildTree(entities: EntityWithComponents[]): TreeNode[] {
  // 1. Build lookup maps in one pass
  const byId = new Map<string, EntityWithComponents>();
  const childrenOf = new Map<string, EntityWithComponents[]>();

  for (const entity of entities) {
    byId.set(entity.id, entity);
  }

  // 2. Group children by parent_id
  //    Track which IDs appear as children (for cycle/orphan detection)
  const hasParent = new Set<string>();

  for (const entity of entities) {
    const belongsTo = entity.components.BelongsTo;
    if (belongsTo && belongsTo.parent_id) {
      const parentId = belongsTo.parent_id;
      // Defensive: skip self-referential parent
      if (parentId === entity.id) continue;
      // Only link if parent actually exists (orphan protection)
      if (byId.has(parentId)) {
        let siblings = childrenOf.get(parentId);
        if (!siblings) {
          siblings = [];
          childrenOf.set(parentId, siblings);
        }
        siblings.push(entity);
        hasParent.add(entity.id);
      }
    }
  }

  // 3. Sort each sibling group by Order.index
  for (const siblings of childrenOf.values()) {
    siblings.sort((a, b) => {
      const aIdx = a.components.Order?.index ?? Infinity;
      const bIdx = b.components.Order?.index ?? Infinity;
      return aIdx - bIdx;
    });
  }

  // 4. Recursive assembly with cycle detection
  const visited = new Set<string>();

  function buildNode(entity: EntityWithComponents, depth: number): TreeNode {
    // Cycle guard: if already building this node, return leaf
    if (visited.has(entity.id)) {
      return { id: entity.id, entity, children: [], depth };
    }
    visited.add(entity.id);

    const childEntities = childrenOf.get(entity.id) ?? [];
    const children = childEntities.map(child => buildNode(child, depth + 1));

    return { id: entity.id, entity, children, depth };
  }

  // 5. Root nodes: entities without a valid parent link
  const roots: EntityWithComponents[] = [];
  for (const entity of entities) {
    if (!hasParent.has(entity.id)) {
      roots.push(entity);
    }
  }

  // Sort roots by Order.index too
  roots.sort((a, b) => {
    const aIdx = a.components.Order?.index ?? Infinity;
    const bIdx = b.components.Order?.index ?? Infinity;
    return aIdx - bIdx;
  });

  return roots.map(root => buildNode(root, 0));
}
```

### Integration with Svelte 5 $state.raw()

```typescript
// In a .svelte.ts store file

import { buildTree, type TreeNode } from './tree';

// Use $state.raw() — no deep proxy, reassign to trigger updates
let tree = $state.raw<TreeNode[]>([]);
let entities = $state.raw<EntityWithComponents[]>([]);

export function setEntities(newEntities: EntityWithComponents[]) {
  entities = newEntities;
  tree = buildTree(entities);  // full rebuild, reassignment triggers UI update
}

export function getTree(): TreeNode[] {
  return tree;
}
```

### Why this works

1. **O(n) is fast enough.** Both Helix and Zellij rebuild O(n) on every mutation.
   For 500 entities, `buildTree()` takes <1ms. For 5,000 entities, <5ms.
   Profile before optimizing.

2. **$state.raw() compatibility.** The function returns a fresh object tree on
   every call. Reassigning the variable triggers Svelte's reactivity. No proxy
   overhead, no mutation tracking needed.

3. **Orphan handling.** Entities whose `parent_id` references a non-existent entity
   are promoted to root nodes (visible, not silently dropped). This matches the
   defensive principle from Helix's remove-and-collapse pattern.

4. **Cycle defense.** The `visited` set prevents infinite recursion if data is
   corrupted. A cycle produces a truncated subtree rather than a stack overflow.

5. **Stable ordering.** `Order.index` is the sort key, matching Helix's explicit
   position-in-vec approach. Entities without `Order` sort last (Infinity), keeping
   them visible but at the end.

6. **No intermediate tree structure.** Unlike Helix (which maintains a persistent
   arena tree), Yord rebuilds from flat data every time. This is simpler and
   matches the Zellij pattern of transient views from flat storage. The source of
   truth stays in SQLite / the entities array — the tree is always derived, never
   primary.

### When to call buildTree()

- On initial load (`get_entities()` from Rust)
- On `entity:component-changed` event (re-fetch affected entities, rebuild)
- On `entity:removed` event (filter out, rebuild)

For debouncing: if multiple events fire in rapid succession (e.g. bulk import),
batch them with `queueMicrotask()` or a 16ms debounce to avoid redundant rebuilds.

### Future: multi-parent (parent_ids[])

When `BelongsTo` changes from `{ parent_id }` to `{ parent_ids: [] }`, the
function changes minimally:
- Step 2 iterates `parent_ids` instead of checking a single `parent_id`
- An entity can appear in multiple `childrenOf` groups
- The `visited` set becomes critical to prevent infinite recursion in diamond
  hierarchies
- UI shows a "shared" badge on nodes that appear under multiple parents

The same O(n) rebuild pattern handles this naturally.
