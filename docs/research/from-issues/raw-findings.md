# Raw Issue Findings

> Based on analysis of 32,297 issues from 9 reference project issue trackers.
> See [docs/refs/README.md](../../refs/README.md) for rules.

**Total relevant issues found: 592**

## PTY / Process Issues

*192 issues*

### canopy

- **[#23](https://github.com/The-Banana-Standard/canopy/issues/23)** (closed) — close_terminal should explicitly kill child process before removing `bug`, `good first issue`
  Keywords: pty, orphan, sigterm, sigkill
  Summary: `close_terminal` in `terminal.rs` (lines 282-287) merely removes the terminal from the HashMap, relying on the PTY master's `Drop` impl to signal t...

### helix

- **[#2029](https://github.com/helix-editor/helix/issues/2029)** (closed) — Typescript Language Sever crashes on 'Move to a new file` `C-bug`, `A-language-server`
  Keywords: zombie
  Summary: When a code action 'Move to a new file` is available, if selected it crashes with:

- **[#4068](https://github.com/helix-editor/helix/issues/4068)** (closed) — hx creates zombie process on exit `C-bug`, `A-language-server`, `A-helix-term`
  Keywords: zombie
  Summary: When closing typescript project, new instance of tsserver.js is not killed properly and still running in the background.

- **[#5424](https://github.com/helix-editor/helix/issues/5424)** (closed) — Clipboard contents lost when `hx` used as scrollback buffer editor within zellij session `C-bug`, `A-helix-term`
  Keywords: process group, setsid
  Summary: I am using [zellij](https://zellij.dev/) as a terminal multiplexer/session manager (e.g. like `tmux`), below I will provide info related to [zellij...

- **[#9776](https://github.com/helix-editor/helix/issues/9776)** (closed) — Aggressive editor.scrolloff causes infinite loop `C-bug`
  Keywords: sigterm, sigkill
  Summary: Setting `editor.scrolloff` to a high number causes helix to freeze in an infinite loop in some (large?) files.

- **[#13853](https://github.com/helix-editor/helix/issues/13853)** (open) — helix does not run setsid on LSPs `C-bug`
  Keywords: setsid
  Summary: This manifests to me while using kitty. In my `.config/kitty/kitty.conf` I have this line:

### tauri-plugin-pty

- **[#2](https://github.com/Tnze/tauri-plugin-pty/issues/2)** (closed) — multiple instance spawned
  Keywords: pty
  Summary: Hey this is a great plugin and thank you for it.

- **[#5](https://github.com/Tnze/tauri-plugin-pty/issues/5)** (closed) — Calling kill() does not terminate the indefinite read loop
  Keywords: pty
  Summary: Why do the `resize`, `write` and `kill` methods in the JS API call the wrapper function `_init` which calls `plugin:pty\|spawn`? See https://github....

### tauri-terminal

- **[#6](https://github.com/marc2332/tauri-terminal/issues/6)** (open) — Bug: not responding pty request from ssh tunnel
  Keywords: pty
  Summary: I connect to an backend SSH tunnel, which runs a `Bubbletea` cli program through a PTY, however the output cannot refresh.

### waveterm

- **[#98](https://github.com/wavetermdev/waveterm/issues/98)** (closed) — Connect failed because remote server doesn't have GLIBC dependency `duplicate`, `legacy`
  Keywords: pty
  Summary: **Describe the bug**

- **[#103](https://github.com/wavetermdev/waveterm/issues/103)** (closed) — "Error, could not connect." for a reason I am not being able to identify. `bug`, `legacy`
  Keywords: pty
  Summary: I'm on openSUSE, as soon as I launch Wave, I get "Error, could not connect.", following by this output messages:

- **[#179](https://github.com/wavetermdev/waveterm/issues/179)** (closed) — Cannot connect to ssh server with banner `bug`, `legacy`
  Keywords: pty
  Summary: **Describe the bug**

- **[#264](https://github.com/wavetermdev/waveterm/issues/264)** (closed) — error connecting to remote: invalid packet received from mshell client `bug`, `legacy`
  Keywords: pty
  Summary: **Describe the bug**

- **[#560](https://github.com/wavetermdev/waveterm/issues/560)** (closed) — no prompt Fail on startup after /connect local or /reset `bug`, `legacy`
  Keywords: pty
  Summary: It does not give a prompt and reports

- **[#630](https://github.com/wavetermdev/waveterm/issues/630)** (closed) — error reading from pty: read /dev/ptmx: input/output error wave> initialized connection state (shell:zsh) `legacy`
  Keywords: pty
  Summary: Whenever I open a new tab in workspace this error pop up

- **[#2768](https://github.com/wavetermdev/waveterm/issues/2768)** (closed) — [Bug]: ping/ICMP fails to LAN hosts on macOS Wi-Fi (raw socket privilege issue) `bug`, `triage`
  Keywords: pty
  Summary: Current Behavior

- **[#3077](https://github.com/wavetermdev/waveterm/issues/3077)** (open) — [Bug]: Touchpad scroll triggers shell history navigation instead of scrolling terminal output
  Keywords: pty
  Summary: **Version:** 0.14.3 (202603122332)

### wezterm

- **[#27](https://github.com/wezterm/wezterm/issues/27)** (closed) — publish the pty module to crates.io
  Keywords: pty
  Summary: Would it be possible to publish the pty module as a crate that can be reused by other programs?

- **[#35](https://github.com/wezterm/wezterm/issues/35)** (closed) — Document how to configure using winpty
  Keywords: pty
  Summary: * Obtain `https://github.com/rprichard/winpty/releases/download/0.4.3/winpty-0.4.3-msys2-2.7.0-x64.tar.gz`

- **[#65](https://github.com/wezterm/wezterm/issues/65)** (closed) — Non-responsive on yes
  Keywords: pty
  Summary: Did something not work the way you expected?

- **[#166](https://github.com/wezterm/wezterm/issues/166)** (closed) — Release new version of portable pty
  Keywords: pty
  Summary: The last release was 7 months back.

- **[#187](https://github.com/wezterm/wezterm/issues/187)** (closed) — pty examples fail on OSX Catalina `bug`, `macOS`
  Keywords: pty
  Summary: Describe the bug

- **[#272](https://github.com/wezterm/wezterm/issues/272)** (closed) — OpenGL failed to initialize `bug`, `Linux`
  Keywords: zombie
  Summary: Describe the bug

- **[#463](https://github.com/wezterm/wezterm/issues/463)** (closed) — Portable Pty hangs on Windows `bug`
  Keywords: pty
  Summary: Describe the bug

- **[#917](https://github.com/wezterm/wezterm/issues/917)** (closed) — Each ssh domain session is by default a mirror clone, can't just close only one's own client session `bug`, `multiplexer`
  Keywords: orphan
  Summary: Describe the bug

- **[#1015](https://github.com/wezterm/wezterm/issues/1015)** (closed) — Wezterm error setting env vars on ssh session: [Session(-22)] Unable to complete request for channel-setenv `bug`
  Keywords: pty
  Summary: Describe the bug

- **[#1038](https://github.com/wezterm/wezterm/issues/1038)** (closed) — Error building wezterm from arch aur `bug`
  Keywords: pty
  Summary: Build Environment (please complete the following information):

- **[#1197](https://github.com/wezterm/wezterm/issues/1197)** (closed) — Closing a pane when connected via `wezterm ssh` isn't working correctly `bug`, `fixed-in-nightly`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#1274](https://github.com/wezterm/wezterm/issues/1274)** (closed) — [portable-pty] no `CommandBuilder::env_clear` function `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#1389](https://github.com/wezterm/wezterm/issues/1389)** (closed) — portable_pty doesn't compile on windows `bug`
  Keywords: pty
  Summary: Build Environment:

- **[#1396](https://github.com/wezterm/wezterm/issues/1396)** (open) — portable-pty example not working properly on Windows
  Keywords: pty
  Summary: (I omittedthe bug report form as this issue concerns `portable-pty`)

- **[#1567](https://github.com/wezterm/wezterm/issues/1567)** (closed) — Using wezterm to ssh stuck at `Authenticating...` `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#2035](https://github.com/wezterm/wezterm/issues/2035)** (closed) — Stretched font (SourceCodePro) `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#2092](https://github.com/wezterm/wezterm/issues/2092)** (closed) — Ssh fails because working directory from local domain doesn't exist `bug`, `fixed-in-nightly`
  Keywords: process group
  Summary: What Operating System(s) are you seeing this problem on?

- **[#2466](https://github.com/wezterm/wezterm/issues/2466)** (closed) — "too many open files" on long running `wezterm ssh` session `bug`, `fixed-in-nightly`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#2980](https://github.com/wezterm/wezterm/issues/2980)** (closed) — Expand environment variables on Windows `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3020](https://github.com/wezterm/wezterm/issues/3020)** (closed) — Error making wezterm-git when updating from AUR `bug`
  Keywords: pty
  Summary: Build Environment (please complete the following information):

- **[#3108](https://github.com/wezterm/wezterm/issues/3108)** (closed) — portable-pty release with `get_termios`
  Keywords: pty
  Summary: Hey there! I've been using and appreciating the `portable-pty` crate and just ran into a use case where I'd like to get access to the termios pty h...

- **[#3223](https://github.com/wezterm/wezterm/issues/3223)** (open) — Wezterm panic for `tmux -CC a` command via ssh `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3345](https://github.com/wezterm/wezterm/issues/3345)** (closed) — KeeAgent on Windows ssh authentication socket of AF_UNIX type is blocked by wezterm `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4205](https://github.com/wezterm/wezterm/issues/4205)** (open) — portable-pty removes custom paths from PATH on windows `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4206](https://github.com/wezterm/wezterm/issues/4206)** (open) — portable-pty windows failed to write to pty `docs`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4718](https://github.com/wezterm/wezterm/issues/4718)** (closed) — portable-pty doesn't report EOF when the proccess is exit, which cause reader thread never returned. `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4784](https://github.com/wezterm/wezterm/issues/4784)** (closed) — [Windows] using portable_pty causes terminal to be cleared while trying to stream to stdout `bug`, `Windows`, `PR-welcome`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4973](https://github.com/wezterm/wezterm/issues/4973)** (closed) — Yes I manage to get it working creating a separated thread, and use pipes to share information.
  Keywords: pty
  Summary: Yes I manage to get it working creating a separated thread, and use pipes to share information.

- **[#5101](https://github.com/wezterm/wezterm/issues/5101)** (closed) — WezTerm killing tmux window when closed `bug`
  Keywords: sighup
  Summary: What Operating System(s) are you seeing this problem on?

- **[#5107](https://github.com/wezterm/wezterm/issues/5107)** (open) — portable-pty: clone_killer gives an invalid handle on windows `bug`
  Keywords: pty
  Summary: portable-pty 0.8.1

- **[#5110](https://github.com/wezterm/wezterm/issues/5110)** (closed) — SSH multiplexing not working between Windows and remote server `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#5479](https://github.com/wezterm/wezterm/issues/5479)** (closed) — ProxyCommand zombie process `bug`, `fixed-in-nightly`
  Keywords: zombie
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6435](https://github.com/wezterm/wezterm/issues/6435)** (open) — Wezterm occasionally interprets single clicks as double clicks (or triple lcicks `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6481](https://github.com/wezterm/wezterm/issues/6481)** (closed) — Please push portable-pty to cargo asap `bug`
  Keywords: pty
  Summary: With the retirement of the serial crate and the inclusion of the serial2 crate, please puch the new version to crates.io asap. thanks.

- **[#6499](https://github.com/wezterm/wezterm/issues/6499)** (open) — `wezterm start --` or `wezterm -e` panic if `PATHEXT` has an empty entry (`;;`) `bug`, `Windows`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6594](https://github.com/wezterm/wezterm/issues/6594)** (closed) — Cannot build from source on Linux with GCC 15 `bug`, `waiting-on-op`, `Stale`
  Keywords: pty
  Summary: Build Environment (please complete the following information):

- **[#6783](https://github.com/wezterm/wezterm/issues/6783)** (open) — portable-pty 0.9.0 doesn't work on windows `bug`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6799](https://github.com/wezterm/wezterm/issues/6799)** (closed) — MUX - version mismatch? `bug`, `needs:triage`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6833](https://github.com/wezterm/wezterm/issues/6833)** (closed) — WezTerm Freezes on Launch with 100% CPU Usage on macOS 15.4 `bug`, `macOS`, `needs:triage`
  Keywords: sigterm
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6855](https://github.com/wezterm/wezterm/issues/6855)** (open) — Wezterm uses a lot of CPU on Linux `bug`, `needs:triage`
  Keywords: zombie
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6946](https://github.com/wezterm/wezterm/issues/6946)** (open) — tauri uses portable_pty with cmd window `bug`, `needs:triage`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7014](https://github.com/wezterm/wezterm/issues/7014)** (closed) — SSH multiplexing stops working after some time `bug`, `needs:triage`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7025](https://github.com/wezterm/wezterm/issues/7025)** (open) — Portably pty fails to launch commands on windows `bug`, `needs:triage`
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7661](https://github.com/wezterm/wezterm/issues/7661)** (open) — tmux CC domain detach deadlocks entire GUI when last window is closed via Ctrl+D
  Keywords: pty
  Summary: What Operating System(s) are you seeing this problem on?

### zed

- **[#6092](https://github.com/zed-industries/zed/issues/6092)** (closed) — Project Diagnostics errors result in Zombie indicator `bug`, `area:diagnostics`
  Keywords: zombie
  Summary: Check for existing issues

- **[#6159](https://github.com/zed-industries/zed/issues/6159)** (closed) — feed command output to zed does not work as expected; zombie zed in-memory/cache reopens instead `bug`
  Keywords: zombie
  Summary: Check for existing issues

- **[#6265](https://github.com/zed-industries/zed/issues/6265)** (closed) — Crash at start `bug`, `crash`
  Keywords: pty
  Summary: Check for existing issues

- **[#7457](https://github.com/zed-industries/zed/issues/7457)** (closed) — shortening window pane causes app crash and prompts macos to force kill `bug`, `crash`
  Keywords: pty
  Summary: Check for existing issues

- **[#9482](https://github.com/zed-industries/zed/issues/9482)** (closed) — Orphan Node Process After Zed Quit (And Sometimes Node 100+% CPU Usage) `bug`, `area:language server`
  Keywords: orphan
  Summary: Check for existing issues

- **[#15138](https://github.com/zed-industries/zed/issues/15138)** (open) — Dependency Dashboard `ignore top-ranking issues`
  Keywords: pty
  Summary: This issue lists Renovate updates and detected dependencies. Read the [Dependency Dashboard](https://docs.renovatebot.com/key-concepts/dashboard/) ...

- **[#15170](https://github.com/zed-industries/zed/issues/15170)** (closed) — Cannot open zed, tmp-socket opens with no GUI `bug`, `platform:linux`, `support`
  Keywords: sigterm
  Summary: Check for existing issues

- **[#19784](https://github.com/zed-industries/zed/issues/19784)** (closed) — Remote development: Daemon does not respond to signals and cannot be shut down `bug`, `platform:linux`, `platform:remote`
  Keywords: sigterm, sigkill
  Summary: Check for existing issues

- **[#20223](https://github.com/zed-industries/zed/issues/20223)** (closed) — Task zed-editor process blocked for over 120 seconds, causing app freeze `bug`, `crash`, `platform:linux`
  Keywords: zombie
  Summary: Check for existing issues

- **[#24963](https://github.com/zed-industries/zed/issues/24963)** (closed) — Random crash while copying function from diff
  Keywords: pty
  Summary: <!-- Please insert a one line summary of the issue below -->

- **[#25843](https://github.com/zed-industries/zed/issues/25843)** (open) — Zed uses 100% CPU when files outside the editor are changed `area:performance`, `meta:regression`, `state:needs repro`
  Keywords: pty
  Summary: <!-- Please insert a one line summary of the issue below -->

- **[#30132](https://github.com/zed-industries/zed/issues/30132)** (closed) — ctrl-backspace in terminal does not delete last word (unlike base Alacritty) `area:integrations/terminal`, `area:controls/keybinds`, `state:reproducible`
  Keywords: pty
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#33144](https://github.com/zed-industries/zed/issues/33144)** (closed) — In vim mode, :r!echo hi adds extra junk to the document after hi.
  Keywords: process group
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#34932](https://github.com/zed-industries/zed/issues/34932)** (closed) — Opening another project (in same window) while running a task doesn't kill process
  Keywords: orphan
  Summary: Zed doesn't terminate a process running in a terminal pane when the project window is switched to another project.

- **[#35489](https://github.com/zed-industries/zed/issues/35489)** (closed) — Cannot stop agent-driven terminal action `area:integrations/terminal`, `area:ai/agent thread`
  Keywords: sigkill
  Summary: As a user, I should be able to stop a terminal action started by an agent.

- **[#35759](https://github.com/zed-industries/zed/issues/35759)** (open) — "Failed to load environment variables" error with Bash `area:integrations/terminal`, `state:needs repro`, `area:integrations/environment`
  Keywords: process group
  Summary: I'm getting an error like this every time I start up Zed:

- **[#35797](https://github.com/zed-industries/zed/issues/35797)** (open) — Terminal Hanging on terraform init after 73421006d5f0826ebe40f876e9a92a38929d1931 `area:performance`, `area:integrations/terminal`, `state:needs repro`
  Keywords: sigkill
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#36934](https://github.com/zed-industries/zed/issues/36934)** (closed) — Windows: Unable open or terminate lingering zombie process after closing and reopening Zed a couple of times. `platform:windows`
  Keywords: zombie
  Summary: After a reboot, Zed installed via scoop will happily function and show no signs of distress for the first 5-10 times it is opened and closed - ulti...

- **[#37458](https://github.com/zed-industries/zed/issues/37458)** (closed) — AI: orphaned gemini-cli processes running after Zed quit (volta) `area:ai`, `area:ai/gemini`, `area:ai/acp`
  Keywords: orphan
  Summary: <!-- Please insert a one line summary of the issue below -->

- **[#39890](https://github.com/zed-industries/zed/issues/39890)** (closed) — Zed Crashes Frequently `area:languages/json`, `frequency:uncommon`, `priority:P2`
  Keywords: process group
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#42365](https://github.com/zed-industries/zed/issues/42365)** (closed) — Terminal resize does not propagate when developing remotely
  Keywords: pty
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#43114](https://github.com/zed-industries/zed/issues/43114)** (closed) — remote (linux to linux) crashes 100% of the time starting with release 0.213.3 `meta:regression`, `state:needs info`, `platform:remote`
  Keywords: zombie
  Summary: Can not longer use zed remote with this release.

- **[#43864](https://github.com/zed-industries/zed/issues/43864)** (open) — Language servers not properly shutting down when closing Zed `area:language server/server failure`, `state:reproducible`, `frequency:common`
  Keywords: zombie
  Summary: Reproduction steps

- **[#44572](https://github.com/zed-industries/zed/issues/44572)** (closed) — Zed update 0.216.0 breaks rust analyzer `area:languages/rust`, `state:needs research`, `stale`
  Keywords: sigkill
  Summary: Reproduction steps

- **[#45211](https://github.com/zed-industries/zed/issues/45211)** (closed) — [Bug] Gemini ACP extension leaks multiple zombie processes causing memory exhaustion (Not reproducible with Codex/Claude) `state:needs info`, `area:ai/gemini`, `frequency:uncommon`
  Keywords: zombie, sigterm
  Summary: Reproduction steps

- **[#45339](https://github.com/zed-industries/zed/issues/45339)** (open) — Codex Agent Loading failed on windows `platform:windows`, `state:needs repro`, `area:ai/openai compatible`
  Keywords: process group
  Summary: Reproduction steps

- **[#45665](https://github.com/zed-industries/zed/issues/45665)** (closed) — Zed Contributing docs should mention "out of memory" issue, suggest appropriate fix `state:needs repro`
  Keywords: sigkill
  Summary: Reproduction steps

- **[#46474](https://github.com/zed-industries/zed/issues/46474)** (open) — Zed does not kill Node.js processes upon exit, creating zombie processes `state:reproducible`, `area:ai/acp`, `area:performance/memory leak`
  Keywords: zombie
  Summary: Reproduction steps

- **[#46551](https://github.com/zed-industries/zed/issues/46551)** (open) — Zed uses terminal.shell settings outside of the terminal, breaking Codex, Claude, and Gemini ACP Agents `area:ai`, `state:reproducible`, `frequency:common`
  Keywords: process group
  Summary: > [!WARNING]

- **[#47045](https://github.com/zed-industries/zed/issues/47045)** (closed) — git stash crach `area:integrations/git`, `state:needs repro`, `frequency:common`
  Keywords: pty
  Summary: Reproduction steps

- **[#47412](https://github.com/zed-industries/zed/issues/47412)** (open) — Child processes not terminated when integrated terminal closes `area:integrations/terminal`, `state:needs repro`, `area:performance/memory leak`
  Keywords: orphan, sigterm, sighup
  Summary: Description

- **[#47455](https://github.com/zed-industries/zed/issues/47455)** (closed) — Claude agent (claude-code-acp) zombie processes locking files on Windows `area:ai/anthropic`, `frequency:common`, `priority:P2`
  Keywords: zombie
  Summary: Reproduction steps

- **[#47543](https://github.com/zed-industries/zed/issues/47543)** (open) — Crash and process hang when opening a database migrated by a newer Zed version with no diagnostics shown `area:diagnostics`, `state:reproducible`, `frequency:common`
  Keywords: zombie
  Summary: Reproduction steps

- **[#48355](https://github.com/zed-industries/zed/issues/48355)** (closed) — Devcontainer integrated terminal prints raw ANSI escape sequences for arrow keys (^[[A, ^[[B, …) `area:integrations/terminal`, `frequency:uncommon`, `priority:P2`
  Keywords: pty
  Summary: Reproduction steps

- **[#48692](https://github.com/zed-industries/zed/issues/48692)** (closed) — Zed crashes on launch on macOS Monterey (x86_64) due to Mach port guard violation `platform:macOS`
  Keywords: sigkill
  Summary: Reproduction steps

- **[#48722](https://github.com/zed-industries/zed/issues/48722)** (closed) — Claude agent (claude-code-acp) zombie node.exe processes persist after Zed exit on Windows (regression) `platform:windows`, `meta:duplicate`, `area:ai/acp`
  Keywords: orphan, zombie
  Summary: Check for existing issues

- **[#49423](https://github.com/zed-industries/zed/issues/49423)** (closed) — Zed crashes on multiple windows as tabs with remote ssh connections `area:editor`, `meta:duplicate`, `frequency:common`
  Keywords: pty
  Summary: Reproduction steps

- **[#51058](https://github.com/zed-industries/zed/issues/51058)** (closed) — Zed UI hangs after long idle in Codex/ACP agent chat; agent continues in background while ACP hits reconnect errors `state:needs repro`, `area:ai/agent thread`, `frequency:uncommon`
  Keywords: zombie
  Summary: Reproduction steps

- **[#51155](https://github.com/zed-industries/zed/issues/51155)** (closed) — Claude agent (claude-agent-acp) spawns orphaned processes that persist indefinitely on macOS `state:needs triage`
  Keywords: orphan
  Summary: Reproduction steps

- **[#51361](https://github.com/zed-industries/zed/issues/51361)** (open) — Session Titles are not editable `state:reproducible`, `area:ai/agent thread`, `area:ai/acp`
  Keywords: sigterm, sigkill
  Summary: Reproduction steps

### zellij

- **[#39](https://github.com/zellij-org/zellij/issues/39)** (closed) — Create a --debug flag that would help communicate compatibility issues
  Keywords: pty
  Summary: When used, this flag would log all STDOUT bytes received on the pty of each pane to a separate log file. This will help debug compatibility problem...

- **[#176](https://github.com/zellij-org/zellij/issues/176)** (open) — Bug: opening a new tab is a little slow `help wanted`
  Keywords: pty
  Summary: When we open a tab, zellij waits until a new pty starts and only then opens the tab. This feels slow to the user.

- **[#315](https://github.com/zellij-org/zellij/issues/315)** (closed) — Opening lots of panes breaks zelliji and terminal `suspected bug`
  Keywords: sigkill
  Summary: zellij 0.5.0, on Arch Linux, Fish shell, KDE Konsole

- **[#417](https://github.com/zellij-org/zellij/issues/417)** (closed) — Doesn't start when used by multiple users `suspected bug`
  Keywords: sigterm, sigkill
  Summary: Description

- **[#509](https://github.com/zellij-org/zellij/issues/509)** (closed) — CPU usage when idle `suspected bug`
  Keywords: pty
  Summary: version: 0.10.0

- **[#518](https://github.com/zellij-org/zellij/issues/518)** (closed) — Zombie child processes are not reaped on pane/tab exit `suspected bug`, `help wanted`
  Keywords: zombie
  Summary: Observed: On macOS 11 and ArchLinux, exiting panes or tabs within a `zellij` session produces zombie child processes that the parent does not reap ...

- **[#525](https://github.com/zellij-org/zellij/issues/525)** (closed) — High memory usage and latency when a program produces output too quickly `suspected bug`
  Keywords: pty
  Summary: version `0.12.0`

- **[#685](https://github.com/zellij-org/zellij/issues/685)** (closed) — Opening too many frames (spawm alt+N) `suspected bug`
  Keywords: pty
  Summary: Error: thread 'pty' panicked at 'called `Option::unwrap()` on a `None` value': zellij-server/src/pty.rs:288

- **[#702](https://github.com/zellij-org/zellij/issues/702)** (closed) — [build] zellij 0.16.0 failed to build on linux `suspected bug`, `build`
  Keywords: sigkill
  Summary: Trying to [upgrade zellij into 0.16.0](https://github.com/Homebrew/homebrew-core/pull/84320), but run into some build as below:

- **[#727](https://github.com/zellij-org/zellij/issues/727)** (closed) — Working directory inheritence isn't quite working
  Keywords: pty
  Summary: I just noticed that when switching between tabs and then splitting into a new pane the current working directory is inherited from the previous tab...

- **[#979](https://github.com/zellij-org/zellij/issues/979)** (closed) — Server pty_thread panicked at failure to spawn for new pane `suspected bug`
  Keywords: pty
  Summary: Zellij should not crash, if it cant spawn another instance.

- **[#1000](https://github.com/zellij-org/zellij/issues/1000)** (closed) — Issue running zellij x86_64-unknown-linux-musl on Alpine Linux `suspected bug`
  Keywords: pty
  Summary: and thanks :100: for the awesome application!

- **[#1286](https://github.com/zellij-org/zellij/issues/1286)** (closed) — Zombie Process on Tab/Pane Close `suspected bug`
  Keywords: zombie
  Summary: I'm seeing zombie processes on tab/pane close.  Found out about it through vim open buffers on reopening a previously closed tab that had the same ...

- **[#1326](https://github.com/zellij-org/zellij/issues/1326)** (closed) — Client process lingers around after ssh disconnect `suspected bug`
  Keywords: sigterm, sighup
  Summary: This problem was already noticed in #1029 and now moved here.

- **[#1414](https://github.com/zellij-org/zellij/issues/1414)** (closed) — Zellij does not exit on reboot/shutdown `suspected bug`
  Keywords: sigkill
  Summary: **Current Behavior**

- **[#1421](https://github.com/zellij-org/zellij/issues/1421)** (closed) — Server doesn't exit anymore when closing last pane
  Keywords: pty
  Summary: #1383 introduced this bug. @tlinford

- **[#1672](https://github.com/zellij-org/zellij/issues/1672)** (closed) — Strider Panics Opening File when Editor not Set `suspected bug`, `duplicate`
  Keywords: pty
  Summary: I just started trying out zellij today. I'm liking it generally, but ran into a repeatable crash in the strider plugin.

- **[#1722](https://github.com/zellij-org/zellij/issues/1722)** (closed) — Could not find the SHELL variable: NotPresent `suspected bug`
  Keywords: pty
  Summary: **Basic information**

- **[#1747](https://github.com/zellij-org/zellij/issues/1747)** (open) — I'm not seeing zellij detach on a SIGHUP `suspected bug`
  Keywords: sighup
  Summary: Should this work? If I start a zellij session and my ssh connection to a remote is terminated for whatever reason (such as my closing my laptop lid...

- **[#1751](https://github.com/zellij-org/zellij/issues/1751)** (closed) — Thread 'pty' panicked `suspected bug`
  Keywords: pty
  Summary: As requested by the error message

- **[#1866](https://github.com/zellij-org/zellij/issues/1866)** (open) — build: macos: cargo: `failed to run custom build command for log v0.4.17`
  Keywords: sigkill
  Summary: _I couldn't decide whether it's bug or becuase of my environment so that I created this as a _blank_ issue._

- **[#1943](https://github.com/zellij-org/zellij/issues/1943)** (open) — SHELL=sh zellij is broken
  Keywords: pty
  Summary: This worked previously:

- **[#1949](https://github.com/zellij-org/zellij/issues/1949)** (closed) — Killing client crashes server
  Keywords: pty, sigkill
  Summary: Killing an attached client brings the server down reliably. I thought we had fixed this?

- **[#1985](https://github.com/zellij-org/zellij/issues/1985)** (open) — zellij server pty crash due to failure on sending message to screen `suspected bug`
  Keywords: pty
  Summary: Behavior observed: `Crash`

- **[#1993](https://github.com/zellij-org/zellij/issues/1993)** (closed) — Server crashed when client tty was closed
  Keywords: pty
  Summary: - I had a zellij session running on my server

- **[#2006](https://github.com/zellij-org/zellij/issues/2006)** (closed) — Thread 'pty' panicked: Cannot find shell zsh `suspected bug`
  Keywords: pty
  Summary: Please attach the files that were created in `/tmp/zellij-1000/zellij-log/` to the extent you are comfortable with.

- **[#2084](https://github.com/zellij-org/zellij/issues/2084)** (open) — wasm plugin causes freeze with 100% cpu load `suspected bug`
  Keywords: pty
  Summary: The issue seems to be sporadic, but luckily/at least I have some logs. Overall does zellij appear to be stable for me.

- **[#2207](https://github.com/zellij-org/zellij/issues/2207)** (open) — Cursor doesn't display(disapear) in blank lines or blank (space) characters in Neovim `suspected bug`
  Keywords: pty
  Summary: Cursor doesn't display in neovim in blanc lines or space character in normal or insert mode

- **[#2248](https://github.com/zellij-org/zellij/issues/2248)** (closed) — Panic on closing a tab `suspected bug`
  Keywords: pty
  Summary: **Basic information**

- **[#2329](https://github.com/zellij-org/zellij/issues/2329)** (closed) — killing ssh connection may crash remote zellij server with "failed to enable mouse mode" `suspected bug`
  Keywords: pty
  Summary: **Basic information**

- **[#2376](https://github.com/zellij-org/zellij/issues/2376)** (open) — Panic occured in a plugin (strider) when opening a folder containing a file with some special chars in its name `suspected bug`
  Keywords: pty
  Summary: **Basic information**

- **[#2435](https://github.com/zellij-org/zellij/issues/2435)** (open) — perf: resonably slow text selection `suspected bug`
  Keywords: pty
  Summary: Compared to bare Alacritty, selecting text in Zellij is noticeably slower. Additionally, when attempting to select text, the cursor does not switch...

- **[#2481](https://github.com/zellij-org/zellij/issues/2481)** (open) — crash: Thread 'screen' panicked `suspected bug`
  Keywords: pty
  Summary: **Basic information**

- **[#2520](https://github.com/zellij-org/zellij/issues/2520)** (open) — `neovim` turns to blank screen when add new pane `suspected bug`
  Keywords: pty
  Summary: **Basic information**

- **[#2553](https://github.com/zellij-org/zellij/issues/2553)** (open) — zellij does not start anymore `suspected bug`
  Keywords: pty
  Summary: Please attach the files that were created in `/tmp/zellij-1000/zellij-log/` to the extent you are comfortable with.

- **[#2563](https://github.com/zellij-org/zellij/issues/2563)** (closed) — [BUG] Neovim + Rust analyzer don't show errors inside a Zellij session `suspected bug`
  Keywords: pty, sighup
  Summary: This doesn't seem to happen with typescripts/pythons LSPs

- **[#2604](https://github.com/zellij-org/zellij/issues/2604)** (open) — Panic while closing main terminal pane `suspected bug`
  Keywords: pty
  Summary: Thank you for taking the time to file this issue! Please follow the instructions and fill in the missing parts below the instructions, if it is mea...

- **[#2613](https://github.com/zellij-org/zellij/issues/2613)** (open) — zellij exit immediately after starting on slurm compute node `suspected bug`
  Keywords: pty
  Summary: I'm using zellij on a compute cluster managed by slurm. When I login to the slurm login node, zellij runs perfectly. But after I login to a compute...

- **[#2640](https://github.com/zellij-org/zellij/issues/2640)** (closed) — No shell displayed, cannot create new pane \| failed to open PTY for command `suspected bug`
  Keywords: pty
  Summary: so when i open zellij there is no shell or prompt or anything, cannot enter any commands, just a blinking cursor, here is a photo of how it looks like

- **[#2648](https://github.com/zellij-org/zellij/issues/2648)** (open) — ZelliJ crashes when create new tab immediately after starting (happen only once) `suspected bug`
  Keywords: pty
  Summary: **Basic information**

- **[#2782](https://github.com/zellij-org/zellij/issues/2782)** (open) — Error on creating new session `suspected bug`
  Keywords: pty
  Summary: Log (`--debug`):

- **[#2794](https://github.com/zellij-org/zellij/issues/2794)** (closed) — Flickering in Nushell when used with Zellij and WezTerm `suspected bug`
  Keywords: pty
  Summary: When e.g. displaying completions over a certain length, the output form Nushell will flicker under Zellij when used in WezTerm.

- **[#2796](https://github.com/zellij-org/zellij/issues/2796)** (open) — strider plugin causes 100% cpu load `suspected bug`
  Keywords: zombie
  Summary: Every time I start Zellij with the strider plugin it causes 100% CPU load.

- **[#2823](https://github.com/zellij-org/zellij/issues/2823)** (open) — Closing a Plugin with worker leads to errors in the logs `suspected bug`
  Keywords: pty
  Summary: **Description**

- **[#2828](https://github.com/zellij-org/zellij/issues/2828)** (closed) — Epilepsy warning: Flickering due to segmented rendering with apps which refresh constantly such as top or pw-top `suspected bug`
  Keywords: pty
  Summary: `zellij --version`: 0.38.2

- **[#2834](https://github.com/zellij-org/zellij/issues/2834)** (closed) — Zellij crashed `suspected bug`
  Keywords: pty
  Summary: **Basic information**

- **[#2847](https://github.com/zellij-org/zellij/issues/2847)** (closed) — Plugin error on start
  Keywords: pty
  Summary: 👋🏼 After reinstalling homebrew for arm64 and reinstalling zellij, I see this error every time I start it:

- **[#2914](https://github.com/zellij-org/zellij/issues/2914)** (closed) — panic in at thread `async-std/runtime` at `rkyv-0.7.42` crate in `src/impls/core/mod.rs:267:17` `suspected bug`
  Keywords: pty
  Summary: Please attach the files that were created in `/tmp/zellij-1000/zellij-log/` to the extent you are comfortable with.

- **[#2931](https://github.com/zellij-org/zellij/issues/2931)** (open) — Unable to use custom config on 0.39.0 `suspected bug`
  Keywords: sigterm, sighup
  Summary: **Basic information**

- **[#3002](https://github.com/zellij-org/zellij/issues/3002)** (closed) — Terminal Shortcuts not working `suspected bug`
  Keywords: sigterm, sighup
  Summary: Some common terminal shortcuts do not work

- **[#3024](https://github.com/zellij-org/zellij/issues/3024)** (closed) — Zombie process after closing a pane command `suspected bug`
  Keywords: zombie, sigterm, sighup
  Summary: 2. Issues with the Zellij UI / behavior

- **[#3117](https://github.com/zellij-org/zellij/issues/3117)** (closed) — Custom keybinds from layout forgotten after detach `suspected bug`
  Keywords: pty
  Summary: <!-- Please choose the relevant section, follow the instructions and delete the other sections:

- **[#3151](https://github.com/zellij-org/zellij/issues/3151)** (open) — Attaching resurrection session crashes `suspected bug`
  Keywords: pty
  Summary: 1. Attaching an exited session crashes on resurrection

- **[#3165](https://github.com/zellij-org/zellij/issues/3165)** (closed) — Zellij not considering window startup mode `suspected bug`
  Keywords: pty
  Summary: I got an issue with Zellij and Alacritty, where Zellij does not appear to consider the startup mode there. I use maximized.

- **[#3272](https://github.com/zellij-org/zellij/issues/3272)** (closed) — Plugin API: open_terminal fails when default_shell is not set `suspected bug`
  Keywords: pty
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#3280](https://github.com/zellij-org/zellij/issues/3280)** (closed) — Cannot resurrect after upgrading to 0.40.0 `suspected bug`
  Keywords: pty
  Summary: Issue description

- **[#3311](https://github.com/zellij-org/zellij/issues/3311)** (closed) — The `zellij action rename-tab` command results in empty tab in zellij 0.40.0 `suspected bug`
  Keywords: pty
  Summary: Issue description

- **[#3372](https://github.com/zellij-org/zellij/issues/3372)** (open) — Sixel support broken since v0.40.0 `suspected bug`
  Keywords: pty
  Summary: **Basic information**

- **[#3419](https://github.com/zellij-org/zellij/issues/3419)** (open) — Unable to enter password in GPG password prompt (pinentry)
  Keywords: pty
  Summary: Attempting to enter GPG password in the terminal (with the `pinentry` program) results in graphical errors (text pasting in the wrong place) in Zel...

- **[#3433](https://github.com/zellij-org/zellij/issues/3433)** (open) — Feedback:  `on_force_close` and `session_serialization` default values
  Keywords: sigterm
  Summary: Hello, I recently switched from tmux to Zellij and I've been loving it! I just wanted to give some feedback on the default values of `on_force_clos...

- **[#3448](https://github.com/zellij-org/zellij/issues/3448)** (closed) — zellij run command in-place with nushell causes crash
  Keywords: pty
  Summary: Basic information

- **[#3507](https://github.com/zellij-org/zellij/issues/3507)** (closed) — `zellij run -i` crashes in Nushell
  Keywords: pty
  Summary: Issue description

- **[#3572](https://github.com/zellij-org/zellij/issues/3572)** (open) — Panic when loading an existing session.
  Keywords: pty
  Summary: Issue description

- **[#3609](https://github.com/zellij-org/zellij/issues/3609)** (open) — Tabs are missing after resurrection
  Keywords: sigterm, sighup
  Summary: 2. Issues with the Zellij behavior

- **[#3762](https://github.com/zellij-org/zellij/issues/3762)** (open) — Sigstop on editing scrollback freezes
  Keywords: sigkill
  Summary: `zellij --version`: for both 0.41.1 & 0.40.1

- **[#3778](https://github.com/zellij-org/zellij/issues/3778)** (closed) — Delete key no longer works
  Keywords: pty
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#3783](https://github.com/zellij-org/zellij/issues/3783)** (open) — CJK font causes blank character blocks when switching buffers in neovim
  Keywords: pty
  Summary: Basic information

- **[#3829](https://github.com/zellij-org/zellij/issues/3829)** (closed) — aarch64: 'async-std/runtime' panicked while or after WASM compilation
  Keywords: pty
  Summary: <!-- Please choose the relevant section, follow the instructions and delete the other sections:

- **[#3918](https://github.com/zellij-org/zellij/issues/3918)** (open) — Zombie sessions refuse to die
  Keywords: zombie
  Summary: Issue description

- **[#3939](https://github.com/zellij-org/zellij/issues/3939)** (open) — Missing lock symbol in password prompt
  Keywords: pty
  Summary: 1. Graphical issue inside a terminal pane (e.g. something does not look as it should)

- **[#3976](https://github.com/zellij-org/zellij/issues/3976)** (closed) — Panic in `grid.rs`
  Keywords: pty
  Summary: Did scroll and got a panic (unwrapping a `None`) in `grid.rs`. I don't know what happened and I'm not able to reproduce it.

- **[#4054](https://github.com/zellij-org/zellij/issues/4054)** (closed) — Copy command leaves a lot of zombie processes
  Keywords: zombie
  Summary: **Basic information**

- **[#4109](https://github.com/zellij-org/zellij/issues/4109)** (open) — fix(session-manager): Unable to use a custom layout if file contains extension (`.kbl`)
  Keywords: pty
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#4126](https://github.com/zellij-org/zellij/issues/4126)** (open) — hide session name and word "Zellij" on left
  Keywords: sigterm, sighup
  Summary: Hi! Just simple question: I use compact view, I want completely hide session name and word "Zellij" on left of it. How to do that ? Ideally also "T...

- **[#4134](https://github.com/zellij-org/zellij/issues/4134)** (open) — `kill $fish_pid` is not a decent way to exit fish
  Keywords: sigterm
  Summary: Issues with the Zellij UI / behavior / crash

- **[#4219](https://github.com/zellij-org/zellij/issues/4219)** (open) — Zellij not working in termux yet it used to work sometime back
  Keywords: pty
  Summary: `cat /data/data/com.termux/files/usr/tmp/zellij-*/zellij-log/zellij.log

- **[#4256](https://github.com/zellij-org/zellij/issues/4256)** (open) — Custom layout unexpectedly overrides default keybindings
  Keywords: sigterm, sighup
  Summary: Issue description

- **[#4389](https://github.com/zellij-org/zellij/issues/4389)** (open) — zellij session stuck at 100% CPU on pane resize sometimes
  Keywords: sigkill
  Summary: <!-- Please choose the relevant section, follow the instructions and delete the other sections:

- **[#4413](https://github.com/zellij-org/zellij/issues/4413)** (open) — Session resurrection completely non-functional
  Keywords: pty
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#4473](https://github.com/zellij-org/zellij/issues/4473)** (open) — zellij-server/src/panes/grid.rs:1639:31: removal index (is 13) should be < len (is 13)
  Keywords: pty
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#4490](https://github.com/zellij-org/zellij/issues/4490)** (open) — Zellij crash after Detach  within a session and then with the 'zellij attach <that session> '
  Keywords: pty
  Summary: Issue with the Zellij crash

- **[#4496](https://github.com/zellij-org/zellij/issues/4496)** (open) — Soft limit for number of files bumped up in new panes (caused by sysinfo crate)
  Keywords: pty
  Summary: <!-- Please choose the relevant section, follow the instructions and delete the other sections:

- **[#4511](https://github.com/zellij-org/zellij/issues/4511)** (open) — Feature: make timeouts for external commands tweakable
  Keywords: sigkill
  Summary: Note on new features: while Zellij tries to be as user friendly as possible, it must also be friendly to its maintainers. We take great care in con...

- **[#4558](https://github.com/zellij-org/zellij/issues/4558)** (open) — The colours used on a fresh install render Zellij unusable
  Keywords: sigkill
  Summary: **Steps to reproduce:**

- **[#4619](https://github.com/zellij-org/zellij/issues/4619)** (open) — zellij server crashes every few days
  Keywords: pty
  Summary: Roughly every 2-4 days zellij server crashes, killing all the running processes inside. The parent terminal displays a message `Received empty mess...

- **[#4658](https://github.com/zellij-org/zellij/issues/4658)** (open) — terminal colors not resetted properly?
  Keywords: pty
  Summary: 1. Graphical issue inside a terminal pane (eg. something does not look as it should)

- **[#4702](https://github.com/zellij-org/zellij/issues/4702)** (open) — Zellij fails to show some accented characters, probably due to encoding `suspected bug`
  Keywords: pty
  Summary: 1. Graphical issue inside a terminal pane (eg. something does not look as it should or as it looks outside of Zellij)

- **[#4756](https://github.com/zellij-org/zellij/issues/4756)** (open) — Web/WebSocket clients never receive Sixel image data
  Keywords: pty
  Summary: Web/WebSocket clients (and any non-native client) never receive Sixel image data from Zellij. The Sixel rendering infrastructure works correctly, b...

- **[#4797](https://github.com/zellij-org/zellij/issues/4797)** (open) — Screen thread does unnecessary render work when no clients are connected
  Keywords: pty
  Summary: Issue description

- **[#4805](https://github.com/zellij-org/zellij/issues/4805)** (open) — Zellij server panics and exits after client disconnection (VSCode Remote SSH terminal) `suspected bug`
  Keywords: pty
  Summary: Environment

- **[#4870](https://github.com/zellij-org/zellij/issues/4870)** (open) — [windows] : unstable over openning same session over ssh and wezterm
  Keywords: pty
  Summary: Reproduction

## Session / Restore Issues

*104 issues*

### waveterm

- **[#1662](https://github.com/wavetermdev/waveterm/issues/1662)** (open) — [Bug]: Terminal folder you were in gets forgotten when reloading app `bug`, `triage`
  Keywords: restore
  Summary: Current Behavior

- **[#3073](https://github.com/wavetermdev/waveterm/issues/3073)** (open) — Reconnect overlay opens new window and blocks input
  Keywords: restore
  Summary: Description

### wezterm

- **[#1319](https://github.com/wezterm/wezterm/issues/1319)** (closed) — Empty window when automatically started after restart on macOS `bug`, `macOS`, `waiting-on-op`
  Keywords: restore
  Summary: What Operating System(s) are you seeing this problem on?

- **[#5901](https://github.com/wezterm/wezterm/issues/5901)** (closed) — Maximized Wayland Wezterm renders as windowed after switching Gnome workspace `bug`
  Keywords: restore
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7315](https://github.com/wezterm/wezterm/issues/7315)** (open) — glow markdown rendering got broken when you scroll up/down in Wezterm `bug`, `needs:triage`
  Keywords: resurrect
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7437](https://github.com/wezterm/wezterm/issues/7437)** (open) — Kitty Keyboard doesn't restore properly `bug`, `needs:triage`
  Keywords: restore
  Summary: What Operating System(s) are you seeing this problem on?

### zed

- **[#4651](https://github.com/zed-industries/zed/issues/4651)** (closed) — Option to allow restore of windows `area:settings`
  Keywords: restore
  Summary: Check for existing issues

- **[#5297](https://github.com/zed-industries/zed/issues/5297)** (closed) — Deleting a file while having project search or multibuffer interface will restore it `bug`, `area:workspace`, `area:multi-buffer`
  Keywords: restore
  Summary: **Describe the bug**

- **[#5919](https://github.com/zed-industries/zed/issues/5919)** (closed) — Open with + Zed session restore = same project opened twice `bug`, `area:workspace`
  Keywords: restore
  Summary: Check for existing issues

- **[#7371](https://github.com/zed-industries/zed/issues/7371)** (open) — Restarts should be non-destructive on workspace restore/reload `area:workspace`, `area:serialization`
  Keywords: restore
  Summary: Check for existing issues

- **[#9101](https://github.com/zed-industries/zed/issues/9101)** (closed) — Saving a multibuffer, with an excerpt to a deleted file, resurrects the file `bug`, `area:workspace`, `area:multi-buffer`
  Keywords: resurrect
  Summary: Check for existing issues

- **[#15551](https://github.com/zed-industries/zed/issues/15551)** (closed) — Restore entire workspace state on reload `meta:duplicate`
  Keywords: restore
  Summary: Check for existing issues

- **[#15959](https://github.com/zed-industries/zed/issues/15959)** (open) — Linux Wayland: "restore_on_startup" == "last_session" does not restore session `area:workspace`, `platform:linux`, `platform:linux/wayland`
  Keywords: restore
  Summary: Description

- **[#16843](https://github.com/zed-industries/zed/issues/16843)** (closed) — Serialize and restore unsent agent messages `area:serialization`, `area:ai/agent thread`, `never stale`
  Keywords: restore
  Summary: If you have unsent messages in the agent panel or in a text thread when you restart zed this content is lost.

- **[#17051](https://github.com/zed-industries/zed/issues/17051)** (closed) — Restore unsaved buffers when no projectis open `meta:duplicate`
  Keywords: restore
  Summary: Check for existing issues

- **[#19103](https://github.com/zed-industries/zed/issues/19103)** (closed) — Edited files not in worktree not restored properly `bug`, `area:workspace`, `area:settings`
  Keywords: restore
  Summary: Check for existing issues

- **[#19496](https://github.com/zed-industries/zed/issues/19496)** (closed) — Workspace Contexts: Save & restore open files & panel states `area:workspace`, `area:serialization`, `area:ui/panel`
  Keywords: restore
  Summary: Check for existing issues

- **[#19997](https://github.com/zed-industries/zed/issues/19997)** (closed) — @ symbol can't be inserted in certain keyboard layouts `bug`, `area:editor`, `area:internationalization`
  Keywords: restore
  Summary: Check for existing issues

- **[#20775](https://github.com/zed-industries/zed/issues/20775)** (open) — Externally deleted files do not have deletion indication after workspace restore `area:workspace`, `area:project panel`, `state:reproducible`
  Keywords: restore
  Summary: Check for existing issues

- **[#23054](https://github.com/zed-industries/zed/issues/23054)** (closed) — Autosave session without save files `meta:duplicate`
  Keywords: restore
  Summary: Check for existing issues

- **[#23288](https://github.com/zed-industries/zed/issues/23288)** (open) — Implement graceful recovery for GPU device loss in Zed `platform:linux`, `area:gpui`, `never stale`
  Keywords: restore
  Summary: Check for existing issues

- **[#24120](https://github.com/zed-industries/zed/issues/24120)** (open) — Ruff format on save doesn't work during zed session until I push changes to server settings `area:editor`, `area:languages/python`, `area:settings`
  Keywords: restore
  Summary: Ruff format on save doesn't work during a zed remote session until I push changes to server settings. In my case, I'm simply deleting all the conte...

- **[#25022](https://github.com/zed-industries/zed/issues/25022)** (open) — Zed stopped restoring sessions after an update `meta:regression`, `area:serialization`, `frequency:always`
  Keywords: restore
  Summary: Zed stopped restoring sessions after an update

- **[#25095](https://github.com/zed-industries/zed/issues/25095)** (closed) — Unable to open or save files after initial edit of any file `platform:linux`
  Keywords: restore
  Summary: I am experiencing a bug in version 0.173.11 where after opening one or more rust files, editing any breaks ability to open others or save that edit...

- **[#30272](https://github.com/zed-industries/zed/issues/30272)** (open) — Frequent Disconnects / Doesn't automatically reconnect `state:reproducible`, `area:auth`, `frequency:common`
  Keywords: restore
  Summary: My team is using Zed, and we are frequently disconnected at random. Zed does not auto-reconnect, nor does it have an option to reconnect that I can...

- **[#30917](https://github.com/zed-industries/zed/issues/30917)** (closed) — Diff indicators not restored when reopening remote project
  Keywords: restore
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#31721](https://github.com/zed-industries/zed/issues/31721)** (closed) — on macOS, restoring Zed window on startup doesn't properly reload state `area:workspace`, `area:ai`, `stale`
  Keywords: restore
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#31917](https://github.com/zed-industries/zed/issues/31917)** (closed) — Can't always use redo after save `meta:regression`, `area:editor`, `state:reproducible`
  Keywords: restore
  Summary: In certain circumstances, saving a buffer incorrectly prevents `redo` after save.

- **[#33191](https://github.com/zed-industries/zed/issues/33191)** (closed) — Windows: Zed window briefly turns black (Intel Graphics) `platform:windows`, `state:reproducible`, `graphics:intel`
  Keywords: restore
  Summary: When restoring the app from a minimized state, the entire app window **sometimes** turns **black** for a moment before rendering properly. The dura...

- **[#33262](https://github.com/zed-industries/zed/issues/33262)** (closed) — AI: "Reject All" is not working as expected - "Restore Checkpoint" sometimes missing and has no function at all `area:ai`
  Keywords: restore
  Summary: As demonstrated in the following video, I noticed the following issues:

- **[#36137](https://github.com/zed-industries/zed/issues/36137)** (open) — Toggling the AI dock does not restore focus to the last active panel `area:controls/keybinds`, `area:accessibility`, `state:needs repro`
  Keywords: restore
  Summary: **Summary:**

- **[#36324](https://github.com/zed-industries/zed/issues/36324)** (open) — Restore closed tab does not work for Zed views (e.g. Extensions page) `area:workspace`, `area:serialization`, `area:extensions/infrastructure`
  Keywords: restore
  Summary: Medium UX bug (inconsistency)

- **[#37348](https://github.com/zed-industries/zed/issues/37348)** (closed) — Windows: Cannot restore opened windows/projects/files after update `platform:windows`
  Keywords: restore
  Summary: Cannot restore opened windows/projects/files after update.

- **[#38946](https://github.com/zed-industries/zed/issues/38946)** (closed) — Zed doesn't reopen previously open files and directories upon restart `area:debugger`
  Keywords: restore
  Summary: When Zed is closed and then reopened, it does not restore the files and directories that were open before the shutdown.

- **[#39820](https://github.com/zed-industries/zed/issues/39820)** (closed) — Zed restart still loses data
  Keywords: restore
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#40314](https://github.com/zed-industries/zed/issues/40314)** (closed) — Windows: Session restore not working when settings UI is open `platform:windows`, `area:settings/ui`
  Keywords: restore
  Summary: After opening the Settings UI, if you click to quit Zed directly, the settings window fails to close along with it; when you restart Zed, all previ...

- **[#40615](https://github.com/zed-industries/zed/issues/40615)** (closed) — Windows: `workspace: reload` opens an empty window `platform:windows`
  Keywords: restore
  Summary: <!-- Please insert a one-line summary of the issue below -->

- **[#40666](https://github.com/zed-industries/zed/issues/40666)** (closed) — Windows: the startup window location and size wrong, and no way to restore to correct one easily `platform:windows`
  Keywords: restore
  Summary: when launch, i CANNOT see the full zed window in one screen, no way to maxmize.

- **[#41039](https://github.com/zed-industries/zed/issues/41039)** (closed) — random files flickering on save
  Keywords: restore
  Summary: <details><summary>demo video</summary>

- **[#41307](https://github.com/zed-industries/zed/issues/41307)** (closed) — Restore Unsaved Buffers option does not work with windows without projects/worktrees `area:settings`
  Keywords: restore
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#41474](https://github.com/zed-industries/zed/issues/41474)** (closed) — [Windows 10] Pinned tab lost on restart `platform:windows`
  Keywords: restore
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#41475](https://github.com/zed-industries/zed/issues/41475)** (closed) — Restore on startup not working `state:needs info`, `area:serialization`, `state:needs repro`
  Keywords: restore
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#43067](https://github.com/zed-industries/zed/issues/43067)** (closed) — On macOS, Zed restart clumps all instances onto one Desktop `platform:macOS`
  Keywords: restore
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#43541](https://github.com/zed-industries/zed/issues/43541)** (closed) — Zed does not restore the inline chat history on close `area:ai`, `stale`, `state:reproducible`
  Keywords: restore
  Summary: Reproduction steps

- **[#43952](https://github.com/zed-industries/zed/issues/43952)** (closed) — Zed doesn't restore window position upon restart `platform:linux`, `platform:linux/wayland`, `area:ui/scaling`
  Keywords: restore
  Summary: Reproduction steps

- **[#44042](https://github.com/zed-industries/zed/issues/44042)** (open) — Tabs of windows aren't working well upon restart `area:editor`, `state:reproducible`, `frequency:common`
  Keywords: restore
  Summary: Reproduction steps

- **[#47075](https://github.com/zed-industries/zed/issues/47075)** (open) — Restore to Launchpad not taking effect `area:workspace`, `platform:linux`, `state:needs repro`
  Keywords: restore
  Summary: Reproduction steps

- **[#47117](https://github.com/zed-industries/zed/issues/47117)** (closed) — Cannot locate the “Restore Checkpoint” button in the Agent Panel `area:ai/agent thread`, `frequency:common`, `priority:P2`
  Keywords: restore
  Summary: Reproduction steps

- **[#47189](https://github.com/zed-industries/zed/issues/47189)** (open) — macos system window tab restore session not working properly `area:workspace`, `area:ui/tabs`, `state:reproducible`
  Keywords: restore
  Summary: Reproduction steps

- **[#47907](https://github.com/zed-industries/zed/issues/47907)** (open) — Restore from checkpoint button no longer appearing in Agent thread when using Dev Containers `state:reproducible`, `area:ai/agent thread`, `frequency:common`
  Keywords: restore
  Summary: Reproduction steps

- **[#48255](https://github.com/zed-industries/zed/issues/48255)** (closed) — Unsaved buffers are not restored without a workspace `state:needs info`, `state:needs triage`
  Keywords: restore
  Summary: Reproduction steps

- **[#48468](https://github.com/zed-industries/zed/issues/48468)** (open) — "Restore Unsaved Buffers" is not respected when closing the window on macOS `area:serialization`, `state:reproducible`, `frequency:common`
  Keywords: restore
  Summary: Reproduction steps

- **[#48498](https://github.com/zed-industries/zed/issues/48498)** (open) — Zed (Vim mode): gd "Go to Definition" stops working after some time - only fixed by restarting `area:parity/vim`, `area:navigation`, `state:needs repro`
  Keywords: restore
  Summary: Reproduction steps

- **[#49461](https://github.com/zed-industries/zed/issues/49461)** (closed) — Unsaved buffer in new window ist lost, when open recent project `area:serialization`, `meta:duplicate`, `frequency:common`
  Keywords: restore
  Summary: Reproduction steps

- **[#49971](https://github.com/zed-industries/zed/issues/49971)** (closed) — launchpad restore on startup not working in linux `state:needs triage`
  Keywords: restore
  Summary: Reproduction steps

- **[#50070](https://github.com/zed-industries/zed/issues/50070)** (closed) — Zed throws away non-project workspaces with unsaved buffers without prompting `area:workspace`, `area:serialization`, `community champion`
  Keywords: restore
  Summary: Reproduction steps

- **[#50409](https://github.com/zed-industries/zed/issues/50409)** (open) — workspace: Create new WorkspaceId(<id>) when there is an actual project `area:workspace`, `state:reproducible`, `frequency:common`
  Keywords: restore
  Summary: Reproduction steps

- **[#50910](https://github.com/zed-industries/zed/issues/50910)** (open) — [Codex] Fatal error when restoring historical session via WSL Remote: cwd path forward slashes converted to backslashes `state:needs repro`, `platform:windows/wsl`, `frequency:common`
  Keywords: restore
  Summary: Reproduction steps

- **[#50994](https://github.com/zed-industries/zed/issues/50994)** (closed) — Workspace restoration fails when no folder is opened (untitled buffer only) `area:workspace`, `state:reproducible`, `frequency:common`
  Keywords: restore
  Summary: Check for existing issues

- **[#52038](https://github.com/zed-industries/zed/issues/52038)** (open) — Restoring Unsaved Session No Longer Works `state:needs triage`
  Keywords: restore
  Summary: Reproduction steps

- **[#52040](https://github.com/zed-industries/zed/issues/52040)** (closed) — "Failed to restore 1 workspace. Check logs for details." message isn't accompanied by useful log information `state:needs triage`
  Keywords: restore
  Summary: Reproduction steps

- **[#52069](https://github.com/zed-industries/zed/issues/52069)** (open) — After the factory droid acp resume session, the mode and model options no longer appear. `state:needs triage`
  Keywords: restore
  Summary: Reproduction steps

### zellij

- **[#1133](https://github.com/zellij-org/zellij/issues/1133)** (open) — Add current directory and command to Plugin API `plugin system`
  Keywords: resurrect
  Summary: This would be the equivalent of `pane_current_path` and `pane_current_command` in tmux.

- **[#1255](https://github.com/zellij-org/zellij/issues/1255)** (open) — Sessions appear lost when zellij is upgraded
  Keywords: restore
  Summary: When zellij is upgraded, for example by the system package manager, any sessions running under the previous version are not visible anymore to new ...

- **[#1524](https://github.com/zellij-org/zellij/issues/1524)** (closed) — Save and auto restore all layout of open tabs and panes after restart `duplicate`
  Keywords: restore, resurrect
  Summary: I currently use tmux with plugins `tmux-resurrect` and `tmux-continuum` and I would be able to switch to zellij if there is a way to achieve simila...

- **[#3012](https://github.com/zellij-org/zellij/issues/3012)** (closed) — Serialize current session layout (expose machinery of resurrection)
  Keywords: resurrect
  Summary: Not sure if this is already doable, but Zellij can already `ressurrect` an old dead session, but the way this is done is black magic for the user. ...

- **[#3057](https://github.com/zellij-org/zellij/issues/3057)** (open) — NO_COLOR or --no-color options
  Keywords: resurrect
  Summary: `zellij` does not seem to support the `NO_COLOR` environment variable, or have a a `--no-color` option, which makes it hard to script the CLI as fo...

- **[#3151](https://github.com/zellij-org/zellij/issues/3151)** (open) — Attaching resurrection session crashes `suspected bug`
  Keywords: resurrect
  Summary: 1. Attaching an exited session crashes on resurrection

- **[#3238](https://github.com/zellij-org/zellij/issues/3238)** (open) — Keymaps/configs don't restore on attach `suspected bug`
  Keywords: restore
  Summary: **Basic information**

- **[#3280](https://github.com/zellij-org/zellij/issues/3280)** (closed) — Cannot resurrect after upgrading to 0.40.0 `suspected bug`
  Keywords: resurrect
  Summary: Issue description

- **[#3371](https://github.com/zellij-org/zellij/issues/3371)** (open) — Sessions not found after updating to a new Zellij version `suspected bug`
  Keywords: resurrect
  Summary: <!-- Please choose the relevant section, follow the instructions and delete the other sections:

- **[#3374](https://github.com/zellij-org/zellij/issues/3374)** (closed) — tabs / panes lost their working directories on session restore after a reboot `suspected bug`
  Keywords: restore
  Summary: Issue description

- **[#3375](https://github.com/zellij-org/zellij/issues/3375)** (open) — Enhance Session Selection with Prefix Matching and Fuzzy Finder
  Keywords: resurrect
  Summary: Instead of typing `zellij a session-name`, wouldn't it be better to use `zellij a se` (using just the first few characters)? If there are multiple ...

- **[#3408](https://github.com/zellij-org/zellij/issues/3408)** (open) — Support hiding session-manager with `Esc` on `New Session` tab
  Keywords: resurrect
  Summary: the session-manager supports `Esc` to hide itself except in `New Session` tab, I've checked the code don't see it calls `hide_self` like others han...

- **[#3410](https://github.com/zellij-org/zellij/issues/3410)** (open) — bug: session-manager session doesn't get cleaned up when resurrecting a session
  Keywords: resurrect
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#3464](https://github.com/zellij-org/zellij/issues/3464)** (closed) — session_serialization stops working
  Keywords: restore
  Summary: `session_serialization` worked somehow before. But suddenly, it stops working.  I can NOT restore exited session (actually, there is no more exited...

- **[#3521](https://github.com/zellij-org/zellij/issues/3521)** (open) — bug: plugin config conflicts between layout and plugin alias on resurrection
  Keywords: resurrect
  Summary: **Basic information**

- **[#3609](https://github.com/zellij-org/zellij/issues/3609)** (open) — Tabs are missing after resurrection
  Keywords: resurrect
  Summary: 2. Issues with the Zellij behavior

- **[#3619](https://github.com/zellij-org/zellij/issues/3619)** (open) — ctrl+alt+punctuation not properly send to the underlying program
  Keywords: resurrect
  Summary: In short `ctrl+alt+key` is not properly send to application when using zelij.

- **[#3731](https://github.com/zellij-org/zellij/issues/3731)** (open) — Session already exists, but is dead
  Keywords: resurrect
  Summary: version: zellij 0.41.1

- **[#3861](https://github.com/zellij-org/zellij/issues/3861)** (open) — Improve Session Resurrection
  Keywords: resurrect
  Summary: In my usual workflow, I source a shell file that exports certain necessary environment variables to initialize the project correctly. After which, ...

- **[#3896](https://github.com/zellij-org/zellij/issues/3896)** (open) — Session with only the default tab/pane does not get serialized correctly or does not serialize at all
  Keywords: resurrect
  Summary: I am very thankful for the new resurrection feature. Unfortunately, there seems to be a bug when trying to get resurrection to work correctly when ...

- **[#3952](https://github.com/zellij-org/zellij/issues/3952)** (open) — Resume command written to session-layout does not match layout command
  Keywords: resurrect
  Summary: Issue description

- **[#4030](https://github.com/zellij-org/zellij/issues/4030)** (open) — Interrupting with Ctrl+z resurrected session hangs
  Keywords: resurrect
  Summary: if I start resurrect a session, then pressing ctrl+z to go to the bash line hangs the terminal.

- **[#4058](https://github.com/zellij-org/zellij/issues/4058)** (open) — default_tab_template pane commands get a comfirmation question before execution on resurrected sessions
  Keywords: resurrect
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#4074](https://github.com/zellij-org/zellij/issues/4074)** (open) — Theme improvement opportunities or muted, zen mode Zellij
  Keywords: resurrect
  Summary: While adopting the new [themes](https://zellij.dev/documentation/themes.html) released in [0.42.0](https://github.com/zellij-org/zellij/releases/ta...

- **[#4130](https://github.com/zellij-org/zellij/issues/4130)** (open) — Feature Request: Session Serialization on Detach / Kill
  Keywords: resurrect
  Summary: Basically what the title says.

- **[#4132](https://github.com/zellij-org/zellij/issues/4132)** (open) — Not able to exit remote ssh session after reboot of host machine
  Keywords: resurrect
  Summary: Issues with the Zellij behavior

- **[#4156](https://github.com/zellij-org/zellij/issues/4156)** (open) — Custom settings not kept after resurrection
  Keywords: resurrect
  Summary: Issue description

- **[#4281](https://github.com/zellij-org/zellij/issues/4281)** (open) — Restore `puma` rails server
  Keywords: restore
  Summary: Restore `puma` rails server

- **[#4304](https://github.com/zellij-org/zellij/issues/4304)** (closed) — Feature Request: Restore the Last Focused Tab When Reattaching to a Session
  Keywords: restore
  Summary: Note on new features: while Zellij tries to be as user friendly as possible, it must also be friendly to its maintainers. We take great care in con...

- **[#4316](https://github.com/zellij-org/zellij/issues/4316)** (open) — Session "Created" times listed in Resurrect Session menu in Session Manager are incorrect
  Keywords: resurrect
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#4318](https://github.com/zellij-org/zellij/issues/4318)** (open) — Thread 'signal_listener' panicked when trying to attach to session
  Keywords: resurrect
  Summary: Previously:

- **[#4350](https://github.com/zellij-org/zellij/issues/4350)** (closed) — Setting session_serialization to false does not work after upgrading to 0.43.0
  Keywords: resurrect
  Summary: After upgrading zellij from 0.42.2 to 0.43.0, the option [session_serialization](https://zellij.dev/documentation/options.html#session_serializatio...

- **[#4358](https://github.com/zellij-org/zellij/issues/4358)** (closed) — BUG: `session_serialization false` still shows exited sessions in `zellij ls` after 0.43.0
  Keywords: resurrect
  Summary: **Description:**

- **[#4413](https://github.com/zellij-org/zellij/issues/4413)** (open) — Session resurrection completely non-functional
  Keywords: resurrect
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#4441](https://github.com/zellij-org/zellij/issues/4441)** (open) — [minor] session-manager plugin cannot be closed when on "New session" tab
  Keywords: resurrect
  Summary: It's just a minor inconvenience, but it sometimes make it more cumbersome to rapidly switch between sessions:

- **[#4451](https://github.com/zellij-org/zellij/issues/4451)** (closed) — Zellij session cannot be deleted; file reappears after deletion.
  Keywords: resurrect
  Summary: **Basic information**

- **[#4485](https://github.com/zellij-org/zellij/issues/4485)** (open) — Resurrect process with a shell instance
  Keywords: resurrect
  Summary: right now zellij can only `run` the command or `drop to the shell` and what I want is for zellij to spawn a shell then run the command.

- **[#4641](https://github.com/zellij-org/zellij/issues/4641)** (open) — How to auto-remove exited sessions?
  Keywords: resurrect
  Summary: I figured out that resurrection is broken atm (at least this is my impression from the many issues here reported about it, and also it didn't work ...

- **[#4673](https://github.com/zellij-org/zellij/issues/4673)** (open) — Zellij creates illegal nested stacked panes
  Keywords: resurrect
  Summary: 2. Issues with the Zellij UI / behavior / crash

- **[#4794](https://github.com/zellij-org/zellij/issues/4794)** (open) — Web client rendering freezes while session stays alive `suspected bug`
  Keywords: restore
  Summary: Issues with the Zellij UI / behavior / crash

- **[#4805](https://github.com/zellij-org/zellij/issues/4805)** (open) — Zellij server panics and exits after client disconnection (VSCode Remote SSH terminal) `suspected bug`
  Keywords: resurrect
  Summary: Environment

- **[#4873](https://github.com/zellij-org/zellij/issues/4873)** (open) — Session resurrection captures wrong command when parent has multiple child processes `suspected bug`
  Keywords: resurrect
  Summary: Issue description

## Terminal Rendering Issues

*95 issues*

### helix

- **[#247](https://github.com/helix-editor/helix/issues/247)** (closed) — Support OSC52 `C-enhancement`, `A-helix-term`
  Keywords: osc 52
  Summary: This would provide a way to copy text to the system clipboard even if you are using it through SSH without X forwarding.  It's also a much cleaner ...

- **[#3954](https://github.com/helix-editor/helix/issues/3954)** (open) — Nix multiline strings don't have syntax highlighting on multiline strings `C-bug`, `A-tree-sitter`, `upstream`
  Keywords: escape sequence
  Summary: When entered mutliline strings in nix:

- **[#4594](https://github.com/helix-editor/helix/issues/4594)** (closed) — Do not use `termux-clipboard-set` on Termux `C-enhancement`
  Keywords: osc 52
  Summary: Currently Helix uses `termux-clipboard-set` for yanking in Termux on Android:

- **[#5913](https://github.com/helix-editor/helix/issues/5913)** (closed) — editor.statusline.separator not working `C-bug`
  Keywords: osc 52
  Summary: Setting the field has no effect.

- **[#6401](https://github.com/helix-editor/helix/issues/6401)** (closed) — clipboard-paste-* not working as intended `C-bug`
  Keywords: osc 52
  Summary: When I tried to copy text from out side back to helix, they just can paste the text, I have tried `space+p` or using all the command with prefix `:...

- **[#6606](https://github.com/helix-editor/helix/issues/6606)** (closed) — Languages.toml  -- git-commit Error `C-bug`
  Keywords: osc 52
  Summary: When opening Helix I get the following message when using the default languages.toml file

- **[#6888](https://github.com/helix-editor/helix/issues/6888)** (closed) — Add yank_joined command `C-enhancement`, `E-easy`, `A-helix-term`
  Keywords: osc 52
  Summary: Your enhancement may already be reported!

- **[#6958](https://github.com/helix-editor/helix/issues/6958)** (closed) — Syntax highlighting error for V language `C-bug`, `E-easy`, `A-tree-sitter`
  Keywords: osc 52
  Summary: So whenever i try to type anything in string literal like: "hello", after one letter syntax highlighting completly brokes. I have language set to v

- **[#7273](https://github.com/helix-editor/helix/issues/7273)** (open) — Helix crashes when editing files with Russian text `C-bug`, `A-tree-sitter`, `E-has-instructions`
  Keywords: osc 52
  Summary: Helix crashes when I edit a Markdown document that is part or all Cyrillic characters. An error is displayed in the terminal:

- **[#7818](https://github.com/helix-editor/helix/issues/7818)** (closed) — copy from ssh-tmux-helix into local system clipboard via osc52 `C-enhancement`
  Keywords: osc 52
  Summary: This is the use case

- **[#8288](https://github.com/helix-editor/helix/issues/8288)** (closed) — OOM crash when performing search on large input file `C-bug`
  Keywords: osc 52
  Summary: Opening a large text file, ~2GB with helix works perfectly fine.

- **[#8715](https://github.com/helix-editor/helix/issues/8715)** (open) — Space-p and Space-R not working inside Tmux
  Keywords: osc 52
  Summary: Discussed in https://github.com/helix-editor/helix/discussions/8470

- **[#9094](https://github.com/helix-editor/helix/issues/9094)** (closed) — Keybinding with C-/ doesn't work on Windows Terminal when SSH `C-bug`
  Keywords: osc 52
  Summary: Want to remap ctrl+/ to toggle comments.

- **[#10034](https://github.com/helix-editor/helix/issues/10034)** (closed) — Helix syntax highlighting doesn't work `C-bug`
  Keywords: osc 52
  Summary: I cloned the repo and built it using instructions from documentation. I fetched and built grammars, installed clangd and rust-analyzer, but I can't...

- **[#10570](https://github.com/helix-editor/helix/issues/10570)** (closed) — Toggling comments inside insert mode `C-bug`
  Keywords: osc 52
  Summary: I've been trying to improve my commenting workflow by making it accessible inside insert mode with this config:

- **[#11027](https://github.com/helix-editor/helix/issues/11027)** (closed) — Using `diff` in gutters twice makes Helix get stuck `C-bug`, `A-helix-term`
  Keywords: osc 52
  Summary: I tried to put `"diff"` twice in the `gutters` option, and after reloading the config, Helix got stuck - I had to kill it with -9. When I then try ...

- **[#11050](https://github.com/helix-editor/helix/issues/11050)** (open) — Enable OSC52 automatically if connected via SSH `C-enhancement`, `A-helix-term`
  Keywords: osc 52
  Summary: When connecting via ssh to a macOS host, helix still uses `pbcopy+pbpaste` as a clipboard provider instead of OSC52.

- **[#11402](https://github.com/helix-editor/helix/issues/11402)** (closed) — Theme problem after su -l USER `C-bug`
  Keywords: osc 52
  Summary: Helix window looks really bad when I login as different user and start it.

- **[#11640](https://github.com/helix-editor/helix/issues/11640)** (closed) — PERF: testing .git and .helix in the parents of $HOME within autofs `C-bug`
  Keywords: osc 52
  Summary: Thank you for creating Helix. I've had a pleasure getting familiar with it locally over the last week.

- **[#12058](https://github.com/helix-editor/helix/issues/12058)** (closed) — Hard Linked Files Break on Write `C-bug`, `R-duplicate`
  Keywords: osc 52
  Summary: When editting a file, consisting of multiple copies via file system hard links, when Helix writes out the changes to the file.  The hard links are ...

- **[#13818](https://github.com/helix-editor/helix/issues/13818)** (closed) — :insert-output breaks key handling (esc) in lazygit `C-bug`
  Keywords: escape sequence
  Summary: I grabbed the [recipes for running yazi and lazygit](https://github.com/helix-editor/helix/wiki/Recipes), but running either and then exiting helix...

- **[#15008](https://github.com/helix-editor/helix/issues/15008)** (open) — Poor Rendering Performance Over Slow SSH `C-bug`
  Keywords: escape sequence
  Summary: When running Helix on a remote machine over a slow and a bit laggy SSH connection, scrolling and other actions that call for redrawing/rerendering ...

- **[#15255](https://github.com/helix-editor/helix/issues/15255)** (closed) — insert-output breaks on using skim (sk) `C-bug`
  Keywords: escape sequence
  Summary: sk interactive view won't work in helix using `insert-output`, instead freezes and insert TUI elements as output when entering a key (such as escap...

- **[#15284](https://github.com/helix-editor/helix/issues/15284)** (open) — Terminal Escape Codes Leak to stdout During Helix Startup `C-bug`
  Keywords: escape sequence
  Summary: When starting Helix, terminal escape sequences are printed to the terminal:

### waveterm

- **[#116](https://github.com/wavetermdev/waveterm/issues/116)** (closed) — Bug with ANSI escape sequence `bug`, `legacy`
  Keywords: escape sequence
  Summary: - Erase screen sequence does not work

- **[#2787](https://github.com/wavetermdev/waveterm/issues/2787)** (open) — [Bug]: TUI animations scroll instead of animating in-place (missing xterm.js Synchronized Output support)
  Keywords: webgl, canvas renderer
  Summary: Description

- **[#2845](https://github.com/wavetermdev/waveterm/issues/2845)** (closed) — [Bug]: OSC 52 clipboard copying fails when window/block not focused `maintainer-interest`
  Keywords: osc 52
  Summary: Describe the bug

- **[#2895](https://github.com/wavetermdev/waveterm/issues/2895)** (open) — [Bug]: Terminals freeze for several minutes after macOS screen lock / sleep `need more info`
  Keywords: webgl
  Summary: After macOS locks the screen or the machine sleeps, all WaveTerm terminal blocks are frozen for **3–5 minutes** on wake before becoming responsive ...

### wezterm

- **[#489](https://github.com/wezterm/wezterm/issues/489)** (closed) — Toast Notifications
  Keywords: escape sequence
  Summary: This issue tracks the state of toast notification support.

- **[#510](https://github.com/wezterm/wezterm/issues/510)** (closed) — Powershell breaks cursor keys by setting DECCKM `bug`
  Keywords: escape sequence
  Summary: Describe the bug

- **[#545](https://github.com/wezterm/wezterm/issues/545)** (closed) — `Ctrl + Shift + Backspace`/`Ctrl + Shift + Delete` not working `bug`, `X11`, `fixed-in-nightly`
  Keywords: escape sequence
  Summary: Describe the bug

- **[#673](https://github.com/wezterm/wezterm/issues/673)** (closed) — Non-ASCII symbols mangled in window title `bug`
  Keywords: escape sequence
  Summary: Describe the bug

- **[#764](https://github.com/wezterm/wezterm/issues/764)** (closed) — OSC 52 does not work in SSH domain `bug`, `fixed-in-nightly`, `multiplexer`
  Keywords: osc 52
  Summary: Describe the bug

- **[#812](https://github.com/wezterm/wezterm/issues/812)** (closed) — Crashing due to unknown escape sequence `bug`
  Keywords: escape sequence
  Summary: Describe the bug

- **[#836](https://github.com/wezterm/wezterm/issues/836)** (closed) — OSC 52 does not work in splitted panes `bug`, `fixed-in-nightly`
  Keywords: osc 52
  Summary: Describe the bug

- **[#999](https://github.com/wezterm/wezterm/issues/999)** (closed) — Terminfo/tmux sync feature does not work `bug`
  Keywords: escape sequence
  Summary: Describe the bug

- **[#1296](https://github.com/wezterm/wezterm/issues/1296)** (closed) — Pasting sometimes does not work correctly `bug`
  Keywords: osc 52
  Summary: What Operating System(s) are you seeing this problem on?

- **[#1528](https://github.com/wezterm/wezterm/issues/1528)** (closed) — Unable to Set User Var When Connected to Unix Domain `bug`, `fixed-in-nightly`, `multiplexer`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#1593](https://github.com/wezterm/wezterm/issues/1593)** (closed) — Is there any way to change Wezterm window or tab title from bash? `docs`
  Keywords: escape sequence
  Summary: I'm using wezterm on Ubuntu 20.04 Mate. I have seen https://wezfurlong.org/wezterm/escape-sequences.html#operating-system-command-sequences - and a...

- **[#1662](https://github.com/wezterm/wezterm/issues/1662)** (closed) — OSC 52 does not work on the first opened tab `bug`
  Keywords: osc 52
  Summary: What Operating System(s) are you seeing this problem on?

- **[#1709](https://github.com/wezterm/wezterm/issues/1709)** (closed) — Background Color Handling `bug`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#1790](https://github.com/wezterm/wezterm/issues/1790)** (closed) — osc52 broken for panes spawned in the gui via the mux (`wezterm cli spawn`, `wezterm cli split-pane`) `bug`, `fixed-in-nightly`, `multiplexer`
  Keywords: osc 52
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3122](https://github.com/wezterm/wezterm/issues/3122)** (closed) — Input lag when typing when webgpu is enabled `bug`, `Wayland`, `fixed-in-nightly`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3258](https://github.com/wezterm/wezterm/issues/3258)** (closed) — iterm2 inline image protocol: Inconsistent behaviour when sizing animated WebP images `bug`, `fixed-in-nightly`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3266](https://github.com/wezterm/wezterm/issues/3266)** (closed) — iterm2 inline image protocol: the screen is scrolled whenever an image (with`doNotMoveCursor=0`) reaches the last line `bug`, `fixed-in-nightly`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3416](https://github.com/wezterm/wezterm/issues/3416)** (closed) — OSC52 clipboard text losing newlines in osx `bug`, `macOS`
  Keywords: osc 52
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3489](https://github.com/wezterm/wezterm/issues/3489)** (closed) — CONTROL + Backspace in Nushell calls for CONTROL + H `bug`
  Keywords: escape sequence
  Summary: I'm really sorry to bother you with such a minor issue, but I wanted to bring it to your attention in case it can be easily fixed. If it cannot, pl...

- **[#3837](https://github.com/wezterm/wezterm/issues/3837)** (closed) — Cannot type numbers in Kakoune when using programmer's Dvorak layout `bug`, `waiting-on-op`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3848](https://github.com/wezterm/wezterm/issues/3848)** (closed) — Copying using OSC 52 is Broken `bug`, `fixed-in-nightly`
  Keywords: osc 52
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4054](https://github.com/wezterm/wezterm/issues/4054)** (closed) — mouse reporting click event freeze wezterm `bug`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4501](https://github.com/wezterm/wezterm/issues/4501)** (closed) — 100% CPU usage on non visible window `bug`, `Wayland`, `waiting-on-op`
  Keywords: webgl
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4750](https://github.com/wezterm/wezterm/issues/4750)** (closed) — Improve resources consumption `waiting-on-op`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4791](https://github.com/wezterm/wezterm/issues/4791)** (closed) — osc52 stops working after a while `bug`, `waiting-on-op`
  Keywords: osc 52
  Summary: I am hitting again the same problem I already reported in https://github.com/wez/wezterm/issues/1790

- **[#4899](https://github.com/wezterm/wezterm/issues/4899)** (open) — wezterm cli set-window-title does not change the displayed window title `bug`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4913](https://github.com/wezterm/wezterm/issues/4913)** (closed) — win 11 wsl2 arch linux wezterm image preview on ranger wrongly positioned `bug`, `conpty`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#5476](https://github.com/wezterm/wezterm/issues/5476)** (closed) — OSC 9 escape code does not display a toast notification, but they are in the notifications list when checked. `bug`, `macOS`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#5756](https://github.com/wezterm/wezterm/issues/5756)** (closed) — `wezterm imgcat` fails when invoked from a "non-Wez" Windows terminal `bug`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#5793](https://github.com/wezterm/wezterm/issues/5793)** (open) — Clipboard in the terminal tab lags behind system's clipboard `bug`, `Wayland`
  Keywords: osc 52
  Summary: What Operating System(s) are you seeing this problem on?

- **[#5917](https://github.com/wezterm/wezterm/issues/5917)** (open) — wezterm with custom config disables OSC52 copy to clipboard `bug`
  Keywords: osc 52
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6291](https://github.com/wezterm/wezterm/issues/6291)** (closed) — Pasting issues in WezTerm with tmux on macOS Sequoia when using extended key settings `bug`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6994](https://github.com/wezterm/wezterm/issues/6994)** (open) — Ctrl+Shift+, not generating proper escape sequence with Kitty keyboard protocol `bug`, `keyboard`, `needs:triage`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7222](https://github.com/wezterm/wezterm/issues/7222)** (open) — kitty image not blending with text `bug`, `needs:triage`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7361](https://github.com/wezterm/wezterm/issues/7361)** (closed) — Wezterm does not properly interpret sequence "\E[;H" `bug`, `needs:triage`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7629](https://github.com/wezterm/wezterm/issues/7629)** (closed) — Cursor flickers when tmux status updates over ssh `bug`, `needs:triage`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7631](https://github.com/wezterm/wezterm/issues/7631)** (open) — Sending Kitty images via shared memory does not work on macOS `bug`, `needs:triage`
  Keywords: escape sequence
  Summary: What Operating System(s) are you seeing this problem on?

### zed

- **[#9742](https://github.com/zed-industries/zed/issues/9742)** (closed) — Github Copilot CLI - Error: could not prompt: unexpected escape sequence from terminal: ['\x1b' ']'] `bug`, `area:integrations/terminal`, `stale`
  Keywords: escape sequence
  Summary: Check for existing issues

- **[#19996](https://github.com/zed-industries/zed/issues/19996)** (open) — Support $PROMPT_COMMAND in terminal tab title `area:integrations/terminal`, `never stale`
  Keywords: escape sequence
  Summary: Describe the feature

- **[#20732](https://github.com/zed-industries/zed/issues/20732)** (closed) — v0.162.1: Vim mode escape sequence prefixes no longer inserted in Insert mode `area:parity/vim`, `meta:duplicate`
  Keywords: escape sequence
  Summary: Check for existing issues

- **[#22437](https://github.com/zed-industries/zed/issues/22437)** (closed) — Escape Sequence Issue in Zed Terminal with JetBrains Mono Nerd Font `bug`, `area:integrations/terminal`, `area:ui/font`
  Keywords: escape sequence
  Summary: Check for existing issues

- **[#31334](https://github.com/zed-industries/zed/issues/31334)** (closed) — Agent Panel: Quotes and file name left on the file after LLM Edits `area:ai`
  Keywords: escape sequence
  Summary: The agentic mode generally performs well. However, when using Gemini models on tsx files, the output almost always includes triple backticks () at ...

- **[#33858](https://github.com/zed-industries/zed/issues/33858)** (closed) — Shift+Enter in built-in terminal sends same escape sequence as Enter, preventing newlines in Claude Code
  Keywords: escape sequence
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#38603](https://github.com/zed-industries/zed/issues/38603)** (open) — Styled Components (CSS in JS): backslash in CSS property value breaks syntax highlighting `area:languages/css`, `state:needs repro`, `frequency:common`
  Keywords: escape sequence
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#45125](https://github.com/zed-industries/zed/issues/45125)** (open) — TypeScript: escaped backtick inside template literal type breaks syntax highlighting `area:languages/typescript`, `area:tree-sitter`, `stale`
  Keywords: escape sequence
  Summary: Reproduction steps

### zellij

- **[#1039](https://github.com/zellij-org/zellij/issues/1039)** (closed) — per-pane titlebar/footerbar?
  Keywords: escape sequence
  Summary: Hi! I'm experimenting with zellij layouts for medium-length-ish semi-interactive running tasks that I want to fire up and be able to monitor at a g...

- **[#1709](https://github.com/zellij-org/zellij/issues/1709)** (closed) — OSC 52 events doesn't be forward when the contents is  more than or equal to 1024 bytes `suspected bug`
  Keywords: osc 52
  Summary: #1644 introduced OSC 52 events forwarding.

- **[#1712](https://github.com/zellij-org/zellij/issues/1712)** (open) — Option to disable select mode
  Keywords: osc 52
  Summary: Similar to #1033, #1673, #1593, #1677, etc, I cannot copy text when using Zellij.

- **[#2637](https://github.com/zellij-org/zellij/issues/2637)** (open) — Zellij Clipboard Functionality: Limitations for Integrating Neovim Registers
  Keywords: osc 52
  Summary: I am a user of **Neovim** and heavily use its registers. It is convenient to press `y` to copy some text from **Neovim** running on the remote host...

- **[#2853](https://github.com/zellij-org/zellij/issues/2853)** (open) — Native terminal selection/copy includes spaces and hard breaks even when pane is fullscreen and borderless `suspected bug`
  Keywords: osc 52
  Summary: Zellij always wants to fill every cell of the screen and do hard wrapping even for panes that have no border and are full-width. This interferes wi...

- **[#2931](https://github.com/zellij-org/zellij/issues/2931)** (open) — Unable to use custom config on 0.39.0 `suspected bug`
  Keywords: osc 52
  Summary: **Basic information**

- **[#3002](https://github.com/zellij-org/zellij/issues/3002)** (closed) — Terminal Shortcuts not working `suspected bug`
  Keywords: osc 52
  Summary: Some common terminal shortcuts do not work

- **[#3133](https://github.com/zellij-org/zellij/issues/3133)** (closed) — Can't paste when connecting via SSH `suspected bug`
  Keywords: osc 52
  Summary: Issue description

- **[#3135](https://github.com/zellij-org/zellij/issues/3135)** (open) — paste_command option
  Keywords: osc 52
  Summary: Hi, guys. Similar to `copy_command`, I'd like to see a command for paste.

- **[#3512](https://github.com/zellij-org/zellij/issues/3512)** (open) — Consider using OSC-7 to get CWD
  Keywords: escape sequence
  Summary: It may be worthwhile investigating using the OSC-7 ANSI escape sequence to get the current working directory, instead of from the process. It seems...

- **[#3516](https://github.com/zellij-org/zellij/issues/3516)** (open) — Jumping words with Ctrl(cmd)+Arrow keys on mac triggers fullscreen
  Keywords: escape sequence
  Summary: Issues with the Zellij UI / behavior / crash

- **[#3597](https://github.com/zellij-org/zellij/issues/3597)** (open) — Ansi escape sequence for multi cursor terminals
  Keywords: escape sequence
  Summary: Currently zellij can detect multiple users attached to it, but it can't detect if there are multiple users behind a single terminal (zellij) attach...

- **[#3609](https://github.com/zellij-org/zellij/issues/3609)** (open) — Tabs are missing after resurrection
  Keywords: osc 52
  Summary: 2. Issues with the Zellij behavior

- **[#3769](https://github.com/zellij-org/zellij/issues/3769)** (open) — Zellij hijacks "ctrl+shift+right" and "alt+shift+right" keybindings defined in iTerm2
  Keywords: escape sequence
  Summary: **Basic information**

- **[#3859](https://github.com/zellij-org/zellij/issues/3859)** (open) — Copy not working on Panic Prompt v3 terminal
  Keywords: osc 52
  Summary: RE: https://zellij.dev/documentation/faq.html#copy--paste-isnt-working-how-can-i-fix-this

- **[#3890](https://github.com/zellij-org/zellij/issues/3890)** (open) — Ctrl-A / Ctrl-E / Ctrl-R not working on terminal command line
  Keywords: escape sequence
  Summary: 1. Graphical issue inside a terminal pane (eg. something does not look as it should)

- **[#3951](https://github.com/zellij-org/zellij/issues/3951)** (open) — Zellij prevents neovim from detecting OSC52 clipboard support
  Keywords: osc 52
  Summary: Zellij prevents neovim 0.10.* from detecting OSC52 clipboard support (which zellij supports).

- **[#3954](https://github.com/zellij-org/zellij/issues/3954)** (open) — Escape code passthrough
  Keywords: escape sequence
  Summary: Note on new features: while Zellij tries to be as user friendly as possible, it must also be friendly to its maintainers. We take great care in con...

- **[#4126](https://github.com/zellij-org/zellij/issues/4126)** (open) — hide session name and word "Zellij" on left
  Keywords: osc 52
  Summary: Hi! Just simple question: I use compact view, I want completely hide session name and word "Zellij" on left of it. How to do that ? Ideally also "T...

- **[#4177](https://github.com/zellij-org/zellij/issues/4177)** (open) — Problem with copy past into LXC - ZELLIJ <> TMUX
  Keywords: osc 52
  Summary: You're working on a MacBook, using iTerm2 to connect via SSH to an LXC container. You do all your development in the terminal, using Neovim as your...

- **[#4256](https://github.com/zellij-org/zellij/issues/4256)** (open) — Custom layout unexpectedly overrides default keybindings
  Keywords: osc 52
  Summary: Issue description

- **[#4320](https://github.com/zellij-org/zellij/issues/4320)** (open) — Support XTGETTCAP
  Keywords: osc 52, escape sequence
  Summary: Zellij supports OSC 52, but some programs like [neovim](https://github.com/neovim/neovim/issues/29504#issuecomment-2226374704) want to query capabi...

## Platform-Specific Issues

*97 issues*

### waveterm

- **[#2787](https://github.com/wavetermdev/waveterm/issues/2787)** (open) — [Bug]: TUI animations scroll instead of animating in-place (missing xterm.js Synchronized Output support)
  Keywords: conpty
  Summary: Description

### wezterm

- **[#244](https://github.com/wezterm/wezterm/issues/244)** (closed) — bold/bright foreground [1m is fixed to white `bug`, `Windows`
  Keywords: conpty
  Summary: Describe the bug

- **[#1927](https://github.com/wezterm/wezterm/issues/1927)** (open) — Track/bundle/ship conpty using the new nuget package `Windows`
  Keywords: conpty
  Summary: * https://github.com/microsoft/terminal/pull/12980

- **[#2294](https://github.com/wezterm/wezterm/issues/2294)** (closed) — not launching hx properly `bug`, `Windows`, `cant-reproduce`
  Keywords: conpty
  Summary: **update:**

- **[#3080](https://github.com/wezterm/wezterm/issues/3080)** (closed) — Wezterm Crashes in WSL2 with WSLg `bug`, `Windows`, `Wayland`
  Keywords: conpty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3531](https://github.com/wezterm/wezterm/issues/3531)** (closed) — conpty: With shell integration enabled, the last line of each output immediately disappears until the screen is filled `bug`, `Windows`
  Keywords: conpty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3624](https://github.com/wezterm/wezterm/issues/3624)** (closed) — conpty: Images rendered by imgcat is cut off by cmd.exe or wsl prompts `bug`, `Windows`, `fixed-in-nightly`
  Keywords: conpty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4783](https://github.com/wezterm/wezterm/issues/4783)** (open) — Undercurl not working - differnt behavior in shell and tmux/neovim `bug`, `Windows`, `conpty`
  Keywords: conpty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4784](https://github.com/wezterm/wezterm/issues/4784)** (closed) — [Windows] using portable_pty causes terminal to be cleared while trying to stream to stdout `bug`, `Windows`, `PR-welcome`
  Keywords: conpty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6670](https://github.com/wezterm/wezterm/issues/6670)** (closed) — tmux -CC not working on windows ConPTY `bug`, `conpty`
  Keywords: conpty
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7025](https://github.com/wezterm/wezterm/issues/7025)** (open) — Portably pty fails to launch commands on windows `bug`, `needs:triage`
  Keywords: conpty
  Summary: What Operating System(s) are you seeing this problem on?

### zed

- **[#8825](https://github.com/zed-industries/zed/issues/8825)** (closed) — Zed main branch build windows crash `bug`, `crash`, `platform:windows`
  Keywords: conpty
  Summary: Check for existing issues

- **[#15991](https://github.com/zed-industries/zed/issues/15991)** (closed) — SSH remoting: Terminal fails to launch `bug`, `area:integrations/terminal`, `platform:windows`
  Keywords: conpty
  Summary: Check for existing issues

- **[#33429](https://github.com/zed-industries/zed/issues/33429)** (closed) — Debugger: cannot found go when debugging `area:debugger`
  Keywords: conpty
  Summary: <!-- Please insert a one line summary of the issue below -->

- **[#33520](https://github.com/zed-industries/zed/issues/33520)** (closed) — Zed crashes on Windows after locking and unlocking the computer `platform:windows`
  Keywords: conpty
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#34626](https://github.com/zed-industries/zed/issues/34626)** (closed) — Crash after typing `]` with a spanish keyboard `platform:windows`
  Keywords: conpty
  Summary: Typing `]` on a spanish keyboard crashes zed on windows.

- **[#36839](https://github.com/zed-industries/zed/issues/36839)** (closed) — Failed to load the database file `platform:windows`
  Keywords: conpty
  Summary: **Summary**

- **[#37263](https://github.com/zed-industries/zed/issues/37263)** (closed) — error: Connection to TCP DAP timeout 127.0.0.1:62447 `area:debugger`, `frequency:uncommon`, `priority:P2`
  Keywords: conpty
  Summary: <!-- Please insert a one line summary of the issue below -->

- **[#39407](https://github.com/zed-industries/zed/issues/39407)** (closed) — Windows Beta: Crash opening project in _Thinkpad E14 Gen 6_
  Keywords: conpty
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#40302](https://github.com/zed-industries/zed/issues/40302)** (open) — Windows: openrouter not working `platform:windows`, `area:ai`, `state:needs repro`
  Keywords: conpty
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#40988](https://github.com/zed-industries/zed/issues/40988)** (closed) — Windows: Language Servers not starting when opening remote in new window
  Keywords: conpty
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#41409](https://github.com/zed-industries/zed/issues/41409)** (closed) — AsciiDoc grammar fails to compile (LLVM OOM) on Windows; logs attached
  Keywords: conpty
  Summary: The AsciiDoc grammar refuses to compile in Zed. Below are the relevant logs from Zed.log showing repeated failures and the LLVM out-of-memory error...

- **[#41830](https://github.com/zed-industries/zed/issues/41830)** (closed) — Project Search causes remote server connection to drop `area:search`, `platform:remote`, `state:needs repro`
  Keywords: conpty
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#42166](https://github.com/zed-industries/zed/issues/42166)** (closed) — Zed crashes when opening the terminal. `platform:windows`
  Keywords: conpty
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#42345](https://github.com/zed-industries/zed/issues/42345)** (closed) — Windows: <Terminal overwrites old lines after switching apps (intermittent)> `platform:windows`
  Keywords: conpty
  Summary: <!-- Please insert a one-line summary of the issue below -->

- **[#42355](https://github.com/zed-industries/zed/issues/42355)** (closed) — Windows: Zed cannot use `rust-analyzer.exe` installed from `rustup component add` `platform:windows`
  Keywords: conpty
  Summary: **Expected Behavior**:

- **[#42713](https://github.com/zed-industries/zed/issues/42713)** (open) — Windows: Git SSH passphrase modal not shown when remote is RedHat `area:integrations/git`, `platform:remote`, `frequency:uncommon`
  Keywords: conpty
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#42805](https://github.com/zed-industries/zed/issues/42805)** (closed) — Windows: Failed to load environment variables after update `platform:windows`
  Keywords: conpty
  Summary: <!-- Please insert a one-line summary of the issue below -->

- **[#42844](https://github.com/zed-industries/zed/issues/42844)** (open) — Windows: Changing the shell setting breaks some things `area:integrations/terminal`
  Keywords: conpty
  Summary: <!-- Please insert a one-line summary of the issue below -->

- **[#42861](https://github.com/zed-industries/zed/issues/42861)** (closed) — Windows: When viewing specific files, cursor movement becomes sluggish `area:performance`, `area:ui/animations`, `state:reproducible`
  Keywords: conpty
  Summary: <!-- Please insert a one-line summary of the issue below -->

- **[#43121](https://github.com/zed-industries/zed/issues/43121)** (closed) — Windows: Always Shows `failed to load environment variables` `area:languages/prisma`, `area:integrations/environment`
  Keywords: conpty
  Summary: It keeps showing failed to load environment variables on the start-up

- **[#43224](https://github.com/zed-industries/zed/issues/43224)** (closed) — Windows: Slow Startup `platform:windows`
  Keywords: conpty
  Summary: <!-- Please insert a one-line summary of the issue below -->

- **[#43252](https://github.com/zed-industries/zed/issues/43252)** (closed) — Windows: running a task should not load the profile by default `platform:windows`, `area:tasks`
  Keywords: conpty
  Summary: <!-- Please insert a one-line summary of the issue below -->

- **[#43335](https://github.com/zed-industries/zed/issues/43335)** (closed) — OpenCode via ACP only works in New Empty Window, but not in existing projects `state:needs info`, `platform:windows`
  Keywords: conpty
  Summary: Add folder to it to actually use Opencode otherwise it just doesn't respond.

- **[#43627](https://github.com/zed-industries/zed/issues/43627)** (closed) — TailwindCSS hover does not work on `.astro` files in a remote project `area:languages/astro`, `platform:remote`, `frequency:common`
  Keywords: conpty
  Summary: Tailwind class hover **exclusively does not work** on Astro files.

- **[#43697](https://github.com/zed-industries/zed/issues/43697)** (closed) — The problem of duplicate logs printed by the built-in terminal of Zed when using git log `area:integrations/terminal`, `state:needs info`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#43882](https://github.com/zed-industries/zed/issues/43882)** (closed) — ctrl-w does not close active terminal tab `area:integrations/terminal`, `platform:windows`, `state:reproducible`
  Keywords: conpty
  Summary: Reproduction steps

- **[#44119](https://github.com/zed-industries/zed/issues/44119)** (closed) — Failed to load wsl enviroment `platform:windows`
  Keywords: conpty
  Summary: Reproduction steps

- **[#44437](https://github.com/zed-industries/zed/issues/44437)** (closed) — CodeLLDB is not available for download for ARM64 Windows `platform:windows`, `area:debugger/dap/CodeLLDB`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#44448](https://github.com/zed-industries/zed/issues/44448)** (closed) — [WSL] Connection fails with "os error 206" (filename too long) when using direnv on NixOS `platform:windows`, `platform:windows/wsl`, `frequency:uncommon`
  Keywords: conpty
  Summary: Reproduction steps

- **[#44541](https://github.com/zed-industries/zed/issues/44541)** (open) — The terminal's vim_mode isn't working properly. `area:integrations/terminal`, `area:parity/vim`, `state:reproducible`
  Keywords: conpty
  Summary: Reproduction steps

- **[#44571](https://github.com/zed-industries/zed/issues/44571)** (open) — Not recognizing SQL as installed language in WSL `area:languages`, `area:language server/server failure`, `state:needs repro`
  Keywords: conpty
  Summary: Reproduction steps

- **[#44635](https://github.com/zed-industries/zed/issues/44635)** (closed) — XDebug no longer initializing `state:unactionable`, `area:debugger`
  Keywords: conpty
  Summary: Reproduction steps

- **[#44815](https://github.com/zed-industries/zed/issues/44815)** (open) — Rust library tests don't run when trying to debug them `area:languages/rust`, `state:needs repro`, `area:debugger`
  Keywords: conpty
  Summary: Reproduction steps

- **[#44839](https://github.com/zed-industries/zed/issues/44839)** (closed) — zed dev version using wsl2 open project file not display
  Keywords: conpty
  Summary: Reproduction steps

- **[#45126](https://github.com/zed-industries/zed/issues/45126)** (open) — Using oh-my-bash on an ssh remote causes the terminal to crash instantly `area:integrations/terminal`, `platform:remote`, `state:needs repro`
  Keywords: conpty
  Summary: Reproduction steps

- **[#45182](https://github.com/zed-industries/zed/issues/45182)** (closed) — Bracket colorization is applied only after the first edit of the file. `state:reproducible`, `frequency:always`, `priority:P3`
  Keywords: conpty
  Summary: Reproduction steps

- **[#45827](https://github.com/zed-industries/zed/issues/45827)** (closed) — Terminal working directory not being honored for Remote SSH project `area:integrations/terminal`, `platform:remote`, `frequency:uncommon`
  Keywords: conpty
  Summary: Reproduction steps

- **[#45832](https://github.com/zed-industries/zed/issues/45832)** (open) — Multiple windows `state:needs repro`, `area:integrations/environment`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#45888](https://github.com/zed-industries/zed/issues/45888)** (closed) — Some setting keys are not recognised `state:needs repro`
  Keywords: conpty
  Summary: Reproduction steps

- **[#45984](https://github.com/zed-industries/zed/issues/45984)** (closed) — Enable all postfix snippets by default for Rust `area:settings`, `state:reproducible`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46307](https://github.com/zed-industries/zed/issues/46307)** (closed) — Canonicalization of VHD mounted directories results in limited terminal functionality `area:integrations/terminal`, `platform:windows`, `area:file finder`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46333](https://github.com/zed-industries/zed/issues/46333)** (closed) — something wrong with ty `area:languages/python`, `platform:remote`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46360](https://github.com/zed-industries/zed/issues/46360)** (open) — md texts with different sizes are 'wiggling' in scrolling `area:ui/font`, `area:gpui`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46514](https://github.com/zed-industries/zed/issues/46514)** (closed) — Terminal pwsh.exe on panel resize moves prompt to top of resized panel and overwrites existing text `platform:windows`, `state:needs repro`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46650](https://github.com/zed-industries/zed/issues/46650)** (open) — Recent Update Broke Windows Custom Shell Settings (os error 123) `meta:regression`, `area:integrations/terminal`, `platform:windows`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46687](https://github.com/zed-industries/zed/issues/46687)** (closed) — hanging when open user profile folder `area:performance/memory leak`, `frequency:always`, `priority:P2`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46871](https://github.com/zed-industries/zed/issues/46871)** (closed) — WIN: Python virtual env activation not working on terminal `state:needs triage`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46933](https://github.com/zed-industries/zed/issues/46933)** (open) — Zed update broke my Go, then my D `area:integrations/terminal`, `platform:windows`, `area:languages/go`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46943](https://github.com/zed-industries/zed/issues/46943)** (open) — Cannot Open any File on a Specific Line `area:cli`, `platform:windows`, `state:needs repro`
  Keywords: conpty
  Summary: Reproduction steps

- **[#46970](https://github.com/zed-industries/zed/issues/46970)** (open) — Terminal freezes and input lags with remote ssh server on first open in new window or Ctrl+Shift+~ until Ctrl+C is pressed `area:performance`, `area:integrations/terminal`, `meta:freeze`
  Keywords: conpty
  Summary: Reproduction steps

- **[#47303](https://github.com/zed-industries/zed/issues/47303)** (closed) — Windows Debugpy launcher fails to connect with virtual environment toolchain `area:languages/python`, `state:needs repro`, `area:debugger`
  Keywords: conpty
  Summary: Reproduction steps

- **[#47340](https://github.com/zed-industries/zed/issues/47340)** (closed) — No external agents work on Windows with 'server shut down unexpectedly' error displayed `platform:windows`, `area:ai/agent thread`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#47585](https://github.com/zed-industries/zed/issues/47585)** (open) — Rust error messages are not showing correctly on hover and making hover popovers unresponsive. `area:languages/rust`, `area:popovers`, `state:needs repro`
  Keywords: conpty
  Summary: Reproduction steps

- **[#47823](https://github.com/zed-industries/zed/issues/47823)** (closed) — "Failed to load environment variables" when `'` symbol in project path `platform:windows`, `state:reproducible`, `area:integrations/environment`
  Keywords: conpty
  Summary: Reproduction steps

- **[#47960](https://github.com/zed-industries/zed/issues/47960)** (closed) — Kaspersky antivirus detects PDM:Trojan-Banker.Win32.ClipBanker.gen when using Claude Code `area:security & privacy`, `frequency:uncommon`, `priority:P2`
  Keywords: conpty
  Summary: Hello. These reproductions steps might not be enough, as it's something that happened to me today but it's the first time. I've been using Zed with...

- **[#47972](https://github.com/zed-industries/zed/issues/47972)** (closed) — Display problem when zooming out to console panel `area:integrations/terminal`, `platform:windows`, `state:reproducible`
  Keywords: conpty
  Summary: Reproduction steps

- **[#47985](https://github.com/zed-industries/zed/issues/47985)** (closed) — Debugger (Rust): unexpected argument '' found `state:needs triage`
  Keywords: conpty
  Summary: Reproduction steps

- **[#47996](https://github.com/zed-industries/zed/issues/47996)** (open) — CPU spike to 40% on all cores from MS Defender if Zed has updates `platform:windows`, `area:security & privacy`, `state:needs repro`
  Keywords: conpty
  Summary: Reproduction steps

- **[#48006](https://github.com/zed-industries/zed/issues/48006)** (open) — On the Windows 11 system, Zed gets stuck when switching between projects. `state:needs repro`, `frequency:common`, `priority:P2`
  Keywords: conpty
  Summary: Reproduction steps

- **[#48284](https://github.com/zed-industries/zed/issues/48284)** (closed) — "Reveal in file explorer" command doesn't work (`project_panel::RevealInFileManager`) `area:project panel`, `state:needs repro`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#48324](https://github.com/zed-industries/zed/issues/48324)** (closed) — Codex is not working with Zed on WSL2. Error: Internal error: "server shut down unexpectedly" `frequency:uncommon`, `priority:P3`, `area:ai/codex`
  Keywords: conpty
  Summary: Reproduction steps

- **[#48326](https://github.com/zed-industries/zed/issues/48326)** (closed) — Zed under WSL can't use agent plugin for ai panel (OpenCode, Github Copilot, etc) `state:needs triage`
  Keywords: conpty
  Summary: Reproduction steps

- **[#48385](https://github.com/zed-industries/zed/issues/48385)** (closed) — Remote Folder (SSH) not being able to connect `platform:windows`, `platform:remote`, `meta:duplicate`
  Keywords: conpty
  Summary: Reproduction steps

- **[#48540](https://github.com/zed-industries/zed/issues/48540)** (open) — OpenCode disregards config files for Ask Permissions `state:reproducible`, `area:ai/acp`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#48754](https://github.com/zed-industries/zed/issues/48754)** (open) — No external agents work in WSL `area:ai/acp`, `platform:windows/wsl`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49072](https://github.com/zed-industries/zed/issues/49072)** (closed) — Terminal freezes in devcontainer on Windows (Podman) `area:integrations/terminal`, `platform:windows`, `frequency:uncommon`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49165](https://github.com/zed-industries/zed/issues/49165)** (closed) — UI of the latest version on Windows ARM virtual machine is glitching `platform:windows`, `meta:duplicate`, `frequency:uncommon`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49406](https://github.com/zed-industries/zed/issues/49406)** (closed) — Unable to log in with Claude code (windows 11/WSL) `state:needs triage`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49442](https://github.com/zed-industries/zed/issues/49442)** (open) — Slow startup on Windows `area:performance`, `platform:windows`, `state:needs repro`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49570](https://github.com/zed-industries/zed/issues/49570)** (closed) — ACP registry agents still not working on wsl `state:needs triage`
  Keywords: conpty
  Summary: Related to https://github.com/zed-industries/zed/issues/47993

- **[#49687](https://github.com/zed-industries/zed/issues/49687)** (open) — Auto-update crash with permission denied on Windows `area:installer-updater`, `platform:windows`, `state:needs research`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49827](https://github.com/zed-industries/zed/issues/49827)** (open) — editor::ExpandMacroRecursively does not seem to work anymore with derive macros `area:languages/rust`, `state:needs repro`, `community champion`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49875](https://github.com/zed-industries/zed/issues/49875)** (closed) — Autosave “on focus change” does not trigger when editing files inside Uncommitted Changes diff view `area:editor`, `area:integrations/git`, `state:needs repro`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49880](https://github.com/zed-industries/zed/issues/49880)** (open) — Module path is not properly escaped when trying to run function/module in Python project `area:languages/python`, `state:needs repro`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49915](https://github.com/zed-industries/zed/issues/49915)** (open) — Drag windows txt file into WSL seems to use the wrong path. `area:navigation`, `state:needs repro`, `platform:windows/wsl`
  Keywords: conpty
  Summary: Reproduction steps

- **[#49956](https://github.com/zed-industries/zed/issues/49956)** (closed) — Connect to remote host failed: program not found. Plear try again. `state:needs info`, `platform:windows`, `platform:remote`
  Keywords: conpty
  Summary: Reproduction steps

- **[#50399](https://github.com/zed-industries/zed/issues/50399)** (open) — Opencode get stuck when write file `state:needs repro`, `area:ai/agent thread`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#50497](https://github.com/zed-industries/zed/issues/50497)** (open) — GIT DIFF VIEW doesn`t work `state:needs repro`, `frequency:common`, `priority:P2`
  Keywords: conpty
  Summary: Reproduction steps

- **[#50561](https://github.com/zed-industries/zed/issues/50561)** (open) — search is not working correctly `area:search`, `state:needs repro`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#50657](https://github.com/zed-industries/zed/issues/50657)** (closed) — Claude Agent not working `meta:duplicate`, `area:ai/agent thread`, `frequency:common`
  Keywords: conpty
  Summary: Reproduction steps

- **[#50696](https://github.com/zed-industries/zed/issues/50696)** (closed) — Zed crashing because of Roslyn crashed `area:languages/c#`, `state:reproducible`, `frequency:uncommon`
  Keywords: conpty
  Summary: Reproduction steps

- **[#51350](https://github.com/zed-industries/zed/issues/51350)** (open) — remote MCP server changed to "This MCP server was installed from an extension" after restart Zed `state:needs repro`, `area:ai/mcp`, `frequency:uncommon`
  Keywords: conpty
  Summary: Reproduction steps

- **[#51552](https://github.com/zed-industries/zed/issues/51552)** (open) — Remote SSH connection fails on Windows — proxy stdin closes immediately, server gets EOF `platform:windows`, `platform:remote`, `state:needs repro`
  Keywords: conpty
  Summary: Reproduction steps

- **[#51589](https://github.com/zed-industries/zed/issues/51589)** (closed) — Project & Git panel doesn't detect changes made from terminal `state:needs triage`
  Keywords: conpty
  Summary: Reproduction steps

- **[#51693](https://github.com/zed-industries/zed/issues/51693)** (closed) — HelixPaste appears broken (for system clipboard) `state:reproducible`, `area:parity/helix`, `frequency:uncommon`
  Keywords: conpty
  Summary: Reproduction steps

- **[#52071](https://github.com/zed-industries/zed/issues/52071)** (closed) — `vim::EndOfDocument` go to the start of document instead `state:needs triage`
  Keywords: conpty
  Summary: Reproduction steps

## Storage Issues

*8 issues*

### zed

- **[#36839](https://github.com/zed-industries/zed/issues/36839)** (closed) — Failed to load the database file `platform:windows`
  Keywords: sqlite
  Summary: **Summary**

- **[#45696](https://github.com/zed-industries/zed/issues/45696)** (closed) — Memory leak: Zed consumes 71GB+ RAM when opened on home directory (~/) causing system freeze and potential filesystem corruption `area:performance`, `state:reproducible`, `area:performance/memory leak`
  Keywords: corrupt
  Summary: Opening Zed on the home directory (~/) causes catastrophic memory consumption, eventually exhausting all system RAM, heavily utilizing swap, and re...

- **[#47109](https://github.com/zed-industries/zed/issues/47109)** (open) — Zed logs flooded with database constraint errors `area:workspace`, `state:needs repro`, `frequency:common`
  Keywords: sqlite
  Summary: Reproduction steps

- **[#47543](https://github.com/zed-industries/zed/issues/47543)** (open) — Crash and process hang when opening a database migrated by a newer Zed version with no diagnostics shown `area:diagnostics`, `state:reproducible`, `frequency:common`
  Keywords: sqlite
  Summary: Reproduction steps

- **[#47576](https://github.com/zed-industries/zed/issues/47576)** (open) — OpenAI API error: tool/function schema invalid when array parameter is missing items `state:needs repro`, `frequency:common`, `priority:P2`
  Keywords: sqlite
  Summary: Reproduction steps

- **[#49162](https://github.com/zed-industries/zed/issues/49162)** (open) — ACP readTextFile returns stale buffer content after terminal commands modify files on disk `state:needs repro`, `area:ai/acp`, `frequency:common`
  Keywords: corrupt
  Summary: When using Claude Code (or other ACP agents) in Zed, `fs/read_text_file` returns stale content from Zed's in-memory buffer after files have been mo...

- **[#49241](https://github.com/zed-industries/zed/issues/49241)** (closed) — "Failed to load the database file." even just by starting up Zed `platform:windows`, `area:serialization`, `state:reproducible`
  Keywords: sqlite, wal
  Summary: Reproduction steps

- **[#49585](https://github.com/zed-industries/zed/issues/49585)** (closed) — Malformed Sqlite DB broke Recent Projects and Workspace Reload after update `platform:macOS`, `area:serialization`, `frequency:uncommon`
  Keywords: sqlite
  Summary: Reproduction steps

## Design Pattern Issues

*96 issues*

### helix

- **[#202](https://github.com/helix-editor/helix/issues/202)** (closed) — Space leak
  Keywords: leak
  Summary: Scrolling through a file leaks memory.

- **[#11213](https://github.com/helix-editor/helix/issues/11213)** (open) — Searching for \0 leaks error message into UI `C-bug`, `A-theme`
  Keywords: leak
  Summary: If you type `/\0<ret>` then this will happen:

- **[#12316](https://github.com/helix-editor/helix/issues/12316)** (open) — Possible deadlock in concurrent write test `C-bug`
  Keywords: deadlock
  Summary: I was rewriting the write command and in the process of testing I noticed that these two tests always seem to timeout. I've never seen it run fully...

### waveterm

- **[#2731](https://github.com/wavetermdev/waveterm/issues/2731)** (open) — [Bug]: Electron memory leak `bug`, `triage`
  Keywords: leak
  Summary: Current Behavior

### wezterm

- **[#238](https://github.com/wezterm/wezterm/issues/238)** (closed) — Memory leak `bug`, `Linux`, `X11`
  Keywords: leak
  Summary: Describe the bug

- **[#534](https://github.com/wezterm/wezterm/issues/534)** (closed) — Image protocol memory leaking ? `bug`
  Keywords: leak
  Summary: Describe the bug

- **[#1860](https://github.com/wezterm/wezterm/issues/1860)** (closed) — Deadlock on ssh_domain connect  when stdout is printed from remote .bashrc `bug`, `fixed-in-nightly`, `multiplexer`
  Keywords: deadlock
  Summary: What Operating System(s) are you seeing this problem on?

- **[#2060](https://github.com/wezterm/wezterm/issues/2060)** (closed) — tmux 3.3 leaks text into shell `bug`
  Keywords: leak
  Summary: What Operating System(s) are you seeing this problem on?

- **[#2453](https://github.com/wezterm/wezterm/issues/2453)** (closed) — Obscene Memory Usage, Possible Memory Leak? `bug`
  Keywords: leak
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3612](https://github.com/wezterm/wezterm/issues/3612)** (closed) — fd leak on wezterm.gui.enumerate_gpus() `bug`, `fixed-in-nightly`
  Keywords: leak
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3771](https://github.com/wezterm/wezterm/issues/3771)** (open) — Memory leak and high CPU usage with actions.RotatePanes with certain pane layouts in `wezterm connect unix` `bug`, `cant-reproduce`
  Keywords: leak
  Summary: What Operating System(s) are you seeing this problem on?

- **[#3815](https://github.com/wezterm/wezterm/issues/3815)** (closed) — Memory Leaks `bug`
  Keywords: leak
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4498](https://github.com/wezterm/wezterm/issues/4498)** (closed) — Inotify leak `bug`, `cant-reproduce`
  Keywords: leak
  Summary: What Operating System(s) are you seeing this problem on?

- **[#4826](https://github.com/wezterm/wezterm/issues/4826)** (open) — Wezterm wgpu VRAM memory leak `bug`, `frontend:webgpu`
  Keywords: leak
  Summary: What Operating System(s) are you seeing this problem on?

- **[#6116](https://github.com/wezterm/wezterm/issues/6116)** (closed) — Thread and memory leak when spawning/terminating `bug`
  Keywords: leak
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7400](https://github.com/wezterm/wezterm/issues/7400)** (open) — Memory leak when using kitty + gif `bug`, `needs:triage`
  Keywords: leak
  Summary: What Operating System(s) are you seeing this problem on?

- **[#7661](https://github.com/wezterm/wezterm/issues/7661)** (open) — tmux CC domain detach deadlocks entire GUI when last window is closed via Ctrl+D
  Keywords: deadlock
  Summary: What Operating System(s) are you seeing this problem on?

### zed

- **[#17](https://github.com/zed-industries/zed/issues/17)** (closed) — Investigate using `scoped_pool` for text layout `area:performance`
  Keywords: thread pool
  Summary: Previously, text layout used `easy_parallel` which had a significant overhead associated with it because of spawning new thread each time. That dro...

- **[#211](https://github.com/zed-industries/zed/issues/211)** (closed) — Deadlock when splitting panes with a shared buffer `bug`
  Keywords: deadlock
  Summary: Deadlock when splitting panes with a shared buffer

- **[#1057](https://github.com/zed-industries/zed/issues/1057)** (closed) — Server leaks memory `bug`
  Keywords: leak
  Summary: Server leaks memory

- **[#1484](https://github.com/zed-industries/zed/issues/1484)** (closed) — RA instance leakage while collaborating
  Keywords: leak
  Summary: Me and @ForLoveOfCats where pairing all day on a GPUI update, we found that RA server processes where leaking again. Not sure how to trigger yet, m...

- **[#1672](https://github.com/zed-industries/zed/issues/1672)** (closed) — `Event::chars_for_modified_key` is leaking `CGEvent`s and `CGEventSource`s `bug`
  Keywords: leak
  Summary: `Event::chars_for_modified_key` is leaking `CGEvent`s and `CGEventSource`s

- **[#1727](https://github.com/zed-industries/zed/issues/1727)** (closed) — Editing a plain text file leaks memory
  Keywords: leak
  Summary: Editing a plain text file leaks memory

- **[#5586](https://github.com/zed-industries/zed/issues/5586)** (closed) — Zed uses 400%+ CPU with possible memory leak `bug`
  Keywords: leak
  Summary: **Describe the bug**

- **[#6336](https://github.com/zed-industries/zed/issues/6336)** (open) — Memory Usage at 174GB (Gigabytes)! `meta:large projects`, `state:needs repro`, `area:performance/memory leak`
  Keywords: leak
  Summary: Check for existing issues

- **[#7939](https://github.com/zed-industries/zed/issues/7939)** (closed) — Zed memory usage greater than vscode and memory leak issue on long run `bug`, `area:performance`, `stale`
  Keywords: leak
  Summary: Check for existing issues

- **[#9716](https://github.com/zed-industries/zed/issues/9716)** (closed) — Streamer mode visually leaks secrets when a tree sitter grammar isn't loaded `bug`, `area:editor`, `priority`
  Keywords: leak
  Summary: This can happen in several cases, e.g. the file isn't covered by a tree sitter grammar or that associated tree sitter grammar hasn't finished initi...

- **[#9744](https://github.com/zed-industries/zed/issues/9744)** (closed) — Big memory leak issue since version 0.126.2 `bug`, `area:performance`, `crash`
  Keywords: leak
  Summary: Check for existing issues

- **[#10525](https://github.com/zed-industries/zed/issues/10525)** (closed) — Zed is leaking memory `bug`, `area:performance`, `crash`
  Keywords: leak
  Summary: Check for existing issues

- **[#12561](https://github.com/zed-industries/zed/issues/12561)** (closed) — serious memory leak issue on windows10 `bug`, `area:performance`, `platform:windows`
  Keywords: leak
  Summary: Check for existing issues

- **[#13361](https://github.com/zed-industries/zed/issues/13361)** (closed) — The remote projects dialogue box "leaks" the panel mouse interactions below `bug`, `area:controls/mouse`, `area:workspace`
  Keywords: leak
  Summary: Check for existing issues

- **[#13544](https://github.com/zed-industries/zed/issues/13544)** (closed) — Possible memory leak `bug`, `area:performance`, `crash`
  Keywords: leak
  Summary: Check for existing issues

- **[#13907](https://github.com/zed-industries/zed/issues/13907)** (closed) — Memory Leak due to missing any git branches `bug`, `area:performance`, `platform:windows`
  Keywords: leak
  Summary: Check for existing issues

- **[#14794](https://github.com/zed-industries/zed/issues/14794)** (closed) — Potential memory leak with the XML extension `bug`, `area:languages`
  Keywords: leak
  Summary: Check for existing issues

- **[#15692](https://github.com/zed-industries/zed/issues/15692)** (closed) — Facing Memory leak issues, High power/memory consumption `bug`, `area:performance`, `platform:linux`
  Keywords: leak
  Summary: Check for existing issues

- **[#17937](https://github.com/zed-industries/zed/issues/17937)** (closed) — Zed Taking 185 GB of Ram on my Macbook due to Potential Memory Leak Issues `bug`, `area:performance`
  Keywords: leak
  Summary: Check for existing issues

- **[#18384](https://github.com/zed-industries/zed/issues/18384)** (closed) — Memory leak `meta:duplicate`
  Keywords: leak
  Summary: Check for existing issues

- **[#21042](https://github.com/zed-industries/zed/issues/21042)** (open) — Zed Leaks Key Events to Chinese Input Method `area:internationalization`, `area:controls/keybinds`, `state:needs repro`
  Keywords: leak
  Summary: Check for existing issues

- **[#23008](https://github.com/zed-industries/zed/issues/23008)** (open) — Memory Leak in Zed Editor's Built-in Terminal When Running yes $(echo -en "\a") `area:performance`, `area:integrations/terminal`, `area:performance/memory leak`
  Keywords: leak
  Summary: Check for existing issues

- **[#23750](https://github.com/zed-industries/zed/issues/23750)** (closed) — > 160GB Memory Usage/Leak With Latest Update
  Keywords: leak
  Summary: Describe the bug / provide steps to reproduce it

- **[#23833](https://github.com/zed-industries/zed/issues/23833)** (closed) — Memory leak
  Keywords: leak
  Summary: Describe the bug / provide steps to reproduce it

- **[#24742](https://github.com/zed-industries/zed/issues/24742)** (closed) — Tree sitter memory leak when using the YAML parser in an extension
  Keywords: leak
  Summary: <!-- Please insert a one line summary of the issue below -->

- **[#25259](https://github.com/zed-industries/zed/issues/25259)** (closed) — Memory leak in Elixir and Rails projects `area:performance`, `stale`
  Keywords: leak
  Summary: <img width="452" alt="Image" src="https://github.com/user-attachments/assets/d37cf81b-fae9-4e3f-bb0e-18fbe007cad5" />

- **[#25270](https://github.com/zed-industries/zed/issues/25270)** (open) — Emacs-style Kill Ring Support `area:editor`, `.contrib/good second issue`, `area:parity/emacs`
  Keywords: ring buffer
  Summary: Implement support for an emacs-style kill-ring buffer with support for multiple "clipboard" items (distinct from the OS clipboard).

- **[#25975](https://github.com/zed-industries/zed/issues/25975)** (closed) — Git Beta: Memory leak upon pushing `area:integrations/git`
  Keywords: leak
  Summary: When I push a commit, regardless of repo, my zed seizes into a memory leak.

- **[#27076](https://github.com/zed-industries/zed/issues/27076)** (closed) — Memory leak with zed window froze in Windows. `area:performance`, `platform:windows`
  Keywords: leak
  Summary: After leaving the Zed window and browsing some web pages, when I switched back to Zed, the entire window froze, and it showed the 'No Response' pro...

- **[#27414](https://github.com/zed-industries/zed/issues/27414)** (closed) — gpui: Image cache memory leak `area:performance`, `area:gpui`
  Keywords: leak
  Summary: GPUI image cache will leak memory when we load a lot of images to window.

- **[#27739](https://github.com/zed-industries/zed/issues/27739)** (closed) — Memory leak on Kubernetes/Nix project
  Keywords: leak
  Summary: <!-- Please insert a one line summary of the issue below -->

- **[#29391](https://github.com/zed-industries/zed/issues/29391)** (closed) — Zed crashing the display driver
  Keywords: leak
  Summary: On linux it seems to memory leak 100+GB. Navigation makes it exponentially worse.

- **[#29530](https://github.com/zed-industries/zed/issues/29530)** (closed) — Agent Panel: Zed leaks MCP servers `area:ai`
  Keywords: leak
  Summary: Description

- **[#30749](https://github.com/zed-industries/zed/issues/30749)** (open) — Generate commit message leaking unrelated code `area:integrations/git`, `area:ai`, `state:needs repro`
  Keywords: leak
  Summary: https://github.com/user-attachments/assets/9d68334f-7ea6-4690-99d0-97a7b47ca922

- **[#31180](https://github.com/zed-industries/zed/issues/31180)** (closed) — Memory leak when opening a file from a mounted Docker volume on MacOS
  Keywords: leak
  Summary: Memory leak when opening a file from a mounted Docker volume on MacOS

- **[#31461](https://github.com/zed-industries/zed/issues/31461)** (open) — Memory leak causing computer to crash `area:performance`, `state:needs repro`, `area:performance/memory leak`
  Keywords: leak
  Summary: Description

- **[#31637](https://github.com/zed-industries/zed/issues/31637)** (open) — High memory consumption in Project Search with large codebases `area:search`, `meta:large projects`, `area:performance/memory leak`
  Keywords: leak
  Summary: Massive memory leak during global search in large projects.

- **[#31791](https://github.com/zed-industries/zed/issues/31791)** (closed) — AI: Copilot memory leak (potential) `area:ai/copilot`, `meta:upstream`
  Keywords: leak
  Summary: Description

- **[#35549](https://github.com/zed-industries/zed/issues/35549)** (closed) — Huge memory leak on git conflict
  Keywords: leak
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#35894](https://github.com/zed-industries/zed/issues/35894)** (open) — Image viewer memory leak with BMP files. `area:performance`, `area:preview/images`, `state:needs repro`
  Keywords: leak
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#37280](https://github.com/zed-industries/zed/issues/37280)** (closed) — App hangs on startup with Hyprland/Wayland on Arch Linux (futex deadlock)
  Keywords: deadlock
  Summary: System Details:

- **[#37291](https://github.com/zed-industries/zed/issues/37291)** (open) — Insane memory and disk usage (leaks) `state:needs info`, `area:performance/memory leak`, `frequency:uncommon`
  Keywords: leak
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#38128](https://github.com/zed-industries/zed/issues/38128)** (closed) — Windows Alpha: memory leak `platform:windows`
  Keywords: leak
  Summary: The memory usage of the Windows version is 2543MB

- **[#38927](https://github.com/zed-industries/zed/issues/38927)** (open) — Find & Replace memory leak on large files `area:performance`, `meta:large projects`, `state:reproducible`
  Keywords: leak
  Summary: Find & Replace on 1 million occurrences works, but when finished still has memory usage at over 3.2GB (and though nothing crashed, my computer was ...

- **[#39776](https://github.com/zed-industries/zed/issues/39776)** (closed) — PHP Actor throws error after installation `area:languages/php`
  Keywords: actor
  Summary: After installing Zed Editor in Windows, throws and error

- **[#40013](https://github.com/zed-industries/zed/issues/40013)** (closed) — Severe memory leak on startup after latest update (Linux).
  Keywords: leak
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#40163](https://github.com/zed-industries/zed/issues/40163)** (closed) — Linux: Zed freezes and leaks memory on startup when a project contains a large `.img` file. `area:workspace`, `platform:linux`
  Keywords: leak
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#40351](https://github.com/zed-industries/zed/issues/40351)** (closed) — Memory leak and Cpu engaged `area:performance`, `platform:windows`
  Keywords: leak
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#40894](https://github.com/zed-industries/zed/issues/40894)** (closed) — Memory leak `platform:windows`
  Keywords: leak
  Summary: <!-- Please insert a one-line summary of the issue below -->

- **[#41240](https://github.com/zed-industries/zed/issues/41240)** (closed) — AI: AI conversation history leaks across projects (threads visible globally instead of scoped to active workspace) `area:ai`
  Keywords: leak
  Summary: Description

- **[#42015](https://github.com/zed-industries/zed/issues/42015)** (closed) — Zed Editor randomly fails to start due to NVIDIA driver deadlock on Linux startup
  Keywords: deadlock
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#42019](https://github.com/zed-industries/zed/issues/42019)** (closed) — Zed editor randomly fails to start due to NVIDIA driver deadlock on Linux startup `platform:linux/wayland`, `state:needs repro`, `graphics:nvidia`
  Keywords: deadlock
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#42290](https://github.com/zed-industries/zed/issues/42290)** (open) — Memory Leaking when copying large files to remote server `platform:remote`, `meta:large projects`, `state:reproducible`
  Keywords: leak
  Summary: <!-- Begin your issue with a one sentence summary -->

- **[#42447](https://github.com/zed-industries/zed/issues/42447)** (open) — Windows: Memory leak with SFTP folders `platform:windows`
  Keywords: leak
  Summary: I have a shared directory (via SFTP) that comes from my server hosted within my own network. It is a folder containing about 20 folders, most of wh...

- **[#43684](https://github.com/zed-industries/zed/issues/43684)** (open) — Memory leak in an F# project `area:performance/memory leak`, `frequency:uncommon`, `priority:P2`
  Keywords: leak
  Summary: Memory Leak (75GB+) with FProject and LSP

- **[#44518](https://github.com/zed-industries/zed/issues/44518)** (closed) — Memory leak in zed crashes whole system on Ubuntu 24.04
  Keywords: leak
  Summary: Reproduction steps

- **[#44641](https://github.com/zed-industries/zed/issues/44641)** (open) — Scroll event leaking outside buffer `area:controls/mouse`, `area:ui/tabs`, `stale`
  Keywords: leak
  Summary: Reproduction steps

- **[#45211](https://github.com/zed-industries/zed/issues/45211)** (closed) — [Bug] Gemini ACP extension leaks multiple zombie processes causing memory exhaustion (Not reproducible with Codex/Claude) `state:needs info`, `area:ai/gemini`, `frequency:uncommon`
  Keywords: leak
  Summary: Reproduction steps

- **[#45696](https://github.com/zed-industries/zed/issues/45696)** (closed) — Memory leak: Zed consumes 71GB+ RAM when opened on home directory (~/) causing system freeze and potential filesystem corruption `area:performance`, `state:reproducible`, `area:performance/memory leak`
  Keywords: leak
  Summary: Opening Zed on the home directory (~/) causes catastrophic memory consumption, eventually exhausting all system RAM, heavily utilizing swap, and re...

- **[#45922](https://github.com/zed-industries/zed/issues/45922)** (closed) — NT /11 pro memory leak - hangs on startup then crashes `state:needs repro`, `area:performance/memory leak`, `frequency:common`
  Keywords: leak
  Summary: Reproduction steps

- **[#46025](https://github.com/zed-industries/zed/issues/46025)** (open) — NT /11 WSL? pro memory leak OOM heap overruns `platform:windows`, `state:needs repro`, `area:performance/memory leak`
  Keywords: leak
  Summary: Reproduction steps

- **[#46741](https://github.com/zed-industries/zed/issues/46741)** (closed) — Memory Leak on Fedora Workstation 43 `area:performance`, `platform:linux`, `state:needs repro`
  Keywords: leak
  Summary: Reproduction steps

- **[#47325](https://github.com/zed-industries/zed/issues/47325)** (open) — Serious memory leaks still when Zed is left running for days `state:needs repro`, `area:performance/memory leak`, `frequency:common`
  Keywords: leak
  Summary: Reproduction steps

- **[#47432](https://github.com/zed-industries/zed/issues/47432)** (closed) — Maybe memory leak `area:performance`, `state:needs repro`, `frequency:common`
  Keywords: leak
  Summary: Reproduction steps

- **[#47612](https://github.com/zed-industries/zed/issues/47612)** (closed) — ACP mode leaks node processes `platform:windows`, `state:reproducible`, `area:ai/acp`
  Keywords: leak
  Summary: Reproduction steps

- **[#47796](https://github.com/zed-industries/zed/issues/47796)** (closed) — Rust: Memory leak with big statics `meta:duplicate`, `frequency:common`, `priority:P2`
  Keywords: leak
  Summary: Reproduction steps

- **[#48263](https://github.com/zed-industries/zed/issues/48263)** (open) — OOM error for long AI Agent conversations `state:needs repro`, `area:performance/memory leak`, `frequency:uncommon`
  Keywords: leak
  Summary: Reproduction steps

- **[#48968](https://github.com/zed-industries/zed/issues/48968)** (open) — Memory leak from rapid FS events in watched directory — 50GB consumption, swap exhaustion, WindowServer crash `state:needs repro`, `area:performance/memory leak`, `frequency:common`
  Keywords: leak
  Summary: Zed consumed ~50GB of memory (16GB physical + ~34GB swap) when a directory it had open contained a lock file being rapidly created/destroyed by an ...

- **[#49776](https://github.com/zed-industries/zed/issues/49776)** (open) — Severe file descriptor leak: 450k+ open /proc/[PID]/stat files after 21 hours `area:performance`, `state:needs repro`, `frequency:common`
  Keywords: leak
  Summary: Zed Memory Leak Bug Report

- **[#50151](https://github.com/zed-industries/zed/issues/50151)** (closed) — UI deadlock in window_did_change_key_status during window activation (macOS, multi-window) `platform:macOS`, `area:performance/memory leak`, `frequency:common`
  Keywords: deadlock
  Summary: Reproduction steps

- **[#50889](https://github.com/zed-industries/zed/issues/50889)** (closed) — Opening and closing Claude Agents leaks memory `state:reproducible`, `area:ai/anthropic`, `area:performance/memory leak`
  Keywords: leak
  Summary: Reproduction steps

- **[#51618](https://github.com/zed-industries/zed/issues/51618)** (closed) — Worktree/background scanner allocates infinite amount of RAM heap while indexing a workspace `state:needs triage`
  Keywords: leak
  Summary: Reproduction steps

- **[#51790](https://github.com/zed-industries/zed/issues/51790)** (open) — Terminal leaks ALACRITTY_WINDOW_ID env var, causing tools to misdetect terminal capabilities `area:integrations/terminal`, `state:needs repro`, `area:integrations/environment`
  Keywords: leak
  Summary: Zed's integrated terminal sets the `ALACRITTY_WINDOW_ID` environment variable (presumably because it uses Alacritty's rendering engine internally)....

### zellij

- **[#796](https://github.com/zellij-org/zellij/issues/796)** (closed) — Zellij leaks file descriptors to the running processes/shell `suspected bug`
  Keywords: leak
  Summary: **Basic information**

- **[#803](https://github.com/zellij-org/zellij/issues/803)** (closed) — Zellij is killed by OOM `suspected bug`
  Keywords: leak
  Summary: **Basic information**

- **[#1625](https://github.com/zellij-org/zellij/issues/1625)** (closed) — Memory leak `suspected bug`
  Keywords: leak
  Summary: Thank you for taking the time to file this issue! Please follow the instructions and fill in the missing parts below the instructions, if it is mea...

- **[#1789](https://github.com/zellij-org/zellij/issues/1789)** (closed) — high memory usage , zellij may has memory leak? `suspected bug`
  Keywords: leak
  Summary: <img width="896" alt="image" src="https://user-images.githubusercontent.com/7645818/195237500-8cd3862d-f6dc-489f-a217-6dff52f39d4c.png">

- **[#1948](https://github.com/zellij-org/zellij/issues/1948)** (closed) — Problems with repeatedly attaching to session - leaks memory and finally crashes
  Keywords: leak
  Summary: Just some stress testing with rapid attaching and detaching yielded this possible server crash locations:

- **[#3598](https://github.com/zellij-org/zellij/issues/3598)** (closed) — RAM and CPU usage grow unbounded
  Keywords: leak
  Summary: I have been observing this problem for a long time, but I struggle find a minimal reproduction, which is why I hesitated to open an issue. But the ...
