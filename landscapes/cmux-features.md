---
title: "cmux — Full Feature Reference (SUPERSEDED — DO NOT USE AS FOUNDATION)"
description: "Comprehensive command, API, and capability reference for cmux (manaflow-ai). Load-bearing document for orchestrator integration decisions."
research_date: 2026-05-04
last_verified: 2026-05-04
status: SUPERSEDED
superseded_at: 2026-05-05
superseded_by: "studio-research/landscapes/orchestration-layer-internals.md (to be authored under AO-264..AO-271 umbrella, AO-270 synthesis)"
superseded_reason: "Predates the paste-the-output discipline established in feedback_charter_before_ratify.md. Synthesized from cmux source-code reading + reasoning, not live --help/probe paste. S-67/S-68 dispatch sequence proved this style of synthesis breaks at first contact (claude-teams binary, ❯ glyph, REPL wait, Stop hook). Per S-69 directive: dump and rebuild from official sources first, with paste-the-output evidence for every load-bearing claim. This file is preserved for provenance + comparison value only."
staleness_warning: "SUPERSEDED — DO NOT USE AS FOUNDATION FOR THE ORCHESTRATOR REDESIGN. Refer to AO-264 + the synthesis doc at AO-270. Original staleness warning below: cmux is in active development. API surface expands with each minor version. Re-verify after any version bump. This doc covers v0.63.2 (latest as of research date) plus in-repo docs that reflect current main."
sources:
  - url: "https://github.com/manaflow-ai/cmux/blob/main/docs/cli-contract.md"
    date: 2026-05-04
    type: authoritative
  - url: "https://github.com/manaflow-ai/cmux/blob/main/docs/v2-api-migration.md"
    date: 2026-05-04
    type: authoritative
  - url: "https://github.com/manaflow-ai/cmux/blob/main/docs/notifications.md"
    date: 2026-05-04
    type: authoritative
  - url: "https://github.com/manaflow-ai/cmux/blob/main/docs/remote-daemon-spec.md"
    date: 2026-05-04
    type: authoritative
  - url: "https://github.com/manaflow-ai/cmux/blob/main/docs/agent-browser-port-spec.md"
    date: 2026-05-04
    type: authoritative
  - url: "https://github.com/manaflow-ai/cmux/blob/main/docs/feed.md"
    date: 2026-05-04
    type: authoritative
  - url: "https://github.com/manaflow-ai/cmux/blob/main/docs/dock.md"
    date: 2026-05-04
    type: authoritative
  - url: "https://github.com/manaflow-ai/cmux/blob/main/README.md"
    date: 2026-05-04
    type: authoritative
  - url: "https://github.com/manaflow-ai/cmux/releases"
    date: 2026-05-04
    type: authoritative
  - path: "studio-research/evaluations/cmux.md"
    date: 2026-04-29
    type: internal-prior-research
related:
  - evaluations/cmux.md
  - landscapes/agentic-substrate-distribution.md
---

# cmux — Full Feature Reference

> **Research date:** 2026-05-04
> **Last verified:** 2026-05-04
> **Version:** v0.63.2 (build 79, released 2026-04-06) — latest as of research date
> **Predecessor doc:** `evaluations/cmux.md` (2026-04-29) covers product identity, persistence model, BackendAdapter fit assessment, and known bugs. This doc extends it with the complete command surface, socket API schema, web/browser primitives, and orchestrator integration guidance.

---

## TL;DR

cmux exposes three integration surfaces for programmatic control:

1. **CLI** — 100+ commands covering windows, workspaces, panes, surfaces, browser, notifications, status, and VM. Full tmux-compat shim included.
2. **JSON-RPC v2 socket** — Handle-based protocol over a Unix socket. Every CLI command has a v2 method equivalent.
3. **Feed / hooks** — Agent hook bridge for permission approvals, plan decisions, and question answers — blocking, with 120s timeout.

**Web/browser panes: YES.** cmux has a full in-app browser (WKWebView) with a scriptable API ported from Vercel's `agent-browser`. Agents can open a URL in a split pane, snapshot the DOM, click, fill forms, evaluate JS, and take screenshots. All accessible via CLI (`cmux browser <subcommand>`) and socket API (`browser.*` methods).

**Named terminal window spawn: YES.** `cmux new-workspace --name <name> --cwd <path> --command <cmd>` creates a named sidebar workspace with an arbitrary command. `cmux new-window` creates a new macOS-level window. Both have v2 socket equivalents.

---

## 1. Overview

cmux is a native macOS terminal (Swift/AppKit + libghostty) built for multi-agent workloads. Its differentiation over standard terminals:

- Sidebar with vertical tabs showing git branch, PR status, cwd, listening ports, latest notification text per workspace
- Notification rings (pane illuminates when agent needs attention)
- In-app browser with agent automation API
- Feed: inline permission/plan approval panel in the right sidebar
- Dock: pinned TUI controls in the right sidebar
- Unix socket JSON-RPC v2 API for full programmatic control
- SSH workspaces with browser traffic proxied through the remote host

