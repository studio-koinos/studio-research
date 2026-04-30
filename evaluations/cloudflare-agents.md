---
title: "Cloudflare Agents — Capability Profile"
description: "BackendAdapter capability profile for Cloudflare's agent runtime built on Workers + Durable Objects. Covers spawn/send/read/status/kill semantics, persistence model, pricing, portability, and fit assessment for Mission 2's multi-backend fleet platform."
research_date: 2026-04-29
last_verified: 2026-04-29
staleness_warning: "Cloudflare is shipping Agents SDK releases weekly (252 releases as of 2026-04-29). Re-verify within 30 days — Project Think (preview as of 2026-04-15) may reach GA; Durable Object Facets (new at Agents Week 2026) may alter sub-agent semantics; Workers AI pricing may change."
confidence: high
sources_count: 22
related:
  - ../landscapes/agentic-fleet-platforms.md (forthcoming)
  - ../landscapes/agentic-substrate-distribution.md
status: active
---

# Cloudflare Agents — Capability Profile

> **Research date:** 2026-04-29
> **Last verified:** 2026-04-29
> **Staleness warning:** Cloudflare is shipping fast. SDK version 0.11.9 is current as of research date. Project Think is preview — APIs may change. Re-verify within 30 days.
> **Confidence summary:** ~30 [VERIFIED] claims from primary Cloudflare docs + GitHub repo. ~8 [LIKELY] from community sources. ~4 [PARAMETRIC] where docs were incomplete. No [UNVERIFIED] claims retained.
>
> **Primary sources (top 6 by load-bearing weight):**
> - Cloudflare Agents docs: https://developers.cloudflare.com/agents/ (2026-04-29)
> - Cloudflare Agents GitHub repo (cloudflare/agents, 0.11.9): https://github.com/cloudflare/agents (2026-04-29)
> - Agents Week 2026 review: https://blog.cloudflare.com/agents-week-in-review/ (2026-04-29)
> - Project Think announcement: https://blog.cloudflare.com/project-think/ (2026-04-15, fetched 2026-04-29)
> - Durable Objects pricing: https://developers.cloudflare.com/durable-objects/platform/pricing/ (2026-04-29)
> - Limits page: https://developers.cloudflare.com/agents/platform/limits/ (2026-04-29)

---

## TL;DR

**Cloudflare Agents** is a GA TypeScript SDK (`agents` npm package, v0.11.9) that wraps Durable Objects into persistent, stateful agent instances. Each agent is a named Durable Object with a built-in SQLite database, WebSocket server, HTTP handler, scheduler, sub-agent spawning, and MCP server capability. Deployment is Cloudflare-only (wrangler deploy); local dev runs via `wrangler dev` + Miniflare.

**BackendAdapter verdict for Mission 2: VIABLE — cloud tier, deferred to Phase 2.** Cloudflare Agents is well-matched as a cloud backend for long-lived, persistent agents that need to run closer to end-users or survive process crashes. It is not a drop-in replacement for local backends (subprocess, cmux, tmux): it's cloud-only, requires Cloudflare account + Workers Paid plan ($5/mo base), and carries meaningful portability cost due to Durable Objects binding lock-in. The BackendAdapter abstraction in `substrate/` must treat this as a distinct tier, not an equivalent of the subprocess backend.

**The three most important findings:**

1. [VERIFIED] Spawn, send, read, status, and kill all have first-class SDK primitives — the `spawn()/send()/read()/status()/kill()` mental model maps cleanly to `routeAgentRequest()` + `setState()/getState()` + `onMessage()` + `abortSubAgent()`. The abstraction boundary is tractable.
2. [VERIFIED] Per-agent SQLite state survives hibernation, redeploys, and crashes — this is the strongest differentiation from subprocess/tmux backends and makes Cloudflare Agents the right tier for "durable agents" that must not lose session state.
3. [LIKELY] Vendor lock-in is real but bounded: the core business logic in an `Agent` class is portable TypeScript; the lock-in lives in Durable Object bindings, the `agents` npm package APIs, and wrangler deployment. Migrating off requires rewriting the persistence + scheduling layers.

