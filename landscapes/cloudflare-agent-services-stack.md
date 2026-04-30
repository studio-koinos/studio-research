---
title: "Cloudflare Agent Services Stack — Vendor Landscape"
description: "Full mapping of Cloudflare's agent-related services as composable primitives for Mission 2's fleet substrate. Covers 19 services across compute, storage, inference, observability, and security."
research_date: 2026-04-29
last_verified: 2026-04-29
staleness_warning: "Cloudflare ships at high velocity. Agents Week 2026 (Apr 13–17) landed ~20 new products or GA promotions. Re-verify monthly. Agent Memory (private beta), AI Search (open beta), and Durable Object Facets are all pre-GA as of research date."
confidence: high
sources_count: 31
related:
  - ../evaluations/cloudflare-agents.md
  - ./agentic-substrate-distribution.md
status: active
---

# Cloudflare Agent Services Stack — Vendor Landscape

> **Research date:** 2026-04-29
> **Last verified:** 2026-04-29
> **Staleness warning:** Re-verify after 30 days. Cloudflare ran Agents Week 2026 (Apr 13–17) and landed ~20 products — several are still in beta (Agent Memory, AI Search, Durable Object Facets). The Unified Inference Layer is accumulating providers monthly.
> **Confidence summary:** ~26 VERIFIED claims (primary Cloudflare docs + official blog posts), ~8 LIKELY (inferred from architecture), 1 PARAMETRIC (Constellation status). Claims are tagged inline.
>
> **Primary load-bearing sources:**
> - Cloudflare Agents Week in Review: https://blog.cloudflare.com/agents-week-in-review/ (2026-04-17)
> - Containers + Sandboxes GA: https://developers.cloudflare.com/changelog/post/2026-04-13-containers-sandbox-ga/ (2026-04-13)
> - Agent Memory private beta: https://blog.cloudflare.com/introducing-agent-memory/ (2026-04-17)
> - Cloudflare AI Platform (Unified Inference Layer): https://blog.cloudflare.com/ai-platform/ (2026-04-15)
> - Workers AI large models (Kimi K2.5/K2.6): https://blog.cloudflare.com/workers-ai-large-models/ (2026-04-16)
> - AI Search (AutoRAG rebrand): https://blog.cloudflare.com/ai-search-agent-primitive/ (2026-04-16)
> - Cloudflare Mesh: https://blog.cloudflare.com/mesh/ (2026-04-14)
> - Workflows v2: https://blog.cloudflare.com/workflows-v2/ (2026-04-15)

---

## TL;DR

Cloudflare has pivoted its entire developer platform toward agents. Agents Week 2026 (Apr 13–17) shipped ~20 products in five days — Sandboxes GA, Workflows v2 (50k concurrency), Agent Memory private beta, AI Search open beta, Cloudflare Mesh, Durable Object Facets, and a Unified Inference Layer covering 14+ providers. The platform is now the most coherent all-in-one edge-agent stack available.

**Top 5 primitives by Mission-2 fleet-substrate fit:**

1. **AI Gateway** — the single most important primitive for fleet observability: unified request logging, caching, fallback routing, rate limiting, and DLP across ALL inference providers (Workers AI, Anthropic, OpenAI, 12+ others). Zero-egress-cost proxy sitting between your agent fleet and every model API. High Mission-2 fit.

2. **Durable Objects** — the backbone of stateful agent coordination. Each agent session gets a Durable Object: strongly-consistent SQLite storage, WebSocket hibernation, per-session key space. Backs the Agents SDK, MCP server sessions, and Workflows state. High Mission-2 fit.

3. **Workflows v2** — durable step-function execution for long-running agent orchestration. 50k concurrent instances, 300 creations/s, independently retryable steps, human-in-the-loop pause/resume, up to weeks-long execution. The right primitive for multi-agent pipelines and fleet dispatch loops. High Mission-2 fit.

4. **Workers** — the entry point for everything. Agent dispatch, webhook ingestion, MCP hosting, Agents SDK runtime. No infrastructure ops. Billable by CPU time (not wall-clock), so long-waiting agents are cheap. High Mission-2 fit.

5. **Sandboxes** — isolated, persistent code-execution environments for agents that need to run arbitrary code (coding agents, eval harnesses, tool execution). GA as of April 13, 2026. Persistent PTY terminals, backup/restore snapshots, zero-trust egress with credential injection. High Mission-2 fit for fleet eval and tool-use agents.

**Recommended composition pattern:**
```
Workers (entry/dispatch)
  → AI Gateway (inference routing, observability, caching)
    → Workers AI / Anthropic / OpenAI (model calls)
  → Durable Objects (agent session state, MCP session backing)
  → Workflows v2 (long-running orchestration loops)
  → AI Search (per-agent RAG memory retrieval)
  → Agent Memory (cross-session context recall)
  → Sandboxes (tool execution, code interpretation)
  → Cloudflare Mesh (private network access to internal APIs/DBs)
```

---

## Service Stack Table

