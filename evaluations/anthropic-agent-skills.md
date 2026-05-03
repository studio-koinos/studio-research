---
title: "Anthropic Agent Skills & Managed Agents — Capability Profile"
description: "Cloud-hosted managed agent runtime from Anthropic. Capability profile across 12 dimensions for Mission 2 BackendAdapter evaluation: spawn, send, read, status, kill, list, persistence, auth, pricing, reach, visibility, portability."
tags: [anthropic, managed-agents, backend-adapter, mission-2, evaluation, capability-profile]
modified: 2026-04-29
status: active
confidence: high
staleness: 30
related:
  - ../landscapes/agentic-fleet-platforms.md
  - ../landscapes/agentic-substrate-distribution.md
---

# Anthropic Agent Skills & Managed Agents — Capability Profile

> **Research date:** 2026-04-29
> **Last verified:** 2026-04-29
> **Staleness warning:** Re-verify after 30 days. Claude Managed Agents launched April 8, 2026 and is in public beta; all endpoints require the `managed-agents-2026-04-01` beta header. Multi-agent (research preview) and memory features require separate access requests. Pricing and rate limits are beta-era numbers subject to change.
> **Confidence summary:** ~60 VERIFIED claims (primary Anthropic docs, fetched 2026-04-29), ~8 LIKELY claims, 3 UNVERIFIED claims, 0 PARAMETRIC claims.
>
> **Primary sources (top 5 by load-bearing weight):**
> - Managed Agents overview: https://platform.claude.com/docs/en/managed-agents/overview (2026-04-29)
> - Sessions API: https://platform.claude.com/docs/en/managed-agents/sessions (2026-04-29)
> - Multi-agent API: https://platform.claude.com/docs/en/managed-agents/multi-agent (2026-04-29)
> - Agent Skills overview: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview (2026-04-29)
> - Pricing page: https://platform.claude.com/docs/en/about-claude/pricing (2026-04-29)

---

## 1. TL;DR

Claude Managed Agents is Anthropic's fully managed cloud agent harness, launched April 8, 2026 (public beta). [VERIFIED, src 1, 2] It exposes four API resources — Agents, Environments, Sessions, and Events — that map cleanly onto the Mission 2 BackendAdapter primitive set (`spawn`/`send`/`read`/`status`/`kill`/`list`). The product is cloud-only, Claude-model-only, and not available on Bedrock or Vertex. [VERIFIED, src 5, 11]

**Verdict on backend fit:** Managed Agents is the highest-impedance-match cloud backend for Mission 2's `spawn/send/read/status/kill/list` abstraction — all six primitives have documented, stable REST endpoints as of the beta launch. The primary impedance mismatch is that Managed Agents is opinionated about container management (you must provision an Environment before spawning a Session), and the multi-agent delegation model (coordinator declares `callable_agents` at definition time, one level deep only) differs from our substrate's more dynamic dispatch model. Vendor lock-in and cloud-only availability are the strategic risks.

---

## 2. Product Identity & Maturity

### What is it?

There are two distinct but related Anthropic products under evaluation. The naming causes confusion:

| Product | What it is | Status |
|---|---|---|
| **Agent Skills** (original) | A packaging format: filesystem-based SKILL.md bundles that extend Claude's capabilities. Introduced October 2025. | GA on claude.ai; beta via API with `skills-2025-10-02` header |
| **Claude Managed Agents** | A cloud agent harness: managed infrastructure for running Claude as a persistent, tool-using agent in a sandboxed container. Launched April 8, 2026. | Public beta, `managed-agents-2026-04-01` header required |

[VERIFIED, src 1, 3] They compose: Managed Agents sessions can have Agent Skills attached to them, but they are independent products. The evaluation request's framing — "Anthropic Agent Skills / Claude managed agents product" — encompasses both, with Managed Agents being the operationally relevant backend candidate for Mission 2.

### Maturity signal

[VERIFIED, src 1] Claude Managed Agents launched April 8, 2026 as a public beta open to all Anthropic API accounts. The beta header `managed-agents-2026-04-01` is required on every request; the Python/TypeScript/Go/Java/C#/Ruby/PHP SDKs set it automatically. Multi-agent sessions and the memory store are in **research preview** (a separate, gated tier) — [request access form required](https://claude.com/form/claude-managed-agents). [VERIFIED, src 6]

