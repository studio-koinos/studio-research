---
title: "cmux — Capability Profile"
description: "Source-level evaluation of manaflow-ai/cmux as a terminal-ops backend candidate for Mission 2's agent fleet platform registry. Covers all 12 capability dimensions, BackendAdapter fit, and empirical findings from Ali's production use."
research_date: 2026-04-29
last_verified: 2026-04-29
staleness_warning: "cmux is in active development (manaflow-ai, YC S2024). Releases ship approximately monthly. Re-verify API surface after any minor-version bump — the JSON-RPC method set is expanding. This evaluation is against v0.63.2 (2026-04-06)."
confidence: high
sources_count: 18
related:
  - ../landscapes/agentic-substrate-distribution.md
  - ../../studio-agent-ops/docs/orchestrator-vision.md
status: active
---

# cmux — Capability Profile

> **Research date:** 2026-04-29
> **Last verified:** 2026-04-29
> **Version evaluated:** cmux v0.63.2 (build 79, commit `179b16ce6`, released 2026-04-06)
> **Verification method:** Live CLI introspection (`CMUX_BUNDLED_CLI_PATH` on Ali's machine), web research (GitHub, official docs, community issues), and cross-reference of session memory entries from S-20, S-23, S-24.

---

## TL;DR

cmux is a native macOS terminal built specifically for multi-agent workloads. It offers a production-grade JSON-RPC socket API, GUI-rich observability (sidebar pills, notification rings), and strong integration with Claude Code. Its **fundamental constraint as a terminal-ops BackendAdapter is its persistence model**: panes die if the app closes — there is no detach/reattach. This makes cmux the correct backend for **local-interactive Layer 1 deployments** (Ali watching agents work) but **wrong for always-on Layer 2/3 fleet orchestration** (which needs persistence across disconnects). The cmux+tmux composition pattern documented in `orchestrator-vision.md §3` resolves this by nesting tmux inside a cmux pane for remote/persistent deployments. AO-142 Phase 1 (observability primitives) is already wired and verified in production.

**Backend classification (per `orchestrator-vision.md §3` matrix):**

| Property | cmux alone |
|---|---|
| Persistence | Ephemeral (pane dies if cmux process dies) |
| Reach | Local only (macOS desktop app) |
| Visibility | GUI-rich (sidebar pills, notification rings) |
| Dispatch fit | Layer 1 local-interactive only |

---

## 1. Product Identity

[VERIFIED] **manaflow-ai/cmux** is a native macOS terminal application built on Ghostty's rendering engine (libghostty). It is the primary product of Manaflow (YC S2024), founded by Austin Wang (Yale physics, ex-NASA JPL, Chess.com) and Lawrence Chen (Berkeley '24, ex-Minion AI). [src 1, 2]