| Service | Maturity | Primary role in agent infra | Mission-2 fit | Notes |
|---|---|---|---|---|
| **Agents SDK** | GA (framework) | Agent runtime: Durable Objects + SQL + WebSocket + scheduling | High | Separate deep dive: `evaluations/cloudflare-agents.md` |
| **Workers** | GA | Entry point: dispatch, webhooks, MCP hosting, agent compute | High | CPU-billed; no wall-clock limit on paid plan |
| **Durable Objects** | GA (SQLite GA) | Stateful session coordination, per-agent storage, MCP sessions | High | Foundation of Agents SDK |
| **Workflows v2** | GA | Durable orchestration: multi-step, retries, human-in-the-loop | High | 50k concurrency, up to weeks duration |
| **AI Gateway** | GA | Inference routing, observability, caching, fallback, DLP | High | 14+ providers; unified logging; free core |
| **Sandboxes** | GA (2026-04-13) | Isolated code execution for coding agents, tool use, eval | High | PTY, backup/restore, zero-trust egress |
| **Workers AI** | GA | Edge inference: 97 models across LLM/embed/audio/vision | Medium-High | $0.011/1k Neurons; Kimi K2.6 1T params |
| **Agent Memory** | Private beta | Managed persistent memory: fact/event/instruction/task recall | High | Backed by Vectorize + full-text; waitlist |
| **AI Search** | Open beta | Managed RAG: hybrid retrieval, per-agent namespaces, indexing | High | Free during beta; formerly AutoRAG |
| **Vectorize** | GA | Vector database for embeddings, semantic search, agent memory | Medium | Backs AI Search and Agent Memory |
| **D1** | GA | Serverless SQLite: agent state, audit logs, structured records | Medium | 10GB/db, Time Travel 30-day recovery |
| **Queues** | GA | Async work distribution, agent task queuing, dead-letter | Medium | Batch + retry + delay; no egress charges |
| **KV** | GA | Eventually consistent config, feature flags, ephemeral state | Medium | Global low-latency reads; strong write consistency |
| **R2** | GA | Object storage: agent artifacts, session archives, model files | Medium | Zero egress; no GA versioning (use Artifacts on top) |
| **Containers** | GA (2026-04-13) | General-purpose container workloads, GPU, custom runtimes | Medium | Requires Workers Paid; active-CPU pricing |
| **Browser Run** | GA | Agent web automation: Puppeteer, Playwright, CDP, MCP | Medium | 4x concurrency increase in Agents Week |
| **Cloudflare Mesh** | Public beta | Private networking for agents: VPC-to-agent, zero-trust | Medium | Replaces manual tunnels for agent→internal-API |
| **Cloudflare Tunnel** | GA | Expose local dev to cloud (webhooks, MCP dev endpoints) | Low-Med | One-way outbound; free for basic use |
| **Pages** | GA | Static frontends; agent-built UIs; dashboard hosting | Low | Standard JAMstack; minimal agent-specific value |
| **Logpush** | GA (Enterprise) | Forward logs to SIEM/storage at enterprise scale | Low | Enterprise-only; no backfill; Workers Trace Events exception |
| **Analytics Engine** | GA | Custom time-series events from Workers code (agent telemetry) | Medium | Unlimited cardinality; SQL query API |
| **Artifacts** | Private beta | Git-protocol versioned storage for agent session repos | Medium | Covered in detail in `./agentic-substrate-distribution.md` |
| **Constellation** | Deprecated [PARAMETRIC] | Edge ML inference (superseded by Workers AI) | N/A | Docs redirect to Workers AI; treat as sunset |

---

## Composition Patterns

### Pattern 1 — Managed agent runtime (lowest ops overhead)

Use the **Agents SDK** directly. Under the hood: Worker (compute) + Durable Object (state + WebSocket) + Workers AI (inference) + AI Gateway (observability). One TypeScript class; Cloudflare owns the rest. Best for: standard chat agents, voice agents, MCP server hosting. [VERIFIED, src 1]

```
User → Worker (Agents SDK)
          ├── Durable Object (SQL state, WebSocket, scheduling)
          ├── AI Gateway → Workers AI / Anthropic / OpenAI
          └── AI Search (RAG)
```

### Pattern 2 — Durable orchestration (long-running fleet tasks)

**Workflows v2** as the outer harness; Workers as step executors; Durable Objects for per-task state. Best for: multi-agent pipelines, fleet dispatch loops, approval-gated tasks, anything that runs hours-to-days. [VERIFIED, src 5]

```
Trigger (HTTP / Queue / Cron)
  → Workflow (Workflows v2)
       ├── step.do("plan") → Worker → AI Gateway → model
       ├── step.do("execute") → Sandbox (code execution)
       ├── step.sleepUntil(human_approval)
       └── step.do("deliver") → R2 (artifact) / D1 (audit log)
```

### Pattern 3 — Fleet observability layer

**AI Gateway** as the universal proxy between all agents and all models. Every inference call logs request, response, tokens, cost, latency. Caching deduplicates repeated prompts. Rate limiting prevents runaway agents. DLP scans prompts/responses for sensitive data before they hit the model. One gateway URL; switch models with one config change. [VERIFIED, src 3, 19]

```
Agent (any language/runtime)
  → AI Gateway endpoint
       ├── Cache hit → return cached response (no model call)
       ├── Rate limit → 429 (agent handled)
       ├── DLP scan → block/redact if triggered
       ├── Provider routing → Workers AI | Anthropic | OpenAI | 12+ others
       └── Log → Analytics Engine / Logpush (Enterprise)
```

### Pattern 4 — Agent with private internal access (Mesh pattern)

**Cloudflare Mesh** + **Workers VPC** grants agent Workers scoped access to internal databases, private APIs, and corporate services — without public exposure or manual tunnel config. The agent's Durable Object session maintains the authenticated VPC connection. [VERIFIED, src 9]

```
External user → Worker (Agents SDK)
                    → Durable Object (session)
                         → Cloudflare Mesh (Workers VPC binding)
                              → Private DB / Internal API
```

### Pattern 5 — Per-agent memory (AI Search + Agent Memory composition)

Two complementary primitives: **AI Search** for document-based retrieval (RAG over files, docs, knowledge bases) and **Agent Memory** for conversation-derived context recall (facts, events, instructions extracted from past sessions). Both can be namespaced per-agent-instance. [VERIFIED, src 10, 11]

```
Agent conversation
  → Agent Memory (background extraction)
       ├── Vectorize (vector index for facts/events)
       └── Full-text index (instructions/tasks)
  → At next session: retrieve relevant memories → inject into context

File-based knowledge
  → AI Search (per-agent namespace via ai_search_namespaces)
       ├── R2 (document storage)
       └── Vectorize (hybrid retrieval)
```

