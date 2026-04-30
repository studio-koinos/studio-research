---
title: "Agentic Fleet Platforms — Capability Registry"
description: "Central registry for Mission 2: 12-dim capability matrix × N backends with verdicts, integration patterns, and BackendAdapter shape implications. Promoted from .context/AO-104/synthesis-progress.md (S-27)."
research_date: 2026-04-29
last_verified: 2026-04-29
staleness_warning: "Cloud-platform surfaces evolve monthly. Re-verify per-platform after 30 days. Anthropic Managed Agents (beta), Cloudflare Agents Week 2026, and OpenAI Symphony are all 2026-Q1/Q2 — high velocity. Maintenance discipline tracked via studio-radar RDR issues."
confidence: high
sources_count: 104
related:
  - ../evaluations/anthropic-agent-skills.md
  - ../evaluations/cloudflare-agents.md
  - ../evaluations/openai-codex-cloud.md
  - ../evaluations/cmux.md
  - ./anthropic-agent-product-spectrum.md
  - ./openai-agent-product-spectrum.md
  - ./cloudflare-agent-services-stack.md
  - ./agentic-substrate-distribution.md
  - ../../studio-pm/research/terminal-agent-orchestration-landscape.md
  - ../../studio-pm/research/terminal-ops-cao-evaluation.md
status: active
---

# Agentic Fleet Platforms — Capability Registry

> **Research date:** 2026-04-29
> **Last verified:** 2026-04-29
> **Staleness warning:** Re-verify per-platform after 30 days. All cloud surfaces (Anthropic Managed Agents in beta, Cloudflare Agents Week 2026 products, OpenAI Symphony) are in high-velocity states. Pricing and API surfaces may change without notice.
> **Sources:** 104 total across 7 input documents (4 deep dives + 3 spectrum/stack landscapes) + 3 cross-reference documents.
> **Promoted from:** `.context/AO-104/synthesis-progress.md` (S-27 working scratchpad)

---

## 1. TL;DR

Mission 2 of studio-agent-ops is building the agent fleet substrate: a `BackendAdapter` abstraction that lets the terminal-ops orchestrator dispatch agent work to any execution backend — local process, managed cloud, or edge runtime — without baking in a single vendor at the call-site.

Seven research documents (4 deep dives + 3 spectrum landscapes) were synthesized across four vendor surfaces: Anthropic, OpenAI, Cloudflare, and cmux. The result is this registry: 12 capability dimensions × 7 backends, with verdicts, per-backend integration patterns, and the architectural tensions that drive the BackendAdapter design.

**Top 3 strategic findings:**

1. **Symphony reframes Layer 1 design.** OpenAI released Symphony (2026-03-05, Apache 2.0) as a reference implementation of what AO-165 (Layer 1 solo-repo orchestrator) is supposed to do: poll Linear → spawn isolated Codex agents via Codex App Server JSON-RPC 2.0 stdio → CI/proof-of-work gate → land PR. The Elixir/BEAM stack is not fleet-portable, but the pattern and SPEC.md are the design document. Layer 1 becomes "Symphony-pattern reimplementation in TypeScript/bun." [VERIFIED, src 37-44]

2. **Cloudflare Agents Week 2026 (Apr 13–17) reshapes the cloud-tier landscape.** ~20 products shipped in 5 days: Sandboxes GA, Workflows v2 (50k concurrency up from 4,500), Agent Memory (private beta), AI Search (open beta), Cloudflare Mesh, Durable Object Facets. AI Gateway now covers 14+ providers. Ali decided AI Gateway is in for fleet observability pilot. [VERIFIED, src 56-104]

3. **MCP-over-stdio is converging as the cloud-agent integration default.** Anthropic references MCP as integration surface; Cloudflare's Workers host MCP servers; OpenAI's Codex recommends `codex mcp-server` for programmatic integration. BackendAdapter cloud tier may collapse to "MCP adapter + platform-specific server config" rather than per-vendor adapter implementations. [VERIFIED, src 1-14, 21-36, 56-104]

**Top 5 backends by Mission-2 fit (ordered):**

1. Anthropic Managed Agents — all 6 primitives have stable REST endpoints; highest primitive coverage
2. Cloudflare Agents (DO + Agents SDK) — durable-state differentiator; fleet observability via AI Gateway
3. subprocess-headless — current Phase 1 substrate; local, ephemeral, zero dependency
4. tmux — persistent, remote-capable, any-layer; correct successor to subprocess for always-on Layer 2/3
5. OpenAI Codex Cloud (via `codex mcp-server`) — viable secondary cloud backend; GitHub-locked; 3/6 primitives clean

**What Ali should decide next:** Whether AO-165 (Layer 1 orchestrator) should adopt the Symphony SPEC.md directly as its design document (TypeScript reimplementation) or design from scratch. The Symphony pattern is validated; the question is how closely to follow the spec.

---

## 2. Backend Roster

Every backend the substrate may target. Mission-2 fit: **High** (primary candidate), **Med** (viable with gaps), **B-tier** (scoped role), **Low** (deferred or out-of-scope), **n/a** (infrastructure layer, not a dispatch backend).

| Backend | Tier | Persistence | Reach | Visibility | Maturity | Mission-2 Fit | Deep-Dive |
|---|---|---|---|---|---|---|---|
| **subprocess-headless** | local | ephemeral | local-only | none/stdout | GA (de facto) | High — Phase 1 | terminal-agent-orchestration-landscape.md §2 |
| **cmux** | local | ephemeral | macOS-only | GUI-rich (sidebar pills) | 0.x beta (v0.63.2) | B-tier — Layer 1 local-interactive only | evaluations/cmux.md |
| **tmux** | local-or-remote | durable (session survives disconnect) | local or remote SSH | terminal-only | GA | High — Layer 2/3 always-on | terminal-agent-orchestration-landscape.md §3 |
| **iTerm2** | local | ephemeral | macOS-only | GUI (pane view) | GA | B-tier — less capable than cmux | terminal-agent-orchestration-landscape.md §3 |
| **Anthropic Managed Agents** | cloud | durable-within-session | cloud-only (direct API) | SSE stream + Console | Beta (Apr 2026) | High — Phase 2 cloud primary | evaluations/anthropic-agent-skills.md |
| **OpenAI Codex Cloud** | cloud | ephemeral (12h cache) | cloud (GitHub-locked) | web UI + CLI list | GA | Med — secondary cloud, via MCP | evaluations/openai-codex-cloud.md |
| **Cloudflare Agents (DO)** | cloud-edge | durable (SQLite, hibernation) | cloud-edge global | dashboard + wrangler tail | GA (SDK v0.11.9) | Med — Phase 2 cloud alternative | evaluations/cloudflare-agents.md |
| **Codex App Server (local)** | local-or-remote | durable (thread-based) | local or remote | JSONL stdout | GA | High — correct Codex integration surface | openai-agent-product-spectrum.md §4.13 |
| **AI Gateway** | cloud (proxy layer) | n/a | cloud-global | dashboard + logs | GA | n/a — observability above substrate, not a backend | cloudflare-agent-services-stack.md §4 |
| **Cloudflare Workflows v2** | cloud | durable (step checkpoints) | cloud-global | dashboard | GA | Med — Layer 2 dispatch engine alternative | cloudflare-agent-services-stack.md §6 |
| **Cloudflare Sandboxes** | cloud | persistent (backup/restore) | cloud | dashboard | GA (Apr 2026) | Med — tool-execution isolation | cloudflare-agent-services-stack.md §7 |