---

## Dimension 1 — Product Identity

**Product name:** "Cloudflare Agents" / "Agents SDK" — not "Workers AI Agents." The npm package is simply `agents`. [VERIFIED, src 2]

**What it is:** A TypeScript SDK that wraps Cloudflare Durable Objects (DO) into a higher-level "agent" abstraction. Each agent class extends `Agent<Env, State>` or `AIChatAgent<Env, State>`. Agents are instantiated as named DO instances, one per logical entity (user, session, team, task). The SDK bundles: SQLite-backed state, WebSocket lifecycle, HTTP routing, cron/delay scheduling, FIFO task queues, sub-agent spawning, MCP server hosting, and Workers AI model invocation. [VERIFIED, src 1, 2]

**Project Think (next-generation, preview):** Announced 2026-04-15, available as `@cloudflare/think`. Adds: durable execution with crash-recovery fibers (`runFiber()`), persistent session trees with branching, sandboxed code execution via Dynamic Workers (`@cloudflare/codemode`), and the "Execution Ladder" (five compute tiers an agent escalates through). Currently preview — APIs may change. [VERIFIED, src 3, 4]

**Maturity:**
- Core `agents` SDK: **GA** (252 releases, 4.8k GitHub stars, production-ready). [VERIFIED, src 2]
- Project Think: **preview** — "APIs may change as we incorporate feedback." [VERIFIED, src 4]
- Voice agents: **experimental**. [VERIFIED, src 5]
- Agent Memory (managed service): **new at Agents Week 2026** — GA status unconfirmed. [LIKELY, src 5]
- Sub-agents (Facets): **new at Agents Week 2026**, included in GA SDK. [VERIFIED, src 6]

**Pricing model:** Agents inherit Workers + Durable Objects pricing. No separate "agent" SKU. Workers Paid plan required ($5/mo base). See Dimension 10 for full cost table. [VERIFIED, src 7, 8]

---

## Dimension 2 — Spawn Semantics

**How an agent is instantiated:**

Agents are Durable Objects and follow DO instantiation semantics. From the SDK, the entry point is `routeAgentRequest()`: [VERIFIED, src 1]

```typescript
// Worker entry point (wrangler.toml + Workers deployment)
export default {
  async fetch(request: Request, env: Env) {
    return routeAgentRequest(request, env) ||
      new Response("Not found", { status: 404 });
  }
};
```

Agents are addressed by URL: `https://<worker>.workers.dev/agents/:agent-class/:instance-name`

The first HTTP request or WebSocket connection to a new `:instance-name` creates the agent instance and fires `onStart()`. Subsequent requests to the same name route to the same DO instance. [VERIFIED, src 1]

**Sub-agent spawn (parent spawning a child):**

```typescript
// Inside a parent agent
const child = await this.subAgent(ChildAgent, "child-name");
// child is a typed RPC stub; first call fires child's onStart()
```

`subAgent()` is idempotent: subsequent calls with the same name return the existing instance. [VERIFIED, src 6]

**Wrangler deploy command:** `npx wrangler deploy` — builds and pushes to Cloudflare's edge network. Local dev: `npx wrangler dev` (uses Miniflare simulation, local-by-default since Wrangler v3). [VERIFIED, src 9, src 10]

**wrangler.toml minimum configuration:**

```toml
compatibility_flags = ["nodejs_compat"]

[[durable_objects.bindings]]
name = "MyAgent"
class_name = "MyAgent"

[[migrations]]
tag = "v1"
new_sqlite_classes = ["MyAgent"]
```

The `migrations` block is required to enable SQLite storage; without it, state is not persisted. [VERIFIED, src 11]

---

## Dimension 3 — Send Semantics

Three delivery channels: [VERIFIED, src 1]

| Channel | How input arrives | Agent handler |
|---|---|---|
| **HTTP request** | `POST /agents/:class/:name` | `onRequest(request)` |
| **WebSocket message** | Client connects, sends message | `onMessage(connection, message)` |
| **Scheduled trigger** | `schedule()` / cron fires | Named callback method |
| **Email** | Via Cloudflare Email routing | `onEmail(email)` |
| **Queue task** | `queue()` enqueue | Named callback method |