**Architecture:**
- Each macOS window contains one or more **workspaces** (sidebar entries)
- Each workspace contains one or more **panes** (split regions)
- Each pane contains one or more **surfaces** (horizontal tabs — terminal or browser)
- The automation target is always a `surface`

**Socket path:** `~/.config/cmux/cmux.sock` (canonical). Fallback: `/Users/<user>/Library/Application Support/cmux/cmux.sock`. The active path is written to `/tmp/cmux-last-socket-path`. Override via `CMUX_SOCKET_PATH` env var.

---

## 2. Full Command Reference

### 2.1 Global Flags

All commands accept these before the command name:

| Flag | Description |
|------|-------------|
| `--socket <path>` | Override socket path for this invocation |
| `--password <value>` | Socket password (overrides `CMUX_SOCKET_PASSWORD`) |
| `--json` | Machine-readable JSON output |
| `--id-format <refs\|uuids\|both>` | Handle format in JSON and supported text output |
| `--window <id\|ref\|index>` | Route through a specific window |

### 2.2 Environment Variables

| Variable | Description |
|----------|-------------|
| `CMUX_BUNDLED_CLI_PATH` | Path to the cmux binary (injected by cmux into child processes) |
| `CMUX_SOCKET_PATH` | Canonical socket path override |
| `CMUX_SOCKET` | Deprecated alias for `CMUX_SOCKET_PATH` (both set = CLI fails) |
| `CMUX_SOCKET_PASSWORD` | Socket password fallback |
| `CMUX_WORKSPACE_ID` | Default workspace context (injected by cmux into child shells) |
| `CMUX_SURFACE_ID` | Default surface context (injected by cmux into child shells) |
| `CMUX_TAB_ID` | Default tab context (injected by cmux into child shells) |
| `CMUX_CLAUDE_HOOK_CMUX_BIN` | cmux binary path for hook scripts |
| `CMUX_FEED_TUI_BUN_PATH` | Explicit Bun path for Feed TUI |
| `CMUX_FEED_TUI_LEGACY` | Set to `1` to force the old Feed TUI |

### 2.3 Window Commands

| Command | Description |
|---------|-------------|
| `list-windows` | List windows |
| `current-window` | Print the selected window ID |
| `new-window` | Create a new macOS-level window |
| `focus-window --window <id\|ref\|index>` | Focus a window |
| `close-window --window <id\|ref\|index>` | Close a window |
| `move-workspace-to-window` | Move a workspace into a target window |

### 2.4 Workspace Commands

Workspaces are sidebar entries (what most terminals call "tabs" at the window level).

| Command | Description |
|---------|-------------|
| `new-workspace [--name <name>] [--cwd <path>] [--command <cmd>] [--description <text>]` | Create a workspace. `--command` sends text+Enter after creation. |
| `list-workspaces` | List workspaces (returns refs + names + current_directory + custom_color) |
| `current-workspace` | Print current workspace info |
| `select-workspace` | Select (focus) a workspace |
| `close-workspace --workspace <id\|ref>` | Close a workspace |
| `rename-workspace` | Rename a workspace |
| `rename-window` | Compatibility alias for `rename-workspace` |
| `reorder-workspace` | Reorder workspace within its window |
| `workspace-action --action <name>` | Run workspace context-menu actions (see action names below) |
| `ssh <destination>` | Open an SSH-backed workspace |
| `remote-daemon-status` | Print bundled remote daemon version and cache status |

**Workspace action names:** `pin`, `unpin`, `rename`, `clear-name`, `set-description`, `clear-description`, `move-up`, `move-down`, `move-top`, `close-others`, `close-above`, `close-below`, `mark-read`, `mark-unread`, `set-color`, `clear-color`

### 2.5 Pane Commands

Panes are split regions within a workspace.

| Command | Description |
|---------|-------------|
| `list-panes` | List panes in a workspace |
| `focus-pane` | Focus a pane |
| `new-pane` | Create a pane with terminal or browser content |
| `new-split` | Split from a surface in a direction |
| `list-panels` | Compatibility alias over pane/surface data |
| `focus-panel` | Compatibility alias for surface focus |

### 2.6 Surface Commands

Surfaces are tabs within a pane (terminal or browser). The primary automation target.

| Command | Description |
|---------|-------------|
| `list-pane-surfaces` | List surfaces in a pane |
| `new-surface` | Create a surface inside a pane |
| `close-surface` | Close a surface |
| `move-surface` | Move a surface to another pane, workspace, window, or index |
| `split-off` | Move a surface into a new split without changing focus |
| `reorder-surface` | Reorder a surface within its pane |
| `drag-surface-to-split` | Move a surface into a split direction |
| `refresh-surfaces` | Ask the app to refresh terminal surfaces |
| `surface-health` | Print terminal surface health information |
| `debug-terminals` | Print debug terminal state |
| `tab-action --action <name>` | Run horizontal tab context-menu actions |
| `rename-tab` | Rename a tab (wrapper for `tab-action rename`) |
| `trigger-flash [--workspace <id>] [--surface <id>]` | Visual flash on a workspace or surface |

