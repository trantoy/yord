# Browser Platform Support

Tauri 2 covers Linux, Windows, macOS, Android, iOS natively. Browser is not supported — the Rust backend (PTY, process management, SQLite) can't run in a browser.

## Future approach

The Svelte frontend is already web-ready. To support browser:

1. Build a server-side backend (could reuse the same Rust code as a web service)
2. Replace Tauri IPC (commands + events) with WebSocket or HTTP API
3. PTY sessions would run server-side, streamed to the browser via WebSocket
4. SQLite could stay server-side or move to a browser-compatible store (IndexedDB, OPFS)

## Considerations

- Server-side PTY = latency-sensitive, needs good WebSocket handling
- Auth/multi-user becomes a concern if the backend is a server
- Mobile web vs native: Tauri already gives native Android/iOS, so browser mainly serves desktop-without-install use case

## Priority

Deferred. Revisit after M3 when the desktop app is stable.

## Referenced from

- `docs/design.md` — target platforms footer