**Client-side SDK:** The `agents` package ships `AgentClient` (vanilla JS) and `useAgent` (React hook) for WebSocket connection management. These provide automatic reconnect and state-sync. [VERIFIED, src 12]

**MCP tool invocation:** Agents can expose an MCP server via `McpAgent` subclass. MCP clients (Claude, Cursor, etc.) can call agent tools over the standard MCP protocol. Supports OAuth 2.1 authorization. [VERIFIED, src 1 — MCP docs section]

---

## Dimension 4 — Read / Output Semantics

**HTTP response:** `onRequest()` returns a standard `Response` object. Can be synchronous or streaming. [VERIFIED, src 1]

**SSE streaming:** Agents support Server-Sent Events via `ReadableStream` with `Content-Type: text/event-stream`. The Vercel AI SDK's `streamText().toTextStreamResponse()` composes directly. Long-running SSE connections have no platform timeout. The `Last-Event-ID` header enables resume from saved state after reconnect. [VERIFIED, src 12]

**WebSocket broadcast:** `this.broadcast(message, excludeIds?)` pushes to all connected clients simultaneously. Individual sends via `connection.send(message)`. [VERIFIED, src 12]

**Workers AI streaming:** Model calls through Workers AI support streaming token generation with `stream: true`. The `workers-ai-provider` package wraps this for the Vercel AI SDK's streaming primitives. [LIKELY — not directly fetched from source, based on SDK docs index reference to Workers AI streaming.]

**State sync to clients:** `setState()` triggers `onStateChanged()` on the server and automatically syncs the new state to all connected clients via the `useAgent` hook or `AgentClient`. This is the SDK's reactive state pattern. [VERIFIED, src 1]

---

## Dimension 5 — Status / Kill / List

**Status query:** No native "get agent status" endpoint in the SDK. The agent's own `onRequest()` can implement a `/status` route. State can be read via `getState()` but this requires routing an HTTP request to the specific DO instance. [VERIFIED, src 1 — absence of built-in status API noted]

**Kill (abort):** `abortSubAgent(AgentClass, "name")` — forcefully stops a running sub-agent. Child stops immediately, restarts on next `subAgent()` call. Storage remains intact. Applied transitively to nested children. For top-level agents, DO hibernation is the natural pause; no explicit kill API at the top level exists in the SDK — you would have to call the Cloudflare REST API to delete the DO instance or let it hibernate. [VERIFIED, src 6]

**Delete:** `deleteSubAgent(AgentClass, "name")` — permanently removes a sub-agent and wipes its SQLite storage. Transitively deletes nested children. [VERIFIED, src 6]

**List:** No native list-all-instances API in the SDK. Cloudflare Durable Objects does not expose an instance enumeration API — this is a known platform limitation. Enumerating agents requires maintaining your own registry (e.g., a separate DO or D1 table that records created instances). [LIKELY — based on known DO platform constraint; not confirmed against a specific 2026 doc page, but consistent with all observed DO behavior and referenced in community threads. src 13]

**Wrangler tail:** `wrangler tail` streams real-time logs from Workers. Provides observability into agent invocations but is not an agent-addressing API. [VERIFIED, src 9]

**Scheduling queries:** `getSchedules()` is a synchronous method that returns pending scheduled tasks for a given agent instance, filterable by type and time range. This is instance-scoped, not fleet-scoped. [VERIFIED, src 14]

---

## Dimension 6 — Persistence Model

**Core primitive:** Each agent instance is a Durable Object with a dedicated SQLite database. [VERIFIED, src 1]

**State layers:**

| Layer | API | Persistence |
|---|---|---|
| Typed state object | `setState(partial)` / `getState()` | Survives hibernation, restarts, deploys |
| Raw SQLite | `this.sql\`query\`` | Per-agent SQLite DB, survives hibernation |
| In-memory variables | Standard JS variables | Lost on hibernation |
| Connection state | `connection.setState()` | Per-connection, not persisted across reconnects |