**Tab action names:** `rename`, `clear-name`, `close-left`, `close-right`, `close-others`, `new-terminal-right`, `new-browser-right`, `reload`, `duplicate`, `pin`, `unpin`, `mark-unread`

### 2.7 Send / Read Commands

| Command | Description |
|---------|-------------|
| `send --surface <id> --workspace <id> "<text>"` | Send text to a terminal surface (`\n` = Enter, `\t` = Tab) |
| `send-key --surface <id> <key>` | Send one key to a terminal surface |
| `send-panel` | Send text to a panel/surface (compatibility form) |
| `send-key-panel` | Send one key to a panel/surface (compatibility form) |
| `read-screen [--surface <id>] [--scrollback] [--lines N]` | Read terminal text from a surface |
| `capture-pane [--surface <id>] [--scrollback] [--lines N]` | tmux-compat alias for `read-screen` |

**Warning:** `send` does not wrap input in bracketed-paste markers. Multi-line content fires Return on each `\n`. For `claude -p "<prompt>"` dispatch (prompt as CLI argument), this is irrelevant. Only bites when injecting interactive multi-line input.

### 2.8 Notification / Status Commands

| Command | Description |
|---------|-------------|
| `notify --title <text> [--subtitle <text>] [--body <text>] [--tab <id>] [--panel <id>]` | Send a notification to a workspace/surface |
| `list-notifications` | List queued notifications |
| `clear-notifications` | Clear queued notifications |
| `set-status <key> <value> [--icon <name>] [--color <hex>]` | Set a sidebar status pill |
| `clear-status <key>` | Remove a sidebar status pill |
| `list-status` | List sidebar status pills |
| `set-progress <value>` | Set sidebar progress indicator |
| `clear-progress` | Clear sidebar progress |
| `log <text>` | Append a sidebar log entry |
| `clear-log` | Clear sidebar log entries |
| `list-log` | List sidebar log entries |
| `sidebar-state` | Dump full sidebar metadata state |

**Note on `set-status`:** Pills persist across cmux restarts (SessionPersistence). Clear stale pills at session start. Keys are workspace-scoped (confirmed by `--workspace` flag behavior). Multiple tools can manage their own keys without collision.

### 2.9 Browser Commands

cmux has a full in-app browser (WKWebView) with a Playwright-style automation API ported from Vercel's `agent-browser`. All commands route through `cmux browser <subcommand>`.

**Opening a browser pane:**
```bash
cmux browser open [--surface <id>]        # open browser in default split position
cmux browser open-split [--surface <id>]  # open in split (same as open)
cmux new-pane --type browser              # create a browser pane
```

**Navigation:**
```bash
cmux browser goto <url>                   # navigate to URL
cmux browser navigate <url>               # alias
cmux browser back
cmux browser forward
cmux browser reload
cmux browser url                          # print current URL
```

**Content inspection:**
```bash
cmux browser snapshot [--surface <id>]    # DOM accessibility tree snapshot with element refs
cmux browser eval "<js>"                  # evaluate JavaScript
cmux browser get text|html|value|attr|url|title|count|box|styles [selector]
cmux browser is visible|enabled|checked [selector]
cmux browser screenshot [--path <file>]
```

**Element interaction:**
```bash
cmux browser click <selector>
cmux browser dblclick <selector>
cmux browser hover <selector>
cmux browser focus <selector>
cmux browser fill <selector> <value>      # set input value
cmux browser type <selector> <text>       # type characters
cmux browser press <key>
cmux browser keydown <key>
cmux browser keyup <key>
cmux browser check <selector>
cmux browser uncheck <selector>
cmux browser select <selector> <value>
cmux browser scroll [selector] [x] [y]
cmux browser scroll-into-view <selector>
```

**Element locators:**
```bash
cmux browser find role|text|label|placeholder|alt|title|testid|first|last|nth <value>
```

**Browser management:**
```bash
cmux browser tab new|list|close|<index>
cmux browser frame <selector>|main
cmux browser dialog accept|dismiss [text]
cmux browser wait <selector|url|text|load-state|function>
cmux browser download wait|save
cmux browser console list|clear
cmux browser errors list|clear
cmux browser highlight <selector>
cmux browser state save|load
cmux browser cookies get|set|clear
cmux browser storage local|session get|set|clear
cmux browser viewport <width> <height>
cmux browser geolocation <lat> <lon>
cmux browser offline <true|false>
cmux browser network route|unroute|requests
cmux browser trace start|stop
cmux browser screencast start|stop
cmux browser input|input_mouse|input_keyboard|input_touch   # low-level input
cmux browser addinitscript|addscript|addstyle <content>     # inject scripts/CSS
cmux browser identify [--surface <id>]                       # identify browser surface context
```

**Legacy aliases** (kept for compatibility):
```bash
cmux open-browser       # → cmux browser open
cmux navigate           # → cmux browser navigate
cmux browser-back       # → cmux browser back
cmux browser-forward    # → cmux browser forward
cmux browser-reload     # → cmux browser reload
cmux get-url            # → cmux browser get-url
cmux focus-webview      # → cmux browser focus-webview
cmux is-webview-focused # → cmux browser is-webview-focused
```

