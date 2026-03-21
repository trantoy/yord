# Multi-Parent Nodes (Shared Entities)

## Context

M1 uses single-parent `BelongsTo: { parent_id }`. The architecture supports multi-parent `BelongsTo: { parent_ids: [] }` for shared nodes (e.g., one agent serving two projects).

## Migration path

1. Change `BelongsTo` data from `{ parent_id }` to `{ parent_ids: [] }`
2. Update SQL query to use `json_each()` instead of `json_extract() = ?`
3. Update `buildTree()` — an entity can appear in multiple `childrenOf` groups
4. Update UI — show "shared" badge on nodes appearing under multiple parents
5. The `visited` set in `buildTree()` prevents infinite recursion in diamond hierarchies

## Storage schema does not change

The ECS tables remain the same. Only the JSON inside `BelongsTo` components changes.

## When to revisit

When the use case arises — likely M4 (agent nodes shared between projects).