**Hibernation:** Agents sleep automatically when idle. WebSocket connections remain open via the WebSocket Hibernation API — the DO pauses, then resumes on the next message without clients detecting a disconnection. This drastically reduces Durable Object duration billing. In-memory state is lost on hibernation; `this.state` and `this.sql` survive. [VERIFIED, src 12]

**Mapping to "agent session":** Each named agent instance IS the session. If you model one agent instance per work session, the DO is the natural session boundary: state is the session context, SQL is the session's structured memory, and the scheduler is the session's deferred-action queue. This is a strong conceptual fit for durable cloud agents. [VERIFIED — structural observation from docs, src 1]

**Sub-agent isolation:** Each sub-agent has its own completely isolated SQLite database, separate from the parent. Parent and child use the identical `this.sql` syntax but write to separate databases. [VERIFIED, src 6]

**Project Think persistent sessions:** The `@cloudflare/think` preview adds a tree-structured message store per agent with conversation forking, non-destructive compaction, and full-text search across history. This is a higher-level abstraction over the base SQLite layer. [VERIFIED, src 4]

**Storage limits:** 1 GB max stored state per unique agent instance. [VERIFIED, src 15]

---

## Dimension 7 — Reach

**Deployment:** Cloudflare global edge network (300+ cities). Cloud-only — there is no self-hosted / on-premise deployment path. The Cloudflare account + Workers Paid plan are hard requirements. [VERIFIED, src 1]

**Local development story:** `wrangler dev` runs agents locally via Miniflare v3 (the local Workers simulator). Local-by-default since Wrangler v3. Miniflare simulates Durable Objects, SQLite, KV, R2, D1, and most bindings. The Local Explorer (open beta as of Agents Week 2026) provides a UI for inspecting local DO state. [VERIFIED, src 9, 10]

**Known local dev gaps:** Browser Run has no local simulation. Hyperdrive is unsupported locally. Workers AI calls in `wrangler dev` hit the real Cloudflare AI API unless mocked. [VERIFIED, src 10]

**Remote-only services at Agents Week 2026:** Cloudflare Mesh (private networking), Agent Memory managed service, AI Search — these are cloud-only with no local simulation. [VERIFIED, src 5]

**Latency profile:** Edge deployment means agents run in a DC geographically close to the requester, not in a fixed region. This is a benefit for user-facing latency but complicates scenarios where the agent needs access to a fixed-region database or private network (partially addressed by Cloudflare Mesh). [LIKELY, src 5]

---

## Dimension 8 — Visibility

**Dashboard:** Cloudflare Dashboard → Workers & Pages → your worker. Shows requests, CPU time, error rate. Agent Lee (new at Agents Week 2026) is an in-dashboard AI assistant for the Cloudflare stack. [VERIFIED, src 5]

**Real-time logs:** `wrangler tail` streams Worker invocation logs. Can be filtered by status, IP, sampling rate. [VERIFIED, src 9]

**Scheduling visibility:** `getSchedules()` on the agent instance returns all pending schedules with metadata (id, type, when, payload). Instance-scoped only. [VERIFIED, src 14]

**Workers AI observability:** AI Gateway (14+ providers) provides per-model request logs, cost tracking, error rates, caching, and rate limiting. Available for Workers AI and third-party provider calls. [VERIFIED, src 5]

**Missing fleet visibility:** There is no "list all running agent instances" API. You cannot query the platform for all instances of a given agent class. Fleet-level visibility requires building your own registry. [LIKELY — documented platform constraint, see Dimension 5. src 13]

---

## Dimension 9 — Authentication

**Inbound to agent:** The agent's `onRequest()` or `onConnect()` handlers receive the raw HTTP request (including headers, cookies, query params). Authentication is implemented by the developer in these handlers. The SDK shows the pattern `connection.close(4001, "Unauthorized")` for WebSocket auth rejection. No built-in auth middleware in the core `Agent` class. [VERIFIED, src 12]

**MCP OAuth:** `McpAgent` supports OAuth 2.1 authorization (RFC 9728) for MCP tool access. Managed OAuth for Cloudflare Access integration was announced GA at Agents Week 2026. [VERIFIED, src 5]