### Pattern 6 — Coding agent / tool-use agent (Sandboxes)

**Sandboxes** provide isolated Linux environments for agents that must execute untrusted code. Each sandbox has a PTY terminal, filesystem, background processes, and live preview URLs. Backup/restore snapshots let expensive setup (installing deps, cloning repos) survive across agent calls. Zero-trust egress with credential injection means the sandbox can call internal APIs without exposing secrets. [VERIFIED, src 6]

```
Agent → Sandbox (isolated container)
            ├── PTY terminal (interactive shell)
            ├── Code interpreter (Python / JS / TS persistent kernel)
            ├── Live preview URL (agent verifies running service)
            └── Outbound Worker (egress policy + credential injection)
```

---

## Per-Service Summary

### 1. Agents SDK (framework)

The Agents SDK is Cloudflare's batteries-included TypeScript framework for building stateful agents on top of Durable Objects. Each agent is a class that extends `Agent`; the framework handles SQL state, WebSocket hibernation, task scheduling (delays, cron, specific times), streaming AI responses, human-in-the-loop approval flows, and MCP client/server interoperability. Voice pipeline (real-time STT/TTS), browser automation via Chrome DevTools Protocol, and multi-model routing (Workers AI, OpenAI, Anthropic, Gemini) are all first-class features.

**Project Think** (announced Agents Week 2026, currently preview) is the next-generation Agents SDK — "batteries-included" with expanded autonomous capabilities beyond the current base class model. [VERIFIED, src 1, 2]

**Mission-2 fit:** High. The framework directly models the BackendAdapter pattern from Mission 2's substrate vision — each agent backend can be a Durable Object, isolated per session, with state synced to connected clients in real time.

**Cross-reference:** See `evaluations/cloudflare-agents.md` for deep dive.

---

### 2. Workers

Cloudflare Workers is the serverless compute foundation. JavaScript/TypeScript/Python/Rust/WebAssembly; deploys in seconds to 330+ cities globally. The billing model is CPU-time-based (not wall-clock), which is economically favorable for I/O-heavy agents: a Worker waiting on an LLM response or a database query bills zero CPU time during the wait. Free tier: 100k requests/day, 10ms CPU. Paid: 10M requests/month + $0.30/M additional; 30M CPU-ms/month + $0.02/M additional. [VERIFIED, src 16, 22]

CPU limit on Standard plan: 5 minutes per invocation (extendable to 15 minutes for Cron/Queue consumers). For longer-running agent tasks, Workflows v2 or Durable Objects with hibernation are the correct abstractions.

**Mission-2 fit:** High. Workers is the universal entry point — every BackendAdapter in Mission 2's fleet substrate ultimately runs on Workers (or can). Webhook ingestion, agent dispatch, MCP server endpoints all live here.

---

### 3. Workers AI

Cloudflare's serverless GPU inference platform. [VERIFIED, src 13] 97 models across text generation, embeddings, image generation, vision, audio (STT/TTS), classification, translation, summarization. Runs on Cloudflare's global GPU network; no infrastructure to manage.

**Pricing:** $0.011 per 1,000 Neurons (GPU compute units). Free: 10,000 Neurons/day. Key LLM rates: $0.027–$0.950 per 1M input tokens depending on model. Kimi K2.5/K2.6 (1 trillion parameters, 256k context) is the frontier model; Cloudflare internally uses it at 77% lower cost than proprietary alternatives for a code-review agent processing 7B tokens/day. [VERIFIED, src 13, 14]

**Async inference API:** For high-throughput agent workloads, Workers AI offers an asynchronous inference path (durable, no capacity errors, typically executes within 5 minutes). This is the right API for batch agent tasks not on the user-facing critical path. [VERIFIED, src 14]

**Prefix caching:** Session affinity headers reduce costs on repeated prompts (cached tokens priced lower than input tokens). [LIKELY — mentioned in large-models blog but not in pricing docs directly]

**Mission-2 fit:** Medium-High. Useful when you want inference co-located with agent Workers (one-hop, no egress). Trade-off: model selection is narrower than frontier providers; Kimi K2.6 fills the frontier gap. Route frontier tasks through AI Gateway to Anthropic/OpenAI; use Workers AI for cost-sensitive background tasks and embeddings.

---

### 4. AI Gateway (Unified Inference Layer)

The single most important Cloudflare primitive for fleet observability and multi-model routing. AI Gateway is a proxy layer that sits between your agent code and any model provider. [VERIFIED, src 3, 19]

**Supported providers:** Workers AI, Anthropic, OpenAI, Google Gemini, Replicate, Alibaba Cloud, AssemblyAI, Bytedance, InWorld, MiniMax, Pixverse, Recraft, Runway, Vidu — 14+ providers, growing. One endpoint URL; switch providers by changing one parameter. [VERIFIED, src 19]

**Core features (free on all plans):**
- Request/response logging with token counts, latency, cost metadata
- Response caching (serve from Cloudflare cache, reduce model calls)
- Rate limiting (prevent runaway agents)
- Fallback routing (automatic failover if provider goes down)
- Custom metadata for cost attribution by agent, user, or workflow

**Premium features (paid):**
- Data Loss Prevention (DLP): scan prompts/responses for PII and sensitive data before they reach the model
- Guardrails: AI-model-evaluated safety checks on inputs/outputs (billed as Workers AI inference)
- Logpush: stream logs to external SIEM (Workers Paid: 10M/month + $0.05/M)

**Agent-specific design:** Streaming resilience (buffers responses independently of agent lifetime; agents can reconnect mid-stream); time-to-first-token optimization for interactive agents; chaining reliability (prevents cascading failures across multi-step sequences). [VERIFIED, src 19]

**Pricing:** Core gateway is free. 10M logs/gateway/month on Paid plan. DLP scanning is complimentary (2 profiles on Free; full suite with Zero Trust). [VERIFIED, src 20]