**Browser enable/disable:**
```bash
cmux disable-browser    # disable browser creation and link interception
cmux enable-browser     # re-enable
cmux browser-status     # print current state (--json supported)
```

### 2.10 Tree / Introspection Commands

| Command | Description |
|---------|-------------|
| `tree [--all]` | Full hierarchy: windows > workspaces > panes > surfaces |
| `identify` | Print caller context: workspace_ref, surface_ref, pane_ref, window_ref, focused surface |
| `capabilities` | Print server capabilities as JSON (includes all v2 methods) |
| `ping` | Check socket connectivity |
| `rpc <method> [json-params]` | Call a raw v2 socket method |

### 2.11 tmux Compatibility Commands

All tmux-compat commands route through `__tmux-compat` internally.

| Command | Status |
|---------|--------|
| `capture-pane` | Functional — reads pane text |
| `resize-pane` | Functional |
| `pipe-pane` | Functional — pipe pane text to shell command |
| `wait-for` | Functional — signal/wait on named sync point |
| `swap-pane`, `break-pane`, `join-pane` | Functional |
| `next-window`, `previous-window`, `last-window` | Functional |
| `last-pane` | Functional |
| `find-window` | Functional — find workspace by title or content |
| `clear-history` | Functional |
| `set-hook` | Functional — tmux-compat hook definitions |
| `respawn-pane` | Functional — send restart command to a surface |
| `display-message` | Functional — print or display a message |
| `set-buffer`, `paste-buffer`, `list-buffers` | Functional |
| `popup` | Placeholder — unsupported |
| `bind-key`, `unbind-key`, `copy-mode` | Placeholders — unsupported |

### 2.12 VM / Cloud Commands

| Command | Description |
|---------|-------------|
| `vm ls` / `vm list` | List VMs |
| `vm new` / `vm create` | Create a VM (`--image`, `--provider`, `--detach`) |
| `vm shell` / `vm attach` | Open interactive shell for a VM |
| `vm rm` / `vm destroy` / `vm delete` | Destroy a VM |
| `vm ssh` / `vm ssh-info` | Print SSH connection info |
| `vm exec <cmd>` | Run a shell command inside a VM |
| `cloud` | Alias for `vm` |

### 2.13 Feed Commands

Feed is the inline permission/plan approval surface in the right sidebar.

```bash
cmux feed tui [--opentui] [--legacy]   # open Feed TUI in terminal
cmux feed clear [--yes]                # clear Feed history
```

**Hook installation for agent integrations:**
```bash
cmux hooks setup [--agent <name>] [--yes]          # install hooks for all agents on PATH
cmux hooks uninstall [--agent <name>] [--yes]       # remove hooks
cmux hooks claude <event>                           # handle Claude Code hook event from stdin
cmux hooks codex <event>                            # handle Codex hook event
cmux hooks feed --source <agent>                    # convert agent hook events to Feed context
cmux hooks <agent> install|uninstall                # per-agent install/uninstall
cmux claude-hook <event>                            # compat alias for hooks claude
```

**Supported agents for hooks:** Claude Code, Codex, Cursor, Gemini, Copilot, CodeBuddy, Factory, Qoder, OpenCode

### 2.14 Markdown / Utility Commands

| Command | Description |
|---------|-------------|
| `markdown open <path>` | Open a markdown file in a formatted viewer panel with live reload |
| `reload-config` | Ask cmux to reload configuration |
| `restore-session` | Restore the previously saved cmux session |
| `themes [list\|set\|clear]` | List, set, or clear Ghostty themes |
| `shortcuts` | Open Settings to Keyboard Shortcuts |
| `settings [open\|path\|docs\|target]` | Open Settings, print paths, or show docs |
| `docs [topic]` | Print canonical docs URLs for a topic |
| `auth status\|login\|logout` | Manage auth state |
| `claude-teams` | Launch Claude Code with cmux/tmux-style agent team integration |
| `omo` | Launch OpenCode with oh-my-openagent integration |
| `omx` | Launch Oh My Codex with cmux pane integration |
| `omc` | Launch Oh My Claude Code with cmux pane integration |

### 2.15 Dock Configuration

Dock pins TUI controls into the right sidebar. Not controlled via CLI commands — configured via JSON files:

- `.cmux/dock.json` in the project or a parent directory (repo-shared, trusted per-fingerprint)
- `~/.config/cmux/dock.json` (personal default, no trust gate)

Config schema:
```json
{
  "controls": [
    {
      "id": "git",
      "title": "Git",
      "command": "lazygit",
      "cwd": ".",
      "height": 300,
      "env": { "KEY": "value" }
    }
  ]
}
```

Fields: `id` (stable key), `title` (sidebar label), `command` (shell command), `cwd` (optional), `height` (optional, points), `env` (optional, non-secret only).

---

## 3. Socket / API Surface

### 3.1 Protocol

v2 JSON-RPC over Unix socket. Each request is one JSON object per line; each response is one JSON object per line.