Early enterprise adopters cited in Anthropic materials include Notion, Asana, Rakuten, and Sentry. [LIKELY, src 9]

---

## 3. The 12 Capability Dimensions

### 3.1 Spawn Semantics

Instantiation is a three-step REST sequence: [VERIFIED, src 4]

**Step 1 — Create an Agent** (define behavior; reusable across sessions):

```bash
POST /v1/agents
{
  "name": "Engineering Lead",
  "model": "claude-opus-4-7",
  "system": "You coordinate engineering work.",
  "tools": [{"type": "bash_20250124"}, {"type": "text_editor_20250429"}],
  "skills": [{"type": "anthropic", "skill_id": "xlsx"}],
  "callable_agents": [...]   # for multi-agent only
}
# → returns agent.id + agent.version
```

**Step 2 — Create an Environment** (define container; reusable across sessions):

```bash
POST /v1/environments
{
  "name": "python-dev",
  "config": {
    "type": "cloud",
    "packages": {"pip": ["pandas", "numpy"]},
    "networking": {"type": "unrestricted"}
  }
}
# → returns environment.id
```

**Step 3 — Create a Session** (spawn a live container instance):

```bash
POST /v1/sessions
{
  "agent": "<agent_id>",        # or {"type":"agent","id":"...","version":1} to pin
  "environment_id": "<env_id>",
  "vault_ids": ["<vault_id>"]   # optional: MCP OAuth credential vaults
}
# → returns session.id, status: "idle"
```

[VERIFIED, src 4] Sessions start in `idle` state. No work begins until the first event is sent (see §3.3). Agents and Environments are reusable durable resources; Sessions are per-task instances.

**Impedance note for BackendAdapter:** There is no single `spawn(command, cwd, env, metadata) → session_id` call. The substrate's BackendAdapter will need to either (a) pre-create Agents and Environments as named resources and cache their IDs, or (b) create them on demand and accept the 2-step latency overhead before the session is `idle`. Option (a) is the natural fit for the fleet registry pattern.

### 3.2 Send Semantics

Input is delivered as typed **events** posted to `/v1/sessions/:id/events`. [VERIFIED, src 4]

```bash
POST /v1/sessions/<session_id>/events
{
  "events": [
    {
      "type": "user.message",
      "content": [{"type": "text", "text": "List the files in the working directory."}]
    }
  ]
}
```

Event types the client can send:

| Type | Description |
|---|---|
| `user.message` | Standard user turn (text, image, document content blocks) |
| `user.tool_confirmation` | Allow/deny a tool that requires confirmation |
| `user.custom_tool_result` | Return result for a custom tool Claude called |
| `user.interrupt` | Stop the currently running agent; session returns to `idle` |

[VERIFIED, src 4] The send call is non-blocking — it enqueues the event; the response is received via streaming (see §3.4). Mid-execution sends (to guide or interrupt) are supported; the session is a state machine that accepts events at `idle` and responds by transitioning to `running`.

### 3.3 Read / Output Semantics

Output is delivered via **Server-Sent Events (SSE)** streamed from: [VERIFIED, src 4, 6]

```bash
GET /v1/sessions/<session_id>/stream   # session-level (primary thread)
GET /v1/sessions/<session_id>/threads/<thread_id>/stream  # per-agent thread
```

Key event types returned on the stream:

| Type | Description |
|---|---|
| `agent.message` | Claude's text response; content blocks (text, tool_use) |
| `agent.tool_result` | Result of a tool call executed in the container |
| `session.idle` | Agent finished its current work; session back to `idle` |
| `session.thread_created` | Sub-agent thread spawned (multi-agent only) |
| `session.thread_idle` | Sub-agent thread finished |
| `agent.thread_message_sent/received` | Inter-agent messaging (multi-agent only) |

[VERIFIED, src 6] Event history is persisted server-side. You can fetch historical events via:

```bash
GET /v1/sessions/<session_id>/threads/<thread_id>/events   # paginated list
```

[VERIFIED, src 4] Structured output contract: `agent.message` events contain typed content blocks. For multi-agent sessions, the primary stream is a condensed aggregate; per-thread streams show full reasoning and tool traces.

**Structured output gap:** Managed Agents does NOT expose the Messages API's `--output-format stream-json` / NDJSON format used by Claude Code CLI headless mode. The SSE event format is its own protocol. BackendAdapters targeting the Claude Code CLI backend and the Managed Agents backend will need separate stream parsers.

### 3.4 Status / Kill / List

[VERIFIED, src 4] All three operations have documented endpoints:

**Status (retrieve one session):**
```bash
GET /v1/sessions/<session_id>
# → {id, status: "idle"|"running"|"rescheduling"|"terminated", ...}
```

Session statuses:

| Status | Meaning |
|---|---|
| `idle` | Waiting for input; starts here, returns here after each agent turn |
| `running` | Agent actively executing (tools, inference) |
| `rescheduling` | Transient error; automatic retry in progress |
| `terminated` | Unrecoverable error; session ended |

**List (all sessions in org):**
```bash
GET /v1/sessions   # paginated
# → {data: [{id, status, ...}], ...}
```

**Kill (interrupt running agent → idle):**
```bash
# Send interrupt event — for graceful stop:
POST /v1/sessions/<session_id>/events
{"events": [{"type": "user.interrupt"}]}

# Archive (freeze, preserve history, block new events):
POST /v1/sessions/<session_id>/archive

# Delete (permanent — cannot delete while running):
DELETE /v1/sessions/<session_id>
```

[VERIFIED, src 4] Running sessions cannot be directly deleted; you must interrupt first, then delete. Session deletion removes the event history and container; Agent and Environment resources are independent and unaffected.

### 3.5 Persistence Model

[VERIFIED, src 1, 4] **Persistent sessions.** Sessions maintain conversation history server-side across multiple user messages. A session is not ephemeral (one-prompt → one-response → done); it is stateful and can receive multiple event batches over its lifetime. File system state in the container persists within a session but does not persist across sessions — each new session gets a fresh container instance. Environments define the template; sessions are instantiated copies.

Memory store (cross-session persistence of knowledge) is in research preview as of research date. [VERIFIED, src 6]

**Session duration:** No documented hard limit found in docs. Runtime billing is per-`running`-millisecond with idle time free, which implies sessions can remain `idle` indefinitely without accruing cost. [LIKELY, src 5, 12]

**Thread persistence in multi-agent:** Sub-agent threads are persistent within a session — the coordinator can send follow-ups to a previously called sub-agent and that agent retains its prior conversation history. [VERIFIED, src 6]

### 3.6 Reach

[VERIFIED, src 1, 11] Cloud-only. No local invocation path. Claude Managed Agents runs exclusively on Anthropic's own infrastructure. The service is NOT available via:
- AWS Bedrock
- Google Vertex AI
- Microsoft Azure Foundry

The Anthropic pricing page states explicitly: "Claude Managed Agents is available only through the Claude API directly." [VERIFIED, src 5]

Claude models themselves are available on all three clouds, but those surfaces expose only the Messages API, not the Managed Agents harness. [VERIFIED, src 5, 11]

**Strategic implication for Mission 2:** For local backends (subprocess-headless, cmux, tmux) the substrate controls the process directly. For Managed Agents, all execution is remote. There is no hybrid path. The BackendAdapter for Managed Agents is a pure HTTP client with no local subprocess involvement.

### 3.7 Visibility

[VERIFIED, src 1, 4] Observability surfaces available:

- **Session event history:** Full event log accessible via `GET /v1/sessions/:id/threads/:tid/events` (paginated). Server-side persistent.
- **Per-thread stream:** Drill into any sub-agent's full reasoning + tool call trace.
- **Session status polling:** `GET /v1/sessions/:id` returns current status.
- **Claude Console:** Anthropic's web console provides organization-level usage and billing dashboards. Specific agent-level log surfacing in the console was not confirmed in docs reviewed. [LIKELY, src 1]