**Mission-2 fit:** High. The fleet needs a single observability chokepoint for all LLM calls across all agents and all providers. AI Gateway is exactly that. This is the primitive that enables cost attribution per agent session, multi-provider failover without agent-side logic, and centralized DLP enforcement.

---

### 5. Durable Objects

The stateful backbone of Cloudflare's agent infrastructure. Each Durable Object is a globally-unique, strongly-consistent micro-server with its own compute, SQLite database, and WebSocket connections. The consistency model is transactional, serializable, and strongly consistent — correct for agent session state where concurrent updates must not produce stale reads. [VERIFIED, src 4]

**SQLite integration (GA):** Each Durable Object has a built-in SQLite database accessible via a Storage API. This is how the Agents SDK persists conversation history, tool call logs, and agent state. Available on Free plan (with limits); standard pricing on Paid.

**WebSocket Hibernation API:** Durable Objects can maintain WebSocket connections across many clients efficiently — the object hibernates between messages (no CPU charge during idle), waking only when a message arrives. Critical for voice agents, streaming chat, and real-time agent monitoring. [VERIFIED, src 4]

**Durable Object Facets** (announced Agents Week 2026, pre-GA): allows Dynamic Workers to instantiate isolated SQLite databases per Durable Object instance. Enables per-tenant agent isolation at the storage layer without pre-provisioning. [VERIFIED, src 1]

**Pricing:** 1M requests/month included on Paid plan; $0.15/M additional. SQLite storage billed separately per-GB.

**Mission-2 fit:** High. Every BackendAdapter that needs per-session state — cmux, tmux session tracking, agent context persistence — maps naturally to a Durable Object per session. The Agents SDK uses this pattern directly; Mission 2 can adopt the same model.

---

### 6. Workflows v2

Cloudflare Workflows provides durable, multi-step execution for long-running agent orchestration — think AWS Step Functions but on the Workers platform with no separate infrastructure. [VERIFIED, src 5, 21]

**Execution model:** A Workflow is a series of `step.do()` calls. Each step is independently retryable; if a step fails, the workflow replays from that step without re-executing completed steps. Steps can sleep (`step.sleep()`, `step.sleepUntil()`), wait for external events (webhooks, human approval), and chain into arbitrarily complex agent loops.

**Workflows v2 (Agents Week 2026):** Complete control-plane rearchitecture. Old limits: 4,500 concurrent instances, 100 creations/10 seconds. New limits: 50,000 concurrent instances, 300 creations/second (100/s per workflow). Execution duration: minutes to weeks. [VERIFIED, src 5, 21]

**Agent orchestration:** Workflows is being used as the durable harness for agent loops that run autonomously for hours/days. Each workflow instance represents one agent task; the step model maps to agent think/act/observe cycles. Human-in-the-loop pause/resume is built-in via event waiting. [VERIFIED, src 21]

**Pricing:** Available on Free and Paid plans. Specific per-step pricing not prominently surfaced in docs (LIKELY billed via Durable Objects + Workers pricing underneath).

**Mission-2 fit:** High. Long-running fleet coordination tasks (cross-repo dispatch, timed maintenance sweeps, session-close workflows) map directly to Workflows. The 50k concurrency limit handles fleet-scale without throttling.

---

### 7. Sandboxes (formerly part of Containers)

Sandboxes are Cloudflare's isolated, persistent compute environments specifically designed for agent tool execution — running untrusted code, code interpretation, shell commands, and web services. [VERIFIED, src 6]

**GA date:** April 13, 2026. [VERIFIED, src 6]

**Key capabilities:**
- **Persistent code interpreters:** Python, JavaScript, TypeScript kernels maintain state across calls (no re-import overhead)
- **Interactive PTY terminals:** Multiple isolated shells per sandbox; browser-accessible terminal
- **Live preview URLs:** Agent can run a web service inside the sandbox and verify it via HTTP without external deployment
- **Backup/restore API:** Snapshot the workspace (installed deps, git state, files) and restore instantly — avoids expensive re-setup across agent calls
- **Outbound Workers (Egress policy):** Zero-trust outbound proxy with: credential injection (secrets never exposed to sandbox), TLS interception, per-instance allow/deny lists, dynamic egress policies. Agents can call internal APIs without the sandbox knowing the credentials. [VERIFIED, src 6]

**Relationship to Containers:** Containers is the general-purpose primitive (arbitrary container images, multi-vCPU, custom runtimes, GPU); Sandboxes is the agent-specific higher-level abstraction built on top of Containers. [LIKELY based on architecture docs]

**Mission-2 fit:** High. Any agent that executes code (eval harnesses, coding agents, tool-use agents) needs sandbox isolation. The backup/restore API is particularly valuable for fleet scenarios where many agent instances share setup overhead.

---

### 8. AI Search (formerly AutoRAG)

Cloudflare's managed RAG service — upload documents to R2, AI Search handles chunking, embedding (via Workers AI), indexing (via Vectorize), and retrieval. Rebranded from AutoRAG to AI Search in April 2026 with the addition of managed storage, vector index, and web crawling for instances created after April 16, 2026. [VERIFIED, src 10]

**Key agent feature:** `ai_search_namespaces` binding allows runtime instance creation — spin up a separate search index per agent, per customer, or per conversation without redeployment. Cross-instance queries are also supported (query all agent instances at once). [VERIFIED, src 10]

**Retrieval:** Hybrid (semantic + keyword) with relevance boosting and metadata filtering. Agents can accumulate context (e.g., customer resolution history) that becomes searchable across future sessions.

**Status:** Open beta. Free during beta (20,000 queries/month on Free; unlimited on Paid). At least 30 days' notice before paid pricing. Post-beta pricing consolidates component charges into unified AI Search pricing. [VERIFIED, src 10]