**Request envelope:**
```json
{"id":"<string-or-number>","method":"<method>","params":{}}
```

**Success response:**
```json
{"id":"<echoed>","ok":true,"result":{...}}
```

**Error response:**
```json
{"id":"<echoed>","ok":false,"error":{"code":"<code>","message":"<message>"}}
```

**Error codes:** `not_found`, `invalid_params`, `invalid_state`, `not_supported`, `internal_error`

**Handle formats:** Short refs (`workspace:N`, `pane:N`, `surface:N`, `window:N`) are the default. UUIDs available via `--id-format uuids`. Short refs are global per daemon, monotonic, never reused until daemon restart.

**Raw invocation:** `cmux rpc <method> '<json-params>'`

### 3.2 Full v2 Method List

**System:**
```
system.ping
system.capabilities         → lists all methods + version
system.identify             → focused window/workspace/pane/surface + caller context
system.tree                 → full hierarchy
```

**Windows:**
```
window.list
window.current
window.focus                params: {window_id}
window.create
window.close                params: {window_id}
```

**Workspaces:**
```
workspace.list              returns: [{workspace_id, name, current_directory, custom_color, ...}]
workspace.create            params: {cwd, name}   returns: {workspace_id, workspace_ref}
workspace.select            params: {workspace_id}
workspace.current
workspace.close             params: {workspace_id}
workspace.move_to_window    params: {workspace_id, window_id}
workspace.remote.reconnect  params: {workspace_id}
workspace.remote.configure  params: {workspace_id, port?, local_proxy_port?, ssh_options?}
workspace.remote.status     returns: {remote, proxy metadata}
```

**Panes:**
```
pane.list                   params: {workspace_id?}
pane.focus                  params: {pane_id}
pane.surfaces               params: {pane_id}
pane.create                 params: {workspace_id, type: "terminal"|"browser"}
```

**Surfaces:**
```
surface.list                params: {workspace_id?, pane_id?}
surface.focus               params: {surface_id}
surface.split               params: {surface_id, direction}
surface.create              params: {pane_id, workspace_id, type: "terminal"|"browser"}
                            returns: {surface_id, surface_ref, pane_id, workspace_id, window_ref}
surface.close               params: {surface_id, workspace_id}
surface.drag_to_split       params: {surface_id, direction}
surface.move                params: {surface_id, pane_id|workspace_id|window_id, before_surface_id|after_surface_id|start|end}
surface.reorder             params: {surface_id, before_surface_id|after_surface_id}
surface.refresh
surface.health              returns: per-surface type + in_window status
surface.send_text           params: {surface_id, workspace_id, text}
surface.send_key            params: {surface_id, key}
surface.read_text           params: {surface_id, workspace_id, scrollback?: bool, lines?: int}
                            returns: {text}
surface.trigger_flash       params: {surface_id}
```

**Notifications:**
```
notification.create             params: {title, subtitle?, body?, workspace_id?}
notification.create_for_surface params: {surface_id, title, subtitle?, body?}
notification.create_for_target  params: {target, title, subtitle?, body?}
notification.list
notification.clear
```

**App:**
```
app.focus_override.set      params: {focus: bool}    (test use)
app.simulate_active                                  (test use)
```

**Browser:**
```
browser.open_split          params: {surface_id?, workspace_id?}
                            returns: {surface_id, pane_id, workspace_id, window_id, ...}
browser.navigate            params: {surface_id, url}
browser.back                params: {surface_id}
browser.forward             params: {surface_id}
browser.reload              params: {surface_id}
browser.url.get             params: {surface_id}  returns: {url}
browser.focus_webview       params: {surface_id}
browser.is_webview_focused  params: {surface_id}  returns: {focused: bool}
browser.snapshot            params: {surface_id}  returns: {snapshot: <accessibility tree with refs>}
browser.eval                params: {surface_id, expression}  returns: {result}
browser.wait                params: {surface_id, selector|url|text|load_state|function, timeout?}
browser.click               params: {surface_id, selector|ref}
browser.dblclick            params: {surface_id, selector|ref}
browser.hover               params: {surface_id, selector|ref}
browser.focus               params: {surface_id, selector|ref}
browser.fill                params: {surface_id, selector|ref, value}
browser.type                params: {surface_id, selector|ref, text}
browser.press               params: {surface_id, key}
browser.keydown             params: {surface_id, key}
browser.keyup               params: {surface_id, key}
browser.check               params: {surface_id, selector|ref}
browser.uncheck             params: {surface_id, selector|ref}
browser.select              params: {surface_id, selector|ref, value}
browser.scroll              params: {surface_id, selector?, x?, y?}
browser.scroll_into_view    params: {surface_id, selector|ref}
browser.get.*               (url, title, text, html, value, attr, count, box, styles)
browser.is.*                (visible, enabled, checked)
browser.screenshot          params: {surface_id, path?}
browser.find.*              (role, text, label, placeholder, alt, title, testid, nth, first, last)
browser.frame.select        params: {surface_id, selector}
browser.frame.main          params: {surface_id}
browser.dialog.respond      params: {surface_id, action: accept|dismiss, text?}
browser.download.wait       params: {surface_id}
browser.console.list        params: {surface_id}
browser.errors.list         params: {surface_id}
browser.highlight           params: {surface_id, selector|ref}
browser.state.save          params: {surface_id, path}
browser.state.load          params: {surface_id, path}
browser.cookies.get|set|clear
browser.storage.local|session.get|set|clear
browser.addinitscript|addscript|addstyle
browser.network.route|unroute|requests
browser.trace.start|stop
browser.screencast.start|stop
browser.input|input_mouse|input_keyboard|input_touch
```