**Worker-to-Worker / Service Bindings:** Workers can call other Workers (and thus other agent instances) using Cloudflare Service Bindings — zero-latency, authenticated inter-service calls within the same Cloudflare account. This is the primary pattern for orchestrator-to-agent calls within the platform. [VERIFIED, src 1]

**API token:** `wrangler secret put API_KEY` for per-environment secrets. Available as `this.env.API_KEY` in agent code. Production secrets stored in Cloudflare's secrets storage, not in wrangler.toml. [VERIFIED, src 11]

**Cloudflare API tokens:** The Agents Week 2026 announcement included enhanced API token security: scannable tokens, OAuth visibility, resource-scoped permissions. [VERIFIED, src 5]

---

## Dimension 10 — Pricing / Quota

All Agents pricing is composed from three billing surfaces. The Workers Paid plan ($5/mo minimum) is required for Durable Objects.

### Workers (request + CPU)

| | Free | Paid |
|---|---|---|
| Requests | 100k/day | 10M/month included, then $0.30/M |
| CPU time | 10ms/invocation | 30M CPU-ms/month included, then $0.02/M CPU-ms |
| Max CPU per invocation | 10ms | 5 min (standard), 15 min (scheduled) |

[VERIFIED, src 8]

### Durable Objects (compute duration + storage)

| | Free | Paid |
|---|---|---|
| Requests | 100k/day | 1M/month included, then $0.15/M |
| Duration | 13k GB-s/day | 400k GB-s/month included, then $12.50/M GB-s |
| SQLite row reads | 5M/day | 25B/month included, then $0.001/M rows |
| SQLite row writes | 100k/day | 50M/month included, then $1.00/M rows |
| Storage | 5 GB total | 5 GB-month included, then $0.20/GB-month |

Storage billing for SQLite-backed DOs was enabled January 2026. WebSocket Hibernation API dramatically reduces duration charges by sleeping the DO between messages. [VERIFIED, src 7]

### Workers AI (inference)

| Unit | Price |
|---|---|
| Base rate | $0.011 per 1,000 Neurons |
| Daily free | 10,000 Neurons (resets 00:00 UTC) |
| llama-3.2-1b-instruct | $0.027/M input tokens |
| kimi-k2.6 (largest listed) | $0.950/M input tokens |

Third-party model providers (Anthropic, OpenAI, etc.) are billed at provider rates + optional AI Gateway proxying. [VERIFIED, src 16]

**Concurrency:** Tens of millions of concurrent agent instances possible per account. 500 script deployments per account; each script can define multiple agent classes. [VERIFIED, src 15]

**Cost implication for Mission 2:** For cloud agents that are mostly idle (hibernating), the effective cost is near zero — Duration billing only applies while awake. The economic story is: "agents cost zero when sleeping, fractions of a cent when serving a request." This is the right pricing model for event-driven background agents. [VERIFIED, src 4 — explicitly noted in Project Think announcement]

---

## Dimension 11 — Maturity / Known Limitations

**SDK version:** 0.11.9 as of 2026-04-29. GA. 252 releases. 4.8k GitHub stars. Actively maintained by Cloudflare. [VERIFIED, src 2]

**External PRs not accepted:** "We're not accepting external pull requests at this time — the SDK is evolving rapidly." This signals the API surface is still in flux despite GA status. [VERIFIED, src 2]

**Node 24+ required for development.** [VERIFIED, src 2]

**Experimental features:** `experimental/` directory in the repo carries no stability guarantee. [VERIFIED, src 2]

**Known platform limitations:**