No native webhook push for session completion was documented; the client is expected to poll the SSE stream or status endpoint. [LIKELY, src 4]

### 3.8 Authentication

[VERIFIED, src 1, 4] Standard Anthropic API key authentication:

```bash
-H "x-api-key: $ANTHROPIC_API_KEY"
-H "anthropic-version: 2023-06-01"
-H "anthropic-beta: managed-agents-2026-04-01"
```

The API key is organization-scoped. No per-agent or per-session token scoping was documented as of the research date. [UNVERIFIED — docs describe API key auth only; whether fine-grained scoping is planned is not stated]

**MCP OAuth credential vaults:** [VERIFIED, src 4] For MCP tools requiring OAuth, Anthropic provides a **Vaults** abstraction — credentials are stored server-side, referenced by `vault_id` at session creation, and Anthropic manages token refresh. This means the client does not need to pass credentials on every session.

### 3.9 Pricing / Quota

[VERIFIED, src 5] Three billing dimensions:

| Dimension | Rate |
|---|---|
| Input tokens | Model-rate (e.g., Opus 4.7: $5/MTok) |
| Output tokens | Model-rate (e.g., Opus 4.7: $25/MTok) |
| Session runtime | $0.08 per session-hour (billed to millisecond; `running` status only; idle is free) |
| Web search | $10 per 1,000 searches |
| Prompt caching | Standard cache multipliers apply (10% of base input price for cache reads) |

[VERIFIED, src 5] Explicitly **NOT** available for Managed Agents sessions:
- Batch API 50% discount (sessions are stateful/interactive)
- Fast mode (inference speed managed by runtime)
- Data residency `inference_geo` parameter
- Third-party platform pricing (Bedrock/Vertex rates do not apply)

**Worked example from pricing docs:** 1-hour coding session, Claude Opus 4.7, 50k input + 15k output tokens = **$0.705 total** ($0.25 input + $0.375 output + $0.08 runtime). With cache hits on 40k of the input tokens: **$0.525 total**. [VERIFIED, src 5]

**Rate limits:** [VERIFIED, src 1, with rate correction from toolworthy.ai src 9]

| Operation | Documented limit |
|---|---|
| Create endpoints (POST agents/sessions/environments) | 300 req/min per org |
| Read endpoints (GET retrieve/list/stream) | 600 req/min per org |

Note: toolworthy.ai cited 60 req/min for creates. The Anthropic docs page stated 300 req/min. The docs value is treated as authoritative. [CONFLICTING between sources; Anthropic docs preferred]

**Concurrency limits:** Not publicly documented as of research date. [UNVERIFIED]

### 3.10 Maturity / Known Limitations

[VERIFIED, src 1] **Beta status.** All endpoints require the beta header. Anthropic states: "Behaviors may be refined between releases to improve outputs."

**Documented constraints:**

| Constraint | Detail |
|---|---|
| Max skills per session | 20 (across all agents for multi-agent sessions) |
| Multi-agent delegation depth | 1 level only — coordinator can call sub-agents; sub-agents cannot call further sub-agents |
| Multi-agent access | Research preview; gated behind access request form |
| Memory store | Research preview; gated |
| Batch API | Not supported for Managed Agents sessions |
| Fast mode | Not supported |
| Data residency | Not supported (`inference_geo` is Messages API-only) |
| Cross-surface Skills sync | Skills uploaded via API are NOT available on claude.ai and vice versa |
| Environments not versioned | No built-in environment version tracking; client must log externally |
| No ZDR eligibility | Agent Skills are not covered by Zero Data Retention arrangements |

[VERIFIED, src 3, 5, 6]

**Community-surfaced concerns:** [VERIFIED, src 10, 11]
- **Vendor lock-in:** VentureBeat and InfoQ coverage both highlighted that Managed Agents is Anthropic-infra-only, with no Bedrock/Vertex support and no multi-model option.
- **Context management risk:** Practitioners noted that Managed Agents manages context compaction automatically (it is advertised as a feature — built-in prompt caching and compaction). For workloads where irreversible context discarding is a failure mode, this is a risk.
- **No open standard:** At least one founder noted that the Skills packaging format, while described by Anthropic as an "open standard," does not currently have cross-vendor adoption or a published standard body governance structure. [LIKELY]