**Feed (v2 socket verbs):**
```
feed.push                   params: {source, event_type, request_id, payload}
                            blocks up to 120s waiting for user decision
feed.permission.reply       params: {request_id, decision}
feed.question.reply         params: {request_id, answers}
feed.exit_plan.reply        params: {request_id, mode}
```

**Debug / test-only:**
```
debug.shortcut.set|simulate
debug.type                  simulate typing
debug.app.activate
debug.terminal.is_focused
debug.terminal.read_text
debug.terminal.render_stats
debug.layout
debug.bonsplit_underflow.*
debug.empty_panel.*
debug.notification.focus
debug.flash.*
debug.panel_snapshot.*
debug.window.screenshot
```

**v1 legacy protocol** (space-delimited, fully working, not documented here — prefer v2)

### 3.3 Socket Authentication

| Mode | Behavior |
|------|----------|
| Off | No socket available |
| cmux processes only (default) | Only processes launched by cmux can connect (fs-perm check on socket ownership) |
| Automation mode | Any process running as the same macOS user |
| Password mode | Requires socket password |
| Full open access | No restrictions |

Password precedence: `--password` flag > `CMUX_SOCKET_PASSWORD` env > `~/Library/Application Support/cmux/socket-control-password` file.

Socket permissions: `srw-------` (0600, owner-only).

---

## 4. Web Window / Browser Pane Support

**Answer: YES — full browser pane support.**

cmux has an integrated WKWebView browser with a scriptable API ported from Vercel's `agent-browser`. Key facts:

- Browser panes are first-class `surface` objects alongside terminal surfaces
- A browser surface is opened in a split pane next to the terminal
- The browser is NOT a separate window — it lives in the same cmux workspace hierarchy
- All browser surfaces are accessible to the v2 socket API via `surface_id`

**How to open a browser pane:**
```bash
# CLI: open browser split adjacent to current terminal
cmux browser open
# or via keyboard: ⌘⇧L

# CLI: create a new pane of type browser
cmux new-pane --type browser

# CLI: open-split in a specific workspace
cmux browser open --workspace workspace:1

# v2 socket:
{"id":"1","method":"browser.open_split","params":{"workspace_id":"workspace:1"}}
# returns: {surface_id, pane_id, workspace_id, window_id, ...}
```

**Then navigate:**
```bash
cmux browser goto https://localhost:3000
# or via socket:
{"id":"2","method":"browser.navigate","params":{"surface_id":"surface:3","url":"https://localhost:3000"}}
```

**Browser placement policy (automatic):** Without an explicit target, placement is caller-relative — reuse the nearest right sibling pane; if none exists, split right from the caller pane. Explicit `surface_id` forces deterministic targeting.

**SSH workspaces + browser:** In SSH workspaces, browser traffic is automatically routed through the remote host via a local SOCKS5 proxy tunnel to `cmuxd-remote`. Localhost URLs from within an SSH workspace resolve on the remote host, not the local machine.

**`import-browser-cookies` analog:** cmux can import cookies, history, and sessions from Chrome, Firefox, Arc, and 20+ browsers so browser panes start authenticated. This is a UI feature, not a CLI command.

**Browser API maturity:** All P0 and P1 methods from the `agent-browser` port are implemented and tested (`tests_v2/test_browser_api_p0.py`, `test_browser_api_comprehensive.py`, `test_browser_api_extended_families.py`). P2 (network interception, emulation settings, trace/video/screencast, har) is partially implemented; unsupported subcommands return `not_supported` error.

**Key agent workflow pattern:**
```bash
# 1. Identify where you are
cmux identify

# 2. Open browser split
BROWSER_SURFACE=$(cmux rpc browser.open_split '{"workspace_id":"workspace:1"}' | jq -r '.result.surface_id')

# 3. Navigate
cmux browser goto https://localhost:3000 --surface "$BROWSER_SURFACE"

# 4. Snapshot DOM (get element refs for interactions)
cmux browser snapshot --surface "$BROWSER_SURFACE"

# 5. Interact using element refs from snapshot
cmux browser click "@e5" --surface "$BROWSER_SURFACE"

# 6. Verify
cmux browser screenshot --path /tmp/after-click.png --surface "$BROWSER_SURFACE"
```

---

## 5. Terminal Window / Workspace Spawn

**Answer: YES — full named workspace spawn with arbitrary commands.**

### 5.1 CLI Spawn