**Not-a-backend clarifications:**
- AI Gateway sits ABOVE the substrate as a fleet observability + routing layer. It is not dispatched to; all inference calls route through it.
- Workflows v2 is an orchestration motor (outer harness), not an agent backend. It calls backends.
- Anthropic Messages API + Claude Code CLI: Claude Code CLI is the fleet's current execution substrate (not a candidate — already in production).

---

## 3. Capability Matrix

12 dimensions × 7 primary backends. Cell values are concrete and source-grounded.

| Dim | subprocess-headless | cmux | tmux | Anthropic Managed Agents | OpenAI Codex Cloud | Cloudflare Agents | Codex App Server |
|---|---|---|---|---|---|---|---|
| **1. Spawn semantics** | one-step: `spawn(argv, env, cwd)` → handle | two-step: `workspace.create` → `surface.create`; handle via list | one-step: `new-session -d -s name -c cwd` | **three-step**: `POST /agents` → `POST /environments` → `POST /sessions` | one-step (CLI): `codex cloud exec --env ENV prompt` / SDK: `startThread()` | request-triggered: first HTTP/WS to `/:class/:name` instantiates DO | one-step: connect stdio → send `{"method":"codex/start"}`  |
| **2. Send semantics** | write to stdin; CLI arg for `claude -p` | `surface.send_text` (JSON-RPC); no bracketed-paste | `send-keys -t session text Enter` | `POST /sessions/:id/events` with typed event objects | `thread.run(prompt)` (SDK); `codex-reply` (MCP tool) | HTTP POST or WebSocket message to agent URL | JSON-RPC `codex/message` over stdio |
| **3. Read semantics** | pull-stream: stdout AsyncIterable NDJSON events | snapshot-poll: `surface.read_text` (plain text, scrollback) | snapshot-poll: `capture-pane -t session -p` | push-SSE: `GET /sessions/:id/stream` (event-typed stream) | snapshot-only: `codex cloud list --json` for status; no streaming cloud event feed | push-callback: `onMessage()` WebSocket or SSE `ReadableStream` | pull-stream: JSONL over stdio (streaming delta events) |
| **4. Status/kill/list** | status: exit-code / process-alive check; kill: SIGTERM; list: substrate registry | status: screen-poll (no native signal); kill: `workspace.close`/`surface.close`; list: `workspace.list` | status: `display-message -p '#{pane_pid}'`; kill: `kill-session`; list: `list-sessions` | status: `GET /sessions/:id` (idle/running/rescheduling/terminated); kill: `user.interrupt` event then `DELETE`; list: `GET /sessions` | status: `codex cloud list --json` (filter by id); kill: **none documented**; list: `codex cloud list --json` | status: custom `/status` route (no SDK native); kill: `abortSubAgent()` (child) / CF REST API (top); list: **none native** — self-built registry required | status: thread state in JSON-RPC responses; kill: close stdio; list: substrate registry |
| **5. Persistence** | ephemeral (dies with parent) | ephemeral (pane dies if cmux closes) | durable (session survives disconnect; lives until `kill-session`) | durable-within-session (conversation history server-side; fresh container per new session; memory store in research preview) | ephemeral-with-cache (task container cached 12h; thread IDs persist across SDK calls) | cloud-durable: SQLite per-DO survives hibernation, redeploys, crashes; in-memory state lost on hibernation | durable (thread IDs persist; `resume` flag reconnects) |
| **6. Reach** | local-only | macOS-only, local | local or remote (SSH); any OS on host | cloud-only (Anthropic API direct; no Bedrock/Vertex) | cloud-only (GitHub-locked; no GitLab/Bitbucket) | cloud-edge global (330+ cities); local dev via `wrangler dev` + Miniflare | local or remote (App Server on any host; `--remote ws://host:port` for remote TUI) |
| **7. Visibility** | none (stdout/stderr redirect) | GUI-rich: sidebar status pills, notification rings, `set-status`, `notify` | terminal-only: pane view, no GUI | SSE event history (server-side); Console usage dashboard [LIKELY] | web dashboard (`chatgpt.com/codex`); CLI `codex cloud list`; OTel (local only) | Cloudflare Dashboard (requests, errors); `wrangler tail` (real-time logs) | JSONL events on stdout; OTel export option |
| **8. Auth model** | n/a (local process) | macOS socket owner-check (default); password-mode available | n/a (local) / SSH key for remote | Anthropic API key (`x-api-key`); beta header required | ChatGPT OAuth + GitHub OAuth; `CODEX_API_KEY` for CI; SSO on Business tier | developer-implemented in `onRequest()`/`onConnect()`; MCP OAuth 2.1 via `McpAgent` | ChatGPT account auth (same as CLI) |
| **9. Pricing** | $0 (local process) | $0 (GPL OSS) | $0 (OSS) | $0.08/session-hour + model token rates; no Batch discount; web search $10/1k | $20/mo Plus (15-80 cloud tasks/5h window) to pay-as-you-go; cloud task costs ~5x local; reasoning-time billing since 2026-04-09 | Workers Paid ($5/mo) + CPU time ($0.02/M CPU-ms) + DO duration ($12.50/M GB-s, hibernation-reduced) | $0 for local App Server; cloud Codex task rates apply |
| **10. Maturity** | GA (de facto standard) | 0.x beta — pre-stable API; 844 open issues; `read-screen` production since PR #219 (Feb 2026) | GA — 30+ years; tmux 3.4 stable | Beta — `managed-agents-2026-04-01` header required; multi-agent in research preview | GA — core task execution; credit metering documented-unreliable; Business tier 3x faster drain since Apr 2026 | GA (SDK v0.11.9, 252 releases) — external PRs not accepted; Project Think in preview | GA (protocol) — actively used by Symphony, Codex desktop, VS Code extension |
| **11. Known limitations (1-line)** | no persistence, no GUI visibility, process-kill loses context | ephemeral hard limit; dead-PTY bug #1472 on background workspaces; no remote surface control yet | no native GUI; requires SSH for remote; no built-in fleet enumeration | beta-only; no Bedrock/Vertex; multi-agent delegation 1-level only; no Batch discount; context compaction automatic | no streaming cloud event feed; no programmatic kill; GitHub hard-dependency (Feb 9 2026 outage); credit metering unreliable | no native fleet enumeration; 6-subrequest limit per Worker invocation; DO APIs are proprietary; Project Think APIs unstable | requires local Codex binary; cloud routing adds GitHub dependency; resuming across machines requires thread ID persistence |
| **12. Vendor lock-in** | not-applicable (local) | medium (macOS-only; custom JSON-RPC wire protocol) | not-applicable (OSS standard) | concentrated (Anthropic-only harness; no multi-cloud; MCP tools portable; session logs exportable) | diffused across two vendors: OpenAI + GitHub (stiff); open-source CLI = escape hatch for local path | concentrated-in-DO (DO APIs proprietary; Workers request model uses Web Standards = portable; AI Gateway swappable) | diffused (open JSON-RPC protocol; Symphony SPEC.md published; open-source MIT) |

