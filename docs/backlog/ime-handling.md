# IME (Input Method Editor) Handling

## Context

CJK input and macOS accent menus use IME composition. Known issues across terminal apps:

## Problems observed

- **waveterm**: xterm.js fires `onData` during IME composition, duplicating CJK characters. Fixed by tracking composition state with 20ms post-composition window.
- **waveterm**: Paste deduplication needed — xterm.js `paste()` triggers internal `onData`, doubling pasted text. Fixed with 30ms suppression timeout.
- **WebKitGTK on Linux**: IME composition events can be dropped or duplicated. IBus and Fcitx behave differently.
- **WKWebView on macOS**: Dead keys (Option+E then E for e-acute) can produce double characters if `keydown` handlers are too aggressive.

## When to revisit

M2 (when layout complexity increases and multi-pane focus makes IME handling more critical). Test CJK input on all platforms early.