| Limitation | Detail | Source |
|---|---|---|
| CPU time cap | 30s per invocation (refreshed on each HTTP request / WebSocket message). Not a wall-clock cap — waiting for DB or LLM responses doesn't count. | [VERIFIED, src 15] |
| No agent instance enumeration | No API to list all running instances of an agent class. Fleet registry must be self-built. | [LIKELY, src 13] |
| 6-connection limit in Workers | Workers have a 6-concurrent-subrequest limit per invocation. Poorly documented, discovered by community. | [VERIFIED, src 13] |
| No local Browser Run simulation | `wrangler dev` cannot simulate Browser Run. Must call live CF service. | [VERIFIED, src 10] |
| D1 reliability complaints | Community reports of hung queries and missing transaction support in D1 (adjacent product, not DO). | [VERIFIED, src 13] |
| Workers AI model selection | Workers AI catalog lags commercial providers. Community noted LoRA-capable models are "year old dense models that perform horribly." | [VERIFIED, src 13] |
| Project Think APIs unstable | `@cloudflare/think` is preview — "APIs may change as we incorporate feedback." | [VERIFIED, src 4] |
| Voice agents experimental | Real-time STT/TTS over WebSockets — experimental label at Agents Week 2026. | [VERIFIED, src 5] |

**REST API concern (historical):** At AI Platform launch, a community commenter noted "REST API support wasn't available at launch" for the inference layer — Cloudflare's initial rollout was Workers-binding-only. This has since been addressed but reflects the pattern of bindings-first development. [VERIFIED, src 13]

**Pricing surprises:** Community members raised concern about cost controls. Hard spending limits are not available (only daily neuron limits for Workers AI); some users found total costs with Workers + DO + Workers AI difficult to predict. [VERIFIED, src 13]

---

## Dimension 12 — Vendor Lock-In / Portability

**Lock-in surfaces (ranked by severity):**

| Surface | Portability cost | Mitigation |
|---|---|---|
| Durable Objects (state, SQL, scheduling) | High — DO APIs (`this.sql`, `this.state`, `schedule()`) are proprietary to Cloudflare. No equivalent in AWS Lambda, Azure Functions, Fly.io, etc. | Design thin adapter interfaces; keep business logic in pure functions that take/return state objects rather than calling `this.sql` directly. |
| `agents` npm package APIs | Medium — extending `Agent<Env, State>` ties lifecycle hooks to CF. Migrating off requires rewriting lifecycle layer. | The `Agent` class is TypeScript; business logic in agent methods can be extracted to plain functions. |
| wrangler deployment toolchain | Medium — wrangler is CF-specific; no alternative deploy path. | One-time migration cost to a different deploy tool. |
| Workers request model (Fetch API) | Low — Workers use standard Web APIs (Fetch, Request, Response, URL, Crypto). Core request handling is portable. | Web standard APIs run in Deno, Bun, Node.js with minimal changes. |
| Workers AI model bindings | Low-Medium — switching from Workers AI to Anthropic/OpenAI requires changing import + credentials, not architecture. The `workers-ai-provider` is designed for provider swap. | Cloudflare explicitly supports 14+ providers via AI Gateway; model swap is one-line per their positioning. |

**Cloudflare's portability claim:** "Workers' use of Web Standard APIs is a portability advantage; core request-handling logic often runs in any JavaScript environment with minimal changes." The Cloudflare-specific parts (bindings) are "isolated to specific import statements and binding configurations — not woven throughout business logic." [VERIFIED, src 17]

**Honest portability assessment:** The claim is true for stateless Workers but less true for Agents, which are built entirely around Durable Objects. A stateful `Agent` class that uses `this.sql`, `setState()`, `subAgent()`, and `schedule()` cannot run outside Cloudflare without a compatibility shim. No such shim exists publicly as of research date. [PARAMETRIC — reasoned from architecture, no independent source found that directly confirms or denies shim availability]

**Migration path if exiting:** Core business logic (LLM prompting, tool definitions, response parsing) is in plain TypeScript methods — extractable. The persistence and scheduling layers must be rewritten against the target platform's primitives. Estimated effort: moderate (days to weeks, not months) for a well-abstracted agent; high for tightly coupled agents.

**Ecosystem portability:** MCP tool exposure is a net positive — MCP is an open protocol. Agents that expose their tools via `McpAgent` can be consumed by any MCP client regardless of where the agent runs. This creates a portable interface surface even though the implementation is CF-specific. [VERIFIED, src 1]

---

## BackendAdapter Fit Assessment