### 3.11 Vendor Lock-In / Portability

[VERIFIED, src 5, 11] Managed Agents is available only on Anthropic's own infrastructure. Migration paths:

| Capability | Portability |
|---|---|
| Claude model itself | Available on Bedrock, Vertex, Foundry via Messages API |
| Managed Agents harness | Anthropic-only; no equivalent harness on Bedrock/Vertex |
| Agent Skills (SKILL.md format) | Filesystem-based; portable to Claude Code (local) or any Messages API usage with code execution |
| Session event history | Fetchable via API (paginated list); JSON format; client can archive |
| MCP tool integrations | MCP is an open standard; tools are re-connectable to any MCP-compatible client |
| Container filesystem | No documented export/snapshot mechanism; session data does not persist across sessions |

**Lock-in assessment:** The harness is locked. The model is not (Messages API on Bedrock/Vertex). Skills packaging is portable. The session event log is exportable. MCP tool definitions are portable. Container state is ephemeral-by-design and not portable. For an organization that needs multi-cloud or model-agnostic agent execution, Managed Agents is a strategic commitment to Anthropic infrastructure specifically.

---

## 4. Mission 2 BackendAdapter Fit Assessment

Mission 2 needs a `BackendAdapter` implementing these six primitives: `spawn / send / read / status / kill / list`.

### Primitive-level mapping

| Primitive | Managed Agents equivalent | Notes |
|---|---|---|
| `spawn(command, cwd, env, metadata) → session_id` | `POST /v1/agents` + `POST /v1/environments` + `POST /v1/sessions` → `session.id` | **3 calls instead of 1.** Agent + Environment can be pre-created and cached to reduce to 1 call at spawn time. |
| `send(session_id, text) → delivery_receipt` | `POST /v1/sessions/:id/events` `{type:"user.message", content:[...]}` | Clean 1:1 match. Supports rich content (text, image, document). |
| `read(session_id, lines?, since?) → structured_events` | `GET /v1/sessions/:id/stream` (SSE) or `GET /v1/sessions/:id/threads/:tid/events` (history) | SSE stream is push-model, not pull. BackendAdapter needs an SSE consumer rather than a polling reader. Event format is Managed-Agents-specific, not the Messages API stream-json format. |
| `status(session_id) → {state, agent_type}` | `GET /v1/sessions/:id` → `{status: "idle"\|"running"\|"rescheduling"\|"terminated"}` | Clean 1:1. Status names differ from substrate's vocabulary (idle/running vs idle/running/waiting/done); mapping layer needed. |
| `kill(session_id) → ok` | `POST /v1/sessions/:id/events` `{type:"user.interrupt"}` then optionally `DELETE /v1/sessions/:id` | Two-step for clean termination. Interrupt → `idle`, then delete. Cannot delete while `running`. |
| `list() → [{session_id, agent_type, metadata, state}]` | `GET /v1/sessions` (paginated list) | Clean 1:1. Pagination must be handled. |

### Impedance mismatches

**1. Pre-provisioned resources:** Unlike subprocess or tmux backends where `spawn` is atomic, Managed Agents requires pre-existing Agent and Environment resources. The BackendAdapter needs to implement a resource-pool or registry pattern: at substrate init time, create named Agents and Environments and cache their IDs. Spawn then becomes a single `POST /v1/sessions`.

**2. SSE vs polling:** The substrate's `read()` primitive assumes a pull model (buffer snapshot or log lines). Managed Agents is push-SSE. The adapter needs to maintain an SSE listener per session and buffer events for `read()` callers. This is implementable but non-trivial.

**3. Event format divergence:** The Claude Code CLI headless backend emits NDJSON events with types like `system`, `assistant`, `tool_result`. Managed Agents emits SSE events with types like `agent.message`, `session.idle`, `agent.tool_result`. The BackendAdapter for each backend will need its own event translator to the substrate's normalized event schema. This is expected for a multi-backend design.

