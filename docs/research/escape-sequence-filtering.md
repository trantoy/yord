# Escape Sequence Filtering Specification

Research based on waveterm, zed, and wezterm codebases. Targeting CVE-2022-45872-class attacks (RCE via terminal escape sequences in iTerm2) and related clipboard/title exfiltration vectors.

---

## Background: How the Attack Works

The fundamental threat model: a malicious program (or crafted file content displayed via `cat`, `curl`, etc.) emits escape sequences that cause the terminal emulator to:

1. **Write to the clipboard** (OSC 52 store) -- attacker plants shell commands
2. **Read from the clipboard and echo it back** (OSC 52 query) -- attacker steals secrets
3. **Report the window title** (CSI 20t / CSI 21t) -- title was previously set to contain shell commands via OSC 0/1/2, and the report is written back to stdin where the shell executes it
4. **Execute OS commands** (OSC 7 with crafted URLs, DCS with device control strings)
5. **Resize/move/iconify the window** (CSI window manipulation) -- UI disruption or social engineering

CVE-2022-45872 (iTerm2): Malicious content could set a window title to a shell command via OSC, then request the title report. The report was written to the terminal input, where the shell executed it as a command. RCE.

---

## 1. Whitelist: Sequences That MUST Be Allowed

These are essential for terminal functionality. xterm.js handles them internally; they must pass through unmodified from PTY to xterm.js.

### Cursor Movement (CSI)
| Sequence | Name | Purpose |
|----------|------|---------|
| CSI n A | CUU | Cursor up |
| CSI n B | CUD | Cursor down |
| CSI n C | CUF | Cursor forward |
| CSI n D | CUB | Cursor backward |
| CSI n ; m H | CUP | Cursor position |
| CSI n ; m f | HVP | Horizontal/vertical position |
| CSI s | SCP | Save cursor position |
| CSI u | RCP | Restore cursor position |
| CSI 6 n | DSR | Device status report (cursor position) |

### Erase / Scroll (CSI)
| Sequence | Name | Purpose |
|----------|------|---------|
| CSI n J | ED | Erase in display (0/1/2/3) |
| CSI n K | EL | Erase in line |
| CSI n S | SU | Scroll up |
| CSI n T | SD | Scroll down |
| CSI n L | IL | Insert lines |
| CSI n M | DL | Delete lines |
| CSI n @ | ICH | Insert characters |
| CSI n P | DCH | Delete characters |

### Graphics Rendition (CSI)
| Sequence | Name | Purpose |
|----------|------|---------|
| CSI n m | SGR | Set graphics rendition (colors, bold, italic, underline) |
| CSI 38;2;r;g;b m | SGR | 24-bit foreground color |
| CSI 48;2;r;g;b m | SGR | 24-bit background color |
| CSI 38;5;n m | SGR | 256-color foreground |
| CSI 48;5;n m | SGR | 256-color background |

### Mode Setting (CSI ? ... h/l)
| Sequence | Name | Purpose |
|----------|------|---------|
| CSI ? 1 h/l | DECCKM | Application cursor keys |
| CSI ? 7 h/l | DECAWM | Auto-wrap mode |
| CSI ? 12 h/l | -- | Cursor blinking |
| CSI ? 25 h/l | DECTCEM | Show/hide cursor |
| CSI ? 47 h/l | -- | Alternate screen buffer (old) |
| CSI ? 1000 h/l | -- | Mouse tracking (basic) |
| CSI ? 1002 h/l | -- | Mouse tracking (button events) |
| CSI ? 1003 h/l | -- | Mouse tracking (any event) |
| CSI ? 1006 h/l | -- | SGR mouse mode |
| CSI ? 1049 h/l | -- | Alternate screen buffer (with save/restore cursor) |
| CSI ? 2004 h/l | -- | Bracketed paste mode |
| CSI ? 2026 h/l | -- | Synchronized output (used by waveterm for repaint transactions) |

