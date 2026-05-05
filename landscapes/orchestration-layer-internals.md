---
title: "Orchestration Layer Internals — Empirically Grounded Synthesis"
description: "Synthesis of 7 surface deep-dives (cmux, Claude Code CLI/SDK, hooks, MCP, auth/connectors, tmux/iTerm2, Codex CLI) for orchestrator v2 design discussion. 14 architectural findings + 9 known gaps + 3 falsifications including a self-referential audit-document falsification. Every load-bearing claim is paste-citation-traced to a singleton deep-dive."
research_date: 2026-05-05
last_verified: 2026-05-05
staleness_warning: "Orchestration-layer surfaces evolve weekly (Claude Code 2.1.x; cmux 0.64.0 released day-of; Codex 0.128.x). Re-verify per surface after 30 days; check binary versions before relying on specific behaviors. Singletons at studio-agent-ops/.context/AO-263/ pin 2026-05-05 versions."
confidence: high
sources_count: 7
related:
  - ../../studio-agent-ops/.context/AO-263/cmux.md
  - ../../studio-agent-ops/.context/AO-263/claude-code.md
  - ../../studio-agent-ops/.context/AO-263/hooks.md
  - ../../studio-agent-ops/.context/AO-263/mcp.md
  - ../../studio-agent-ops/.context/AO-263/auth.md
  - ../../studio-agent-ops/.context/AO-263/tmux.md
  - ../../studio-agent-ops/.context/AO-263/codex.md
  - ../../studio-agent-ops/.context/AO-263/PHASE-1.5-REVIEW.md
  - ../../studio-agent-ops/decisions/008_s68-pivot-conflict-audit.md
  - ./agentic-fleet-platforms.md
  - ./cmux-features.md
status: active
supersedes: landscapes/cmux-features.md
---

# Orchestration Layer Internals — Empirically Grounded Synthesis

**Version:** v0
**Date:** 2026-05-05
**Author:** AO-270 Pass 2 synthesis agent
**Source set:** 7 singletons at `studio-agent-ops/.context/AO-263/<surface>.md` (cmux, claude-code, hooks, mcp, auth, tmux, codex), `PHASE-1.5-REVIEW.md`, `decisions/008_s68-pivot-conflict-audit.md`, 3 STATUS-tagged memory files
**Research dates:** All singletons probed 2026-05-05; binary versions pinned per §2
**Supersedes:** `landscapes/cmux-features.md` (S-65 landscape, which contained 2 falsified claims — Feed CLI surface and workstream.jsonl audit log; do NOT inherit claims from that file)
**Defensibility contract:** Every load-bearing claim cites its source as `[surface.md §section]`. When claim differs from vendor documentation, paste evidence wins; vendor doc annotated as stale/wrong. Claims without paste confirmation are labeled **[VENDOR-DOC-ONLY]**. Claims from blocked probes are labeled **[HYPOTHESIS — probe blocked]**.

---

## Navigation Guide

This document has 9 sections. Entry points by reader goal:

- **"I need the 5 most important facts before the design meeting"** → §1 Executive Summary only
- **"I'm the v2 architect, tell me what the singletons say about each open question"** → §1 (scan) → §7 directly; §3 and §4 are the factual backing on demand
- **"I need to know about secrets hygiene gaps found in this phase"** → §6 (standalone section)
- **"What prior decisions or memories are now factually wrong?"** → §5 Falsification Register
- **"I need to understand a specific surface (hooks, MCP, auth, etc.)"** → §2 Per-Surface Primer (orientation only; singletons are the primary source)
- **"What are the known empirical gaps that might affect my design?"** → §8 Open Questions

---

## §1 Executive Summary

This phase (AO-263) spent 7 deep-dive sessions empirically probing every orchestration-layer surface that a cmux-orchestrated Claude Code + Codex fleet depends on. The purpose was to provide the orchestrator v2 design discussion with grounded facts, not vendor-documentation assumptions.

**The 5 most important facts:**

**Fact 1 — Subagent hooks DO fire (FALSIFIES a prior decision).**
Claude Code 2.1.128 PreToolUse, PostToolUse, and SubagentStop hooks fire for subagents spawned via the Task tool. `agent_id` and `agent_type` fields identify subagent origin. This FALSIFIES GH issue #34692 (still open as of 2026-05-05, but empirically wrong on 2.1.128) and FALSIFIES `feedback_subagent_hook_inheritance_gap.md`. More critically, it FALSIFIES a load-bearing premise in `decisions/008_s68-pivot-conflict-audit.md §2`, which states: *"Hooks do NOT fire for subagent tool calls (per `feedback_subagent_hook_inheritance_gap.md`) — orchestration must enforce at parent boundary."* The expertise phase itself falsified the constraint that motivated the S-68 pivot. The implication for v2: hook-based subagent observability is NEWLY VIABLE for Task-tool dispatches (custom-agent/SDK Agent() paths need re-probe — see §8 gap #7). [hooks.md §6.2, §D2]

**Fact 2 — MCP `${VAR}` header substitution WORKS (FALSIFIES a prior memory).**
`${VAR}` substitution in `.mcp.json` `headers` field works on Claude Code 2.1.128 for both pure (`${TOKEN}`) and partial (`Bearer ${TOKEN}`) forms. Server-side capture confirmed the expanded value. This FALSIFIES `reference_claude_code_mcp_headers_env_var_broken.md` (S-41). The `headersHelper` workaround continues to work but is no longer the only viable path. [mcp.md §"Transport: HTTP", §Divergences §1]

**Fact 3 — Codex does NOT strip OPENAI_API_KEY from spawned subprocesses.**
Codex 0.128.0 inherits `OPENAI_API_KEY` into every subprocess by default — the opposite of Claude Code's post-AO-213 behavior. A live test with a synthetic key confirmed the subprocess saw the value. Fleet configurations that spawn Codex must explicitly configure `[shell_environment_policy].exclude = ["OPENAI_API_KEY"]` or use `env -u OPENAI_API_KEY codex ...` at spawn time. [codex.md §5.4]

**Fact 4 — `claude mcp get <server>` outputs static bearer tokens in plaintext.**
Running `claude mcp get linear` prints `Authorization: Bearer lin_api_<VALUE>` to stdout. The `lin_api_*` token format is NOT covered by the `secrets-redaction-runtime.md` regex table. This is a new secrets-hygiene gap in the AO-153/AO-154 lineage. The `~/.claude/claude_code_config.json` legacy file also stores a plaintext GitHub PAT. See §6 for the full gap analysis. [mcp.md §Divergences §3]

