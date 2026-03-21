# WASM Plugin System

## Context

Yord's ECS architecture naturally supports extensibility — new component types = new node types. A plugin system would let third parties add node types without modifying core code.

## Recommended approach (from research)

- WASM-based plugins via Extism or Wasmtime in the Rust backend
- Capability-based permissions: plugins declare what they need (`filesystem:read`, `network:http`, `editor:decorations`)
- Host grants only what's declared
- Each plugin runs in its own WASM instance with defined memory limits
- Plugin manifest (`plugin.toml`) with metadata, version, required capabilities

## Anti-patterns to avoid

- **"Just import JS modules"** (Atom pattern) — no isolation, no versioning, killed the platform
- **Over-engineered module system** (Eclipse OSGi) — dependency hell
- **Full DOM access for extensions** (Atom) — monkey-patching, no stability

## Key insight

Build features through the same extension points plugins will use. When you build a node type, use `registerNodeType()`. This guarantees the plugin API works because you're its first consumer.

## When to revisit

M5+. The ECS model means plugins are just new component types + systems, so the architecture is ready.