**4. Multi-agent delegation model:** Managed Agents uses a declared `callable_agents` list at Agent definition time, with one delegation level. Our substrate's dispatch model is more dynamic (any session can be told to spawn a sub-session at runtime). The mismatch means multi-agent coordination via Managed Agents is pre-wired at agent creation, not dynamic at dispatch time.

**5. No local execution path:** All compute runs in Anthropic's cloud container. There is no hybrid model where some tools run locally and Managed Agents handles inference. For workloads that need local file access or local tool execution, Managed Agents is not usable as the backend.

### Overall assessment

**High fit for cloud-hosted, long-running, asynchronous agent work.** All six primitives are supported with documented stable endpoints. The mismatch points (resource pre-provisioning, SSE consumer, event translation) are addressable in a single adapter implementation (estimated 2–3 days of implementation work for a competent engineer). The strategic risks (cloud-only, Claude-only, no Bedrock/Vertex) are policy decisions, not technical blockers.

**Not a fit for:** local-first workloads, multi-model comparisons, Bedrock/Vertex deployments, workloads requiring container state persistence across sessions.

---

## 5. Open Questions / Unknowns

1. **Session duration limits:** No documented maximum session lifetime was found. Unclear whether a session that stays `idle` for hours/days eventually expires or is terminated by the platform. [UNVERIFIED]

2. **Concurrency limits:** The rate limits table covers request/minute rates, not concurrent running sessions. How many concurrent `running` sessions an org-tier account can sustain is not publicly documented. [UNVERIFIED]

3. **Agent SDK vs Managed Agents distinction for BackendAdapter:** The Claude Agent SDK (`@anthropic-ai/claude-agent-sdk` / `claude-agent-sdk-python`) is a separate product that wraps Claude Code CLI capabilities programmatically. It is NOT the same as Managed Agents. For Mission 2, Managed Agents (REST API) is the correct cloud backend surface. The Agent SDK targets a different use case (programmatic Claude Code invocation, not cloud container orchestration). The two should not be conflated in dispatch routing.

4. **Skills cross-surface sync:** Whether the roadmap includes a unified Skills registry (API ↔ claude.ai ↔ Claude Code) was not documented. Today each surface is separate. [UNVERIFIED]

5. **Anthropic-managed vs customer-managed containers:** The environment config API accepts `"type": "cloud"` — whether customer-managed compute (BYOC) is planned was not found in docs. All container execution is Anthropic-hosted as of the research date. [UNVERIFIED]

6. **Outcome and memory store APIs:** Docs reference "outcomes" and a "memory store" as research preview features. Their API shape was behind the access gate; no docs were available for review. [UNVERIFIED]

---

## 6. Sources

| # | URL | Accessed |
|---|---|---|
| 1 | https://platform.claude.com/docs/en/managed-agents/overview | 2026-04-29 |
| 2 | https://www.infoq.com/news/2026/04/anthropic-managed-agents/ | 2026-04-29 |
| 3 | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview | 2026-04-29 |
| 4 | https://platform.claude.com/docs/en/managed-agents/sessions | 2026-04-29 |
| 5 | https://platform.claude.com/docs/en/about-claude/pricing | 2026-04-29 |
| 6 | https://platform.claude.com/docs/en/managed-agents/multi-agent | 2026-04-29 |
| 7 | https://platform.claude.com/docs/en/managed-agents/skills | 2026-04-29 |
| 8 | https://platform.claude.com/docs/en/managed-agents/environments | 2026-04-29 |
| 9 | https://www.toolworthy.ai/tool/claude-managed-agents | 2026-04-29 |
| 10 | https://venturebeat.com/orchestration/anthropics-claude-managed-agents-gives-enterprises-a-new-one-stop-shop-but | 2026-04-29 |
| 11 | https://www.cloudproinc.com.au/index.php/2026/04/08/anthropic-openai-and-google-are-all-locking-in-enterprise-customers-how-to-manage-vendor-risk/ | 2026-04-29 |
| 12 | https://www.finout.io/blog/anthropic-just-launched-managed-agents.-lets-talk-about-how-were-going-to-pay-for-this | 2026-04-29 |