**Fact 5 — Completion detection for interactive REPLs remains architecturally open.**
The Stop hook is insufficient for interactive REPL completion detection (fires when the model yields control; doesn't confirm the REPL session has ended). Two new paths are viable: SubagentStop provides `agent_transcript_path` for transcript-tail completion detection; a hypothetical cmux blocking-call primitive (name unverified — may be `wait-for`, may not exist; cmux.md does not enumerate it) **[HYPOTHESIS — name and existence both unverified in AO-264 scope]**. The explicit-Bash-emit pattern (emit-event.sh) remains viable but in-band. [decisions/008 §5.Q1, hooks.md §3.6, codex.md §3]

---

## §2 Per-Surface Primer

Single orientation table — one row per surface. For depth, read the singleton directly.

| Surface | Binary / version | Path | Key empirical findings | Open questions |
|---|---|---|---|---|
| **cmux** | 0.63.2 | `$CMUX_BUNDLED_CLI_PATH` (NOT on `$PATH`) | Hidden subcommands: `set-status`, `clear-status`, `list-status` (not in `--help`, functional); `CMUX_TAB_ID == CMUX_WORKSPACE_ID` (doc-vs-impl bug); `cmux codex install-hooks --help` executes instead of printing help | `cmux codex-hook` subcommand surface not fully enumerated; `cmux feed` is NOT a CLI subcommand (GUI-only) |
| **Claude Code CLI** | 2.1.128 | `/Users/aliargun/.local/bin/claude` | ~50 top-level flags live-catalogued; `--bare` strips hooks/LSP/OAuth/plugins; `--max-budget-usd` enforced post-turn not pre-turn; REPL prompt glyph U+276F (0xE2 0x9D 0xAF) confirmed | `claude install` + `claude setup-token` blocked by --HELP guard |
| **Hooks** | claude 2.1.128 hooks system | `~/.claude/settings.json` | 6/28 events paste-confirmed; settings cascade layering confirmed (1+6=7 SessionStart fires); SubagentStop fires with `agent_transcript_path`; FALSIFICATION of #34692 | Custom-agent/SDK Agent() subagent hook coverage not re-probed |
| **MCP** | claude 2.1.128 | `.mcp.json`, `~/.claude/claude_code_config.json` | OAuth/PKCE full discovery dance captured; `${VAR}` in headers WORKS (FALSIFICATION); `--strict-mcp-config` partially enforced (honored by runtime, not `mcp list`); plaintext bearer via `mcp get` | Plugin-MCP OAuth token storage location (`~/.claude/.credentials.json` likely, harness-blocked) |
| **Auth/Connectors** | claude 2.1.128 + codex 0.128.0 | Keychain + `~/.codex/auth.json` | 6-tier auth precedence live-mapped; `authMethod` is configured method not active source; cross-CLI Keychain fully isolated; Codex env-strip asymmetry (no OPENAI_API_KEY strip by default) | Agent SDK `query()` env inheritance defaults |
| **tmux/iTerm2** | tmux NOT installed; iTerm2 present, Python API NOT installed | BackendAdapter in `substrate/` | `--tmux=classic` syntax FALSIFIED (parser rejects; actual enum `auto\|tmux\|in-process`); 3 backend classes decompiled (TmuxBackend, ITermBackend, InProcessBackend); BackendAdapter fully mapped | All tmux live probes blocked; iTerm2 Python API not installed |
| **Codex CLI** | 0.128.0 | `/opt/homebrew/bin/codex` | `codex exec --json` event stream live-captured; OPENAI_API_KEY NOT stripped by default; `codex mcp-server` = Codex as MCP stdio server; `--remote ws://` WebSocket experimental | `bearer_token_env_var` literal-fallback behavior uncharacterized |

---

## §3 Boundary Contracts — Integration Seams

This section synthesizes the 16 bilateral-sufficient edges from the Phase 1.5 coverage audit into explicit seam contracts. Each contract states: what one side provides, what the other consumes, and the failure modes that matter for v2 design.

### 3.1 ClaudeCode × cmux — Environment injection, PTY, REPL boot, glyph detection

**Contract:** cmux provides the runtime environment in which Claude Code runs. The cmux bundled wrapper (`$CMUX_BUNDLED_CLI_PATH`) performs 4 actions at launch: (1) sets all CMUX_* env vars (16 vars, including CMUX_WORKSPACE_ID, CMUX_SURFACE_ID, CMUX_TAB_ID); (2) injects 6 hooks via `--settings` argument (additive on top of user-level settings); (3) auto-generates `--session-id` unless resume flags are present; (4) runs NODE_OPTIONS restoration via `restore-node-options.cjs`.

**What Claude Code consumes:** CMUX_BUNDLED_CLI_PATH (path to the real cmux binary for `cmux notify` calls), CMUX_SURFACE_ID (surface identity for set-status pills), CMUX_WORKSPACE_ID (workspace context). CLAUDECODE=1 is set by Claude Code itself; the wrapper `unset CLAUDECODE` before spawn to defeat nested-session detection, then Claude Code re-sets it.

**REPL-ready detection:** The REPL prompt glyph is U+276F (❯), 3-byte UTF-8 (0xE2 0x9D 0xAF). This is the signal an orchestrator can tail-and-grep when spawning a REPL and waiting for it to become interactive. Confirmed empirically: `printf '❯\n' | xxd → e2 9d af`. [claude-code.md §"× cmux"; cmux.md §"Init mechanics"]

**Failure modes:** If `CMUX_SURFACE_ID` is absent, cmux notify/set-status calls silently no-op (both `mcp notify` and `set-status` check this gate). If the bundled wrapper is NOT used (claude launched directly), the 6 injected hooks do not fire and the session-id is not set. CMUX_TAB_ID == CMUX_WORKSPACE_ID (not the surface UUID) — any code routing on tab ID is getting workspace granularity, not surface granularity. [cmux.md §Divergences §3]

### 3.2 ClaudeCode × hooks — Settings cascade, invocation semantics, exit-code contract

**Contract:** Claude Code discovers hooks from the settings cascade. When `--settings <path>` is passed, those settings are applied ON TOP OF user-level `~/.claude/settings.json` (not replacing it). A test with 1 custom + 6 cmux-injected hooks confirmed 7 SessionStart fires — both layers fire concurrently. `disableAllHooks: true` is the global kill-switch.

**Exit-code semantics matter for orchestrators:** On PreToolUse hooks, exit 0 = allow, exit 2 = block (tool call is prevented; stderr sent to model as system message). On SessionStart, exit 2 = `additionalContext` injection (NOT a block). On Stop, exit code affects flow control. The stream-json `outcome` field maps exit 2 to `"error"` — this label is semantically misleading (it means "blocked" or "whisper injected," not an error state). Orchestrators reading stream-json must test `outcome == "error"` AND check event context to distinguish. [hooks.md §4.1; claude-code.md §Divergences §3]

**`--include-hook-events` toggle:** When Claude Code runs in `--output-format stream-json` mode, hook events are NOT included by default. Add `--include-hook-events` to surface `hook_response` events in the stream. This is the only way to observe hook outcomes from the orchestrator's stream reader. [claude-code.md §"stream-json"]

**Input schema divergences:** PostToolUse actual schema: `tool_response: {stdout, stderr, interrupted, isImage, noOutputExpected}`. Vendor docs say `tool_output: {type, text}`. Read `tool_response`. SessionEnd actual field: `reason` (not `end_reason` per vendor docs). [hooks.md §D1, §D3]

### 3.3 ClaudeCode × MCP — Config precedence, strict-mode partial enforcement

**Contract:** `--mcp-config <path>` takes precedence over project `.mcp.json` (paste-confirmed: `STATIC_VAR=literal-value` from `--mcp-config` wins over project-file value). The variadic-greedy parsing trap: `--mcp-config <file> --strict-mcp-config` requires correct argument ordering; putting `mcp list` after the flags causes greedy parsing to consume `mcp` as the config file path.

**`--strict-mcp-config` partial enforcement (PREVIOUSLY UNDOCUMENTED):** Honored by agent-runtime tool list — if `--strict-mcp-config` is active, only tools from the specified config are exposed to the model. Silently no-ops on `claude mcp list` and `claude mcp get` enumeration subcommands. Validation MUST use `-p` print mode with "list your tools" prompt, NOT `mcp list`. [mcp.md §"--mcp-config and --strict-mcp-config behavior", §Divergences §2]

**`${VAR}` substitution works in headers (FALSIFICATION):** Pure form (`${TOKEN}`) and partial form (`Bearer ${TOKEN}`) both correctly substituted in HTTP MCP `headers` field. Server-side capture confirmed expanded values. Prior memory (`reference_claude_code_mcp_headers_env_var_broken.md`) is FALSIFIED. [mcp.md §Divergences §1]

### 3.4 ClaudeCode × auth — Env-strip, apiKeySource as truth field, `--bare` semantics

**Contract:** Auth resolution order (6 tiers, live-verified): Cloud providers (claude.ai) > ANTHROPIC_AUTH_TOKEN > ANTHROPIC_API_KEY > apiKeyHelper > CLAUDE_CODE_OAUTH_TOKEN > OAuth/Keychain.

**Critical: `authMethod` is NOT the active auth method.** When ANTHROPIC_API_KEY is set in env, `claude auth status --json` shows `authMethod: "claude.ai"` (the configured subscription method) but adds `apiKeySource: "ANTHROPIC_API_KEY"`. Live truth is `apiKeySource`. Orchestrators checking auth state must read `apiKeySource`, not `authMethod`. [auth.md §1 Test 3, §10 Divergence §1]

**ANTHROPIC_API_KEY env-strip (post-AO-213):** Claude Code strips ANTHROPIC_API_KEY from the env passed to spawned subprocesses (builders, tools) per [claude-code.md §"× auth"; auth.md §1 Test 5]. The in-process Agent SDK `query()` call (e.g., for verifier or koinos-context MCP) inherits parent env and bills via API key per `memory/project_ao213_scope_gap.md` (S-51) — tracked as AO-214. This asymmetry affects billing discipline (subprocess vs in-process; the singletons cover the subprocess side; the in-process detail is captured in the memory).

**`--bare` semantics:** Strips hooks, LSP, OAuth, plugins. `--bare` + inline ANTHROPIC_API_KEY=fake → `authMethod: "apiKeyHelper"` but 401 (confirms auth path, confirms bare strips cloud auth). [claude-code.md §"× auth", auth.md §1 Test 4]

### 3.5 hooks × subagents — FALSIFICATION of #34692, SubagentStop, agent_id field

**Contract:** On Claude Code 2.1.128, PreToolUse and PostToolUse hooks fire for subagents spawned via the Task tool. The hook input includes `agent_id` and `agent_type` fields that identify the subagent origin. SubagentStop fires when a subagent completes, with `agent_transcript_path` pointing to the per-subagent JSONL at `<session>/subagents/agent-<id>.jsonl`.

**FALSIFICATION:** GitHub issue #34692 (still OPEN as of 2026-05-05) claims hooks do not fire for subagent tool calls. This is empirically WRONG on 2.1.128. PreToolUse stdin paste captured `agent_id: "a970c15f4d4daf3ee"`, `agent_type: "general-purpose"`. The conclusion that hooks reliably cover subagent telemetry is SUPPORTED for Task-tool dispatches.

**Caveat (BLOCKING gap):** Only Task-tool spawned subagents were probed. Custom-agent (`--agents` flag) and SDK Agent() dispatches were NOT re-probed. The "hooks reliable for subagent telemetry" recommendation must not be generalized until gap #7 is resolved. See §7 Q5 and §8. [hooks.md §6.2, §D2]

### 3.6 MCP × auth — OAuth/PKCE dance, static bearer plaintext exposure

**Contract:** Remote OAuth-protected MCP servers are discovered via the full RFC dance: 401 + WWW-Authenticate → `/.well-known/oauth-protected-resource` (RFC 9728) → `/.well-known/oauth-authorization-server` (RFC 8414) → `/register` (RFC 7591 dynamic client registration, public-client `none` method, PKCE-S256 required, refresh_token grant supported). Token storage: per-server Keychain entries `Claude Code-credentials-<8-hex>`.

**koinos-context is NOT OAuth-discoverable:** Returns 401 without WWW-Authenticate header (spec violation). Workaround: `headersHelper` or `${VAR}` in headers field (both now confirmed working). [mcp.md §"OAuth/PKCE flow"; auth.md §8]

**Plaintext bearer exposure:** `claude mcp get <server>` outputs the static bearer token in plaintext to stdout. The format `lin_api_[A-Za-z0-9]{40}` for Linear PATs is NOT in the `secrets-redaction-runtime.md` regex table. Any orchestrator that calls `mcp get` for inspection will leak the token to transcript. See §6.1 for gap analysis. [mcp.md §Divergences §3]

### 3.7 Codex × cmux — install-hooks, CMUX_SURFACE_ID gate, hooks.json contract

**Contract:** `cmux codex install-hooks` writes `~/.codex/hooks.json` with 3 hooks (SessionStart, Stop, UserPromptSubmit) and adds `codex_hooks = true` to `~/.codex/config.toml`. All 3 hooks are gated on `[ -n "$CMUX_SURFACE_ID" ] && command -v cmux >/dev/null 2>&1` — they silently no-op outside a cmux session.

**Sharp edge (FOOTGUN):** `cmux codex install-hooks --help` does NOT print help. It EXECUTES the install. Confirmed: files were created when the `--help` flag was passed. Any orchestrator or runbook that mentions `--help` for this subcommand will trigger an install. [cmux.md §Sharp Edges §1; codex.md §10]

**Codex hook taxonomy differs from Claude Code:** Codex hooks include PermissionRequest (no Claude Code equivalent). No SubagentStop or SessionEnd in Codex hooks. Codex hooks fire via `cmux codex-hook` (not `cmux claude-hook`). The `cmux codex-hook` subcommand surface is NOT fully enumerated — gap #1 in §8. [codex.md §7]

### 3.8 Codex × auth — env-strip asymmetry, subscription vs API key

**Contract:** Codex auth stores credentials at `~/.codex/auth.json` (mode 600, 3 keys: OPENAI_API_KEY, last_refresh, tokens). Fully isolated from Claude Code's Keychain entries.

**CRITICAL env-strip asymmetry:** Claude Code strips ANTHROPIC_API_KEY from spawned subprocesses by default (post-AO-213). Codex 0.128.0 does NOT strip OPENAI_API_KEY. Live test: `OPENAI_API_KEY=fake-test-key-xyz codex exec ... 'env | grep OPENAI_API_KEY'` — subprocess saw the synthetic key. This is a cross-fleet security gap. Orchestrators that spawn Codex must explicitly strip: `env -u OPENAI_API_KEY codex ...` or configure `[shell_environment_policy].exclude = ["OPENAI_API_KEY", "OPENAI_*"]`. [codex.md §5.4; auth.md §5]

### 3.9 tmux/iTerm2 × BackendAdapter — interface mappability

**Contract:** The BackendAdapter interface (decompiled from claude binary `strings` + in-repo `substrate/src/backends/index.ts`) defines 8 methods: `prepareResources`, `spawn`, `send`, `read`, `status`, `kill`, `list`, `supports`. All 6 core primitives (spawn, send, read-snapshot, read-stream, status, kill) are implementable via tmux CLI commands. Gaps in read-stream (requires `pipe-pane -o` vs direct streaming), status busy/idle distinction (requires polling `show-options activity-action`), and list namespace-filtering. [tmux.md §3]

**iTerm2 fills more cmux gaps than tmux:** iTerm2 Python API supports push notifications (vs tmux polling), streaming output (`async for` on session buffer), and session focus — closer to cmux's full feature surface. cmux provides ~30 additional commands beyond what either backend offers natively (notify, set-status, browser pane, RPC `await_screen_change`, claude-hook/codex-hook ecosystems). [tmux.md §4]

**`--tmux=classic` FALSIFICATION:** `claude --help` shows `--tmux=classic` syntax but the parser rejects it. Actual enum: `auto|tmux|in-process`. Any runbook using `--tmux=classic` is broken. [tmux.md §1.4, §Part 6]

### 3.10 auth × cmux — Env passthrough to spawned Claude

**Contract:** cmux passes the parent shell's auth env unchanged to the spawned Claude process. If ANTHROPIC_API_KEY is set in the parent shell, it reaches Claude and API-key precedence wins over OAuth subscription — even inside cmux. Confirmed live: `env -u ANTHROPIC_API_KEY claude -p 'echo $CMUX_BUNDLED_CLI_PATH'` returns the cmux path, confirming nested claude inherits the outer-shell env. [auth.md §8; cmux.md §"Env injection chain"]

**Low-priority seams (one-liner treatment):**
- **hooks × MCP:** PreToolUse fires on `mcp__linear__list_teams` with full tool namespace in `tool_name` field; matcher supports `mcp__server__*` glob. [hooks.md §6.3]
- **MCP × Codex:** Plugin `.mcp.json` HTTP MCP honored by Codex for Codex-schema fields; Claude Code's `headersHelper` silently dropped. [mcp.md §"× Codex CLI"; codex.md §4.2]
- **tmux × cmux:** cmux exposes tmux compat subcommands (capture-pane, resize-pane, pipe-pane, etc.) as thin wrappers; all RPC surface accessible via socket. [cmux.md §"CLI subcommand surface"]
- **auth × Codex:** `~/.codex/auth.json` vs Claude Code Keychain — fully isolated. [auth.md §5; codex.md §5]
- **ClaudeCode × Codex (via `codex mcp-server`):** Codex can run as MCP stdio server, making it callable from Claude Code as a tool. Recursion primitive. [codex.md §8]

---

## §4 Architectural Findings — The 14

Organized by severity category. Within each category, ordered by v2 design implication severity.

---

### SECURITY (3 findings)

**Finding S-1: Plaintext bearer-token exposure via `claude mcp get`**
*Source: mcp.md §Divergences §3*

`claude mcp get <server>` outputs the MCP server's static bearer token in plaintext to stdout. Live capture: `claude mcp get linear` prints `Authorization: Bearer lin_api_<VALUE>`. The `lin_api_*` Linear PAT format (40 alphanumeric chars) is NOT in the `secrets-redaction-runtime.md` regex table. Any orchestrator pipeline that calls `mcp get` for inspection, any transcript containing this output, and any logging that captures CLI stdout will expose the token. The legacy `~/.claude/claude_code_config.json` file also stores a plaintext GitHub PAT discovered at rest.

**v2 implication:** Orchestrators must not call `claude mcp get` in any code path whose stdout lands in a JSONL transcript or log. Extend `secrets-redaction-runtime.md` to cover `lin_api_[A-Za-z0-9]{40}` (see §6). If the MCP config inspection is needed programmatically, parse the source `.mcp.json` file directly rather than using `mcp get`.

---

**Finding S-2: Codex 0.128.0 does NOT strip OPENAI_API_KEY from spawned subprocesses**
*Source: codex.md §5.4*

Opposite of Claude Code's post-AO-213 behavior. Live test: `OPENAI_API_KEY=fake-test-key-xyz codex exec ... 'env | grep OPENAI_API_KEY'` — subprocess saw the synthetic key. Any process spawned BY Codex (sub-tools, shell commands, hooks, MCP stdio servers) inherits OPENAI_API_KEY if present in the parent env. This creates an exfiltration surface that Claude Code's design closed but Codex leaves open.

**v2 implication:** All orchestrator code that spawns Codex must use `env -u OPENAI_API_KEY codex ...` OR configure `[shell_environment_policy].exclude = ["OPENAI_API_KEY", "OPENAI_*"]` in `~/.codex/config.toml`. Document in the wrapper's contract that OPENAI_API_KEY leaks to Codex's subprocesses by default. Treat Codex differently from Claude Code in env-passthrough design.

---

**Finding S-3: `authMethod` is the configured method, not the active auth source**
*Source: auth.md §1 Test 3, §10 Divergence §1*

`claude auth status --json` returns `authMethod: "claude.ai"` even when `ANTHROPIC_API_KEY` is overriding auth in the current env. The active auth source is reflected only in `apiKeySource: "ANTHROPIC_API_KEY"`. Live test: `ANTHROPIC_API_KEY=fake-test-key claude auth status` showed `authMethod: "claude.ai"` + `apiKeySource: "ANTHROPIC_API_KEY"` + zeroed identity fields.

**v2 implication:** Any orchestrator that checks whether a session is using subscription vs API-key billing must read `apiKeySource`, not `authMethod`. Alerting and billing audits must not rely on `authMethod` alone. This asymmetry is not documented by Anthropic.

---

### FALSIFICATIONS (2 findings)

**Finding F-1: Subagent hooks DO fire on Claude Code 2.1.128**
*Source: hooks.md §6.2, §D2*

GitHub issue #34692 (still open) claims PreToolUse and PostToolUse hooks do not fire for subagent tool calls. This is empirically WRONG on 2.1.128. Live test: Task-tool subagent Bash call triggered PreToolUse with `agent_id: "a970c15f4d4daf3ee"` and `agent_type: "general-purpose"` in the hook stdin. SubagentStop fires with `agent_transcript_path: "~/.claude/projects/<slug>/<session>/subagents/agent-<id>.jsonl"`.

**v2 implication:** Hook-based subagent telemetry is NEWLY VIABLE for Task-tool dispatches. The architecture constraint "orchestration must enforce at parent boundary because hooks miss subagents" is no longer empirically supported for this dispatch path. Caveat: custom-agent and SDK Agent() paths not re-probed — see §8 gap #7. The `rules/linear.md` "Subagent-hook caveat" is reconsiderable. [See also §5.1]

---

**Finding F-2: MCP `${VAR}` substitution in `headers` field WORKS in 2.1.128**
*Source: mcp.md §"Transport: HTTP", §Divergences §1*

Prior memory `reference_claude_code_mcp_headers_env_var_broken.md` (from S-41) cited 4 open GitHub issues claiming `${VAR}` substitution is broken in `.mcp.json` `headers` field. Live test on 2.1.128: both pure form (`${HTTP_PROBE_TOKEN}`) and partial form (`Bearer ${HTTP_PROBE_TOKEN}`) correctly substituted. Server-side capture confirmed expanded values were delivered. The 4 GitHub issues reference real bug reports but appear resolved in the current binary.

**v2 implication:** `headersHelper` workaround is no longer the only viable path for auth-header injection in HTTP MCP servers. `${VAR}` in `headers` is empirically reliable on 2.1.128. Verify on the consumer's Claude Code version before depending on this. The `decisions/007` parity claim between Claude Code and Codex CLI for HTTP MCP auth-headers is RECONSIDERABLE. [See also §5.2]

---

### DESIGN (5 findings)

**Finding D-1: `--max-budget-usd` enforced post-turn, not pre-turn**
*Source: claude-code.md §Divergences §2*

`--max-budget-usd 0.0001` (one cent) did not prevent a turn from running. The model ran, spent $0.38 (380× the budget), and THEN the budget gate fired and stopped execution at the `result` event level. First turn always completes regardless of budget; budget gate fires AFTER the turn.

**v2 implication:** `--max-budget-usd` cannot be used as a pre-execution spend guard. It is a stop-after-overage mechanism. Orchestrator cost controls must use different mechanisms (turn count limits, session-level monitoring, or pre-flight cost estimation with `--dry-run` if available).

---

**Finding D-2: `--strict-mcp-config` partially enforced — `mcp list`/`mcp get` ignore it**
*Source: mcp.md §"--mcp-config and --strict-mcp-config behavior", §Divergences §2*

`--strict-mcp-config` is honored by the agent runtime (only tools from the specified config file are exposed to the model). But `claude mcp list` and `claude mcp get` do NOT respect it — they enumerate all configured servers regardless. This means using `mcp list` to validate isolation is misleading.

**v2 implication:** Orchestrators that use `--strict-mcp-config` for tool-surface isolation must validate with `claude -p "list your tools"` (agent-runtime path), not `claude mcp list` (enumeration path). Do not rely on `mcp list` output to audit what the model can actually access during a session.

---

**Finding D-3: `--tmux=classic` syntax in `--help` does not parse**
*Source: tmux.md §1.4, §Part 6*

`claude --help` text uses `--tmux=classic` syntax in its example. The parser rejects this form: `error: unknown option '--tmux=classic'`. Actual accepted enum (from binary strings): `auto|tmux|in-process`. The help text is stale or the parser was updated.

**v2 implication:** Any runbook, script, or documentation referencing `--tmux=classic` syntax is broken. Use `--tmux tmux` (space-separated) with enum values `auto`, `tmux`, or `in-process`.

---

**Finding D-4: Connector ↔ plugin endpoint dedup behavior inconsistent with vendor docs**
*Source: auth.md §6, §10 Divergence §3*

Vendor docs claim connector endpoints are "hidden" from MCP server list when a plugin provides the same endpoint. Live: `claude mcp list` showed BOTH `claude.ai Figma` (connector) AND `plugin:figma:figma` (plugin) simultaneously, despite same URL. Either dedup logic wasn't firing or "duplicate" means something other than "deduplicated."

**v2 implication:** Orchestrators that enumerate MCP tools must handle namespace ambiguity: same capability may appear under both `claude.ai <name>` and `plugin:<vendor>:<server>` prefixes. Do not assume connector and plugin tool names are mutually exclusive.

---

**Finding D-5: `cmux codex install-hooks --help` executes, does not print help**
*Source: cmux.md §Sharp Edges §1; codex.md §10*

`cmux codex install-hooks --help` is NOT a help flag for this subcommand. It executes the install. Side effects: created `~/.codex/hooks.json` (3 hooks with CMUX_SURFACE_ID gate), set `codex_hooks = true` in `~/.codex/config.toml`. The integration is benign but the flag-parsing is footgun-shaped — any agent or runbook that "peeks" at this subcommand with `--help` will trigger a permanent config change.

**v2 implication:** Document in orchestrator runbooks and agent prompts that `cmux codex install-hooks` must be treated as idempotent-but-irreversible. Use `--help` before first run to preview behavior is the normal workflow convention; that convention is broken for this specific subcommand.

---

### BEHAVIOR / PROTOCOL DIVERGENCE (4 findings)

**Finding B-1: `SessionEnd` field is `reason`, not `end_reason`**
*Source: hooks.md §3.7, §D3*

Vendor docs specify `end_reason` in the SessionEnd hook payload. Live paste shows `"reason": "other"`. This divergence was first observed in S-42 on claude 2.1.123 (memory `reference_claude_code_sessionend_reason_diverges_docs.md`). RE-CONFIRMED on 2.1.128 — still divergent.

**v2 implication:** Any hook script or transcript parser that reads `end_reason` will get `null`. Read `reason`. This affects non-interactive CLI exit detection (see `secrets-redaction-runtime.md` §"Non-interactive-skip" for the `transcript_path` existence discriminator that works around the `reason` unreliability).

---

**Finding B-2: `PostToolUse` field naming diverges from vendor docs**
*Source: hooks.md §3.4, §D1*

Vendor docs: `tool_output: {type, text}`. Live (Bash 2.1.128): `tool_response: {stdout, stderr, interrupted, isImage, noOutputExpected}` plus a sibling `duration_ms` field. Different field name AND different shape.

**v2 implication:** Any hook that reads tool output must use `tool_response.stdout` and `tool_response.stderr`. Any schema validator or hook script built against vendor docs will silently fail to find `tool_output`. Test hook scripts against live paste, not vendor schema docs.

---

**Finding B-3: Hook `exit_code:2` maps to `outcome:"error"` in stream-json (semantically misleading)**
*Source: claude-code.md §Divergences §3; hooks.md §4.1*

When a hook exits with code 2, the stream-json `hook_response` event has `outcome: "error"`. But exit 2 is documented as the "block or whisper" semantic — not an error. On PreToolUse, exit 2 prevents the tool call; on SessionStart, exit 2 injects stderr as `additionalContext`. The semantic IS correctly executed (confirmed: whisper content surfaced in next assistant message), but the `outcome` label misleads.

**v2 implication:** Orchestrators consuming stream-json must not treat `outcome: "error"` as a system failure. They must cross-reference the event type to determine whether exit 2 means "tool blocked" vs "additional context injected" vs a genuine hook failure.

---

**Finding B-4: `CMUX_TAB_ID` holds workspace UUID, not tab UUID**
*Source: cmux.md §Divergences §3*

cmux documentation implies CMUX_TAB_ID is a tab-level alias. Live env scan: `CMUX_TAB_ID == CMUX_WORKSPACE_ID` (same UUID). The surface UUID is `CMUX_SURFACE_ID`. Tab-action `--tab` resolver presumably falls through to `CMUX_SURFACE_ID` when CMUX_TAB_ID doesn't resolve to a tab.

**v2 implication:** Orchestrators that track individual Claude Code surfaces must use CMUX_SURFACE_ID. CMUX_TAB_ID provides workspace granularity only, despite its name. Any routing logic that branches on "which tab" is actually branching on "which workspace."

---

## §5 Falsification Register

This section documents findings that require ACTION: updating prior JSONL decision entries, amending rules files, and closing memory files. Distinct from §4 (orientation) because these generate artifact changes.

### 5.1 FALSIFIED: Subagent hook inheritance gap

**Memory falsified:** `feedback_subagent_hook_inheritance_gap.md`
**Original claim:** "PreToolUse and PostToolUse hooks configured in `~/.claude/settings.json` fire ONLY for tool calls made by the main session thread. When the main agent spawns a subagent via the Agent tool, any Bash/Edit/Write/Read/Grep/MCP calls made by that subagent do not trigger the hooks."
**Source of original claim:** S-18, 2026-04-22, GH issue #34692.

**Falsification evidence:** AO-266 live test on Claude Code 2.1.128. PreToolUse hook fired for a Task-tool subagent Bash call. SubagentStop fired with agent_id, agent_type, and agent_transcript_path fields. Three hooks confirmed firing: PreToolUse, PostToolUse, SubagentStop. [hooks.md §6.2, §D2]

**Scope of falsification:** Task-tool (`general-purpose` subagent type) confirmed. Custom-agent (`--agents` flag) and SDK Agent() dispatch paths NOT re-probed — see §8 gap #7.

**What changes:**
- `feedback_subagent_hook_inheritance_gap.md` — STATUS box added (FALSIFIED 2026-05-05 by AO-266). Historical content preserved per `feedback_never_delete_repo_feedback.md`.
- `rules/linear.md` "Subagent-hook caveat" — reconsiderable; do NOT update until gap #7 is resolved (custom-agent/SDK Agent() paths unverified).
- `decisions/008_s68-pivot-conflict-audit.md §3.5` — classifies `feedback_subagent_hook_inheritance_gap.md` as ALIGNED/KEPT; this alignment is now incorrect.

### 5.2 The Decisions/008 Premise Falsification — Self-Referential

**This deserves explicit treatment because it is the document that motivated the S-68 pivot and the AO-263 expertise phase.**

`decisions/008_s68-pivot-conflict-audit.md §2` ("Cross-cutting agent execution model") states verbatim:

> "Hooks do NOT fire for subagent tool calls (per `feedback_subagent_hook_inheritance_gap.md`) — orchestration must enforce at parent boundary"

This claim was the load-bearing constraint that motivated the S-68 pivot from patch-iteration to expertise-first redesign. The AO-263 expertise phase — which this synthesis documents — was commissioned to gain comprehensive platform knowledge before redesigning the orchestrator under this constraint.

AO-266 (one of the 7 AO-263 sub-issues) empirically FALSIFIED this exact claim on the platform version the orchestrator targets (Claude Code 2.1.128).

**Implication:** The primary constraint cited for the S-68 pivot ("orchestration must enforce at parent boundary") is no longer an architectural necessity for Task-tool dispatches. The pivot to an expertise-first redesign remains the right call (the expertise gained is genuine and the v2 design benefits from it). But the redesign must NOT treat "hook-based subagent observability is impossible" as a given. The hook-based completion detection path (see §7 Q5) is newly viable for Task-tool subagents.

**Artifact requirement:** A new entry in `decisions.jsonl` is needed to record this finding (once bootstrapped). The `decisions/008` document should be annotated with a dated note that §2's "no subagent hooks" premise was falsified by AO-266 on 2026-05-05.

### 5.3 FALSIFIED: MCP `${VAR}` header substitution broken

**Memory falsified:** `reference_claude_code_mcp_headers_env_var_broken.md`
**Original claim:** "`${VAR}` substitution in `.mcp.json` `headers` field is broken. Literal `${VAR}` string is sent to the server unsubstituted."
**Source of original claim:** S-41, 2026-04-30, 4 GitHub issues cited.

**Falsification evidence:** AO-267 live test on Claude Code 2.1.128. HTTP MCP server captured `authorization: "Bearer THE_HTTP_TOKEN_zzz"` (expanded value, not literal `${TOKEN}`). Both pure and partial forms confirmed working. [mcp.md §"Transport: HTTP", §Divergences §1]

**What changes:**
- `reference_claude_code_mcp_headers_env_var_broken.md` — STATUS box added (FALSIFIED 2026-05-05 by AO-267). Historical content preserved.
- `decisions/007` parity claim — the "Codex > Claude Code for HTTP MCP auth-headers" framing is RECONSIDERABLE as of 2.1.128. Verify on consumer's Claude Code version before updating.
- `headersHelper` workaround — still works, now strictly "more powerful sibling" rather than "only option."

### 5.4 NEW: Codex OPENAI_API_KEY env-strip asymmetry

**Memory added:** `feedback_codex_does_not_strip_openai_api_key.md`
**Status:** ACTIVE (new finding, not a falsification of a prior memory — a new fact about Codex behavior).
**Finding:** Codex 0.128.0 does NOT strip OPENAI_API_KEY from the env passed to spawned subprocesses. Claude Code strips ANTHROPIC_API_KEY post-AO-213. Asymmetric by design (Codex's design) or by omission — live test confirms.
**Mitigation:** `[shell_environment_policy].exclude = ["OPENAI_API_KEY", "OPENAI_*"]` in `~/.codex/config.toml` or `env -u OPENAI_API_KEY codex ...` at spawn time.

**What changes:**
- `feedback_codex_does_not_strip_openai_api_key.md` — new file created at session end (2026-05-05).
- Orchestrator dispatch code — must add explicit strip when spawning Codex in any context where OPENAI_API_KEY is present in parent env.
- Fleet runbooks — document asymmetry explicitly; agents must not assume symmetric env-strip behavior across CLIs.

### 5.5 Artifact changes required — Summary table

| Artifact | Change type | Urgency | Status |
|---|---|---|---|
| `feedback_subagent_hook_inheritance_gap.md` | STATUS box: FALSIFIED 2026-05-05 | Done (added this session) | Complete |
| `reference_claude_code_mcp_headers_env_var_broken.md` | STATUS box: FALSIFIED 2026-05-05 | Done (added this session) | Complete |
| `feedback_codex_does_not_strip_openai_api_key.md` | New file created | Done (added this session) | Complete |
| `decisions/008` §2 annotation | Dated note that "no subagent hooks" premise falsified by AO-266 | High — affects v2 design discussion | Pending |
| `decisions.jsonl` | New entry for falsification (once bootstrapped) | Medium | Pending |
| `rules/linear.md` "Subagent-hook caveat" | Reconsider AFTER gap #7 resolved | Medium — blocked on gap #7 | Blocked |
| `decisions/007` parity claim | Update when gap #7 resolved + Codex-side re-verify | Medium | Blocked |
| `secrets-redaction-runtime.md` | Extend regex table with `lin_api_*` pattern | High (secrets-hygiene) | See §6.4 |

---

## §6 Secrets-Hygiene Gap Analysis

This section documents new gaps discovered during AO-263 that fall within the AO-153/AO-154 lineage (secrets-redaction-runtime discipline). Three distinct deliverables: gap documentation, regex extension proposal, and a Codex-specific hygiene note.

### 6.1 Gap: `claude mcp get` outputs static bearer tokens in plaintext

**Evidence:** `claude mcp get linear` printed `Authorization: Bearer lin_api_<REDACTED>` to stdout (redacted here; full paste in mcp.md §Divergences §3). The Linear PAT format is `lin_api_` followed by approximately 40 alphanumeric characters.

**Blast radius:** Any orchestrator, script, or Claude session that calls `claude mcp get` produces transcript output containing the full bearer token. The token flows into the session JSONL, which is scrubbed by `secrets-redaction-runtime.md` patterns at session end. But the CURRENT regex table does NOT include `lin_api_*` — the token would survive the scrub.

**Affected flows:**
1. Any orchestrator that calls `mcp get` to inspect or debug MCP server configuration
2. Any Claude session where the user or agent runs `claude mcp get` interactively
3. Any hook or tool that uses `mcp get` output programmatically

**Mitigation (immediate):** Parse `.mcp.json` source files directly to inspect MCP configuration rather than using `claude mcp get`. Do not add `mcp get` to any automated pipeline.

### 6.2 Gap: `~/.claude/claude_code_config.json` stores plaintext GitHub PAT

**Evidence:** `~/.claude/claude_code_config.json` was observed during the MCP probe phase to contain a plaintext `GITHUB_PERSONAL_ACCESS_TOKEN` value. This is a legacy config file format predating the `~/.claude/.env` canonical secrets store. [mcp.md §"Plaintext bearer-token exposure" + §"open questions" Q3]

**Blast radius:** Any process that reads or logs this config file exposes the GitHub PAT. The file is in the `~/.claude/` directory, which is in-scope for the session JSONL scrub (if any tool reads its contents into a transcript). `github_pat_*` format IS in the current `secrets-redaction-runtime.md` regex table — but only if the token follows the `ghp_` prefix format. Classic tokens (40-hex format) may not match.

**Mitigation:** Migrate this credential to `~/.claude/.env` per `reference_canonical_secrets_store_env.md`. Verify the token format matches a covered regex class or extend coverage.

### 6.3 Proposed regex extension: Linear PAT coverage

**Gap:** `lin_api_[A-Za-z0-9]{40}` format is not in `secrets-redaction-runtime.md` regex table.

**Proposed addition to the `JQ_PROGRAM` block in `hooks/secrets-redaction-scrub.sh`:**

```
# Linear PAT — lin_api_ + 40 alphanumeric chars
gsub("lin_api_[A-Za-z0-9]{40}"; "<REDACTED-lin_api-\(.[0:8])>")
```

**Replacement format:** `<REDACTED-lin_api-{first8}>` — 8 chars for audit disambiguation (longer than the standard 4 because `lin_api_` prefix is already fixed, so 4 chars would always start with `lin_`).

**Extension procedure** (per `secrets-redaction-runtime.md` §"Extension procedure"):**
1. Document new class in `~/.claude/.env.template` (add `LINEAR_API_KEY=` entry with precedence note).
2. Add regex to `hooks/secrets-redaction-scrub.sh` JQ_PROGRAM block. Place after `github_pat_` (shorter prefix, so longer prefix `lin_api_` should come earlier per ordering rule).
3. Add entry to the regex table in `rules/secrets-redaction-runtime.md`.
4. Add verification scenario in `.context/<follow-up-issue>/test-plan.md`.
5. Run dry-run against synthetic JSONL containing a `lin_api_<40-char>` value.

**Linear follow-up:** File a new issue in AGENTOPS: "Extend secrets-redaction-runtime.md to cover Linear PATs (`lin_api_*` format)." Priority: High (security, AO-153 lineage). Assignee: Ali.

### 6.4 Codex credential hygiene

Two Codex-specific hygiene notes beyond the env-strip finding in §5.4:

**Note 1: `bearer_token_env_var` field behavior in Codex config.** The `bearer_token_env_var` field in Codex MCP server config is documented as accepting an environment variable NAME (the env var whose VALUE is the bearer token). In the live probed config, the field contained the literal token value directly, not an env var name. Codex's behavior when this field contains a literal token (rather than a name) is uncharacterized — it may silently use the literal, it may fail auth, or it may expose the literal via error messages. This is gap #6 in §8. [codex.md §10 Divergence §1]

**Note 2: `~/.codex/auth.json` is file-based (not Keychain).** Codex stores auth at `~/.codex/auth.json` (mode 600), not in Keychain. Any script that reads this file (or `jq` output containing the OPENAI_API_KEY field) will expose the API key if the ChatGPT auth path is not active. The `OPENAI_API_KEY` key in the file appeared as `null` in the probed instance (ChatGPT/tokens path active), but fleet configurations using API key auth will have a non-null value. Include `~/.codex/auth.json` reads in scrub scope consideration. [auth.md §5]

---

## §7 Design Implications for Orchestrator v2

This section directly addresses the 8 open questions from `decisions/008 §5` plus one new question surfaced by the AO-263 expertise phase. The singletons' grounded facts are applied to each question. No design decisions are made here — that is the architect's job. Facts are made legible for decision-making.

### Q1: Completion detection mechanism

**decisions/008 §5 question:** "How does the orchestrator know a spawned builder or REPL session has completed its work?"

**What the singletons now tell us:**

The Stop hook is insufficient for interactive REPL completion detection. Stop fires when the model yields control back to the user prompt — it does not confirm the REPL session has ended or that the work is done. A Stop hook on a long-running REPL will fire on every model turn, not just the final one.

Three viable paths have been empirically characterized:

1. **SubagentStop + transcript-tail.** SubagentStop fires when a subagent (Task-tool) completes, and its `agent_transcript_path` field points to the per-subagent JSONL. Orchestrator can tail the JSONL for a completion sentinel. This is the most structured path. Requires the work to be dispatched as a subagent (Task tool), not as a standalone REPL. [hooks.md §3.6]

2. **Explicit in-band signal via emit-event.sh.** Builder emits a structured completion event to `events.jsonl` via a Bash tool call at work completion. Orchestrator polls or tails `events.jsonl`. Existing implementation in substrate. Works for any dispatch model but requires builder discipline. [decisions/008 §5.Q1]

3. **A hypothetical cmux blocking-call primitive (name unverified).** Could be named `wait-for` or something else; cmux.md does NOT enumerate this primitive in its CLI subcommand surface or RPC API documentation. **[HYPOTHESIS — name and existence both unverified; the architect-tier review of AO-270 grep-confirmed `wait-for` is not in cmux.md.]** IF such a primitive exists in cmux 0.63.2 and accepts a regex or glyph condition (e.g., wait until REPL prompt glyph appears), it would be a zero-configuration completion detector for cmux-hosted REPL sessions. Needs a fresh empirical probe (`cmux --help` re-walk + RPC method enumeration) to confirm or rule out before relying on it.

**For interactive REPL sessions (the hardest case):** The REPL prompt glyph U+276F (confirmed 0xE2 0x9D 0xAF) is a reliable "model is ready for input" signal. A pane-content grep for this glyph after a quiet period indicates the REPL is idle. This is the tmux `capture-pane` pattern and is implementable via cmux's compat surface or directly via tmux if installed. [claude-code.md §"REPL glyph"]

### Q2: Workspace naming (`builder-builder` bug)

**decisions/008 §5 question:** "Workspace names collide when multiple builders use the same naming scheme."

**What the singletons now tell us:**

cmux `workspace-action rename` and `new-workspace --name <name>` are confirmed primitives. The fix is a one-line substrate change: generate unique workspace names at spawn time (e.g., `builder-<issue-id>-<timestamp>` or `builder-<uuid[0:8]>`). No architectural decision needed — just implementation discipline in the spawn call. [cmux.md §"workspace lifecycle"]

### Q3: claude.ai connector model

**decisions/008 §5 question:** "How does the orchestrator interact with claude.ai connectors vs plugins?"

**What the singletons now tell us:**

Claude.ai connectors and plugins may expose the same MCP endpoint under different namespaces simultaneously. Live: `claude.ai Figma` (connector) and `plugin:figma:figma` (plugin) were both present in `mcp list` output despite same URL. The dedup logic may not have fired, or "hidden" has a different meaning than assumed. For orchestrator MCP tool routing: namespace ambiguity is a real condition, not a theoretical one.

The connector namespace prefix is `claude.ai <name>`. Plugin namespace prefix is `plugin:<vendor>:<server>`. For MCP PreToolUse matchers, both must be accounted for if they can coexist. [auth.md §6, §10 Divergence §3]

### Q4: tmux/iTerm2 as second InteractiveMultiplexer backend

**decisions/008 §5 question:** "Should the substrate support tmux or iTerm2 as a BackendAdapter alongside cmux?"

**What the singletons now tell us:**

The BackendAdapter interface is fully mapped for both hypothetical backends (§3.9). All 6 core primitives are implementable via tmux CLI. iTerm2 Python API fills more cmux-gap primitives than tmux (streaming, push notifications, session focus). cmux provides ~30 additional commands beyond what either backend offers natively.

Recommendation (from tmux.md §4): tmux is better for headless/durable/SSH-capable contexts; iTerm2 for local/GUI-rich contexts; cmux remains the primary for the current single-machine setup. The AO-163 deferral is worth revisiting when the Layer 1 substrate needs headless or remote execution.

Blocking note: live tmux probes were impossible (tmux not installed on the probe machine). `brew install tmux` is the gate before AO-163 implementation. [tmux.md §3, §4]

### Q5: Hooks vs in-band signaling for subagent observability

**decisions/008 §5 question:** "Should the orchestrator use hooks or in-band signaling (emit-event.sh) to track subagent work?"

**What the singletons now tell us:**

The AO-266 FALSIFICATION (Finding F-1) changes this design space significantly. Hooks are NOW VIABLE for Task-tool subagent telemetry:
- PreToolUse fires for subagent tool calls with `agent_id` + `agent_type` fields
- PostToolUse fires for subagent tool completions with `tool_response` (actual output)
- SubagentStop fires at subagent completion with `agent_transcript_path`

Hook `exit_code:2` maps to `outcome: "error"` in stream-json (semantically misleading — see Finding B-3). Orchestrators reading stream-json must not treat this as a system failure.

28 event types are vendor-documented; 6 are paste-confirmed. The 22 unconfirmed events may offer additional observability primitives (e.g., PermissionRequest, Notification) that could enrich the dispatch telemetry.

**BLOCKING PRECONDITION — Gap #7:** The hook-based subagent observability recommendation applies ONLY to Task-tool dispatched subagents. Custom-agent dispatch (via `--agents` flag) and SDK Agent() dispatch paths were NOT re-probed in AO-266. Before generalizing "hooks reliable for subagent telemetry" to all dispatch models, gap #7 (§8) must be resolved with a targeted probe. Proceeding with hook-based completion detection for Task-tool dispatches only is safe. Generalizing is NOT safe until gap #7 is closed.

### Q6: MCP integration shape for koinos-context

**decisions/008 §5 question:** "What is the MCP integration shape between the substrate and koinos-context?"

**What the singletons now tell us:**

koinos-context is NOT OAuth-discoverable: returns 401 WITHOUT WWW-Authenticate header. This is a spec violation (RFC 9728 requires WWW-Authenticate on 401 responses for protected resources) but is the current behavior. Two working workarounds:
1. `headersHelper` script (writes `{"Authorization": "Bearer <token>"}` to stdout)
2. `${VAR}` in headers field (now confirmed working on 2.1.128 per Finding F-2)

Both paths are empirically reliable. The `headersHelper` approach has an additional process spawn per MCP request; `${VAR}` in headers is lighter-weight. New integrations should prefer `${VAR}` headers if the Claude Code version is 2.1.128+.

For the substrate's Layer 1 in-process Agent SDK `query()` path (used for verifier + koinos-context MCP per AO-214): this inherits parent env and bills via API key, not subscription. This is the known AO-213/AO-214 gap — the env-strip applies to subprocess spawns, not in-process calls. The MCP path for koinos-context remains the preferred integration surface; the gap is billing discipline, not capability. [mcp.md §"OAuth/PKCE flow"; auth.md §8]

### Q7: Layer 2 trigger conditions

**decisions/008 §5 question:** "What conditions trigger the transition from Layer 1 to Layer 2 orchestration?"

**What the singletons now tell us:**

The AO-263 expertise phase reveals which design assumptions were gaps vs confirmed capabilities:

**Confirmed capabilities (Layer 2 can build on these):**
- Spawn contract: cmux `new-workspace` + bundled claude wrapper — confirmed [cmux.md §"Init mechanics"]
- Env injection: CMUX_* vars + hooks via --settings — confirmed [cmux.md §"Env injection chain"]
- set-status observability: hidden subcommands working, two pills live-confirmed [cmux.md §set-status]
- Worktree pipeline: git worktree + builder isolation — confirmed [tmux.md §3 context]
- SubagentStop-based completion: NEWLY CONFIRMED via AO-266 for Task-tool dispatches [hooks.md §6.2]

**Remaining gap for Layer 1 E2E test:** Completion detection for interactive REPL sessions (not SubagentStop — that's for Task-tool). The `cmux wait-for` hypothesis needs a probe. Without resolving REPL completion detection, the Layer 1 E2E test cannot close its loop.

**Layer 2 trigger:** "Layer 1 E2E test passes with completion detection solved." The completion detection gap is the remaining blocker. Infrastructure is ready; the gap is a specific primitive, not a broader architectural gap.

### Q8: Falsification triggers for D-S68-015

**decisions/008 §5 question:** "What would falsify the D-S68-015 design assumptions (the current orchestration design)?"

**What the singletons now tell us:**

Design assumptions that STAND (confirmed):
- Spawn contract exists (cmux primitives confirmed)
- Env injection works (16-var chain confirmed)
- set-status + sidebar observability works (live roundtrip confirmed)
- Worktree isolation is viable
- SubagentStop-based completion detection is NEWLY VIABLE for Task-tool dispatches

Design assumptions that do NOT stand:
- "Stop hook reliably detects session completion" — FALSIFIED for interactive REPLs (Stop fires on each model yield, not session end)
- "`--max-budget-usd` as pre-turn cost gate" — FALSIFIED (post-turn enforcement confirmed)

**Strongest falsification trigger for D-S68-015:** If Task-tool subagent hook coverage is found to NOT generalize to custom-agent or SDK Agent() dispatches (gap #7), then hook-based completion detection applies only to the Task-tool dispatch model and the design must either restrict to that model or maintain both hook-based and in-band signal paths in parallel.

### Q9: Codex as alternative compute substrate

**New question from the AO-263 expertise phase (not in decisions/008 §5).**

**What the singletons now tell us:**

Codex 0.128.0 has three dispatch modes:
1. `codex exec --json` — headless task execution; event stream is paste-confirmed
2. `codex review` — read-only code review; useful as a reviewer agent alternative
3. `codex mcp-server` — Codex as MCP stdio server; recursion primitive (Codex callable as a tool from Claude Code)
4. `--remote ws://<endpoint> --jwt <token>` — experimental WebSocket remote orchestration

The `codex exec --json` path is the viable headless dispatch path. Event stream shape differs from Claude Code stream-json but is structurally similar.

**Constraints for Codex as alternative compute:**
- OPENAI_API_KEY not stripped from subprocesses (Finding S-2) — must configure `shell_environment_policy`
- MCP parity with Claude Code is conditional (plugin HTTP MCP injected only for Codex-schema fields; `headersHelper` dropped)
- cmux hooks (3 hooks) fire only when CMUX_SURFACE_ID is set — Codex agents behave differently outside cmux
- `bearer_token_env_var` literal-fallback behavior uncharacterized — potential silent misconfiguration

**Orchestrator decision:** Codex as alternative compute is viable for code-review tasks (`codex review`) and self-contained execution tasks (`codex exec`). Verify env-strip and MCP config before deploying in any path that handles sensitive context.

---

## §8 Open Questions

The 9 known empirical gaps from the Phase 1.5 coverage audit. Each rated BLOCKING, MATERIAL, or LOW for v2 design implications.

### Gap 1: `cmux codex-hook` subcommand surface not enumerated
**Rating:** MATERIAL
**Source:** cmux.md §open questions; codex.md §7 (receiving-end documented, sending-end not)
**Why it matters:** If `cmux codex-hook` has richer primitives than `cmux claude-hook`, the Codex dispatch path may have different observability properties. The SessionStart and Stop hooks installed via `codex install-hooks` are gated on CMUX_SURFACE_ID — but the full event type menu for `cmux codex-hook` is unknown.
**Recommended action:** Low-effort targeted probe: `cmux codex-hook --help` followed by each enumerated subcommand. One agent session.

### Gap 2: `claude install` + `claude setup-token` blocked by --HELP guard
**Rating:** LOW
**Source:** claude-code.md §"blocked probes"
**Why it matters:** These are one-time flows (installation, token setup), not orchestration-loop primitives. Not blocking for v2 design.
**Recommended action:** Accept as-is for v2. Use vendor docs or GitHub source if needed. No Linear issue required.

### Gap 3: Agent SDK `query()` defaults (maxTurns, system prompt composition, env inheritance)
**Rating:** MATERIAL
**Source:** claude-code.md §"× Agent SDK"; auth.md §8 (AO-214 context)
**Why it matters:** If the substrate's Layer 1 uses in-process SDK `query()` for verifier or koinos-context MCP calls, the env inheritance behavior (no ANTHROPIC_API_KEY strip for in-process calls per AO-214) affects billing. maxTurns default and system prompt composition defaults affect context isolation between orchestrator and tool calls.
**Recommended action:** File a Linear issue: "Probe Agent SDK query() defaults — maxTurns, system prompt, env inheritance." Blocks AO-214 resolution.

### Gap 4: Plugin-MCP OAuth token storage location
**Rating:** LOW for v2 design; MATERIAL for secrets-hygiene audit
**Source:** mcp.md §"open questions"; auth.md §"× MCP"
**Why it matters:** `~/.claude/.credentials.json` is inferred as the likely location (file present at mode 600, 5526 bytes) but harness-blocked from inspection.
**Recommended action:** Request Ali authorization for `cat ~/.claude/.credentials.json | jq keys` (no values, keys only). Trivial resolution; 5-minute probe.

### Gap 5: Settings.json hot-reload not re-verified on 2.1.128
**Rating:** LOW
**Source:** hooks.md §7 (flagged as not re-verified in this session)
**Why it matters:** Memory `reference_settings_json_hot_reload.md` from S-24 (claude 2.1.119) says hot-reload works. If this behavior regressed in 2.1.128, hooks injected mid-session would not fire.
**Recommended action:** Accept for now. Re-verify if hook dispatch fails unexpectedly after an in-session settings edit. The S-24 memory is the current best evidence.

### Gap 6: `bearer_token_env_var` literal-fallback behavior in Codex
**Rating:** MATERIAL
**Source:** codex.md §10 Divergence §1
**Why it matters:** The probed Codex config had a literal token value in the `bearer_token_env_var` field (not an env var name as documented). If Codex silently uses the literal value, misconfigured fleet agents may work by accident but expose credentials. If Codex fails silently, agents using this config pattern will have broken MCP auth with no error message.
**Recommended action:** File a Linear capture: "Probe Codex `bearer_token_env_var` behavior with literal value vs env var name." Priority: Medium.

### Gap 7: Subagent-hook coverage for custom-agent / SDK Agent() dispatches
**Rating:** BLOCKING for generalizing hook-based subagent telemetry
**Source:** hooks.md §D2 (Task-tool confirmed; other paths not probed)
**Why it matters:** The "hooks reliable for subagent telemetry" recommendation (Finding F-1, §7 Q5) is confirmed only for Task-tool dispatched subagents (`agent_type: "general-purpose"`). Custom-agent dispatch via `--agents` flag and SDK Agent() dispatch paths were NOT re-probed in AO-266. If hooks do NOT fire for custom-agent dispatches, the observability model for the substrate's builder/verifier agents (which use custom-agent paths) is different from the Task-tool path.

**This gap is BLOCKING for:**
- Updating `rules/linear.md` "Subagent-hook caveat"
- Recommending hook-based completion detection for all dispatch models in orchestrator v2
- Closing `decisions/007` parity claim update that depends on subagent behavior

**Recommended action:** High-priority targeted probe: spawn a custom-agent (via `--agents` flag) and observe whether PreToolUse + PostToolUse + SubagentStop fire for its tool calls. One focused agent session. File a Linear issue before beginning the probe.

*Note: This gap also appears in §7 Q5 as a BLOCKING PRECONDITION for the hook-based observability recommendation.*

### Gap 8: iTerm2 Python API live-probe gap
**Rating:** LOW (cmux is primary backend; iTerm2 is future-phase)
**Source:** tmux.md §2.3
**Why it matters:** The iTerm2 Python API mapping in §3.9 is vendor-docs-based, not paste-evidenced. Full characterization of the streaming, push-notification, and session-focus primitives requires `pip install iterm2` + iTerm2 prefs API toggle.
**Recommended action:** Accept for v2. Re-open when AO-163 (iTerm2 BackendAdapter) triggers. `pip install iterm2` is the gate.

### Gap 9: Live tmux probes impossible (tmux not installed)
**Rating:** LOW for v2 design given BackendAdapter mapping is complete; BLOCKING for AO-163 implementation
**Source:** tmux.md §1
**Why it matters:** All tmux.md characterization of live behavior is sourced from man.openbsd.org, GitHub wiki, and claude binary `strings` decompilation, not from live execution. The hypothetical TmuxBackend mappings are well-grounded but not paste-confirmed.
**Recommended action:** `brew install tmux` before starting AO-163 implementation. Not blocking v2 design discussion. Low priority until AO-163 is triggered.

---

## §9 References — Section-Level Index

Section-to-singleton mapping. For granular claim tracing, read the singleton directly; the source citation in each finding indicates the specific section.

| Synthesis section | Primary singleton sources | Key paste-evidence location |
|---|---|---|
| §1 Executive Summary | All 7 singletons | Per facts 1–5: hooks.md §6.2, mcp.md §Divergences §1, codex.md §5.4, mcp.md §Divergences §3, decisions/008 §5.Q1 |
| §2 Per-Surface Primer | cmux.md §1–§3, claude-code.md §1–§2, hooks.md §1–§3, mcp.md §1–§4, auth.md §1–§5, tmux.md §1–§2, codex.md §1–§5 | Orientation table derived from PHASE-1.5-REVIEW.md coverage matrix |
| §3.1 ClaudeCode × cmux | claude-code.md §"× cmux"; cmux.md §"Init mechanics", §"Env injection chain", §"The bundled claude wrapper" | REPL glyph xxd paste (claude-code.md); env injection 16-var list (cmux.md) |
| §3.2 ClaudeCode × hooks | claude-code.md §"× hooks", §"stream-json"; hooks.md §1–§4, §6.1 | 7 SessionStart fires (hooks.md); PostToolUse tool_response schema (hooks.md §D1) |
| §3.3 ClaudeCode × MCP | claude-code.md §"× MCP"; mcp.md §"--mcp-config", §"Transport: HTTP", §Divergences §1–§2 | ${VAR} server-side capture (mcp.md §Divergences §1); strict-mcp-config `-p` vs `mcp list` (mcp.md §Divergences §2) |
| §3.4 ClaudeCode × auth | claude-code.md §"× auth"; auth.md §1 Tests 1–5 | authMethod vs apiKeySource paste (auth.md §1 Test 3); --bare paste (auth.md §1 Test 4) |
| §3.5 hooks × subagents | hooks.md §6.2, §3.6, §D2 | PreToolUse subagent stdin with agent_id (hooks.md §6.2); SubagentStop schema (hooks.md §3.6) |
| §3.6 MCP × auth | mcp.md §"OAuth/PKCE flow"; auth.md §2, §6, §8 | OAuth discovery dance 4-step paste (mcp.md); Keychain entries list (auth.md §2) |
| §3.7 Codex × cmux | codex.md §7; cmux.md §"codex install-hooks", §"Sharp Edges §1" | hooks.json 3-hook paste (codex.md §7); install-hooks accidental execution evidence (cmux.md §Sharp Edges §1) |
| §3.8 Codex × auth | codex.md §5.4; auth.md §5 | `env \| grep OPENAI_API_KEY` live test (codex.md §5.4); auth.json jq keys (auth.md §5) |
| §3.9 tmux × BackendAdapter | tmux.md §3, §4 | BackendAdapter interface verbatim (tmux.md §3); --tmux enum rejection (tmux.md §1.4) |
| §3.10 auth × cmux | auth.md §8; cmux.md §"Env injection chain" | CMUX_* present + ANTHROPIC_API_KEY absent in live env scan (auth.md §8) |
| §4 Findings | Per finding: hooks.md, mcp.md, codex.md, claude-code.md, auth.md, tmux.md, cmux.md — see inline citations | Full paste evidence in the singleton; finding rows cite specific §section locations |
| §5 Falsification Register | feedback_subagent_hook_inheritance_gap.md; reference_claude_code_mcp_headers_env_var_broken.md; feedback_codex_does_not_strip_openai_api_key.md; decisions/008_s68-pivot-conflict-audit.md §2 | STATUS boxes in the 2 falsified memories (added 2026-05-05); §2 verbatim quote from decisions/008 |
| §6 Secrets-Hygiene | mcp.md §Divergences §3; mcp.md §"open questions" (claude_code_config.json); codex.md §10 Divergence §1 | `claude mcp get linear` output paste (mcp.md §Divergences §3); `bearer_token_env_var` pitfall (codex.md §10) |
| §7 Design Implications | decisions/008 §5; hooks.md §3.6, §6.2; cmux.md §"workspace lifecycle"; mcp.md §"OAuth/PKCE"; auth.md §8; codex.md §3, §7, §8; tmux.md §3, §4 | SubagentStop agent_transcript_path (hooks.md §3.6); cmux new-workspace confirmed (cmux.md); Codex exec --json event stream (codex.md §3) |
| §8 Open Questions | PHASE-1.5-REVIEW.md §"Gaps to flag" (all 9 gaps verbatim); hooks.md §D2 (gap 7 BLOCKING); codex.md §10 (gap 6) | PHASE-1.5-REVIEW.md is the primary source; each gap back-references the singleton that flagged it |

---

*End of document. Research date: 2026-05-05. Binary versions pinned in §2. This document supersedes `landscapes/cmux-features.md`.*
