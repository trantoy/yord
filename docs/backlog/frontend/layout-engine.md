# Tiling Layout Engine

Recursive split/tabbed layout renderer for the content area. LayoutNode tree stored as a component on the project entity.

- Split (h/v) with draggable dividers
- Tabbed panes
- Attach/detach entities to/from panes
- Layout persistence via Layout component
- Empty panes with "+" button

## Priority

M2.

## Referenced from

- `docs/design.md` — section 4.2, 8.3
- `docs/refs/projects/helix.md` — SlotMap tree, recalculate pattern
- `docs/refs/projects/zed.md` — PaneGroup recursive enum
- `docs/refs/projects/zellij.md` — TiledPaneLayout + flat BTreeMap hybrid