```bash
# Create a named workspace with a command
cmux new-workspace \
  --name "agent-build-123" \
  --cwd /path/to/worktree \
  --command "claude -p 'implement feature X'"
```

- Returns control immediately (stdout is empty on success)
- `--command` sends text + Enter into the new surface after creation
- The workspace appears in the sidebar with the given name

**Caveat:** `new-workspace` does not return the workspace ID on stdout. Retrieve it afterward:
```bash
cmux list-workspaces --json | jq '.[] | select(.name == "agent-build-123") | .workspace_id'
```

### 5.2 v2 Socket Spawn (recommended for orchestration)

```json
// Step 1: Create workspace
{"id":"1","method":"workspace.create","params":{"cwd":"/path/to/worktree","name":"agent-build-123"}}
// Response: {"ok":true,"result":{"workspace_id":"<uuid>","workspace_ref":"workspace:7"}}

// Step 2: Get the surface that was auto-created
{"id":"2","method":"pane.surfaces","params":{"pane_id":"<pane-id-from-workspace>"}}

// Step 3: Send command to the surface
{"id":"3","method":"surface.send_text","params":{"surface_id":"surface:14","workspace_id":"workspace:7","text":"claude -p 'implement feature X'\n"}}
```

**Sharp edge — dead PTY on background workspaces:** When a workspace is created programmatically and not immediately focused, `ghostty_surface_t` (the PTY object) may not be initialized. `read-screen` or `send-key` calls return `"internal_error: Surface not ready"`. Workaround: focus the workspace for ~1s after creation, then switch back. GitHub issue #1472 — open.

### 5.3 New Window

```bash
cmux new-window                              # create a new macOS window
cmux focus-window --window window:1          # focus a specific window
cmux close-window --window window:1          # close a window
```

v2 socket:
```json
{"id":"1","method":"window.create","params":{}}
{"id":"2","method":"window.focus","params":{"window_id":"<id>"}}
```

### 5.4 Claude Code Teams Mode

```bash
cmux claude-teams
```

Runs Claude Code's teammate mode with one command. Spawns teammates as native cmux splits with sidebar metadata and notifications. No tmux required.

---

## 6. Feed / Permission Approval Bridge

Feed is the inline permission/plan/question approval panel for AI agents. It is the primary mechanism for human-in-the-loop control without leaving the terminal.

**What Feed handles:**
- **PermissionRequest** — Agent wants to run a tool, edit a file, execute a shell command. Options: Once / Always / All tools / Bypass / Deny.
- **ExitPlanMode** — Agent finished planning and is ready to edit. Options: Ultraplan / Manual / Auto.
- **AskUserQuestion** — Agent's multiple-choice question. Pick answers and submit.

**Mechanism:** Agent hooks pipe events to `cmux hooks feed --source <agent>`. Feed blocks the hook for up to 120s. User decision unblocks it; hook returns the decision JSON to the agent.

**Supported agents:** Claude Code, Codex, Cursor, Gemini, Copilot, CodeBuddy, Factory, Qoder, OpenCode

**Storage:**
- Audit log: `~/.cmuxterm/workstream.jsonl` (append-only, all events)
- Session mapping: `~/.cmuxterm/<agent>-hook-sessions.json` (for "jump to agent terminal" from Feed)
- Socket: `~/.config/cmux/cmux.sock`

---

## 7. Remote / SSH Support

`cmux ssh <destination>` creates a workspace for a remote machine. The remote daemon (`cmuxd-remote`) is auto-uploaded and bootstrapped. Key capabilities:

- **CLI relay from remote:** Running `cmux ping`, `cmux list-workspaces`, `cmux new-workspace`, `cmux rpc ...` from within an SSH session works — the daemon reverse-forwards a TCP port so CLI commands from the remote shell reach the local cmux socket
- **Browser proxy:** Browser panes in SSH workspaces route traffic through the remote host via SOCKS5 tunnel through `cmuxd-remote`. Localhost URLs resolve on the remote.
- **PTY resize:** "smallest screen wins" semantics across multiple attachments
- **Reconnect:** Context menu "Reconnect Workspace(s)" / `workspace.remote.reconnect` API

