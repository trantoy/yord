# Domain-Based SQLite Migrations

## Context

M1 uses a simple append-only `&[&str]` migration system with a `_migrations` version table. This is sufficient for a single-schema ECS store.

## Zed's pattern

Each crate declares a `Domain` with `NAME` and ordered `MIGRATIONS` (SQL strings). Domains compose via tuple types. Benefits:
- Independent evolution per domain sharing a SQLite file
- Change detection: stored migration text compared to code on startup
- Composability: `Migrator` implemented for tuples of up to 5 domains
- Retry logic for concurrent process scenarios

## When to revisit

When Yord's codebase becomes modular enough to warrant per-domain migrations. Likely M3+ when web nodes add their own storage needs, or if a plugin system (M5+) needs per-plugin schema evolution.

## Decision

Deferred. Simple append-only is sufficient for M1's single schema.