**Mission-2 fit:** High (pending GA). Per-agent namespaced knowledge bases are exactly the pattern Mission 2's fleet substrate needs for per-repo or per-task context retrieval.

---

### 9. Agent Memory

Cloudflare's managed persistent memory service — extracts structured memories from agent conversations and makes them retrievable across sessions without filling the context window. [VERIFIED, src 11]

**Memory types:** Facts (atomic stable knowledge), Events (timestamped occurrences), Instructions (procedural/runbook knowledge), Tasks (ephemeral work items, searchable but not vector-indexed). [VERIFIED, src 11]

**Extraction pipeline:** Content-addressed by SHA-256 (idempotent); full extraction pass (~10k char chunks, 4 concurrent) plus a detail pass for concrete values (names, prices, version numbers); 8-check verifier (entity identity, temporal accuracy, relational context, etc.); asynchronous vectorization via Vectorize. [VERIFIED, src 11]

**Retrieval:** Five parallel channels at query time — direct vector search, full-text, recency-biased, instruction-specific, task-specific — results merged and ranked. [VERIFIED, src 11]

**Integration:** Works independently of Agents SDK; compatible with Anthropic Managed Agents. Distinct from AI Search: "AI Search is for finding results across files; Agent Memory is for context recall." [VERIFIED, src 11]

**Status:** Private beta, waitlist. No pricing disclosed. [VERIFIED, src 11]

**Mission-2 fit:** High (when GA). Cross-session memory recall for fleet agents is a clear need — agent-per-repo context (what was done last session, standing decisions, known constraints) is exactly the Instructions memory type. Currently blocked on GA.

---

### 10. Vectorize

Cloudflare's globally-distributed vector database. General availability. Used internally by AI Search (document vectors) and Agent Memory (fact/event vectors). Accessible directly from Workers for custom RAG pipelines. [VERIFIED, src 12, 15]

Supports embeddings from Workers AI models (BGE, EmbeddingGemma, Qwen3 Embedding) and third-party models. Vector search results link to concrete objects in R2, KV, or D1.

**Mission-2 fit:** Medium. Direct Vectorize use is most relevant if you build custom RAG pipelines that AI Search doesn't cover. For standard agent memory/retrieval, prefer AI Search (which manages Vectorize for you).

---

### 11. D1

Serverless SQLite. GA. No server to provision; per-database 10GB limit designed for horizontal sharding across many small databases. [VERIFIED, src 17]

Pricing: based on query and storage costs (no separate charge for multiple databases). Time Travel: point-in-time recovery within 30 days.

**Agent use cases:** Structured per-agent state (task metadata, config, decisions); audit logs (immutable append with time-travel recovery); agent registry (fleet of agent instances with their DO IDs and status).

**Mission-2 fit:** Medium. D1 is the right choice for structured, queryable agent state that doesn't fit into Durable Objects' key-value model. Particularly useful for fleet-level audit trails and cross-agent query patterns.

---

### 12. Queues

Cloudflare's managed message queue for async coordination. GA on Free and Paid plans. Guaranteed delivery; configurable batch processing, retry logic, delay, and dead-letter queues. No egress charges. [VERIFIED, src 18]

**Agent use cases:** Distributing tasks across an agent fleet; async tool calls where result is not needed immediately; inter-agent messaging; rate-limited ingestion of external events (webhooks, emails); decoupled orchestration (trigger → queue → worker → result).

**Mission-2 fit:** Medium. Queues complement Workflows — use Queues for fan-out (dispatch many independent tasks) and Workflows for stateful sequencing (steps that must happen in order).

---

### 13. Workers KV

Eventually-consistent global key-value store. GA on Free and Paid plans. High read throughput globally; writes are consistent within a region but take up to 60 seconds to propagate globally (this is a known limitation for use cases requiring read-your-writes). [VERIFIED for architecture; propagation time LIKELY based on Cloudflare docs pattern]

**Pricing:** 10M reads/month included; $0.50/M additional. $5/M writes. [VERIFIED, src 22]