### Character Sets / Encoding (ESC)
| Sequence | Purpose |
|----------|---------|
| ESC ( 0 | DEC line drawing character set |
| ESC ( B | ASCII character set |
| ESC 7 / ESC 8 | Save/restore cursor (DECSC/DECRC) |
| ESC M | Reverse index |
| ESC D | Index (line feed) |
| ESC E | Next line |
| ESC c | Full terminal reset (RIS) |

### OSC (Allowed, with restrictions)
| Sequence | Name | Restrictions |
|----------|------|-------------|
| OSC 0 ; text ST | Set window title + icon | Allow set, block report (see blacklist) |
| OSC 1 ; text ST | Set icon name | Allow set, block report |
| OSC 2 ; text ST | Set window title | Allow set, block report |
| OSC 4 ; index ; color ST | Set/query color palette | Allow, see color handling below |
| OSC 7 ; url ST | Current directory | Allow only `file:` protocol, validate path (see waveterm implementation) |
| OSC 8 ; params ; uri ST | Hyperlinks | Allow |
| OSC 10-19 ; color ST | Dynamic colors | Allow set; query responses handled carefully |
| OSC 52 ; sel ; data ST | Clipboard write | RESTRICTED -- see section 2 |
| OSC 104 | Reset color | Allow |
| OSC 110-119 | Reset dynamic colors | Allow |

### Control Characters
| Character | Purpose |
|-----------|---------|
| BEL (0x07) | Bell notification |
| BS (0x08) | Backspace |
| HT (0x09) | Horizontal tab |
| LF (0x0A) | Line feed |
| CR (0x0D) | Carriage return |
| ESC (0x1B) | Escape (sequence initiator) |

---

## 2. Blacklist: Sequences That MUST Be Stripped or Restricted

### CRITICAL: Title Report Sequences (RCE Vector)

**Attack**: Set title to shell command via OSC 0/2, then request title report via CSI 20t/21t. Report is written to PTY input; shell executes it.

| Sequence | Name | Action | CVE/Reference |
|----------|------|--------|---------------|
| CSI 20 t | Report icon name | **BLOCK entirely** | CVE-2003-0063, CVE-2022-45872 |
| CSI 21 t | Report window title | **BLOCK entirely** | CVE-2003-0063, CVE-2022-45872 |

**Wezterm approach**: `enable_title_reporting` config option, defaults to `false`. Comment reads: "Disabled by default for security concerns with shells that might otherwise attempt to execute the response." Reference: <https://marc.info/?l=bugtraq&m=104612710031920&w=2>

**Implementation**: Strip these in the Rust PTY output filter. Do not rely solely on xterm.js.

### CRITICAL: OSC 52 Clipboard Query (Clipboard Theft)

**Attack**: OSC 52 with `?` as data requests the terminal to respond with clipboard contents. Attacker reads your passwords/tokens.

| Sequence | Action | Reference |
|----------|--------|-----------|
| OSC 52 ; sel ; ? ST | Clipboard query/read | Clipboard exfiltration |

**Waveterm approach**: Explicitly checks for `?` and blocks it. Comment: "clipboard query ('?') is not supported for security (prevents clipboard theft)".

**Wezterm approach**: `OperatingSystemCommand::QuerySelection(_) => {}` -- silently ignores clipboard queries (no-op).

**Implementation**: Block in both Rust filter AND xterm.js OSC 52 handler.

### CRITICAL: OSC 52 Clipboard Write (Command Injection via Paste)

Not blocked outright, but must be restricted. An attacker writes shell commands to clipboard; user pastes them later.

| Control | Implementation |
|---------|---------------|
| Size limit | Max 75KB decoded (waveterm: `Osc52MaxDecodedSize = 75 * 1024`) |
| Raw length limit | Max 128KB raw (waveterm: `Osc52MaxRawLength = 128 * 1024`) |
| Focus gating | Only allow when terminal block is focused (waveterm `osc52Mode: "focus"` option) |
| User notification | Consider a toast/indicator when clipboard is written via OSC 52 |

### HIGH: Window Manipulation Sequences (UI Disruption)

| Sequence | Name | Action | Risk |
|----------|------|--------|------|
| CSI 1 t | De-iconify | **BLOCK** | UI manipulation |
| CSI 2 t | Iconify/minimize | **BLOCK** | UI manipulation |
| CSI 3;x;y t | Move window | **BLOCK** | UI manipulation |
| CSI 4;h;w t | Resize window (pixels) | **BLOCK** | UI manipulation |
| CSI 5 t | Raise window | **BLOCK** | UI manipulation |
| CSI 6 t | Lower window | **BLOCK** | UI manipulation |
| CSI 7 t | Refresh window | **BLOCK** | UI manipulation |
| CSI 8;r;c t | Resize window (chars) | **BLOCK** | UI manipulation |
| CSI 9;n t | Maximize/fullscreen | **BLOCK** | UI manipulation |
| CSI 10;n t | Fullscreen toggle | **BLOCK** | UI manipulation |
| CSI 11 t | Report window state | **BLOCK** | Information leak |
| CSI 13 t | Report window position | **BLOCK** | Information leak |
| CSI 14 t | Report window size (px) | **BLOCK** | Information leak |
| CSI 15 t | Report screen size (px) | **BLOCK** | Information leak |
| CSI 16 t | Report cell size (px) | Allow (needed by some TUIs) |
| CSI 18 t | Report text area size (chars) | Allow (needed by some TUIs) |
| CSI 19 t | Report screen size (chars) | **BLOCK** | Information leak |
| CSI 22;n t | Push title | Allow |
| CSI 23;n t | Pop title | Allow |

**Wezterm approach**: `Window::ResizeWindowCells { .. } => { /* We don't allow the application to change the window size */ }`, `Window::Iconify | Window::DeIconify => {}` (silently ignored).

### HIGH: DECRQSS (Status String Request)

**Attack**: DECRQSS responses are DCS sequences written to PTY input. Malformed handling or unexpected responses could be exploited.

| Sequence | Action | Risk |
|----------|--------|------|
| DCS $ q ... ST | DECRQSS | **BLOCK in Rust filter** | Response written to PTY input |

**Wezterm approach**: Handles only 3 specific DECRQSS sub-commands (`"p`, `r`, `s`) and replies with "invalid" for everything else.

**Yord approach**: Since we use xterm.js (not a custom terminal state machine), DECRQSS is not natively handled. Block DCS sequences from PTY output in the Rust filter to prevent xterm.js from receiving them at all. If needed in the future, whitelist specific DCS sub-types.

### MEDIUM: Bracketed Paste Injection

**Attack**: Attacker embeds `ESC[201~` (end bracketed paste) in clipboard content, followed by raw commands. Shell thinks paste ended and executes the commands.

| Sequence | Action | Reference |
|----------|--------|-----------|
| ESC [ 200 ~ / ESC [ 201 ~ in pasted content | **Strip from paste input** | Paste injection |

**Wezterm approach**: `send_paste()` explicitly de-fangs: `canon.replace("\x1b[200~", "").replace("\x1b[201~", "")`.

**Implementation**: In Yord's paste handler (TS side), strip bracketed paste markers from clipboard content before sending to PTY.

### MEDIUM: DCS (Device Control Strings) -- General

DCS sequences can carry arbitrary payloads. Most are not needed for basic terminal operation.

| Sequence | Name | Action |
|----------|------|--------|
| DCS ... ST (general) | Device control | **BLOCK by default** unless specifically whitelisted |
| DCS + q ... ST | XTGETTCAP | Allow if needed for terminfo queries |
| DCS $ q ... ST | DECRQSS | See above |
| DCS tmux; ... ST | Tmux control mode | Block (Yord manages its own sessions) |

### LOW: OSC 7 URL Validation

**Attack**: OSC 7 with non-file: protocol or crafted path could trigger unintended behavior.

**Waveterm approach**: Validates `file:` protocol, decodes URL, normalizes path separators, strips double slashes, handles Windows paths. Maximum data length of 1024 bytes.

**Implementation**: Same validation in Rust-side filter. Reject non-file: protocols. Cap at 1024 bytes.

---

## 3. Implementation: Where to Filter

Yord's data flow for terminal output:

```
PTY (child process)
  |
  | raw bytes
  v
Rust PTY reader (Tauri command / actor)
  |
  | --- FILTER LAYER 1: Rust-side escape sequence filter ---
  |
  | filtered bytes
  v
Tauri event / channel to frontend
  |
  v
xterm.js terminal.write(data)
  |
  | --- FILTER LAYER 2: xterm.js parser handlers ---
  |
  v
Rendered terminal output
```

### Layer 1: Rust-Side Filter (Primary Defense)

Location: Between PTY read and Tauri event emission. This is the chokepoint where ALL output passes.

**What to filter here:**
- Strip CSI 20t, CSI 21t (title reports) -- these never reach xterm.js
- Strip all window manipulation CSI t sequences except 16t, 18t, 22t, 23t
- Strip DCS sequences entirely (unless whitelist needed later)
- Validate OSC 7 (reject non-file: protocol, length cap)
- Strip OSC 52 clipboard queries (data = "?")
- Enforce OSC 52 size limits

**How to implement:**

Use a lightweight state machine or the `vte` crate (the same VTE parser that alacritty uses) to parse the byte stream. The filter operates as a transform: it reads PTY output, parses escape sequences, and emits only allowed sequences to the output buffer.

```rust
// Pseudocode
struct EscapeFilter {
    parser: vte::Parser,
    performer: FilterPerformer,
}

impl EscapeFilter {
    fn filter(&mut self, input: &[u8]) -> Vec<u8> {
        let mut output = Vec::with_capacity(input.len());
        for byte in input {
            self.parser.advance(&mut self.performer, *byte);
        }
        self.performer.take_output()
    }
}

struct FilterPerformer {
    output: Vec<u8>,
    // State for tracking current sequence
}

impl vte::Perform for FilterPerformer {
    fn print(&mut self, c: char) {
        // Always pass through
        self.output.extend(c.to_string().as_bytes());
    }

    fn csi_dispatch(&mut self, params: &[i64], intermediates: &[u8], ignore: bool, action: char) {
        if action == 't' {
            // Window manipulation -- check param[0]
            match params.first() {
                Some(20) | Some(21) => return,  // Block title reports
                Some(1..=10) | Some(11) | Some(13..=15) | Some(19) => return,  // Block window manip
                _ => {}  // Allow 16, 18, 22, 23 and others
            }
        }
        // Re-emit the CSI sequence
        self.emit_csi(params, intermediates, action);
    }

    fn osc_dispatch(&mut self, params: &[&[u8]], bell_terminated: bool) {
        // Check OSC number
        match params.first().and_then(|p| std::str::from_utf8(p).ok()) {
            Some("52") => {
                // Check for clipboard query
                if let Some(data) = params.get(1) {
                    if data.ends_with(b"?") { return; }  // Block query
                    if data.len() > 128 * 1024 { return; }  // Size limit
                }
            }
            Some("7") => {
                // Validate file: protocol
                if let Some(url) = params.get(1) {
                    let url_str = String::from_utf8_lossy(url);
                    if !url_str.starts_with("file:") { return; }
                    if url.len() > 1024 { return; }
                }
            }
            _ => {}
        }
        // Re-emit
        self.emit_osc(params, bell_terminated);
    }

    fn hook(&mut self, params: &[i64], intermediates: &[u8], ignore: bool, action: char) {
        // DCS start -- block by default
        // Could whitelist XTGETTCAP if needed
    }

    // ... other vte::Perform methods pass through
}
```

**Performance note**: The `vte` crate is zero-copy and fast (used by alacritty for real-time terminal rendering). The overhead of parsing + filtering is negligible compared to PTY I/O latency.

### Layer 2: xterm.js Handlers (Secondary Defense / Application Logic)

Location: `terminal.parser.registerOscHandler()` / `registerCsiHandler()` in the frontend.

**What to handle here:**
- OSC 52: Application-level clipboard write with focus gating, user notification
- OSC 7: Update Yord's internal CWD tracking for the terminal entity
- OSC custom (Yord-specific): Shell integration, prompt markers
- CSI J (param 3): Clear scrollback detection (waveterm uses this for repaint transactions)
- CSI ? 2026 h/l: Synchronized output transaction tracking

These are NOT security filters. They are application-level handlers that process allowed sequences for Yord-specific behavior. The security filtering already happened in Layer 1.

---

## 4. xterm.js Configuration

### Terminal Options

```typescript
const terminal = new Terminal({
    // Core rendering
    allowTransparency: true,
    fontFamily: "...",
    fontSize: 14,

    // Proposed API needed for search addon decorations
    allowProposedApi: true,

    // Scrollback
    scrollback: 10000,

    // Security-relevant: do NOT set windowOptions
    // xterm.js windowOptions controls which CSI t sequences
    // get responses. Leaving it unset = all blocked by default.
    // windowOptions: {}  // DO NOT SET

    // Bracketed paste mode
    // Do not disable; let the shell control it
    // ignoreBracketedPasteMode: false,
});
```

### Parser Handler Registrations

```typescript
// --- OSC 7: Current Working Directory ---
terminal.parser.registerOscHandler(7, (data: string) => {
    // Validate: file: protocol only, max 1024 chars
    // Update entity CWD component via Tauri command
    return true;  // Consume the sequence
});

// --- OSC 52: Clipboard Operations ---
terminal.parser.registerOscHandler(52, (data: string) => {
    // This handler runs AFTER Rust-side filtering, which already:
    //   - Blocked clipboard queries ("?")
    //   - Enforced size limits
    //
    // Here we implement application-level policy:
    //   - Focus gating: only write clipboard if terminal pane is focused
    //   - Optional user notification
    //   - Parse selector;base64 format
    //   - Decode and write to clipboard via navigator.clipboard.writeText()
    return true;  // Consume the sequence
});

// --- OSC <yord-custom>: Shell Integration ---
terminal.parser.registerOscHandler(YORD_OSC_NUMBER, (data: string) => {
    // Shell integration protocol (prompt markers, command tracking)
    // Similar to waveterm's OSC 16162
    return true;
});

// --- CSI J (param 3): Clear Scrollback Detection ---
terminal.parser.registerCsiHandler({ final: "J" }, (params) => {
    if (params[0] === 3) {
        // Scrollback clear detected -- may be part of repaint transaction
    }
    return false;  // Let xterm.js process it normally
});

// --- CSI ? 2026 h/l: Synchronized Output ---
terminal.parser.registerCsiHandler({ prefix: "?", final: "h" }, (params) => {
    if (params[0] === 2026) {
        // Sync transaction start
    }
    return false;  // Let xterm.js handle it
});

terminal.parser.registerCsiHandler({ prefix: "?", final: "l" }, (params) => {
    if (params[0] === 2026) {
        // Sync transaction end -- scroll to bottom if repaint
    }
    return false;  // Let xterm.js handle it
});
```

### Paste Handling

```typescript
// Intercept paste events to de-fang bracketed paste injection
function handlePaste(event: ClipboardEvent): void {
    event.preventDefault();
    const text = event.clipboardData?.getData("text/plain") ?? "";

    // Strip embedded bracketed paste markers (wezterm approach)
    const sanitized = text
        .replace(/\x1b\[200~/g, "")
        .replace(/\x1b\[201~/g, "");

    // Send to PTY via Tauri command
    // xterm.js will add its own bracketed paste markers if the mode is active
    sendToPty(sanitized);
}
```

---

## 5. Summary Table

| Sequence | Category | Layer 1 (Rust) | Layer 2 (xterm.js) | Risk |
|----------|----------|----------------|-------------------|------|
| CSI 20t / 21t | Title report | **STRIP** | N/A | RCE (CVE-2022-45872) |
| CSI 1-10t, 11t, 13-15t, 19t | Window manipulation | **STRIP** | N/A | UI manipulation, info leak |
| OSC 52 query (?) | Clipboard read | **STRIP** | **BLOCK** | Clipboard theft |
| OSC 52 store | Clipboard write | Size limit | Focus gate, notify | Paste injection |
| DCS (general) | Device control | **STRIP** | N/A | Various |
| DECRQSS | Status query | **STRIP** | N/A | PTY input injection |
| OSC 7 | CWD report | Validate | App logic | Path injection |
| Bracketed paste markers in paste content | Paste | N/A | **STRIP from paste** | Command injection |
| CSI cursor/erase/SGR | Core terminal | Pass through | xterm.js handles | None |
| CSI mouse modes | Mouse tracking | Pass through | xterm.js handles | None |
| CSI ? 2004 h/l | Bracketed paste mode | Pass through | xterm.js handles | None |
| CSI ? 2026 h/l | Synchronized output | Pass through | App logic | None |

---

## 6. Implementation Priority

**M1 (must-have for first terminal):**
1. Rust-side filter using `vte` crate: strip CSI 20t/21t, window manipulation CSI t, DCS
2. xterm.js OSC 52 handler with query blocking, size limits, focus gating
3. xterm.js OSC 7 handler with protocol validation
4. Paste de-fanging (strip bracketed paste markers from clipboard content)

**M2 (before public release):**
1. OSC 52 user notification (toast when clipboard written by escape sequence)
2. Configuration: per-terminal OSC 52 policy (always / focus / never)
3. Audit against updated CVE database
4. Shell integration OSC handler (Yord-custom protocol)

---

*References:*
- CVE-2022-45872 (iTerm2 RCE via title reporting)
- CVE-2003-0063 (xterm title reporting injection)
- <https://marc.info/?l=bugtraq&m=104612710031920&w=2> (original title reporting advisory)
- Waveterm: `frontend/app/view/term/termwrap.ts`, `frontend/app/view/term/osc-handlers.ts`
- Zed: `crates/terminal/src/terminal.rs` (alacritty_terminal event processing)
- Wezterm: `term/src/terminalstate/performer.rs`, `config/src/config.rs`
- Wezterm escape sequences doc: `docs/escape-sequences.md`