**What remote does NOT support:**
- `cmux read-screen` targeting remote surfaces from the local machine (surface control is local-only; the CLI relay runs on the remote and controls the local socket)
- Full two-way remote surface control as first-class orchestration targets (GitHub issue #2673, open)

---

## 8. Gaps / Unknowns

| Gap | Status | Notes |
|-----|--------|-------|
| Background PTY initialization (GitHub #1472) | Open | Created-but-not-focused workspaces get `internal_error: Surface not ready`. Workaround: focus for ~1s. |
| `read-screen` after display sleep (GitHub #2004) | Open | `caffeinate -i` not enough; use `caffeinate -d` |
| Non-ASCII chars in `send_text` (GitHub #2756) | Open | Key-event path may drop non-ASCII; use ASCII-safe prompts or the `-p` CLI arg |
| Remote surface control from local (GitHub #2673) | Open | `cmuxd-remote` exists but not wired for two-way surface control |
| `surface.read_text` streaming / subscribe | Unknown | Not in capabilities list; polling is currently the only read pattern |
| `set-status` cross-workspace race | Confirmed isolated | Keys are workspace-scoped; parallel orchestrators in different workspaces do not collide |
| Founder's Edition / commercial pricing | Not publicly disclosed | Contact `founders@manaflow.com` |
| P2 browser network interception in WKWebView | Partial | Some subset feasible; unsupported subcommands return `not_supported` |
| `popup` tmux-compat command | Placeholder | Listed in CLI contract as unsupported |
| `bind-key`, `unbind-key`, `copy-mode` | Placeholder | Listed as unsupported |

---

## 9. Implications for Orchestrator Integration

### 9.1 Spawn Pattern (recommended)

Use the v2 socket for orchestration — it returns workspace/surface IDs directly:

```bash
# Create workspace
WS=$(cmux rpc workspace.create '{"cwd":"/path","name":"builder-AO-999"}' | jq -r '.result.workspace_ref')

# Wait for PTY (workaround for #1472 if creating unfocused workspaces)
cmux rpc workspace.select "{\"workspace_id\":\"$WS\"}"
sleep 1
cmux rpc workspace.select '{"workspace_id":"workspace:1"}'  # refocus original

# Send command
cmux rpc surface.send_text "{\"surface_id\":\"surface:N\",\"workspace_id\":\"$WS\",\"text\":\"claude -p 'task'\n\"}"
```

### 9.2 Status Pattern

Use `set-status` pills for live agent phase tracking — each workspace gets its own pills:

```bash
cmux set-status build_phase "planning" --icon hammer --color "#ff9500"
# ... later ...
cmux set-status build_phase "done" --icon checkmark --color "#30d158"
cmux clear-status build_phase   # at session end
```

### 9.3 Completion Detection Pattern

cmux has no native process-alive signal. Two patterns:

1. **Notification relay** (cleaner): Agent hooks fire `cmux notify` on completion. Orchestrator polls `cmux rpc notification.list '{}'`.
2. **Screen polling** (fallback): `cmux rpc surface.read_text` + parse shell prompt appearance.

### 9.4 Browser Integration Pattern

For agents that need to verify web output:

```bash
# Open browser pane in the agent's workspace
BROWSER_SURFACE=$(cmux rpc browser.open_split "{\"workspace_id\":\"$WS\"}" | jq -r '.result.surface_id')
cmux rpc browser.navigate "{\"surface_id\":\"$BROWSER_SURFACE\",\"url\":\"http://localhost:3000\"}"

# Snapshot for agent context
cmux rpc browser.snapshot "{\"surface_id\":\"$BROWSER_SURFACE\"}"
```

### 9.5 Feed Integration Pattern

For permission-gated agent workflows, install Feed hooks once and leave them active:

```bash
cmux hooks setup --agent claude   # one-time per machine
```

Feed then intercepts PermissionRequest events without any orchestrator code changes. The orchestrator does not need to poll or handle approvals — Feed is the human-in-the-loop layer.

### 9.6 Layer Fit

| Orchestrator layer | cmux fit | Reasoning |
|-------------------|----------|-----------|
| Layer 1 local-interactive (Ali watching short-lived builders) | Correct fit | Rich visibility (sidebar pills, notification rings, Feed), workspace spawn, browser panes |
| Layer 2 always-on fleet (survives disconnects) | Wrong alone | No detach/reattach; needs tmux inside a cmux pane for persistence |
| Layer 3 cloud-persistent | Wrong | macOS-only; use api-driven backend |

cmux is the **environment layer** (observability, identity, approval UI), orthogonal to which terminal-ops backend is used for process dispatch. Even if Layer 1 dispatches via subprocess-headless, it can publish `set-status` pills to the cmux sidebar using `CMUX_BUNDLED_CLI_PATH`.

---

## Key Sources

| Source | URL / Path | Date |
|--------|-----------|------|
| CLI contract (authoritative) | `github.com/manaflow-ai/cmux/blob/main/docs/cli-contract.md` | 2026-05-04 |
| v2 API migration + method list | `github.com/manaflow-ai/cmux/blob/main/docs/v2-api-migration.md` | 2026-05-04 |
| Browser port spec | `github.com/manaflow-ai/cmux/blob/main/docs/agent-browser-port-spec.md` | 2026-05-04 |
| Notifications doc | `github.com/manaflow-ai/cmux/blob/main/docs/notifications.md` | 2026-05-04 |
| Remote daemon spec | `github.com/manaflow-ai/cmux/blob/main/docs/remote-daemon-spec.md` | 2026-05-04 |
| Feed doc | `github.com/manaflow-ai/cmux/blob/main/docs/feed.md` | 2026-05-04 |
| Dock doc | `github.com/manaflow-ai/cmux/blob/main/docs/dock.md` | 2026-05-04 |
| README | `github.com/manaflow-ai/cmux/blob/main/README.md` | 2026-05-04 |
| Capability profile (prior research) | `studio-research/evaluations/cmux.md` | 2026-04-29 |