**Agent use cases:** Feature flags (sub-millisecond global read); agent config distribution (JSON config blobs); ephemeral rate-limit counters; lookup tables for routing decisions. Flagship (Cloudflare's new native feature flag service, announced Agents Week 2026) is built on KV + Durable Objects.

**Mission-2 fit:** Medium. Good for global config/feature-flag reads. Wrong for per-session state that requires strong consistency — use Durable Objects for that.

---

### 14. R2

Object storage with zero egress fees. GA. No object versioning at the GA tier (confirmed gap — see `./agentic-substrate-distribution.md` for detailed analysis). [VERIFIED, src 15, 26]

**Agent use cases:** Agent artifact storage (generated files, session exports); model files; large document corpora for AI Search; backup targets for Sandbox snapshots. Cloudflare Artifacts (private beta) layers git-protocol versioning on top of R2 for the versioning story.

**Mission-2 fit:** Medium. Excellent cost profile (zero egress). Versioning gap is bridged by Artifacts (when GA) or application-level versioning. Use as the blob storage layer; don't rely on it for atomic state.

---

### 15. Containers

General-purpose serverless containers on the Workers platform. GA as of April 13, 2026. [VERIFIED, src 6] Six instance types from 1/16 vCPU (lite) to 4 vCPU / 12 GiB RAM (standard-4). Active-CPU pricing: $0.000020/vCPU-second; memory: $0.0000025/GiB-second. Network egress: $0.025/GB (NA/EU) with 1TB included. Requires Workers Paid plan.

**GPU access:** Cloudflare has confirmed GPU-equipped container instances ("It has GPUs" per the container platform preview blog). [VERIFIED, src 27]

**Agent use cases:** Long-running model servers; custom inference runtimes (PyTorch, custom CUDA); CLI tools requiring full Linux; agents that need parallel CPU cores; any workload that doesn't fit Workers' V8 isolate model.

**Mission-2 fit:** Medium. Most of Mission 2's fleet substrate fits in Workers + Durable Objects. Containers are the escape hatch for workloads that need a real OS or parallel CPU — e.g., running a Python-based agent framework that can't be compiled to Workers-compatible WASM.

---

### 16. Browser Run (formerly Browser Rendering)

Headless browser automation on Cloudflare's network. GA on Free and Paid plans. [VERIFIED, src 8] 4x concurrency increase shipped during Agents Week 2026.

**Supported automation frameworks:**
- Puppeteer (standard browser control)
- Playwright (testing and automation)
- Chrome DevTools Protocol / CDP (direct control from any environment)
- Stagehand (AI-powered element selection by intent, not selectors)

**MCP integration:** Browser Run supports Playwright MCP or CDP with MCP clients — enabling LLMs to control browsers via MCP. Integrated directly with the Agents SDK. [VERIFIED, src 8]

**New in Agents Week 2026:** Live View (observe the browser session in real time); Human in the Loop (pause and hand off to a human for steps the agent can't complete); session recordings. [VERIFIED, src 1]

**Mission-2 fit:** Medium. Relevant if fleet agents need web automation (research tasks, form submission, content extraction). Not on the critical path for Mission 2's BackendAdapter substrate, but highly relevant for agent task types the fleet will execute.

---

### 17. Cloudflare Mesh

Secure private networking launched during Agents Week 2026 (April 14, 2026, public beta). [VERIFIED, src 9] Post-quantum encrypted mesh networking for users, devices, nodes, and autonomous AI agents. Every enrolled participant gets a private Mesh IP; peer-to-peer connectivity without deploying infrastructure.

**Workers VPC integration:** Workers and Durable Objects can bind to the Mesh network via `cf1:network`, reaching any Mesh node, subnet route, or private database in the account. This gives agent Workers scoped access to internal APIs, databases, and services without public exposure.

**Use cases for agents:** Agent Worker → internal PostgreSQL (no public endpoint); agent Worker → corporate SSO service; fleet coordinator → Durable Object in another VPC zone; coding agent → private GitHub Enterprise.

**Mission-2 fit:** Medium. Not immediately critical for single-developer fleet substrate, but important once fleet agents need to reach private infrastructure (production DBs, internal APIs, koinos-context's private endpoints). Replaces the per-tunnel fragility of Cloudflare Tunnel for agent-scale use.

---

### 18. Cloudflare Tunnel

Outbound-only secure tunnel from private infrastructure to Cloudflare's network. GA, free for basic use. [VERIFIED, src 23]

**Mechanism:** `cloudflared` daemon establishes an outbound-only connection; Cloudflare routes inbound traffic through the tunnel. No public IP required; firewall can block all inbound.

**Agent use cases:** Expose local agent dev environment to receive webhook callbacks; expose local MCP server endpoint to cloud clients during development; expose local Postgres/Redis to Workers in dev mode.

**Mission-2 fit:** Low-Medium. Primarily a development convenience. For production fleet infrastructure, Cloudflare Mesh (above) is the right primitive. Tunnel is the right tool for the dev/test loop: develop locally, receive real webhooks, debug against real cloud infrastructure.

---

### 19. MCP Server Hosting (via Agents SDK + Durable Objects)

Cloudflare enables hosting remote MCP servers as a first-class product. Each MCP client session is backed by a Durable Object (strongly consistent, WebSocket-hibernating state per client). Authentication via OAuth (RFC 9728 compliant). Supports Streamable HTTP transport (current MCP spec). [VERIFIED, src 24, 25]

**Cloudflare's own MCP catalog:** Cloudflare hosts and operates managed remote MCP servers (Workers, D1, R2, Vectorize, AI Gateway, Browser Run, etc.) — 13+ Cloudflare-operated MCP servers available. Partners include Anthropic, Stripe, Linear, Sentry, Atlassian, Asana, Block, Intercom, PayPal, Webflow. [VERIFIED, src 25]

**Enterprise MCP reference architecture:** Cloudflare Access (OAuth/OIDC), AI Gateway (observability + rate limiting), and MCP server portals compose into an enterprise-grade MCP deployment pattern. [VERIFIED, src 1]

**Mission-2 fit:** High. Remote MCP servers on Cloudflare are a direct primitive for fleet substrate — each BackendAdapter can expose its capabilities as an MCP server, discoverable by any MCP client in the fleet. The Durable Object backing provides per-session state with WebSocket hibernation.

---

### 20. Analytics Engine + Logpush

**Analytics Engine:** Custom time-series telemetry from Workers code. Unlimited cardinality (any dimension combination). SQL query API for analysis. GA. The right primitive for agent-custom event logging (tool calls, model choices, latency by provider, error rates per agent type). [VERIFIED, src 28]

**Logpush:** Enterprise-only (exception: Workers Trace Events on Workers Paid). Near-real-time log push to external SIEM/storage. Useful for feeding agent logs to Datadog, Splunk, or BigQuery. Cannot backfill; one-time delivery. [VERIFIED, src 29]

**Mission-2 fit:** Analytics Engine: Medium (custom telemetry from agent Workers). Logpush: Low (Enterprise gate and operational limitations limit its role in Mission 2's near-term fleet substrate).

---

### 21. Cloudflare Artifacts

Git-protocol versioned storage for agent session repos. Private beta as of April 16, 2026. Built on Durable Objects + R2 + SQLite for chunked git object storage. Implements git protocol v1 and v2 (ls-refs, shallow clones, delta encoding, git-notes for agent metadata). ArtifactFS (open-sourced at github.com/cloudflare/artifact-fs) enables blobless mount of large repos.

Pricing: $0.15 per 1k ops (10k included), $0.50/GB-mo (1GB included). [VERIFIED]

**Mission-2 fit:** Medium — detailed analysis in `./agentic-substrate-distribution.md`. Key finding: architecturally on-target for per-session agent repo versioning; defer commitment until public beta + pricing pilot.

---

### 22. Cloudflare Pages

Standard JAMstack platform. GA, all plans. Git-integrated deployment, global CDN, Functions for server-side logic. [VERIFIED]

**Agent use cases:** Deploy agent-generated static frontends; host agent result dashboards; provide human-in-the-loop approval UIs as a lightweight web app. Composable with D1 and R2 for data-backed frontends.

**Mission-2 fit:** Low. Not on the critical path for fleet substrate. Relevant only when Mission 2 agents need to expose a browser-accessible UI.

---

### 23. Constellation (Deprecated)

Cloudflare's original edge ML inference service — allowed running ONNX models on Workers. Superseded by Workers AI (launched 2023). The `developers.cloudflare.com/constellation/` URL returns a 403. [PARAMETRIC — 403 error on fetch; web search confirms Workers AI supersedes it but no explicit deprecation notice found]

**Mission-2 fit:** N/A. Treat as sunset; use Workers AI for edge inference.

---

## Recommended Deep-Dive Candidates

Services that warrant their own `evaluations/<name>.md` beyond the Agents SDK deep-dive:

1. **AI Gateway** — Most actionable immediate win for fleet observability. Deep-dive should cover: routing rule configuration for multi-provider fallback, DLP profile setup, Analytics Engine integration for custom agent metrics, cost-attribution patterns (metadata by agent session ID). Priority: High.

2. **Workflows v2** — The right orchestration primitive for Mission 2's long-running dispatch loops. Deep-dive should cover: step design patterns for agent loops, event-wait semantics for human-in-the-loop, vs. Durable Objects for short-lived coordination, Workflows v2 vs v1 migration path. Priority: High.

3. **Durable Objects in agent context** — The foundational state primitive, but the rules of use are non-obvious (the "rules of Durable Objects" doc is load-bearing). Deep-dive should cover: when to use DO vs D1 vs KV, SQLite storage patterns for agent conversation history, WebSocket hibernation for streaming agents, Facets for per-tenant isolation. Priority: Medium.

4. **Sandboxes** — The code-execution primitive for eval agents and tool-use agents. Deep-dive should cover: backup/restore cost model, egress policy configuration, integration with Agents SDK, comparison to E2B and Daytona (competitor sandboxes). Priority: Medium.

5. **Cloudflare Mesh** — Will matter when fleet agents need private infrastructure access. Deep-dive should cover: Workers VPC binding syntax, authentication model, cost vs. Cloudflare Tunnel, comparison to Tailscale for agent use. Priority: Low (defer until Mission 2 needs private infra access).

---

## Cloudflare-Specific Risks + Lock-In

### Vendor lock-in surface

**High lock-in risk:**
- **Durable Objects** — the state model is proprietary. If you build deep on DO (SQLite schema, hibernation patterns, strong-consistency semantics), migrating to another provider requires rewriting your entire state layer. No equivalent exists in AWS Lambda, Fly.io, or Vercel at parity.
- **Agents SDK** — the `Agent` class and its primitives (SQLite binding, WebSocket hibernation, scheduling) are Cloudflare-only abstractions. Migrating agents to another runtime requires a full rewrite of state management and WebSocket handling.
- **Workflows v2** — the `step.do()` durable execution model is Cloudflare-specific. AWS Step Functions and Inngest are functional equivalents but require different SDKs.

**Low lock-in risk:**
- **AI Gateway** — the proxy pattern is portable. If you outgrow Cloudflare AI Gateway, you can replace it with LiteLLM, Helicone, or a self-hosted proxy. Your agent code doesn't change (one URL swap).
- **R2** — S3-compatible API. Zero migration friction if you move to S3, B2, or Tigris.
- **Workers AI** — inference is stateless. Swap to direct Anthropic/OpenAI calls with a one-line change through AI Gateway.
- **Queues** — standard queue semantics. Portable to SQS, BullMQ, or Kafka with adapter rewrites.

### Egress costs

Cloudflare's zero-egress-cost positioning is a genuine differentiator: [VERIFIED, src 30]
- R2: no egress fees (vs S3's $0.09/GB)
- Workers: no egress fees
- D1: no egress fees
- Containers: egress charged at $0.025/GB (NA/EU) with 1TB included — not zero, but competitive vs AWS

**The risk:** Cloudflare has not committed to keeping egress-free pricing permanent. It is a strategic decision, not a technical constraint. If pricing changes, the lock-in depth above means switching costs are high.

### Workers execution model constraints

- **V8 isolate model:** No arbitrary binaries, no file system, limited CPU per invocation. Containers and Sandboxes are the escape hatches but add cost and cold-start latency.
- **No GPU on standard Workers:** GPU inference goes through Workers AI (opaque pricing) or Containers (GPU instances, separate pricing). You can't run a local PyTorch model on a standard Worker.
- **5-minute CPU limit on standard Workers:** Long agent tasks must use Workflows or Durable Objects with hibernation. Not a showstopper but requires architectural awareness.

### Data residency

Cloudflare's global network spans 330+ cities. By default, Workers run in the PoP closest to the user — data may be processed in any jurisdiction. [VERIFIED — this is Cloudflare's standard deployment model] For agents handling GDPR-sensitive data, explicit data localization (Smart Placement override, per-region DO placement) is required. This is a real constraint for EU-bound agent workloads.

---

## Sources

All sources accessed on 2026-04-29.

1. https://blog.cloudflare.com/agents-week-in-review/ — Agents Week 2026 full recap (2026-04-17) [VERIFIED]
2. https://developers.cloudflare.com/agents/ — Cloudflare Agents framework docs (2026-04-29) [VERIFIED]
3. https://developers.cloudflare.com/ai-gateway/ — AI Gateway overview (2026-04-29) [VERIFIED]
4. https://developers.cloudflare.com/durable-objects/ — Durable Objects docs (2026-04-29) [VERIFIED]
5. https://blog.cloudflare.com/workflows-v2/ — Workflows v2 architecture (2026-04-15) [VERIFIED]
6. https://developers.cloudflare.com/changelog/post/2026-04-13-containers-sandbox-ga/ — Containers + Sandboxes GA (2026-04-13) [VERIFIED]
7. https://blog.cloudflare.com/sandbox-ga/ — Sandboxes GA announcement (2026-04-13) [VERIFIED]
8. https://developers.cloudflare.com/browser-rendering/ — Browser Run docs (2026-04-29) [VERIFIED]
9. https://blog.cloudflare.com/mesh/ — Cloudflare Mesh announcement (2026-04-14) [VERIFIED]
10. https://blog.cloudflare.com/ai-search-agent-primitive/ — AI Search (AutoRAG rebrand) (2026-04-16) [VERIFIED]
11. https://blog.cloudflare.com/introducing-agent-memory/ — Agent Memory private beta (2026-04-17) [VERIFIED]
12. https://developers.cloudflare.com/vectorize/ — Vectorize docs (2026-04-29) [VERIFIED]
13. https://developers.cloudflare.com/workers-ai/platform/pricing/ — Workers AI pricing (2026-04-29) [VERIFIED]
14. https://blog.cloudflare.com/workers-ai-large-models/ — Kimi K2.5/K2.6 on Workers AI (2026-04-16) [VERIFIED]
15. https://community.cloudflare.com/t/r2-object-versioning-and-replication/524025 — R2 no GA versioning (confirmed) [VERIFIED, also in agentic-substrate-distribution.md src 16]
16. https://developers.cloudflare.com/workers/platform/pricing/ — Workers pricing (2026-04-29) [VERIFIED]
17. https://developers.cloudflare.com/d1/ — D1 docs (2026-04-29) [VERIFIED]
18. https://developers.cloudflare.com/queues/ — Queues docs (2026-04-29) [VERIFIED]
19. https://blog.cloudflare.com/ai-platform/ — Cloudflare AI Platform / Unified Inference Layer (2026-04-15) [VERIFIED]
20. https://developers.cloudflare.com/ai-gateway/reference/pricing/ — AI Gateway pricing (2026-04-29) [VERIFIED]
21. https://developers.cloudflare.com/changelog/post/2026-04-15-workflows-limits-raised/ — Workflows v2 concurrency limits (2026-04-15) [VERIFIED]
22. https://developers.cloudflare.com/workers/platform/pricing/ — Workers + KV pricing detail (2026-04-29) [VERIFIED]
23. https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/ — Cloudflare Tunnel (2026-04-29) [VERIFIED]
24. https://developers.cloudflare.com/agents/model-context-protocol/ — MCP hosting on Cloudflare (2026-04-29) [VERIFIED]
25. https://blog.cloudflare.com/thirteen-new-mcp-servers-from-cloudflare/ — 13 Cloudflare MCP servers (2026-04-14) [VERIFIED]
26. https://blog.cloudflare.com/artifacts-git-for-agents-beta/ — Cloudflare Artifacts beta (2026-04-16) [VERIFIED — also covered in `./agentic-substrate-distribution.md`]
27. https://blog.cloudflare.com/container-platform-preview/ — Container platform GPU preview [VERIFIED]
28. https://developers.cloudflare.com/analytics/analytics-engine/ — Analytics Engine docs (2026-04-29) [VERIFIED]
29. https://developers.cloudflare.com/logs/logpush/ — Logpush docs (2026-04-29) [VERIFIED]
30. https://developers.cloudflare.com/r2/pricing/ — R2 zero-egress pricing (2026-04-29) [VERIFIED]
31. https://developers.cloudflare.com/agents/model-context-protocol/mcp-servers-for-cloudflare/ — Cloudflare's own MCP servers (2026-04-29) [VERIFIED]

---

## Summary (250 words)

**Top-5 Cloudflare primitives by Mission-2 fleet-substrate fit:**

1. **AI Gateway** — Proxy for all inference calls fleet-wide. Unified logging, caching, DLP, fallback routing across 14+ providers. Zero-configuration observability for the fleet's entire model surface. Free core.
2. **Durable Objects** — Per-session stateful coordination with strongly-consistent SQLite. The foundation of the Agents SDK and MCP server sessions. No external state store needed.
3. **Workflows v2** — Durable orchestration for long-running fleet tasks. 50k concurrency, independently retryable steps, human-in-the-loop pause, weeks-long duration. Maps directly to Mission 2's dispatch and session-lifecycle flows.
4. **Workers** — Entry point for everything. CPU-billed (not wall-clock), so idle I/O-waiting agents are cheap. The universal compute substrate for the fleet.
5. **Sandboxes** — GA as of April 13, 2026. Isolated, persistent code execution with PTY, backup/restore snapshots, and zero-trust egress. The right primitive for tool-use agents and eval harnesses in the fleet.

**Recommended composition patterns:** (a) Agents SDK for standard chat/voice/MCP agents; (b) Workflows v2 as the outer harness for multi-step fleet orchestration; (c) AI Gateway as the universal inference proxy regardless of which pattern runs beneath.

**Recommended deep-dive candidates beyond Agents SDK:** AI Gateway (highest immediate value for fleet observability), Workflows v2 (orchestration patterns for Mission 2 dispatch), Sandboxes (code-execution isolation for tool-use agents).

**What I might have missed:** Cloudflare Flagship (feature flags on KV + DO — announced Agents Week, minimal public docs), detailed Workflows v2 pricing (not prominently surfaced), Durable Object Facets pricing and GA timeline, and any agent-specific Containers GPU pricing. Also: the "Project Think" next-gen SDK (preview only as of research date) warrants monitoring — it may significantly change the Agents SDK surface before Mission 2 adopts it.
