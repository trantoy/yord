# bevy_ecs as Runtime Query Engine

## Context

M1 uses plain SQLite with a thin Rust `EntityStore` trait. The ECS *concepts* (entities, components, systems) are the mental model, but no ECS *library* is used.

## When to revisit

M4 (Agent nodes). If 1000+ agents communicate frequently, the access pattern changes:
- Frequent cross-entity reads (agents checking sibling state via `ContextOf`)
- Frequent writes (`Running` status updates, agent output streaming)
- Complex queries ("all agents with access to terminal X across all projects")

At this scale, SQL joins with JSON extraction get ugly. bevy_ecs would give:
- `Query<(&Pty, &Running)>` — real typed queries
- Change detection built-in
- System scheduling with ordering
- ~60fps tick-based updates if needed

## Migration path

The `EntityStore` trait is the swap boundary. M4 implementation:
1. Add `bevy_ecs` as dependency
2. Implement `EntityStore` trait backed by bevy_ecs `World`
3. SQLite becomes persist-only (load on startup, save on change)
4. No frontend changes needed — same IPC surface

## Trade-offs

| | SQLite only | bevy_ecs + SQLite |
|---|---|---|
| Complexity | Low | Medium (two systems to sync) |
| Query power | SQL + JSON extract | Typed Rust queries |
| Performance | Good for <100 entities | Good for 1000+ entities at 60fps |
| Binary size | ~0 (rusqlite) | ~10MB+ |
| Learning curve | SQL | ECS concepts |
| Change detection | Manual (diff on persist) | Built-in |

## Decision

Deferred. Start with SQLite. The trait boundary makes this a non-breaking upgrade.