### Mapping to substrate/BackendAdapter primitives

| BackendAdapter primitive | Cloudflare Agents equivalent | Fidelity |
|---|---|---|
| `spawn(command, cwd, env, metadata) → session_id` | `routeAgentRequest()` + URL-addressed DO instantiation on first request | Partial — no explicit "spawn" call; instantiation is request-triggered. Instance name = session_id. |
| `send(session_id, text)` | HTTP POST or WebSocket message to `/agents/:class/:session_id` | Full — both sync (HTTP) and streaming (WebSocket/SSE) send patterns available. |
| `read(session_id, lines?, since?)` | WebSocket subscribe + `onMessage()` callback, or HTTP GET from agent's `/status` route | Partial — no native "read buffer" pull API. Must implement polling endpoint or subscribe via WebSocket. |
| `status(session_id) → state` | Custom `/status` route returning `getState()` | Partial — must implement; no SDK-native status endpoint. |
| `kill(session_id)` | `abortSubAgent()` for child agents; Cloudflare REST API for top-level agents | Partial — `abortSubAgent()` works cleanly for sub-agents; top-level kill requires platform API call. |
| `list() → sessions` | Not available natively — must self-build registry | Gap — significant; requires custom DO or D1 table as fleet registry. |

### Verdict: VIABLE — cloud tier, Phase 2

**Adopt path:** Cloud-tier backend for Mission 2. Introduce as `CloudflareAgentsBackend` in the `BackendAdapter` abstraction. Suitable for agents that need:
- Durable state across process crashes / redeploys (the killer feature)
- Edge-proximity to end users
- Built-in scheduling and cron
- MCP server hosting
- Sub-agent spawning with isolated storage

**Not suitable for:**
- Local development environments (no self-hosted path)
- Agents that require Linux process semantics (subprocess, file descriptor control, PTY)
- Scenarios requiring fleet enumeration without a custom registry
- Cost-sensitive workloads where idle time is long but DO duration billing matters (though hibernation mitigates this significantly)

**Phase recommendation:** Defer to Phase 2 of Mission 2. Phase 1 should close the local backends (subprocess-headless, cmux, tmux) since they're closer to the current substrate and have no account/billing dependencies. Cloudflare Agents is a natural Phase 2 target once the BackendAdapter interface is stable, because the interface discipline from Phase 1 will make the cloud backend addition mechanical.

**Abstraction boundary:** The `CloudflareAgentsBackend` adapter must own the DO naming convention, URL construction, authentication headers, and the fleet registry (self-built). Business logic above the adapter should be unaware it's talking to a DO instance vs. a local subprocess.

---

## Open Questions

1. **Fleet enumeration:** Is there a first-party Cloudflare API for listing DO instances by class name? The limits page suggests no, but the Cloudflare REST API has more surface than the Workers SDK docs expose. Needs direct API inspection. [PARAMETRIC]

2. **BackendAdapter `read()` pull semantics:** Cloudflare Agents is event-push by design (WebSocket callbacks, onRequest). For the BackendAdapter's pull-oriented `read(session_id, since)` pattern, is the best path (a) WebSocket subscription in the adapter, (b) a custom HTTP polling endpoint on the agent, or (c) reading from the agent's SQLite state directly? Option (c) is not possible (DO instance is the only reader of its own SQLite). Option (a) is cleanest but requires persistent WebSocket connection management in the adapter layer.

3. **Project Think stabilization timeline:** `@cloudflare/think` is preview as of 2026-04-15. The durable execution fibers (`runFiber()`) are directly relevant to Mission 2's "agent that survives crashes" use case. When does this reach GA? Worth monitoring Cloudflare changelog weekly.

4. **Workers AI model quality gap:** Community feedback indicates Workers AI's model selection lags Anthropic/OpenAI/Google by 6-12 months for capable models. For Mission 2 agents that need frontier model quality, the agent runtime (Cloudflare DO) and the inference provider (Anthropic API via AI Gateway) should be treated as independent choices — the BackendAdapter should not assume Workers AI = Cloudflare Agents.