---

## 4. Per-Backend Short Summaries

### 4.1 subprocess-headless

**Verdict: High fit — Phase 1 primary backend**

The current fleet execution primitive. `spawn(argv, cwd, env)` → process handle → read stdout NDJSON events → SIGTERM on kill. Zero dependencies, zero cost, works in CI and cron contexts. All 6 BackendAdapter primitives have direct mappings to POSIX process semantics. The limitation is full: ephemeral (dies with parent), local-only, no GUI visibility, no persistence.

Mission 2 Phase 1 closes here: get the BackendAdapter interface right against subprocess-headless, then Phase 2 extends to cloud backends. The `claude -p --output-format stream-json` invocation is the current dispatch primitive per `studio-pm/research/terminal-agent-orchestration-landscape.md §2`. [VERIFIED, src 10]

**What Mission 2 should do:** Implement `SubprocessBackend` first. Treat its primitive shape as the interface definition point. Every subsequent backend must match the interface, not the semantics.

---

### 4.2 cmux

**Verdict: B-tier — Layer 1 local-interactive only**

cmux is the richest visibility backend in the roster: sidebar status pills, notification rings, `set-status` / `notify` primitives, and automatic git/PR metadata in the sidebar. It is already partially wired (AO-142 Phase 1 complete, observability primitives verified in production). The hard architectural limit is **ephemeral persistence**: pane processes die if cmux closes. This disqualifies it for always-on Layer 2/3 work.