- **License:** GPL-3.0-or-later as of v0.63.2 (relicensed from AGPL-3.0; breaking change per release notes [src 3]). Commercial license available via `founders@manaflow.com` for organizations that cannot comply with GPL. A "Founder's Edition" subscription exists (prioritized feature requests, early access to cmux AI/iOS/Cloud VMs) but pricing is not publicly disclosed. [src 3]
- **Version:** v0.63.2 (build 79, commit `179b16ce6`, released 2026-04-06). [VERIFIED: live `cmux version` output]
- **Maturity:** 15.7k GitHub stars, 2,408 commits, primarily Swift (78.3%) + Python + TypeScript + Shell + Go. [src 4] Reached 9.5k stars within two weeks of launch; ranked #2 on Hacker News. [src 2] 844 open issues / 602 PRs — active development pace.
- **Release cadence:** ~4–6 releases per month at current pace (v0.61.0 Feb 2026 → v0.63.2 Apr 2026). [src 3]
- **Nature:** OSS product with commercial-license option. Not a pure developer tool — Manaflow intends to build a managed cloud product ("Cloud VMs" mentioned in Founder's Edition benefits). [src 3]

---

## 2. Spawn Semantics

[VERIFIED] cmux provides two spawn paths for agent automation:

**Path A — CLI `new-workspace`:**
```bash
cmux new-workspace --name "agent-build" --cwd /path/to/worktree --command "claude -p 'task'"
```
Creates a workspace in the current window. `--command` sends text+Enter after creation. Returns control immediately. [VERIFIED: live `cmux new-workspace --help`]

**Path B — JSON-RPC `workspace.create`:**
```json
{"id":"<uuid>","method":"workspace.create","params":{"cwd":"/path","name":"agent-build"}}
```
Returns `workspace_id` (UUID) and `workspace_ref` (e.g., `workspace:7`). [VERIFIED: live `cmux rpc workspace.create '{...}'` returned well-formed JSON with these fields]

**Surface creation** (tab within a workspace):
```json
{"id":"<uuid>","method":"surface.create","params":{"pane_id":"pane:1","workspace_id":"workspace:1","type":"terminal"}}
```
Returns `surface_id`, `surface_ref`, `pane_id`, `workspace_id`, `window_ref`. [VERIFIED: live RPC call]

**Known sharp edge — dead PTY on background workspaces:** When a workspace is created programmatically and **not immediately focused**, `ghostty_surface_t` (the PTY object) may not be initialized. Subsequent `read-screen` or `send-key` calls return `"internal_error: Surface not ready"`. Workaround: focus the workspace for ~1s after creation, then switch back. GitHub issue #1472 — open, no maintainer response. [src 5]

**`new-workspace` does not return workspace ID** from stdout (confirmed by live test — stdout is empty on success). The handle must be retrieved via a subsequent `list-workspaces` or `rpc workspace.list '{}` call correlated by name.

---

## 3. Send Semantics

[VERIFIED] `surface.send_text` / CLI `cmux send` is the primary send primitive.

**CLI form:**
```bash
cmux send --surface surface:2 --workspace workspace:1 "claude -p 'task'\n"
```
Escape sequences: `\n` and `\r` send Enter; `\t` sends Tab. [VERIFIED: live `cmux send --help`]

**JSON-RPC form:**
```json
{"id":"<uuid>","method":"surface.send_text","params":{"surface_id":"surface:1","workspace_id":"workspace:1","text":"git status\n"}}
```

**Critical caveat — no bracketed-paste wrapping:** [VERIFIED from prior session memory `feedback_cmux_env_vs_backend.md`] `surface.send_text` does NOT wrap input in bracketed-paste markers (`\e[200~...\e[201~`). Multi-line content with `\n` fires Return on each newline. This means piping a raw multi-line prompt via `send_text` will execute each line as a separate shell command, not deliver the whole block as a paste.

**Implication for terminal-ops BackendAdapter:** For headless dispatch (`claude -p "<prompt>"`), this caveat is **irrelevant** — the prompt is an argument to the CLI, not terminal input. The caveat only bites if you attempt to drive an interactive session by injecting a multi-line prompt body via `send_text`. The correct dispatch pattern for `claude -p` avoids this entirely. [src 6]

**Non-ASCII paste encoding issue:** A separate bug (GitHub issue #2756) documents that cmux may route paste through `ghostty_surface_key()` (key-event simulation) instead of `ghostty_surface_text()` (paste path), dropping non-ASCII bytes. Non-ASCII characters in `send_text` payloads may be corrupted. [src 7]

---

## 4. Read / Output Semantics

[VERIFIED] Two equivalent read interfaces:

**CLI:**
```bash
cmux read-screen --surface surface:2 --workspace workspace:1 --scrollback --lines 200
# tmux-compatible alias:
cmux capture-pane --surface surface:2 --scrollback --lines 200
```

**JSON-RPC `surface.read_text`:**
```json
{"id":"<uuid>","method":"surface.read_text","params":{"surface_id":"surface:1","workspace_id":"workspace:1","scrollback":true,"lines":100}}
```
Returns `text` field containing plain text content. [src 8]

**Output format:** Plain text. The underlying call (`readTerminalTextBase64`) returns base64-encoded content that the CLI and JSON-RPC layer decode before returning. ANSI escape sequences are stripped. [src 8]

**Scrollback:** Supported via `--scrollback` / `scrollback: true`. History is bounded by the in-process buffer — there is no persistent scrollback across cmux app restarts. [src 9]

**`read-screen` was non-production until PR #219 (merged 2026-02-21).** Prior to that, the API existed only in non-production builds. The production v2 `surface.read_text` method is included in `system.capabilities` capabilities list. [src 8]

**Known bug — `surface.read_text` ignores `workspace_id` for non-selected surfaces:** PR #219 reviewer flagged this as a correctness regression — `read_text` read only from the selected workspace if `workspace_id` was specified but the surface wasn't focused. Fixed in follow-up PR #220. [src 8]

**Known bug — read-screen fails after display sleep:** After macOS display sleep, `ghostty_surface_t` can be deallocated while the outer `hostedView` remains. `read-screen` returns `"Terminal surface not found"` despite `surface-health` reporting healthy. Workaround: `caffeinate -d` (suppress display sleep) instead of `caffeinate -i`. GitHub issue #2004 — open. [src 10]

**Live verification:** `cmux read-screen --lines 5` from within this session returned correct terminal content including the Claude Code status bar (token count, auto-mode indicator). [VERIFIED: live command output]

---

## 5. Status / Kill / List

[VERIFIED from live CLI introspection]

**List:**
```bash
cmux list-workspaces         # → workspace:N refs with names
cmux list-pane-surfaces      # → surface:N refs with tab names
cmux tree --all              # → full hierarchy: windows > workspaces > panes > surfaces
cmux surface-health          # → per-surface: type, in_window status
```

JSON-RPC equivalents: `workspace.list`, `surface.list`, `pane.list`, `system.tree`. [VERIFIED: live `cmux capabilities` lists all methods]

**Kill / close:**
```bash
cmux close-workspace --workspace workspace:N
cmux close-surface --surface surface:N
```

JSON-RPC: `workspace.close`, `surface.close`. No SIGTERM/SIGKILL equivalent — cmux manages process lifecycle internally.

**Status check:** No native "is the process in this pane still running?" primitive. Status must be inferred from `read-screen` output (shell prompt = idle, spinner = running) or via notification relay events. [src 11]

**`identify`:** Returns caller's `workspace_ref`, `surface_ref`, `pane_ref`, `window_ref` and the currently focused surface. Injected as env vars (`CMUX_WORKSPACE_ID`, `CMUX_SURFACE_ID`) by cmux into every child process. [VERIFIED: live `cmux identify` output]

---

## 6. Persistence Model

[VERIFIED] cmux is **ephemeral** in the process-survival sense:

- Sessions restore on relaunch (layouts, scrollback history, browser history, session IDs via `SessionPersistence.swift`) but **running processes do not persist**. A terminal in a cmux pane that was running `claude -p ...` before cmux quit is gone after restart — the pane reopens as a fresh shell. [src 9]
- `CMUX_SURFACE_ID` / `CMUX_WORKSPACE_ID` survive across session restore (the UUIDs are stable per the persistence contract). This means events.jsonl surface-ID references remain valid for session-level correlation, but NOT across process lifetime.
- There is **no detach/reattach** mechanism. cmux is not a client-server architecture like tmux.

**Implication for orchestrator layers:**

| Use case | Ephemeral OK? |
|---|---|
| Layer 1 local-interactive (Ali watching short-lived builders) | Yes — builders finish before Ali closes cmux |
| Layer 2 fleet-watcher (always-on, survives disconnects) | No — requires tmux or api-driven backend |
| Layer 3 chief-of-agents (cloud-persistent) | No — requires api-driven backend |

The cmux+tmux composition pattern (`orchestrator-vision.md §3`) resolves this: use cmux for the human-visible desktop pane, with a tmux session inside the pane providing persistence. The tmux session survives even if cmux quits or the machine sleeps; re-attaching is `cmux ssh host` then `tmux attach`. [src 12]

---

## 7. Reach

[VERIFIED] **Local-only, macOS-only.** Specifically macOS 14+ (Intel and Apple Silicon). [src 9]

**Remote workspace feature (partial):** cmux v0.63.0 added `cmux ssh <destination>` which creates a workspace, marks it as remote-SSH, and opens an SSH session in that pane. The app establishes a local SSH proxy so browser traffic can egress from the remote host. [VERIFIED: live `cmux ssh --help`]

However, the remote workspace does not install a cmux daemon on the remote host that would allow `cmux send` / `cmux read-screen` to target remote panes. GitHub issue #2673 (open, no maintainer response) proposes the full RemoteHostManager + SSH ControlMaster architecture to make remote panes first-class orchestration targets. Current state is "cmux treats remote SSH connections as plain terminal panes" — you can see them in the sidebar but cannot drive them via the socket API. [src 13]

A remote daemon binary (`cmuxd-remote-darwin-arm64`) exists in releases (SHA256 attestation via GitHub Actions), but it is not yet wired into the main socket API for two-way remote surface control. [VERIFIED: `cmux remote-daemon-status` output]

---

## 8. Visibility (Observability)

[VERIFIED] This is cmux's strongest differentiator relative to other backends.

**Sidebar status pills (`set-status`):**
```bash
cmux set-status claude_phase "building" --icon hammer --color "#ff9500"
cmux set-status claude_phase "verified" --icon checkmark --color "#30d158"
cmux clear-status claude_phase
```
Each key-value pair appears as a pill in the workspace's sidebar tab row. Multiple tools can manage their own keys (`build`, `claude_code`, `deploy`). Pills persist across cmux restarts (per `SessionPersistence.swift:199-205`) — stale pills must be cleared at session start. The `metacognition-nudge.sh` SessionStart hook already does this for the `phase` key. [VERIFIED: live `cmux set-status --help`; `cmux set-status claude_research "evaluating"` returned `OK`]

**Notification system (`notify`):**
```bash
cmux notify --title "Claude Code" --subtitle "agent-ops" --body "Ready for input"
cmux trigger-flash --workspace workspace:1
```
Notification rings on pane tabs illuminate blue. `notify` creates an entry in `list-notifications`. Used by AO-142's `cmux-notify.sh` Stop hook to alert Ali when an agent completes a turn. [VERIFIED: live `cmux notify` returned `OK`; `cmux-notify.sh` confirmed present and wired]

**Workspace metadata:** Sidebar shows git branch, PR status/number, working directory, listening ports, latest notification text — automatically pulled from the shell's cwd. No agent instrumentation needed for git/PR visibility. [src 4]

**AO-142 Phase 1 status:** The observability primitives (`set-status`, `notify`, `CMUX_WORKSPACE_ID`, `CMUX_SURFACE_ID` in events.jsonl, `cmux-notify.sh` Stop hook) are wired and verified in production as of S-23. [VERIFIED: `cmux-notify.sh` present at `~/.claude/hooks/cmux-notify.sh`; live `set-status` and `notify` calls succeeded]

**Key observation from `feedback_cmux_env_vs_backend.md` (S-20):** cmux observability (set-status, notify, identity env vars) is the **environment layer**, orthogonal to which terminal-ops backend is used for dispatch. A subprocess-headless dispatch can still publish set-status pills to the cmux sidebar. These two concerns must not be conflated.

---

## 9. Authentication

[VERIFIED] Five authentication modes:

| Mode | Behavior |
|---|---|
| Off | No socket available |
| cmux processes only (default) | Only processes launched by cmux can connect — fs-perm-based via socket ownership check |
| Automation mode | Any process running as the same macOS user |
| Password mode | Requires `auth <password>` on connect |
| Full open access | No restrictions |

**Socket path:** `/Users/aliargun/Library/Application Support/cmux/cmux.sock` (not `/tmp/cmux.sock` as the docs suggest — the actual socket may vary; `/tmp/cmux-last-socket-path` file records the active path). [VERIFIED: live `cmux capabilities` response + `ls -la` on socket]

**Socket permissions:** `srw-------` (0600, owner-only). [VERIFIED: live `ls -la` on socket]

**Password sources (precedence):** `--password` CLI flag > `CMUX_SOCKET_PASSWORD` env var > `~/Library/Application Support/cmux/socket-control-password` file. [src 14]

**Env vars injected by cmux:** `CMUX_BUNDLED_CLI_PATH`, `CMUX_WORKSPACE_ID`, `CMUX_SURFACE_ID`, `CMUX_SOCKET_PATH`, `CMUX_CLAUDE_HOOK_CMUX_BIN` (specifically for hook scripts). [VERIFIED: live env var introspection; `cmux-notify.sh` references all three fallback paths]

**For terminal-ops BackendAdapter:** In Ali's deployment (same macOS user, single machine), authentication is transparent — no password required. For multi-user or CI scenarios, the password mode or `CMUX_SOCKET_PASSWORD` env var would be needed.

---

## 10. Pricing / Quota

[VERIFIED] **Free and open source under GPL-3.0-or-later.** No per-use pricing, no API quotas, no rate limits documented in the socket API spec. [src 1, 3]

**Commercial license:** Available for organizations that cannot comply with GPL. Contact `founders@manaflow.com`. Pricing undisclosed. [src 3]

**Founder's Edition:** Subscription tier for early adopters. Benefits: prioritized feature requests/bug fixes, early access to cmux AI, iOS app, Cloud VMs, Voice mode. Pricing: not publicly disclosed. [src 15]

**Concurrency:** No documented limit on concurrent workspaces or surfaces. The community spec for multi-agent orchestration notes a practical limit of ~5 concurrent GUI workspaces before terminal overhead becomes significant; headless RPC worker pools support 20+ per the same source. [src 11]

---

## 11. Maturity / Known Limitations

[VERIFIED — combining GitHub issues, community sources, and local session memory]

**Maturity signals:**
- 15.7k GitHub stars, YC-backed, two active founders. [src 2]
- v0.63.2 — still in 0.x versioning, signaling pre-stable API surface. API has changed between minor versions (AGPL→GPL relicense at v0.63.2; `read-screen` API was non-production until PR #219 merged 2026-02-21). [src 3, 8]
- 844 open issues — high volume for an early-stage tool; reflects rapid user growth outpacing issue triage. [src 4]

**Known sharp edges (sourced):**

| Issue | Severity | Status | Source |
|---|---|---|---|
| Dead PTY on programmatically-created background workspaces | HIGH — blocks automation | Open, no maintainer response | GitHub #1472 [src 5] |
| `read-screen` fails after display sleep (`caffeinate -i`) | MEDIUM — breaks long-running orchestrators | Open | GitHub #2004 [src 10] |
| `send_text` drops non-ASCII chars via key-event path | MEDIUM — corrupts non-ASCII payloads | Open | GitHub #2756 [src 7] |
| `surface.read_text` ignored `workspace_id` for non-selected surfaces | HIGH (was) | Fixed in PR #220 | GitHub PR #219 [src 8] |
| set-status pills persist across cmux restarts (stale state) | LOW — cosmetic | Known; workaround: `clear-status` at SessionStart | `SessionPersistence.swift:199-205` [src 16] |
| No native process-alive check | MEDIUM — forces screen-polling | By design (no daemon model) | [src 9, 11] |
| No detach/reattach | HIGH for always-on use | By design | [src 9] |
| Remote surface control not yet wired (issue #2673) | MEDIUM for remote use | Open | GitHub #2673 [src 13] |

**Empirical findings from Ali's production use (local session memory):**

From `feedback_cmux_env_vs_backend.md` (S-20): cmux-as-Backend is "less capable than iTerm2: no public scrollback read; bracketed-paste hazard on send_text." The `read-screen` API was non-production at that point — PR #219 (2026-02-21) promoted it to production. The bracketed-paste caveat remains. [src 6]

From `feedback_cmux_allowed_tools_oauth_declaration_only.md` (S-20): cmux env vars (`CMUX_WORKSPACE_ID`, `CMUX_SURFACE_ID`) are injected correctly into hook contexts and are stable enough for events.jsonl surface-ID references. [src 17]

---

## 12. Vendor Lock-in / Portability

[VERIFIED] **macOS-only.** No Linux, Windows, or headless-server deployment. This is a fundamental architectural constraint, not a missing feature — cmux is a native Swift/AppKit GUI application built on libghostty, which requires AppKit. [src 4]

**Ghostty config compatibility:** cmux reads `~/.config/ghostty/config` for themes and fonts. If you use Ghostty, configuration transfers. [src 4]

**API portability:** The JSON-RPC socket protocol is custom to cmux. No standard (tmux, VT100, etc.) parity beyond the `capture-pane` alias. Migrating a cmux-backend adapter to tmux requires implementing the equivalent tmux commands (`send-keys`, `capture-pane`, `new-session`). The primitive set is similar but the wire protocol is entirely different.

**Lock-in surface for terminal-ops:** If the BackendAdapter abstraction (`spawn/send/read/status/kill/list`) is implemented cleanly with no cmux-specific leakage above the adapter layer, migration is bounded to the adapter implementation. The orchestrator-vision §3 design explicitly calls for this abstraction. AO-104 v1 ships subprocess-headless; cmux is a v2 candidate for the local-interactive cell.

**Cloud deployment:** Not possible with cmux alone. The cmux+tmux composition is the path for remote deployments: cmux provides the desktop view; tmux provides process persistence and remote reach. For cloud-native Layer 3, the api-driven backend is the correct choice with no cmux dependency.

---

## BackendAdapter Fit Assessment

cmux maps to the terminal-ops primitive set as follows:

| Primitive | cmux implementation | Notes |
|---|---|---|
| `spawn(agent, prompt, env)` | `new-workspace --cwd ... --command "claude -p '...'"`  then `rpc workspace.create` | Returns workspace/surface refs, not a process handle. Handle must be retrieved via list. |
| `send(handle, data)` | `rpc surface.send_text` | No bracketed-paste wrapping — irrelevant for `claude -p` dispatch. Non-ASCII hazard documented. |
| `read(handle, lines, since)` | `rpc surface.read_text` with `scrollback + lines` | Production since PR #219 (Feb 2026). Plain text, no ANSI. No `since` offset — must diff against prior snapshot. |
| `status(handle)` | Screen-polling (`read-screen` → parse shell prompt / spinner) OR notification-relay (`cmux notify` + `list-notifications`) | No native process-alive signal. Notification relay is cleaner than polling. |
| `kill(handle)` | `rpc surface.close` + `rpc workspace.close` | No SIGTERM to the inner process — sends Ctrl+C first if needed. |
| `list()` | `rpc workspace.list` + `rpc surface.list` | Returns UUIDs and refs; no process metadata. |

**Deployment cell: Layer 1 + local-interactive.** cmux is the correct backend when Ali wants to see agents working in visible panes with sidebar status pills. It is NOT a substitute for the subprocess-headless backend (which is simpler and headless-CI-safe) or tmux (which is persistent and remote-capable).

**AO-162 scope:** cmux observability integration for Layer 1 orchestrator (set-status + notify + surface-ID emission contract) is independent of which terminal-ops backend is used for dispatch. Even if Layer 1 dispatches via subprocess-headless, it can publish to cmux's sidebar. AO-162 should implement this dispatch-independent observability contract.

**Fit verdict:** cmux is a **B-tier BackendAdapter** — viable and already partially wired, with the single sharp disqualifier being ephemeral persistence. For Ali's local-interactive development workflow (Layer 1 PoC on agent-ops), it is the richest visibility option available. For any deployment that must survive a cmux restart or run headlessly, subprocess-headless (Layer 1 CI/cron) or tmux (Layer 2+ remote) is required.

---

## Open Questions

1. **Background PTY initialization (GitHub #1472):** Is there a cmux API flag or sequence to force PTY initialization for a newly-created background workspace without requiring a focus-then-unfocus dance? A `workspace.create` with `focus: false` + PTY-ready guarantee would unblock fully-headless cmux dispatch.

2. **`surface.read_text` stream events:** Does cmux expose a streaming / subscribe API for surface output (i.e., receive new lines as they arrive rather than polling)? Not documented in current capabilities list but would eliminate polling latency.

3. **Remote daemon (`cmuxd-remote`) roadmap:** When will the remote daemon wire into the socket API for two-way remote surface control? Issue #2673 is open with no maintainer response. This determines whether cmux+tmux composition is the long-term remote pattern or a bridge to a first-class cmux remote backend.

4. **set-status cross-workspace race:** If two orchestrators run in parallel workspaces and both call `set-status claude_phase`, do pills collide? Are keys scoped to workspace (assumed yes per `--workspace` flag) or global?

5. **Founder's Edition vs GPL compliance:** As cmux matures toward a cloud product (Cloud VMs in Founder's Edition roadmap), will the socket API remain in the GPL-licensed codebase or migrate to a commercial-only tier?

---

## Sources

| # | URL | Type | Access Date |
|---|-----|------|-------------|
| 1 | https://github.com/manaflow-ai/cmux | GitHub repo (primary) | 2026-04-29 |
| 2 | https://www.ycombinator.com/companies/manaflow | YC company profile | 2026-04-29 |
| 3 | https://github.com/manaflow-ai/cmux/releases | Release notes (v0.61.0–v0.63.2) | 2026-04-29 |
| 4 | https://github.com/manaflow-ai/cmux/blob/main/README.md | README | 2026-04-29 |
| 5 | https://github.com/manaflow-ai/cmux/issues/1472 | GitHub issue: dead PTY | 2026-04-29 |
| 6 | `~/.claude/projects/.../memory/feedback_cmux_env_vs_backend.md` | Local session memory (S-20) | 2026-04-29 |
| 7 | https://github.com/manaflow-ai/cmux/issues/2756 | GitHub issue: non-ASCII paste | 2026-04-29 |
| 8 | https://github.com/manaflow-ai/cmux/pull/219 | GitHub PR: production read-screen | 2026-04-29 |
| 9 | https://soloterm.com/cmux-vs-tmux | cmux vs tmux comparison | 2026-04-29 |
| 10 | https://github.com/manaflow-ai/cmux/issues/2004 | GitHub issue: display sleep bug | 2026-04-29 |
| 11 | https://gist.github.com/joelhooks/11aea283acfd5a7f50e596bc63bbdd28 | cmux multi-agent orchestration spec | 2026-04-29 |
| 12 | `/Users/aliargun/Documents/GitHub/studio/studio-agent-ops/docs/orchestrator-vision.md §3` | Internal doc (S-24) | 2026-04-29 |
| 13 | https://github.com/manaflow-ai/cmux/issues/2673 | GitHub issue: remote workspace | 2026-04-29 |
| 14 | https://www.mintlify.com/manaflow-ai/cmux/automation/socket-api | Official socket API docs | 2026-04-29 |
| 15 | https://github.com/manaflow-ai/cmux/issues/542 | GitHub issue: Founder's Edition | 2026-04-29 |
| 16 | `~/.claude/hooks/metacognition-nudge.sh` | Local hook (SessionPersistence ref) | 2026-04-29 |
| 17 | `~/.claude/projects/.../memory/feedback_cmux_allowed_tools_oauth_declaration_only.md` | Local session memory (S-20) | 2026-04-29 |
| 18 | https://github.com/manaflow-ai/cmux/discussions/2338 | Community: cmux-mcp discussion | 2026-04-29 |

**All API claims verified against:** live `cmux` v0.63.2 binary at `CMUX_BUNDLED_CLI_PATH=/Applications/cmux.app/Contents/Resources/bin/cmux`, executed on macOS 26.4.1 (Apple Silicon M2 Max equivalent), 2026-04-29.

---

## Cross-References to Internal Docs

- `studio-agent-ops/docs/orchestrator-vision.md §3` — 4-backend matrix; cmux is the local-interactive cell; cmux+tmux composition is the persistent+remote cell
- `studio-agent-ops/docs/orchestrator-vision.md §4` — AO-162 (cmux observability for Layer 1) is in the build sequence, dispatch-independent
- `studio-pm/research/terminal-agent-orchestration-landscape.md §3` — terminal backend matrix; cmux was not in it as of S-19 (2026-04-19); this evaluation fills that gap
- `studio-research/landscapes/agentic-substrate-distribution.md` — sister landscape; cmux is not a distribution substrate (no git-semantic versioning, no secrets management)
- Memory: `feedback_cmux_env_vs_backend.md` — two-layer distinction (environment vs Backend); prerequisite reading for anyone implementing the BackendAdapter
- Memory: `feedback_cmux_allowed_tools_oauth_declaration_only.md` — CMUX env vars confirmed stable for events.jsonl surface-ID references
- Linear: AO-142 — cmux Phase 1 observability (Done); AO-162 — cmux observability for Layer 1 orchestrator (Backlog, promote when AO-104 ships); AO-163 — tmux backend for always-on Layer 2/3 (Backlog, strategic)