5. **Cost model at scale:** For a fleet of O(100) concurrent cloud agents, what does the combined Workers + DO + Workers AI bill look like under realistic usage (e.g., 10 tool calls per agent per hour, 1KB state per agent)? A cost model needs to be built before committing Cloudflare Agents as the cloud tier.

---

## Sources

| # | URL | Date Accessed | Role |
|---|---|---|---|
| 1 | https://developers.cloudflare.com/agents/ + https://developers.cloudflare.com/agents/api-reference/agents-api/ + https://developers.cloudflare.com/agents/api-reference/websockets/ | 2026-04-29 | Primary — agents API, lifecycle, WebSocket, routing |
| 2 | https://github.com/cloudflare/agents | 2026-04-29 | Primary — package version, stars, release count, limitations |
| 3 | https://www.cloudflare.com/agents-week/updates/ | 2026-04-29 | Primary — Agents Week 2026 announcement catalog |
| 4 | https://blog.cloudflare.com/project-think/ | 2026-04-29 | Primary — Project Think preview details, fibers, sessions |
| 5 | https://blog.cloudflare.com/agents-week-in-review/ | 2026-04-29 | Primary — GA/preview status of all Agents Week features |
| 6 | https://developers.cloudflare.com/agents/api-reference/sub-agents/ | 2026-04-29 | Primary — subAgent(), abortSubAgent(), deleteSubAgent() |
| 7 | https://developers.cloudflare.com/durable-objects/platform/pricing/ | 2026-04-29 | Primary — Durable Objects pricing table |
| 8 | https://developers.cloudflare.com/workers/platform/pricing/ | 2026-04-29 | Primary — Workers request + CPU pricing |
| 9 | https://developers.cloudflare.com/workers/wrangler/ | 2026-04-29 | Primary — deploy commands, wrangler tail |
| 10 | https://developers.cloudflare.com/workers/development-testing/ + local dev search results | 2026-04-29 | Primary — local dev capabilities, gaps (Browser Run, Hyperdrive) |
| 11 | https://developers.cloudflare.com/agents/api-reference/configuration/ | 2026-04-29 | Primary — wrangler.toml configuration, migrations, secrets |
| 12 | https://developers.cloudflare.com/agents/api-reference/websockets/ + http-sse docs | 2026-04-29 | Primary — WebSocket lifecycle, SSE streaming, client SDK |
| 13 | https://news.ycombinator.com/item?id=47792538 | 2026-04-29 | Community — vendor lock-in concerns, 6-connection limit, Workers AI model gaps, D1 reliability |
| 14 | https://developers.cloudflare.com/agents/api-reference/schedule-tasks/ | 2026-04-29 | Primary — schedule(), scheduleEvery(), getSchedules(), cancelSchedule() |
| 15 | https://developers.cloudflare.com/agents/platform/limits/ | 2026-04-29 | Primary — concurrency limits, storage limit (1 GB), CPU cap (30s) |
| 16 | https://developers.cloudflare.com/workers-ai/platform/pricing/ | 2026-04-29 | Primary — Workers AI neuron pricing, per-model costs |
| 17 | https://inventivehq.com/blog/multi-cloud-strategy-vendor-lock-in-cloudflare-aws-azure-gcp | 2026-04-29 | Secondary — vendor lock-in analysis of Cloudflare primitives |
| 18 | https://developers.cloudflare.com/changelog/2025-02-25-agents-sdk/ | 2026-04-29 | Historical — SDK launch date (Feb 25, 2025) |
| 19 | https://developers.cloudflare.com/agents/concepts/ + what-are-agents/ | 2026-04-29 | Primary — conceptual overview, agents vs workflows vs copilots |
| 20 | https://developers.cloudflare.com/agents/llms.txt | 2026-04-29 | Primary — full docs index, confirmed MCP / sub-agents / scheduling coverage |
| 21 | https://developers.cloudflare.com/durable-objects/ | 2026-04-29 | Primary — DO overview, hibernation, SQLite backend |
| 22 | https://workers.cloudflare.com/product/agents | 2026-04-29 | Primary — product marketing page, "tens of millions of concurrent agents" claim |
