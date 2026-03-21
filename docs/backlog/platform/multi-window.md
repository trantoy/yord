# Multi-Window Support

Detach a project into its own OS window. Useful for multi-monitor setups. Tauri 2's multi-webview is unstable on Linux (WebKitGTK doesn't support positioned child webviews). May need single-webview-per-window approach.

## Priority

Future. Must be compatible with the layout model.

## Referenced from

- `docs/design.md` — open question #2
- `docs/research/landscape/tauri-pitfalls.md` — multi-webview instability
- `docs/refs/projects/clif-code.md` — window-scoped PTY management pattern