Three active bugs affect automation: dead-PTY on programmatically-created background workspaces (#1472, open); `read-screen` fails after display sleep (#2004, open); `send_text` drops non-ASCII (#2756, open). The socket path is `~/Library/Application Support/cmux/cmux.sock` (not `/tmp/cmux.sock` as docs suggest). [VERIFIED, src 16-33]

**Memory drift:** S-20 memory `feedback_cmux_env_vs_backend.md` claimed "no public scrollback read" — this is outdated. PR #219 (2026-02-21) promoted `read-screen`/`surface.read_text` to production. Update or annotate the memory file.

**Important distinction:** cmux observability (set-status, notify, surface-ID env vars) is the **environment layer** and is orthogonal to which terminal-ops backend dispatches agents. A subprocess-headless dispatch can still publish cmux sidebar pills. Don't conflate the two concerns (per `feedback_cmux_env_vs_backend.md`). [VERIFIED, src 17]

**What Mission 2 should do:** Scope AO-162 to cmux observability integration (dispatch-independent). Build `CmuxBackend` only for Layer 1 local-interactive GUI deployments. Require cmux+tmux composition for persistence in that cell (per orchestrator-vision.md §3).

---

### 4.3 tmux

**Verdict: High fit — Layer 2/3 always-on backend**

tmux is the correct backend for always-on agents: sessions persist across disconnects, work on any OS (remote SSH), and have been stable for 15+ years. Primitive mapping: `new-session` = spawn; `send-keys` = send; `capture-pane` = read; `list-sessions` = list; `kill-session` = kill; session exits = status. The substrate's poll-based read model (capture-pane snapshot diffing) has latency (~1s per poll), but that is acceptable for background fleet agents.

Linear AO-163 (tmux backend for always-on Layer 2/3) is in Backlog — promote when AO-104 ships its subprocess-headless v1.

**What Mission 2 should do:** Implement `TmuxBackend` in Phase 1 or early Phase 2 as the persistence-capable local backend. Gate on AO-104 v1 interface stability.

---

### 4.4 Anthropic Managed Agents

**Verdict: High fit — Phase 2 cloud primary**

All 6 BackendAdapter primitives have documented, stable REST endpoints as of the April 8, 2026 public beta launch. `POST /agents` + `POST /environments` + `POST /sessions` = spawn (3-step, pre-cache agents and environments to reduce to 1 call at runtime). `POST /sessions/:id/events` with `user.message` = send. `GET /sessions/:id/stream` SSE = read (push model, needs SSE consumer in adapter). `GET /sessions/:id` = status (4 states: idle/running/rescheduling/terminated). `user.interrupt` event then `DELETE /sessions/:id` = kill (2-step). `GET /sessions` = list. [VERIFIED, src 1-14]

Three impedance mismatches: (1) pre-provisioned Agent and Environment resources (resource-pool pattern at substrate init time); (2) SSE push vs substrate pull-read semantics (adapter needs SSE listener per session + buffer); (3) multi-agent delegation is pre-wired at Agent definition time (`callable_agents`), not dynamic — differs from fleet's dynamic dispatch model.

Strategic risk: Anthropic-only (no Bedrock/Vertex). Policy constraint, not technical. Pricing: $0.08/session-hour + model tokens; no Batch discount. 1-hour Opus 4.7 session = ~$0.705 (with caching ~$0.525). [VERIFIED, src 5, 12]

**What Mission 2 should do:** Implement `AnthropicManagedBackend` in Phase 2. Pre-create named Agents and Environments at substrate init; spawn = `POST /sessions`. Build SSE consumer in the adapter; normalize Managed Agents event types to substrate's unified event schema. Document Anthropic-only as a policy constraint, not a technical gap.

---

### 4.5 OpenAI Codex Cloud

**Verdict: Viable secondary cloud backend — via `codex mcp-server`**

Codex Cloud is not a REST API you call from your orchestrator. There is no `POST /v1/codex/cloud/tasks` endpoint. The programmatic surfaces are: CLI (`codex cloud exec`), TypeScript SDK (`startThread`), MCP server (`codex mcp-server`). [VERIFIED, src 21-36] The MCP path is the correct BackendAdapter integration surface — it avoids CLI subprocess fragility, uses an open protocol, and returns `threadId` as the substrate's `session_id`.

Primitive gaps: `read` has no streaming cloud event feed (snapshot-only via `codex cloud list --json`); `kill` has no documented programmatic cancellation. Both are documented gaps to accept and surface as `not-supported` via capability flags.

Two hard constraints: GitHub dependency (Feb 9, 2026 outage took Codex Cloud down entirely); credit metering is documented-unreliable (Business tier drains ~3.2x faster than Plus since Apr 9, 2026 pricing change; improper 429 errors; silent quota drops). [VERIFIED, src 22, 27-36]

**What Mission 2 should do:** Integrate via `codex mcp-server` as the `CodexCloudBackend`. Treat `kill` and streaming `read` as `not-supported` (capability flags). Document GitHub as a hard dependency — gate fleet use on GitHub availability. Add cost monitoring before fleet adoption at scale.

---

### 4.6 Cloudflare Agents (Durable Objects)

**Verdict: Viable cloud alternative — Phase 2, durable-state differentiator**

Cloudflare's durable-state model (per-agent SQLite, WebSocket hibernation, pay-while-awake) is the strongest persistence differentiator in the roster — sessions survive hibernation, redeploys, and crashes without losing state. This is a meaningful architectural win over Anthropic Managed Agents' "ephemeral container per session" model for long-running cloud agents.

Primitive gaps: no native fleet enumeration (substrate must self-build a DO or D1 registry); no SDK-native status endpoint (custom `/status` route required); kill for top-level agents requires the Cloudflare REST Management API (not yet confirmed for fleet enumeration). [VERIFIED, src 37-55]

Lock-in assessment: Durable Objects APIs (`this.sql`, `this.state`, `schedule()`, `subAgent()`) are Cloudflare-proprietary. Business logic in agent methods is extractable TypeScript; the persistence and scheduling layers must be rewritten on migration. [VERIFIED, src 37-55]

AI Gateway complement: the fleet observability layer (AI Gateway) sits above the Cloudflare Agents backend and is separately adopted. AI Gateway works regardless of whether the agent backend is Cloudflare DO or Anthropic Managed — it's the universal inference proxy. [VERIFIED, src 56-104]

**What Mission 2 should do:** Implement `CloudflareAgentsBackend` in Phase 2, after BackendAdapter interface is stable from Phase 1. Build a self-maintained fleet registry (D1 or separate DO). AI Gateway adoption is independent and can start earlier (fleet observability pilot per Ali's decision).

---

### 4.7 Codex App Server (local/remote)

**Verdict: High fit — correct OpenAI programmatic dispatch surface**

The Codex App Server is the JSON-RPC 2.0 over stdio protocol that decouples Codex's core logic from its client surfaces (CLI, VS Code, web app, macOS desktop, JetBrains). Symphony is built on it. Any fleet component dispatching Codex agents should speak this protocol, not parse CLI string output. [VERIFIED, src 37-55]

April 2026 additions: Unix socket transport for same-host integration; pagination-friendly resume/fork; sticky environments. [VERIFIED, src 44] The protocol is open (JSON-RPC 2.0, SPEC.md published with Symphony). Switching from OpenAI's Codex to a competing CLI agent requires only a new client binding, not a full re-architecture.

**What Mission 2 should do:** Make `CodexAppServerBackend` the local Codex integration surface (not `codex exec` CLI). Study Symphony SPEC.md for the proof-of-work and WORKFLOW.md patterns. If implementing AO-165 Layer 1, this is the dispatch protocol to implement against.

---

## 5. Architectural Tensions

Eight cross-cutting tensions the research surfaced. Each has a BackendAdapter design implication.

### T1 — Spawn shape: one-step vs three-step pre-provisioning

Local subprocess: single `spawn(argv, env, cwd) → handle`. Anthropic Managed Agents: three-step REST sequence (Create Agent → Create Environment → Create Session). The BackendAdapter interface must accommodate pre-provisioned resources without forcing callers to manage them. Solution: `prepareResources(spec) → resourceRef` at substrate init time; `spawn(resourceRef, prompt) → handle` at dispatch time. Local backends collapse both into one call via a no-op `prepareResources`.

### T2 — Read shape: pull-stream vs push-callback vs snapshot-only

- subprocess-headless: pull-stream (AsyncIterable from stdout)
- Cloudflare Agents: push-first via WebSocket `onMessage()` callback
- Anthropic Managed Agents: push-SSE (`GET /sessions/:id/stream`)
- Codex Cloud: snapshot-only (no streaming cloud event feed)

BackendAdapter `read()` cannot mandate pull semantics. Either expose both shapes (pull + push-callback) as a discriminated return type, or model push-callbacks as an internal pull-buffered wrapper. For snapshot-only backends (Codex Cloud), `read()` returns the latest snapshot with no delta-streaming guarantee.

### T3 — List API: native vs substrate-required registry

- Anthropic Managed Agents: native `GET /sessions` (paginated list)
- Cloudflare Agents: none — Durable Objects exposes no instance enumeration; substrate must maintain its own registry (D1 table or dedicated DO)
- subprocess / tmux: substrate maintains in-memory map or `list-sessions` output

BackendAdapter `list()` is substrate-native for some backends, native for others. Spec must declare `list.semantics: "native" | "registry"` per backend so the fleet knows which backend requires the substrate's fleet registry and which can delegate.

### T4 — Delegation depth: none / one-level / recursive

- subprocess with Claude Code: recursive (spawned `claude -p` can call Skill→Agent on subagents)
- Anthropic Managed Agents: pre-wired one-level only (`callable_agents` list at Agent definition time; sub-agents cannot call further sub-agents)
- Cloudflare Agents: typed sub-agents via `subAgent()` (SDK-managed idempotent instantiation)

The fleet's dynamic dispatch model (any session can be told to spawn a sub-session at runtime) is not universally supported. Managed Agents requires static delegation graphs declared at agent creation time. BackendAdapter must expose `delegationModel: "dynamic" | "static-one-level" | "recursive"` as a capability flag.

### T5 — Persistence granularity: ephemeral / durable-within-session / cloud-managed

Three tiers found:
- ephemeral (subprocess, cmux, Codex Cloud task containers after 12h)
- durable-within-session (Anthropic Managed Agents: conversation history server-side; fresh container per new session)
- cloud-managed durable (Cloudflare DO: SQLite per agent survives hibernation, redeploys, crashes)

`SessionStatus` in the substrate must extend past `running/completed/failed` to include `hibernated` (Cloudflare) and `idle` (Anthropic Managed Agents session waiting for next event). These are not the same as `completed`.

### T6 — Lock-in concentration: concentrated vs diffused

Lock-in surfaces mapped:
- Cloudflare: concentrated in DO-specific APIs (`this.sql`, `schedule()`, `subAgent()`); Workers Web-standard request layer is portable
- Anthropic: concentrated in Managed Agents harness; Claude model itself is multi-cloud; MCP tools are portable
- Codex App Server: diffused (open protocol, open-source CLI, SPEC.md published)

BackendAdapter must enforce: every platform-specific surface (DO state, Anthropic resource pool, GitHub dependency) lives BEHIND the adapter interface. Caller code never touches. This is the primary architectural contract for lock-in discipline.

### T7 — Capability-budget intersection with cloud built-in toolsets

Anthropic Managed Agents comes with built-in tools: bash, read, write, edit, glob, grep, web_fetch, web_search. Capability budget intersection becomes `(parent budget) ∩ (Anthropic's built-in toolset)` rather than just the parent budget alone. The substrate's `resolveCapabilityBudget` currently computes pure intersection over caller-supplied lists; cloud backends introduce a third operand (platform's built-in tools). [VERIFIED, src 13]

BackendAdapter should expose `backend.builtinTools: string[]` so the substrate's budget resolver can intersect correctly.

### T8 — External dependency reliability

Codex Cloud has a hard GitHub dependency: Feb 9, 2026 GitHub outage took Codex Cloud down entirely. BackendAdapter's `status` primitive needs to surface backend-level reliability state (e.g., "dependency unavailable"), distinct from session-level state. A fallback chain concept follows: try cloud backend → degrade to local backend on reliability signal. This is a substrate-level concern, not per-adapter.

---

## 6. BackendAdapter Shape Implications

What the research concretely tells us about the BackendAdapter interface design for `terminal-ops/substrate/`.

### 6.1 Capability flags per backend

Every adapter declares supported primitives. Callers check before invoking; unsupported primitive returns `InvocationError: not-supported` rather than silent no-op.

```typescript
interface BackendCapabilities {
  spawn: true;            // required for all backends
  send: boolean;
  read: "streaming" | "snapshot" | "push-callback";
  status: boolean;
  kill: boolean;
  list: "native" | "registry";  // native: backend API; registry: substrate-maintained
  persistence: "ephemeral" | "durable-session" | "cloud-managed";
  reach: "local" | "local-or-remote" | "cloud-only";
  delegationModel: "dynamic" | "static-one-level" | "recursive";
  builtinTools: string[];  // cloud backends with pre-wired toolsets
  reliabilityDependencies: string[];  // e.g., ["github.com"] for Codex Cloud
}
```

### 6.2 Two-step spawn for cloud backends

```typescript
interface BackendAdapter {
  prepareResources(spec: AgentSpec): Promise<ResourceRef>;  // cloud: creates Agent + Environment; local: no-op
  spawn(ref: ResourceRef, prompt: string): Promise<SessionHandle>;
  // ... remaining primitives
}
```

`prepareResources` runs at substrate init time. Cloud backends pre-create named Agents and Environments and cache their IDs. Local backends return a `ResourceRef` that is the `spawn` arguments themselves.

### 6.3 Tri-modal read signature

```typescript
type ReadResult =
  | { mode: "streaming"; stream: AsyncIterable<NormalizedEvent> }
  | { mode: "snapshot"; events: NormalizedEvent[] }
  | { mode: "push-callback"; register: (cb: EventCallback) => Unsubscribe };
```

BackendAdapter returns the mode it supports; the substrate's `read()` caller dispatches on mode. Push-callback adapters (Cloudflare WebSocket) maintain internal buffering and expose a pull-stream shim if the caller needs pull semantics.

### 6.4 Extended SessionStatus

```typescript
type SessionStatus =
  | "idle"          // Anthropic: waiting for next event; tmux: no active process
  | "running"       // all backends: actively executing
  | "hibernated"    // Cloudflare: DO sleeping, WebSocket connections maintained
  | "completed"     // process exited cleanly / task finished
  | "failed"        // unrecoverable error
  | "unavailable";  // backend dependency down (Codex Cloud + GitHub outage)
```

### 6.5 MCP-over-stdio as cloud-tier collapse pattern

Three cloud backends converge on MCP-over-stdio as their integration surface:
- Anthropic: MCP referenced; Vaults for credential injection; MCP servers declared in Agent configs
- Cloudflare: Workers/DO host MCP servers (DO-backed per-session state, OAuth 2.1)
- Codex Cloud: `codex mcp-server` is the recommended programmatic integration surface

This suggests the cloud-tier BackendAdapter may collapse from three separate vendor adapters into a shared `MCPBackend` (MCP client connecting to a locally-running MCP server) + per-vendor server config. The adapter connects to a local `codex mcp-server` or a remote Anthropic MCP server; the protocol is identical. Pressure-test this during Phase 2 implementation.

### 6.6 Status as reliability surface

```typescript
interface StatusResult {
  sessionState: SessionStatus;
  backendHealth: "ok" | "degraded" | "unavailable";
  backendMessage?: string;  // e.g., "GitHub API unavailable (codex-cloud dependency)"
}
```

Substrate fallback chain evaluates `backendHealth` before dispatching: if `unavailable`, route to next backend in the chain (cloud → local fallback).

### 6.7 Lock-in discipline enforcement

The `BackendAdapter` interface is the enforcement mechanism. Every platform-specific surface (DO SQLite APIs, Anthropic session resource IDs, cmux JSON-RPC methods, GitHub OAuth) lives inside the adapter implementation. The calling site in the substrate sees only `SessionHandle` (opaque string), `ResourceRef` (opaque), and the normalized event types. No platform-specific objects cross the adapter boundary.

---

## 7. Strategic Constraints + Decisions

Surfaced and confirmed across all 7 research inputs.

| Constraint | Type | Detail | Source |
|---|---|---|---|
| **Anthropic Managed Agents = Anthropic-only** | Policy | No Bedrock, Vertex, Azure. Multi-cloud fleet scenarios cannot use it as a cloud backend. | evaluations/anthropic-agent-skills.md §3.6 |
| **Codex Cloud = GitHub-locked** | Hard dependency | GitHub outage on Feb 9, 2026 took Codex Cloud down entirely. Non-GitHub repos + high-availability needs = blockers. | evaluations/openai-codex-cloud.md §3.10 |
| **Codex Cloud = unreliable credit metering** | Operational risk | Apr 9, 2026 pricing change; Business tier drains ~3.2x faster; silent 429s; quota drops to zero. Fleet use needs out-of-band cost monitoring. | evaluations/openai-codex-cloud.md §3.9 |
| **cmux = ephemeral hard limit** | Architectural | Wrong tier for always-on Layer 2/3. cmux+tmux composition required for persistence. | evaluations/cmux.md §6 |
| **cmux dead-PTY bug** | Implementation bug | GitHub #1472 (open). Background-created workspaces may not initialize PTY. Workaround: focus+defocus dance. BackendAdapter must handle `"Surface not ready"`. | evaluations/cmux.md §2 |
| **cmux socket path mismatch** | Implementation | Docs say `/tmp/cmux.sock`; reality is `~/Library/Application Support/cmux/cmux.sock`. Derive dynamically via `CMUX_SOCKET_PATH` env var. | evaluations/cmux.md §9 |
| **Cloudflare DO lock-in is concentrated** | Lock-in risk | `this.sql`, `schedule()`, `subAgent()` are proprietary. Migration requires rewriting persistence + scheduling layers. | evaluations/cloudflare-agents.md §Dim 12 |
| **Symphony = reference implementation, not maintained product** | Adoption | OpenAI invites reimplementations via SPEC.md. We own our TypeScript reimplementation. No upstream support. | openai-agent-product-spectrum.md §3 |
| **OpenAI Assistants API + Custom GPTs sunset 2026-08-26** | Deadline | Hard deprecation. Any fleet component using either needs migration before then. File RDR monitoring issue. | openai-agent-product-spectrum.md §4 |
| **AI Gateway role: above substrate, not a backend** | Architecture | AI Gateway is the inference observability/routing layer. Substrate remains responsible for spawning; AI Gateway aggregates inference traces. Adoption independent of backend choice. | cloudflare-agent-services-stack.md §4 |
| **Cloudflare Agents Week 2026: Workflows v2 = credible Layer 2 alternative** | Architecture option | 50k concurrency, independently retryable steps, weeks-long execution. If Mission 2 builds Layer 2 (AO-166), Workflows v2 is a viable orchestration motor without building our own. Ali decision when Layer 2 design begins. | cloudflare-agent-services-stack.md §6 |
| **GPT-5.5 doubled per-token prices at launch** | Cost | Input $2.50 → $5.00/M. Batch API (50% off) + prompt caching (up to 90% off cached input) are primary mitigations at fleet scale. | openai-agent-product-spectrum.md §6 |
| **Layer-tier specificity is mandatory, not optional** | Architecture | subprocess/cmux = Layer 1 only; tmux = any layer; Anthropic Managed / Cloudflare DO = Layer 2/3 cloud; cmux+tmux composition = persistent Layer 1. Backend selection is layer-dependent. | synthesis-progress.md finding #10 |

**Open Ali decisions surfaced:**

1. **Symphony SPEC.md adoption for AO-165**: Adopt Symphony pattern directly (TypeScript reimplementation) or design Layer 1 from scratch? Symphony is validated and SPEC.md is ready. Recommendation: adopt the pattern, implement in TypeScript/bun.

2. **Cloudflare AI Gateway pilot**: Already decided (per synthesis-progress). Timeline and scope TBD. Highest immediate fleet observability ROI.

3. **Cloudflare Workflows v2 for Layer 2 (AO-166)**: When Layer 2 design begins, evaluate Workflows v2 as the outer dispatch harness vs building our own orchestration code. Decision point: when AO-166 enters design.

---

## 8. Recommended Deep-Dive Candidates

Products warranting their own `evaluations/<product>.md` beyond what currently exists. Prioritized by Mission-2 forcing function.

| Priority | Product | File | Rationale |
|---|---|---|---|
| 1 | **Symphony SPEC + Codex App Server protocol** | `evaluations/symphony-spec.md` | Direct AO-165 input. Study proof-of-work contract, WORKFLOW.md schema, JSON-RPC 2.0 protocol shape. Forcing function: AO-165 design. |
| 2 | **Cloudflare AI Gateway** | `evaluations/cloudflare-ai-gateway.md` | Ali already decided to pilot for fleet observability. Needs: routing rules, cost attribution patterns, DLP profile setup, Analytics Engine integration for custom agent metrics. |
| 3 | **Cloudflare Workflows v2** | `evaluations/cloudflare-workflows-v2.md` | Credible Layer 2 motor alternative. Needs: step design patterns for agent loops, event-wait for human-in-the-loop, v1→v2 migration, vs Durable Objects for short-lived coordination. |
| 4 | **Anthropic Managed Agents standalone** | `evaluations/anthropic-managed-agents.md` | Current deep-dive is merged with Agent Skills. A scoped evaluation should cover: session lifecycle at fleet scale, pricing model vs raw API, container constraints, SSE event taxonomy, head-to-head vs rolling-your-own loop. |
| 5 | **Cloudflare Sandboxes** | `evaluations/cloudflare-sandboxes.md` | GA as of Apr 13, 2026. Backup/restore for eval agent setup, egress policy, E2B/Daytona comparison. Relevant for Phase 3+ tool-use isolation. |
| 6 | **OpenAI Agents SDK Sandbox Agents** | `evaluations/openai-agents-sdk-sandbox.md` | Manifest-defined container workspaces, resumable sessions. Compare against Codex Cloud containers and fleet worktree model. |
| 7 | **OpenAI Workspace Agents** | `evaluations/openai-workspace-agents.md` | Research preview, launched 2026-04-22. Credit pricing from 2026-05-06. Evaluate post-GA for potential fleet dispatch surface. |
| 8 | **Amazon Bedrock Managed Agents (OpenAI-powered)** | `evaluations/bedrock-managed-agents.md` | AWS Bedrock Managed Agents (limited preview, April 2026) wraps OpenAI models. May expose REST API that Codex Cloud proper does not. Relevant if GitHub lock-in is a hard blocker. |

**Also worth tracking but not a dedicated evaluation yet:**
- Cloudflare Project Think (preview; `@cloudflare/think`) — durable execution fibers; monitor for GA
- Cloudflare Agent Memory (private beta) — cross-session memory recall; monitor for GA
- MCP via OpenAI Responses API integration pattern — fleet tool registry as remote MCP server
- Anthropic outcomes API + memory store API (research preview, gated)

---

## 9. Maintenance Discipline

### Staleness triggers (per platform)

| Platform | Staleness signal | Action |
|---|---|---|
| Anthropic Managed Agents | Any update to beta header, pricing, or session API | Re-verify evaluations/anthropic-agent-skills.md |
| Cloudflare Agents | SDK version bump (currently 0.11.9; external PRs not accepted) | Check changelog for breaking API changes |
| Cloudflare Project Think | Preview → GA transition | Full evaluation update |
| OpenAI Codex | Monthly changelog (`developers.openai.com/codex/changelog`) | Credit model + CLI flags most volatile |
| OpenAI Assistants API | Sunset date: 2026-08-26 | Confirm no fleet components using it before then |
| cmux | Version bump (currently v0.63.2; 4-6 releases/month) | Check for changes to JSON-RPC method set |
| Symphony | Any SPEC.md update | Re-evaluate AO-165 design alignment |

### RDR monitoring surface area

File RDR issues for each of the following (not blocking current work — capture discipline):
- Anthropic Managed Agents → GA announcement (remove beta header requirement)
- Cloudflare Project Think → GA announcement
- Cloudflare Agent Memory → GA announcement with pricing
- OpenAI Assistants API / Custom GPTs → sunset date tracking (2026-08-26)
- OpenAI Workspace Agents → pricing announcement (2026-05-06)
- cmux dead-PTY bug #1472 → resolution (unblocks fully-headless cmux dispatch)

### Promotion path from research to substrate code

```
Research input (evaluations/*.md or landscapes/*.md)
  → Architectural tension identified → this registry (Section 5)
    → BackendAdapter shape implication → this registry (Section 6)
      → Interface spec in terminal-ops/substrate/BackendAdapter.ts
        → Implementation in terminal-ops/substrate/backends/<name>.ts
          → Verification (per verification.md — live call + response)
            → Linear issue closed (AO-104 or child issue)
```

Memory files that carry implementation-critical findings (socket path, dead-PTY workaround, credit metering issues) should be updated when the underlying condition changes, with `supersedes` annotations per `decision-log.md`.

---

## 10. Sources

Sources are numbered sequentially. Cross-reference against in-text citations. All sources verified 2026-04-29 unless noted.

### Deep Dive Sources (1–55)

**Anthropic Agent Skills & Managed Agents deep dive** (`evaluations/anthropic-agent-skills.md`):
1. https://platform.claude.com/docs/en/managed-agents/overview (2026-04-29)
2. https://www.infoq.com/news/2026/04/anthropic-managed-agents/ (2026-04-29)
3. https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview (2026-04-29)
4. https://platform.claude.com/docs/en/managed-agents/sessions (2026-04-29)
5. https://platform.claude.com/docs/en/about-claude/pricing (2026-04-29)
6. https://platform.claude.com/docs/en/managed-agents/multi-agent (2026-04-29)
7. https://platform.claude.com/docs/en/managed-agents/skills (2026-04-29)
8. https://platform.claude.com/docs/en/managed-agents/environments (2026-04-29)
9. https://www.toolworthy.ai/tool/claude-managed-agents (2026-04-29)
10. https://venturebeat.com/orchestration/anthropics-claude-managed-agents-gives-enterprises-a-new-one-stop-shop-but (2026-04-29)
11. https://www.cloudproinc.com.au/index.php/2026/04/08/anthropic-openai-and-google-are-all-locking-in-enterprise-customers-how-to-manage-vendor-risk/ (2026-04-29)
12. https://www.finout.io/blog/anthropic-just-launched-managed-agents.-lets-talk-about-how-were-going-to-pay-for-this (2026-04-29)

**Cloudflare Agents deep dive** (`evaluations/cloudflare-agents.md`):
13. https://developers.cloudflare.com/agents/ + API reference + WebSocket docs (2026-04-29)
14. https://github.com/cloudflare/agents (v0.11.9) (2026-04-29)
15. https://www.cloudflare.com/agents-week/updates/ (2026-04-29)
16. https://blog.cloudflare.com/project-think/ (2026-04-29)
17. https://blog.cloudflare.com/agents-week-in-review/ (2026-04-29)
18. https://developers.cloudflare.com/agents/api-reference/sub-agents/ (2026-04-29)
19. https://developers.cloudflare.com/durable-objects/platform/pricing/ (2026-04-29)
20. https://developers.cloudflare.com/workers/platform/pricing/ (2026-04-29)
21. https://developers.cloudflare.com/workers/wrangler/ (2026-04-29)
22. https://developers.cloudflare.com/agents/api-reference/schedule-tasks/ (2026-04-29)
23. https://developers.cloudflare.com/agents/platform/limits/ (2026-04-29)
24. https://developers.cloudflare.com/workers-ai/platform/pricing/ (2026-04-29)
25. https://news.ycombinator.com/item?id=47792538 (2026-04-29)
26. https://inventivehq.com/blog/multi-cloud-strategy-vendor-lock-in-cloudflare-aws-azure-gcp (2026-04-29)

**OpenAI Codex Cloud deep dive** (`evaluations/openai-codex-cloud.md`):
27. https://developers.openai.com/codex/cloud (2026-04-29)
28. https://developers.openai.com/codex/changelog (2026-04-29)
29. https://en.wikipedia.org/wiki/OpenAI_Codex_(AI_agent) (2026-04-29)
30. https://developers.openai.com/codex/cloud/environments (2026-04-29)
31. https://developers.openai.com/codex/pricing (2026-04-29)
32. https://developers.openai.com/codex/sdk (2026-04-29)
33. https://developers.openai.com/codex/guides/agents-sdk (2026-04-29)
34. https://status.openai.com/incidents/01KH1KTV2VBD62R0MRZFJ13XNE + https://serenitiesai.com/articles/github-down-ai-coding-tools-dependency-2026 (2026-04-29)
35. https://community.openai.com/t/understanding-the-new-codex-limit-system-after-the-april-9-update/1378768 (2026-04-29)
36. https://github.com/openai/codex (2026-04-29)

**cmux deep dive** (`evaluations/cmux.md`):
37. https://github.com/manaflow-ai/cmux (2026-04-29)
38. https://www.ycombinator.com/companies/manaflow (2026-04-29)
39. https://github.com/manaflow-ai/cmux/releases (v0.61.0–v0.63.2) (2026-04-29)
40. https://github.com/manaflow-ai/cmux/blob/main/README.md (2026-04-29)
41. https://github.com/manaflow-ai/cmux/issues/1472 (dead PTY) (2026-04-29)
42. https://github.com/manaflow-ai/cmux/issues/2756 (non-ASCII paste) (2026-04-29)
43. https://github.com/manaflow-ai/cmux/pull/219 (production read-screen) (2026-04-29)
44. https://soloterm.com/cmux-vs-tmux (2026-04-29)
45. https://github.com/manaflow-ai/cmux/issues/2004 (display sleep bug) (2026-04-29)
46. https://gist.github.com/joelhooks/11aea283acfd5a7f50e596bc63bbdd28 (multi-agent spec) (2026-04-29)
47. https://github.com/manaflow-ai/cmux/issues/2673 (remote workspace) (2026-04-29)
48. https://www.mintlify.com/manaflow-ai/cmux/automation/socket-api (2026-04-29)

### Spectrum/Stack Landscape Sources (49–104)

**Anthropic product spectrum** (`landscapes/anthropic-agent-product-spectrum.md`):
49. https://platform.claude.com/docs/en/api/getting-started (2026-04-29)
50. https://platform.claude.com/docs/en/docs/about-claude/models/overview (2026-04-29)
51. https://platform.claude.com/docs/en/managed-agents/quickstart (2026-04-29)
52. https://platform.claude.com/docs/en/managed-agents/tools (2026-04-29)
53. https://modelcontextprotocol.io/docs/learn/architecture (2026-04-29)
54. https://platform.claude.com/docs/en/build-with-claude/prompt-caching (2026-04-29)
55. https://code.claude.com/docs/en/discover-plugins (2026-04-29)

**OpenAI product spectrum** (`landscapes/openai-agent-product-spectrum.md`):
56. https://github.com/openai/symphony (Apache 2.0) (2026-04-29)
57. https://www.infoq.com/news/2026/02/opanai-codex-app-server/ (2026-04-29)
58. https://openai.com/index/introducing-workspace-agents-in-chatgpt/ (2026-04-29)
59. https://community.openai.com/t/assistants-api-beta-deprecation-august-26-2026-sunset/1354666 (2026-04-29)
60. https://www.marktechpost.com/2026/03/05/openai-releases-symphony... (2026-04-29)
61. https://opentools.ai/news/openai-symphony-turns-linear-boards-into-autonomous-coding-agent-orchestration (2026-04-29)
62. https://openai.com/index/open-source-codex-orchestration-symphony/ (2026-04-29)
63. https://github.com/openai/openai-agents-python (2026-04-29)
64. https://github.com/openai/openai-agents-js (2026-04-29)
65. https://community.openai.com/t/introducing-the-responses-api/1140929 (2026-04-29)
66. https://community.openai.com/t/introducing-support-for-remote-mcp-servers... (2026-04-29)
67. https://openai.com/index/new-tools-for-building-agents/ (2026-04-29)
68. https://apidog.com/blog/gpt-5-5-pricing/ (2026-04-29)
69. https://techcrunch.com/2026/04/15/openai-updates-its-agents-sdk... (2026-04-29)
70. https://temporal.io/blog/announcing-openai-agents-sdk-integration (2026-04-29)
71. https://developers.openai.com/codex/changelog (April 2026) (2026-04-29)
72. https://techcrunch.com/2025/10/06/openai-launches-agentkit... (2026-04-29)
73. https://venturebeat.com/orchestration/openai-unveils-workspace-agents... (2026-04-29)
74. https://developers.openai.com/api/docs/guides/batch (2026-04-29)
75. https://developers.openai.com/api/docs/guides/function-calling (2026-04-29)

**Cloudflare services stack** (`landscapes/cloudflare-agent-services-stack.md`):
76. https://blog.cloudflare.com/agents-week-in-review/ (2026-04-17)
77. https://developers.cloudflare.com/agents/ (2026-04-29)
78. https://developers.cloudflare.com/ai-gateway/ (2026-04-29)
79. https://developers.cloudflare.com/durable-objects/ (2026-04-29)
80. https://blog.cloudflare.com/workflows-v2/ (2026-04-15)
81. https://developers.cloudflare.com/changelog/post/2026-04-13-containers-sandbox-ga/ (2026-04-13)
82. https://blog.cloudflare.com/sandbox-ga/ (2026-04-13)
83. https://developers.cloudflare.com/browser-rendering/ (2026-04-29)
84. https://blog.cloudflare.com/mesh/ (2026-04-14)
85. https://blog.cloudflare.com/ai-search-agent-primitive/ (2026-04-16)
86. https://blog.cloudflare.com/introducing-agent-memory/ (2026-04-17)
87. https://developers.cloudflare.com/vectorize/ (2026-04-29)
88. https://developers.cloudflare.com/workers-ai/platform/pricing/ (2026-04-29)
89. https://blog.cloudflare.com/workers-ai-large-models/ (2026-04-16)
90. https://developers.cloudflare.com/d1/ (2026-04-29)
91. https://developers.cloudflare.com/queues/ (2026-04-29)
92. https://blog.cloudflare.com/ai-platform/ (2026-04-15)
93. https://developers.cloudflare.com/ai-gateway/reference/pricing/ (2026-04-29)
94. https://developers.cloudflare.com/changelog/post/2026-04-15-workflows-limits-raised/ (2026-04-15)
95. https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/ (2026-04-29)
96. https://developers.cloudflare.com/agents/model-context-protocol/ (2026-04-29)
97. https://blog.cloudflare.com/thirteen-new-mcp-servers-from-cloudflare/ (2026-04-14)
98. https://blog.cloudflare.com/artifacts-git-for-agents-beta/ (2026-04-16)
99. https://blog.cloudflare.com/container-platform-preview/ (2026-04-29)
100. https://developers.cloudflare.com/analytics/analytics-engine/ (2026-04-29)
101. https://developers.cloudflare.com/logs/logpush/ (2026-04-29)
102. https://developers.cloudflare.com/r2/pricing/ (2026-04-29)
103. https://developers.cloudflare.com/agents/model-context-protocol/mcp-servers-for-cloudflare/ (2026-04-29)
104. https://developers.cloudflare.com/workers/platform/pricing/ (2026-04-29)

### Cross-Reference Documents (not counted in sources_count)
- `studio-pm/research/terminal-agent-orchestration-landscape.md` — S-19 local backends + CLI agents (49 sources, 2026-04-19)
- `studio-pm/research/terminal-ops-cao-evaluation.md` — CAO source-level eval, 3 borrowable primitives (2026-04-21)
- `studio-research/landscapes/agentic-substrate-distribution.md` — substrate distribution landscape (38 sources, 2026-04-27)
- `.context/AO-104/synthesis-progress.md` — working scratchpad (S-27) — promoted and superseded by this document
