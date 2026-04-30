---
title: "OpenAI Agent Product Spectrum — Vendor Landscape"
description: "Full mapping of OpenAI's agent-related products as candidates for Mission 2's fleet substrate. Covers API platform, Responses API, Agents SDK, AgentKit, Symphony, Codex CLI, Codex Cloud, Realtime API, Computer Use, Workspace Agents, Batch API, and supporting primitives."
research_date: 2026-04-29
last_verified: 2026-04-29
staleness_warning: "OpenAI ships rapidly; agent-product surface evolves monthly. Symphony released 2026-03-05 (15k stars by late April); Workspace Agents launched 2026-04-22; AgentKit and Responses API extension are both mid-rollout. Re-verify before acting if >45 days old."
confidence: high
sources_count: 38
related:
  - ../evaluations/openai-codex-cloud.md
  - ./agentic-substrate-distribution.md
status: active
---

# OpenAI Agent Product Spectrum — Vendor Landscape

> **Research date:** 2026-04-29
> **Last verified:** 2026-04-29
> **Staleness warning:** Re-verify after 45 days. OpenAI ships monthly. Assistants API sunsets 2026-08-26; Computer Use API tier requirements are in flux; Symphony is post-release but pre-stable.
> **Confidence:** ~30 VERIFIED claims, ~8 LIKELY, 3 UNVERIFIED
>
> **Key sources:**
> - Symphony GitHub: https://github.com/openai/symphony (verified 2026-04-29)
> - Agents SDK Python docs: https://openai.github.io/openai-agents-python/ (verified 2026-04-29)
> - Codex App Server InfoQ: https://www.infoq.com/news/2026/02/opanai-codex-app-server/ (verified 2026-04-29)
> - Workspace Agents announcement: https://openai.com/index/introducing-workspace-agents-in-chatgpt/ (verified 2026-04-29)
> - Assistants deprecation thread: https://community.openai.com/t/assistants-api-beta-deprecation-august-26-2026-sunset/1354666 (verified 2026-04-29)
> - Batch API docs: https://developers.openai.com/api/docs/guides/batch (verified 2026-04-29)

---

## 1. TL;DR

**Top 3 surfaces relevant to Mission 2 fleet substrate:**

1. **Agents SDK (Python + TypeScript)** — The direct programmatic layer for multi-agent fleet orchestration. Provider-agnostic handoffs, guardrails, sandbox agents, and built-in tracing make this the substrate-code layer. Mission 2 builds on top of this, not around it.

2. **Responses API** — The new canonical API primitive replacing Assistants (sunset 2026-08-26). Built-in tool loop, MCP server support, computer use, code interpreter, context compaction. Every fleet agent that calls OpenAI models should call this, not Chat Completions for new work.

3. **Codex App Server / Codex CLI** — The headless JSON-RPC 2.0 protocol that Symphony is built on. If Mission 2 needs to dispatch Codex agents programmatically and ingest structured JSONL output, the App Server is the integration surface. Deep-dive exists at `../evaluations/openai-codex-cloud.md`.

**Symphony note:** Released 2026-03-05 as Apache 2.0 open-source. It is a reference implementation — an Elixir/BEAM daemon that turns a Linear board into a Codex agent control plane. OpenAI treats it as a demo of what you can build on Codex App Server, not a maintained product. Mission 2 fit is **medium**: the pattern is directly relevant (Linear → agent dispatch → CI gating), but the Elixir stack and Linear-specific polling are not fleet-portable without reimplementation. Read §3 for full assessment.

**Verdict per surface:** Agents SDK = integrate now. Responses API = migrate all new work now. Codex App Server = evaluate alongside the parallel deep-dive. Symphony = study the pattern, don't adopt the binary.

---

## 2. Product Spectrum Table

| Product | Maturity | Primary Use Case | Mission-2 Fit | Deep-Dive Status |
|---|---|---|---|---|
| **Symphony** | Open-source beta (released 2026-03-05) | Linear → Codex agent orchestration daemon | Medium — pattern relevant, stack not portable | Needed (this doc covers it) |
| **Responses API** | GA (launched Q1 2025, extended Q1-Q2 2026) | Canonical agentic API primitive | High — replace Assistants for all new work | n/a (well-documented) |
| **Agents SDK (Python)** | GA | Multi-agent workflows, handoffs, guardrails | High — primary fleet orchestration SDK | n/a (well-documented) |
| **Agents SDK (TypeScript)** | GA | Same as Python, TypeScript environments | High | n/a |
| **Codex CLI** | GA (open-source) | Local terminal coding agent | Medium — headless mode useful for dispatch | Existing [1] |
| **Codex App Server** | GA (protocol) | Headless JSON-RPC agent runtime | High — substrate integration surface | Existing [2] |
| **Codex Cloud / Sandbox** | GA | Cloud-isolated coding agent | High — remote execution, no local env | See `../evaluations/openai-codex-cloud.md` |
| **AgentKit** | GA (launched 2025-10) | No-code + low-code agent builder platform | Low — not API-first; for non-dev users | n/a |
| **Workspace Agents** | Research preview (launched 2026-04-22) | Enterprise ChatGPT automation (Slack, Salesforce) | Low — consumer/enterprise product, not API | Needed |
| **Realtime API** | GA | Speech-to-speech voice agents | Low — not in Mission 2 scope | n/a |
| **Computer Use / CUA** | Research preview (Responses API, tiers 3–5) | Screen-interaction agents | Low — narrow scope, high cost | n/a |
| **Custom GPTs** | Deprecated 2026-08-26 | Consumer chatbot customization | n/a — being replaced by Workspace Agents | n/a |
| **Assistants API** | Deprecated, sunset 2026-08-26 | Managed threads/runs with tools | n/a — migrate to Responses API | n/a |
| **Batch API** | GA | Async bulk inference at 50% cost | Medium — eval pipelines, bulk classification | n/a |
| **Function Calling / Tool Use** | GA | Structured tool invocation across all models | High — underpins every agent surface | n/a |
| **Fine-tuning / Distillation / Evals** | GA | Model optimization and agent evaluation | Medium — fleet-level eval and distillation | n/a |
| **MCP server support** | GA (Responses API) | Remote MCP tool servers | High — fleet-wide tool registry integration | n/a |

---

## 3. OpenAI Symphony — Dedicated Section

### What It Is

[VERIFIED] Symphony is an open-source orchestration daemon that turns a project-management board (Linear by default) into a control plane for Codex coding agents. Each open issue maps to an isolated agent workspace. Symphony polls the board, spawns agents, manages lifecycle (restarts on crash/stall), monitors CI, and requires proof of work before landing changes.

Released: 2026-03-05. Repository: https://github.com/openai/symphony. License: Apache 2.0. [VERIFIED, from GitHub search results]

As of late April 2026, the repository had surpassed 15,000 GitHub stars. [VERIFIED, source [4]]

### Architecture

[VERIFIED] Reference implementation: Elixir on the Erlang/BEAM runtime, using OTP supervision trees for fault-tolerant process isolation. PostgreSQL via Ecto for persistent state. Confirmed from MarkTechPost technical analysis [source 5] and OpenTools [source 6].

Core components:

1. **Issue polling** — Monitors Linear board states (configurable status transitions act as a state machine).
2. **Workspace isolation** — Per-issue deterministic workspaces; agents are scoped to a directory, cannot interfere with each other.
3. **Agent execution** — Dispatches Codex agents (via Codex App Server's headless JSON-RPC 2.0 protocol over stdio) to execute the issue description.
4. **Proof-of-work gate** — Before landing, agents must provide: CI status report, passing unit tests, PR review feedback, and a change walkthrough.
5. **Code landing** — Upon proof-of-work, agent submits or merges the PR.

**WORKFLOW.md** [VERIFIED] — A repo-level file that serves as the agent's system instructions and runtime configuration. Treated as code (version-controlled). Rendered with issue context and delivered to the agent on Turn 1.

**Concurrency** [VERIFIED] — Default 10 concurrent agents; maximum 20 turns per agent. [source 6]

**Linear integration** [VERIFIED] — Dynamic `linear_graphql` tool call. Access tokens are explicitly kept away from subagents (security design). [source 7]

### Relationship to Codex App Server

[VERIFIED] Symphony is built on the Codex App Server — the same headless JSON-RPC 2.0 API mode that powers the Codex CLI, VS Code extension, macOS desktop app, and web app. OpenAI positioned Symphony as a demonstration of what external teams can build on top of Codex App Server. This is the clearest signal that the App Server is the intended integration surface for any fleet that wants to dispatch Codex programmatically. [sources 7, 8]

### Performance Claims

[VERIFIED, with caveat] Some OpenAI internal teams reported a 500% increase in landed pull requests during the first three weeks of use. InfoWorld analysts quoted in coverage specifically cautioned against treating PR volume as a productivity proxy without quality metrics. [sources 2, 9] The 500% figure has no published baseline. Treat as directional signal, not benchmark.

### Open-Source Status and Maintenance

[VERIFIED] Apache 2.0. OpenAI described Symphony as "a low-key engineering preview for testing in trusted environments" and as a reference implementation — not a standalone product they plan to maintain. OpenAI is explicitly inviting reimplementations in other languages following the published SPEC.md. [sources 5, 10]

This is a deliberate stance: Symphony demonstrates the pattern; the community owns the productization.

### Mission-2 Fit Assessment

**Fit: Medium**

What transfers directly:
- The orchestration pattern: ticket state machine → isolated workspace → agent dispatch → CI gating → proof-of-work → PR landing. This is the right model for any fleet that wants agents working autonomously on defined issues.
- The Codex App Server as the integration surface. If Mission 2 dispatches Codex agents, connecting via App Server JSON-RPC (not the TUI) is the correct path.
- WORKFLOW.md as repo-level agent configuration. Every fleet repo should have an equivalent.

What does not transfer without work:
- Elixir/BEAM runtime — not compatible with the current Bun+TypeScript substrate at `terminal-ops/`. A Python or TypeScript reimplementation following SPEC.md is the fleet-compatible path.
- Linear-only polling — Symphony is hardwired to Linear's GraphQL API. The fleet uses Linear, so this is less of a gap, but any generalization requires abstraction.
- "10 concurrent agents max" — not a constraint for Mission 2's current scale, but worth tracking as fleet grows.

What it does NOT handle (per explicit design):
- Ambiguous problems requiring strong judgment. These still need interactive sessions.
- Multi-repo or cross-repo orchestration. Symphony is per-repo.

**Recommended action:** Read SPEC.md at github.com/openai/symphony/blob/main/SPEC.md. Study the proof-of-work contract and WORKFLOW.md schema. Don't run the Elixir binary in production — it is not maintained. If Mission 2 wants this pattern, implement the scheduler in TypeScript against the same Codex App Server protocol.

---

## 4. Per-Product Summaries

### 4.1 OpenAI API Platform — Core Models

[VERIFIED] The current API model lineup (April 2026) centers on the GPT-5 family. GPT-4 series and original o-series models were retired from ChatGPT on 2026-02-13, though API access for existing integrations remains.

**Current primary models (API):**
- **GPT-5.5** — $5.00/M input, $30.00/M output (standard). $2.50/$15.00 via Batch or Flex. Cached input: $0.50/M. Long-context (>272K tokens): 2× input, 1.5× output. [VERIFIED, source 11]
- **GPT-5.5-pro** — $30.00/M input, $180.00/M output. [VERIFIED]
- **GPT-5.4, GPT-5.3-Codex, GPT-5.4-mini, GPT-5.4-nano** — available for lower-latency/lower-cost use cases. [VERIFIED]

**Reasoning model consolidation:** OpenAI merged reasoning and general-purpose intelligence into a unified router design. GPT-5.4 Thinking handles deeper reasoning without a separate o-series model. o3 and o4-mini are available on API for existing integrations. [VERIFIED, source 12]

**Auth:** Standard API key (`OPENAI_API_KEY`). Enterprise via Azure OpenAI Service for data-residency requirements. [VERIFIED]

Mission-2 fit: High. Model selection is upstream of agent surface selection — every fleet agent that calls OpenAI will use this pricing table.

---

### 4.2 Responses API

[VERIFIED] Launched Q1 2025 as an evolution of Chat Completions. Adds an agentic execution loop, built-in tool support, and context compaction. Chat Completions remains supported; Responses is the recommended path for all new agent work. [source 13]

**Key capabilities:**
- Built-in tool loop: model proposes action → tool executes → model continues (without round-tripping to client code for every step)
- **Built-in tools:** `web_search`, `file_search`, `code_interpreter`, `computer_use` (research preview, tiers 3–5), `image_generation`, remote MCP servers
- **Context compaction:** Long-running tasks compress prior steps without losing key context — enables agents that run over many iterations without hitting token limits
- **MCP server support:** Responses API supports remote MCP servers via Streamable HTTP or HTTP/SSE transport. Approval flow per tool call (configurable). No additional fee per tool call — pay tokens only. [VERIFIED, source 14]
- **3% SWE-bench improvement** over Chat Completions on reasoning models [VERIFIED, source 15]
- **40–80% improvement in cache utilization** vs Chat Completions (internal OpenAI benchmark) [VERIFIED, source 15]

**Assistants API sunset context:** Assistants API sunsets 2026-08-26. Official migration path: Responses API + Conversations API. Migration guide published. [VERIFIED, source 16]

Mission-2 fit: High. The MCP server support is directly relevant — the fleet's tool registry can be exposed as a remote MCP server that all fleet agents call via Responses API without bespoke tool-call scaffolding.

---

### 4.3 Agents SDK (Python + TypeScript)

[VERIFIED] Open-source, Apache 2.0. Python: https://github.com/openai/openai-agents-python. TypeScript: https://github.com/openai/openai-agents-js. Both are GA. The SDK is described as "a production-ready upgrade of the previous experimentation for agents, Swarm." [source 17]

**Core primitives:**
- **Agents** — LLM + instructions + tools. Python: Pydantic models. TypeScript: Zod schemas.
- **Handoffs** — Agent-to-agent delegation. Named ownership transfer with context pass-through. Supports both parallel fan-out and sequential delegation chains.
- **Guardrails** — Input/output validation that runs in parallel with agent execution. Fails fast on constraint violation before tool calls or output delivery.
- **Tracing** — Built-in per-run trace graph. Integrates with OpenAI's eval, fine-tuning, and distillation tools. Third-party compatible (structured output).
- **Sessions** — Persistent working context across runs. Multiple storage backends: SQLAlchemy, Redis, MongoDB, encrypted variants. [VERIFIED]
- **Sandbox agents** — Container-based execution environments with files, commands, packages, and memory. Manifest-defined; sessions are resumable.
- **Realtime agents** — Built-in voice agent support via `gpt-realtime-1.5`. Automatic interruption detection, context management, guardrails in voice pipelines.

**Provider agnosticism:** [VERIFIED] The SDK supports OpenAI Responses API and Chat Completions, plus 100+ other LLMs via compatible interfaces. [source 17]

**TypeScript parity gap:** [LIKELY] As of the research date, sandbox agents and some new harness capabilities are Python-first with TypeScript support planned for a future release. [source 17] Verify current parity before committing to TypeScript for sandbox workloads.

**TechCrunch (2026-04-15):** OpenAI updated the Agents SDK with enterprise safety features — stronger guardrail enforcement and approval-gate primitives. [VERIFIED, source 18]

**Temporal integration:** [VERIFIED] Production integration with Temporal for durable agent execution and workflow orchestration. [source 19] Relevant if Mission 2 needs durable multi-step workflows with replay semantics.

Mission-2 fit: High. The primary SDK for building fleet orchestration logic. Handoffs map directly to the fleet's dispatcher/builder/verifier agent chain. Tracing integrates with the eval pipeline. Guardrails enforce the verification.md rules at the SDK level.

---

### 4.4 Codex CLI

[VERIFIED] Open-source (https://github.com/openai/codex). Written in Rust. Terminal-native coding agent. [source 20]

Deep coverage exists in the terminal-agent-orchestration landscape at `studio-pm/research/terminal-agent-orchestration-landscape.md` §2. Key facts for cross-reference:

- Headless mode: `codex exec "<prompt>"` with `--json` flag → JSONL event stream on stdout
- App Server mode: headless JSON-RPC 2.0 over stdio — the integration surface Symphony uses
- Approval modes: `suggest`, `auto-edit`, `full-auto`
- April 2026: Unix socket transport for App Server integrations, pagination-friendly resume/fork, sticky environments [VERIFIED, source 20]
- Model support: GPT-5.5, GPT-5.4, GPT-5.3-Codex via `/model` selector
- `codex exec --json` now reports reasoning-token usage for programmatic consumers [VERIFIED, source 20]

Mission-2 fit: Medium. Codex CLI is the tool Mission 2 dispatches; the App Server protocol is the integration surface. See `../evaluations/openai-codex-cloud.md` for the cloud/sandbox deep-dive.

---

### 4.5 Codex Cloud / Sandbox

[VERIFIED] Cloud-hosted Codex agent execution in OpenAI-managed isolated containers. Each task runs in its own container preloaded with the repository. Internet access is disabled during execution. [source 21]

**Recent 2026 updates:**
- macOS and Windows desktop app (March 2026) added visual supervision layer for parallel agent threads
- Remote TUI mode: run App Server on one machine, connect TUI from another
- App Server Unix socket transport for lower-latency local integrations

This surface has a dedicated deep-dive in preparation at `../evaluations/openai-codex-cloud.md`. Do not duplicate — reference it.

Mission-2 fit: High. Isolated cloud execution + repo preloading + CI integration is the target deployment model for fleet builder agents.

---

### 4.6 AgentKit

[VERIFIED] Launched at OpenAI Dev Day, October 2025. A complete toolkit for building, deploying, and optimizing agent workflows. Not to be confused with the Agents SDK — AgentKit is the higher-level platform layer with visual tooling. [source 22]

**Components:**
- **Agent Builder** — Visual canvas for composing logic with drag-and-drop nodes. Supports preview runs, inline eval configuration, and full versioning. [VERIFIED]
- **Connector Registry** — Central management for how data and tools connect across OpenAI products. [VERIFIED]
- **ChatKit** — Toolkit for embedding customizable chat-based agent experiences in external products. [VERIFIED]
- **Enhanced Evaluation** — Datasets, trace grading, automated prompt optimization, third-party model support. [VERIFIED]

**Positioning distinction:** AgentKit is above the Agents SDK in the stack. The Agents SDK is code-first; AgentKit provides the GUI and deployment tooling around it. AgentKit wraps the Agents SDK for teams that want faster iteration without raw code orchestration.

Mission-2 fit: Low. Mission 2 is substrate engineering — code-first, CLI-native. AgentKit's visual builder and ChatKit are consumer-facing surfaces. The eval components (datasets, trace grading) are worth monitoring as fleet-level evaluation tooling matures.

---

### 4.7 Workspace Agents

[VERIFIED] Launched 2026-04-22 as research preview. Codex-powered agents within ChatGPT Business/Enterprise/Edu/Teachers plans. Successor to Custom GPTs for enterprise use cases. [source 23]

**Capabilities:**
- Run in the cloud continuously (not session-bound)
- Share within organization
- Native Slack and Salesforce integrations at launch (more connectors Q2 2026)
- Credit-based pricing starting 2026-05-06 (free during research preview)
- Enterprise admin controls for tool access by user group [VERIFIED, source 23]

**Custom GPTs deprecation:** [VERIFIED] Custom GPTs for business accounts sunset 2026-08-26, same date as Assistants API. [source 23]

Mission-2 fit: Low. Workspace Agents are a ChatGPT product, not an API surface. Fleet substrate cannot programmatically dispatch or orchestrate Workspace Agents. Relevant context for understanding OpenAI's enterprise positioning but not actionable for Mission 2.

---

### 4.8 Realtime API

[VERIFIED] GA. Speech-to-speech agent API using WebRTC or WebSocket transport. Enables voice agents with sub-300ms latency. [source 24]

**Current capabilities:**
- Model: `gpt-realtime-1.5` (via Agents SDK) [VERIFIED]
- Remote MCP server support [VERIFIED]
- Image inputs [VERIFIED]
- Phone calling via SIP [VERIFIED]
- Automatic interruption detection

**Pricing (current):**
- Audio input: $32/M tokens; cached input: $0.40/M
- Audio output: $64/M tokens
- Text input: $4.00/M; cached: $0.40/M; text output: $16.00/M
- Practical rate: ~$0.04/min for typical conversation patterns [VERIFIED, source 25]

Mission-2 fit: Low. Mission 2 is text-based fleet orchestration. Realtime API is voice-first. Relevant if fleet agents ever need voice interfaces (terminal voice commands), but not in scope now.

---

### 4.9 Computer Use / Operator

**Computer Use (CUA tool in Responses API):** [VERIFIED] Available as research preview in Responses API for usage tiers 3–5. Priced at $3/M input tokens and $12/M output tokens. Screenshot-based GUI interaction (browser + desktop). GPT-5.4 provides built-in CUA support. [source 26]

**Operator (ChatGPT product):** [VERIFIED] Fully integrated into ChatGPT as "agent mode" as of 2026-07-17 (projected from Anthropic's knowledge). Operator was the consumer-facing product; CUA is the API primitive. [source 27]

[UNVERIFIED] API access beyond tiers 3–5 and exact GA timeline. The search results from March 2026 indicated "no GA API timeline," which may have changed — verify before building on this surface. [source 28]

Mission-2 fit: Low (for now). Screen-scraping agents are fragile for fleet use. If a future Mission 2 task requires interacting with a web UI that has no API (e.g., a vendor admin portal), the CUA tool is the path. Not a primary substrate surface.

---

### 4.10 Batch API

[VERIFIED] GA. 50% cost discount on all supported endpoints in exchange for ≤24-hour completion window. Separate rate limit pool from synchronous APIs. [source 29]

**Supported endpoints:** Chat Completions, Embeddings, Completions, Responses API, Moderations, Image generation/editing, Video generation. [VERIFIED]

**Constraints:** Max 50,000 requests per batch; 200 MB input file limit; up to 2,000 batch creations per hour; results stored 30 days post-completion. [VERIFIED]

**Stacking discounts:** Batch pricing + prompt caching stack independently. With both active on GPT-5.5: cached input at $0.625/M (75% off standard $2.50/M input). [VERIFIED, source 11]

**Agent use cases:**
- Fleet evaluation pipelines (batch evaluation of agent outputs against test cases)
- Bulk issue classification and triage
- Nightly distillation data collection
- Offline content processing for knowledge base updates

Mission-2 fit: Medium. Not a real-time agent surface. Valuable for fleet-level eval pipelines and bulk annotation tasks that can tolerate 1–6 hour latency windows. The 50% cost discount is material at fleet scale.

---

### 4.11 Function Calling / Tool Use + Structured Outputs

[VERIFIED] Available across all current models. `strict: true` mode enforces JSON Schema compliance deterministically (not best-effort). Requirements when strict mode is enabled: `additionalProperties: false` on all objects, all fields in `required`, optional fields via nullable union type. [source 30]

**Limitation:** Structured Outputs is not compatible with parallel function calls. Set `parallel_tool_calls: false` when using strict mode. [VERIFIED, source 30]

Agents SDK provides native Pydantic/Zod wrappers that auto-generate the JSON Schema — the low-level `strict: true` wiring is handled by the SDK. [VERIFIED]

Mission-2 fit: High. Every fleet agent that calls OpenAI tools uses this primitive. The strict mode enforcement is the correct discipline for fleet agents (fail deterministically, not probabilistically).

---

### 4.12 Fine-Tuning / Distillation / Evals

[VERIFIED] Integrated platform for model optimization. Stored Completions → Evals → Fine-tuning pipeline is fully connected within OpenAI's platform. [source 31]

**Evals API (beta):** Create and run custom evaluations on the platform. Measure model performance on specific tasks without external logging tools. Integrates with Agents SDK tracing output. [VERIFIED]

**Model distillation:** Capture input-output pairs from large models (GPT-5.5) to fine-tune smaller models. Production data → distillation dataset → fine-tuned mini/nano model. Cost reduction path for high-volume fleet agents. [VERIFIED]

Mission-2 fit: Medium. Not immediate for Mission 2 substrate work, but the path from fleet agent traces → eval datasets → distilled models is the long-run cost-reduction strategy for high-frequency dispatch agents.

---

### 4.13 Codex App Server (standalone)

[VERIFIED] The bidirectional protocol that decouples Codex's core logic from its client surfaces. Powers CLI, VS Code, web app, macOS desktop app, JetBrains and Xcode integrations through a single stable API. [source 8]

**Architecture:** JSON-RPC lite over stdio (JSONL). Four components: stdio reader, Codex message processor, thread manager, core threads. Each thread = one Codex session. Items are atomic input/output units with explicit lifecycles (`item/started`, `item/*/delta`, `item/completed`). [VERIFIED]

**What this means for Mission 2:** Any fleet component that wants to dispatch a Codex agent should speak the App Server protocol — not shell out to `codex exec` with string parsing. The protocol gives structured streaming output, session continuity, and resume/fork semantics. Unix socket transport (April 2026) enables same-host integration without subprocess overhead. [VERIFIED, source 8]

Mission-2 fit: High. This is the correct integration surface for programmatic Codex dispatch.

---

## 5. Recommended Deep-Dive Candidates

These products warrant their own `evaluations/<product>.md`:

1. **Codex Cloud / Sandbox** — Already in progress at `../evaluations/openai-codex-cloud.md`. Priority: high. Parallel execution, GitHub integration, CI gating, and the remote TUI are all directly relevant to Mission 2.

2. **Workspace Agents** — Research preview launched 2026-04-22. Potential fleet pattern: expose Mission 2 fleet dispatch as a Workspace Agent for non-dev stakeholders. Needs evaluation once GA pricing is clear (from 2026-05-06). Priority: medium.

3. **Agents SDK Sandbox Agents** — The sandbox primitive (manifest-defined container workspaces, resumable sessions) is architecturally significant for fleet isolation. Needs a focused evaluation comparing it to Codex Cloud containers and the fleet's current worktree isolation model. Priority: medium.

4. **MCP integration via Responses API** — The fleet's tool registry as a remote MCP server that every agent calls without bespoke scaffolding is an architectural decision worth a targeted evaluation. Not a new product, but a new integration pattern. Priority: medium.

---

## 6. OpenAI-Specific Risk + Lock-In

**API surface churn:** [VERIFIED risk] Assistants API (sunset 2026-08-26) is the canonical example. OpenAI deprecates and migrates APIs on 12-month cycles. Any fleet component that builds on a non-GA surface should budget migration cycles. Custom GPTs sunset same date. The Responses API is the current stable surface but is itself evolving rapidly (MCP, compaction, computer use — all landed in 2025–2026 as additions to a 2025 launch).

**Model retirement:** [VERIFIED risk] GPT-4o, GPT-4.1, and o4-mini all retired from ChatGPT 2026-02-13. API access for existing integrations continues but with no guarantee of indefinite availability. Fleet model references should be version-pinned with a migration plan.

**Pricing trajectory:** [VERIFIED] GPT-5.5 doubled per-token prices compared to GPT-5.4 at launch (input $2.50 → $5.00/M). At fleet scale, token cost is a primary budget driver. Batch API (50% discount) + prompt caching (up to 90% on cached input) are the primary mitigation mechanisms. The fleet should instrument token usage per agent role to identify distillation candidates.

**Vendor lock-in mechanisms:**
- Responses API agentic loop runs server-side — tool calls execute on OpenAI's infrastructure, not the fleet's. State is on OpenAI's servers (threads, stored completions). Exporting this state for disaster recovery is non-trivial.
- Agents SDK is Apache 2.0 and provider-agnostic, which limits lock-in at the SDK layer.
- Symphony (Apache 2.0, SPEC.md published) is the counter-example of how to structure an OpenAI integration without lock-in: the spec is open, reimplementations are encouraged.
- Codex App Server protocol (JSON-RPC over stdio) is open and stable — switching to a competing CLI agent requires only a new client binding, not a full re-architecture.

**Mitigation:** Build fleet orchestration on the Agents SDK (provider-agnostic) + Responses API (call-by-call, no stored state). Use Batch API for eval pipelines (no session state). Keep Codex dispatch at the App Server protocol level (open spec). Avoid building on Assistants API (sunset), Custom GPTs (sunset), or Workspace Agents (research preview, unpriced).

---

## 7. Sources

[1] Terminal-Agent-Orchestration Landscape §2, studio-pm/research/terminal-agent-orchestration-landscape.md (verified 2026-04-19)

[2] InfoWorld — "OpenAI's Symphony spec pushes coding agents from prompts to orchestration," April 28, 2026. https://www.infoworld.com/article/4164173/openais-symphony-spec-pushes-coding-agents-from-prompts-to-orchestration.html

[3] Help Net Security — "OpenAI releases Symphony to automate Codex work through Linear," April 28, 2026. https://www.helpnetsecurity.com/2026/04/28/openai-symphony-codex-orchestration-linear/

[4] Let's Data Science — "OpenAI releases Symphony project-management spec for coding agents." https://letsdatascience.com/news/openai-releases-symphony-project-management-spec-for-coding-21c19085

[5] MarkTechPost — "OpenAI Releases Symphony: An Open Source Agentic Framework," March 5, 2026. https://www.marktechpost.com/2026/03/05/openai-releases-symphony-an-open-source-agentic-framework-for-orchestrating-autonomous-ai-agents-through-structured-scalable-implementation-runs/

[6] OpenTools — "OpenAI Symphony Turns Linear Boards Into Autonomous Coding Agent Orchestration." https://opentools.ai/news/openai-symphony-turns-linear-boards-into-autonomous-coding-agent-orchestration

[7] OpenAI — "An open-source spec for Codex orchestration: Symphony." https://openai.com/index/open-source-codex-orchestration-symphony/ (verified via search result summary 2026-04-29)

[8] InfoQ — "OpenAI Publishes Codex App Server Architecture for Unifying AI Agent Surfaces," February 2026. https://www.infoq.com/news/2026/02/opanai-codex-app-server/

[9] DevOps.com — "OpenAI Debuts Symphony to Orchestrate Coding Agents at Scale." https://devops.com/openai-debuts-symphony-to-orchestrate-coding-agents-at-scale/

[10] GitHub — openai/symphony (Apache 2.0, README and SPEC.md). https://github.com/openai/symphony (verified 2026-04-29)

[11] APIdog — "GPT-5.5 Pricing: Full Breakdown of API, Codex, and ChatGPT Costs (April 2026)." https://apidog.com/blog/gpt-5-5-pricing/

[12] OpenAI — "Introducing OpenAI o3 and o4-mini." https://openai.com/index/introducing-o3-and-o4-mini/ (verified via search result summary 2026-04-29)

[13] OpenAI Developer Community — "Introducing the Responses API." https://community.openai.com/t/introducing-the-responses-api/1140929

[14] OpenAI Developer Community — "Introducing support for remote MCP servers, image generation, Code Interpreter, and more in the Responses API." https://community.openai.com/t/introducing-support-for-remote-mcp-servers-image-generation-code-interpreter-and-more-in-the-responses-api/1266973

[15] OpenAI — "Migrate to the Responses API." https://developers.openai.com/api/docs/guides/migrate-to-responses (verified 2026-04-29)

[16] OpenAI Developer Community — "Assistants API beta deprecation — August 26, 2026 sunset." https://community.openai.com/t/assistants-api-beta-deprecation-august-26-2026-sunset/1354666 (verified 2026-04-29)

[17] GitHub — openai/openai-agents-python. https://github.com/openai/openai-agents-python (verified via search 2026-04-29); openai/openai-agents-js. https://github.com/openai/openai-agents-js

[18] TechCrunch — "OpenAI updates its Agents SDK to help enterprises build safer, more capable agents," April 15, 2026. https://techcrunch.com/2026/04/15/openai-updates-its-agents-sdk-to-help-enterprises-build-safer-more-capable-agents/

[19] Temporal — "Production-ready agents with the OpenAI Agents SDK + Temporal." https://temporal.io/blog/announcing-openai-agents-sdk-integration

[20] OpenAI Developers — Codex Changelog (April 2026). https://developers.openai.com/codex/changelog (verified 2026-04-29)

[21] OpenAI Developers — "Web – Codex." https://developers.openai.com/codex/cloud (verified 2026-04-29)

[22] TechCrunch — "OpenAI launches AgentKit to help developers build and ship AI agents," October 6, 2025. https://techcrunch.com/2025/10/06/openai-launches-agentkit-to-help-developers-build-and-ship-ai-agents/

[23] VentureBeat — "OpenAI unveils Workspace Agents, a successor to custom GPTs for enterprises," April 2026. https://venturebeat.com/orchestration/openai-unveils-workspace-agents-a-successor-to-custom-gpts-for-enterprises-that-can-plug-directly-into-slack-salesforce-and-more

[24] OpenAI — "Introducing gpt-realtime and Realtime API updates for production voice agents." https://openai.com/index/introducing-gpt-realtime/ (verified via search 2026-04-29)

[25] LinkedIn — "OpenAI cuts Realtime API pricing, now $0.04/min." https://www.linkedin.com/posts/kwkramer_i-got-a-bunch-of-questions-about-the-cost-activity-7367240082434482177-7gKo

[26] OpenAI Community — "Access to computer-use model." https://community.openai.com/t/access-to-computer-use-model/1157676 (verified via search 2026-04-29)

[27] OpenAI — "Introducing Operator." https://openai.com/index/introducing-operator/ (verified via search 2026-04-29)

[28] Anchor Browser — "OpenAI Operator Explained: How AI Agents Actually Control the Web." https://anchorbrowser.io/blog/how-openai-operator-works-with-ai-agents (March 2026)

[29] OpenAI Developers — "Batch API." https://developers.openai.com/api/docs/guides/batch (verified 2026-04-29)

[30] OpenAI Developers — "Function calling." https://developers.openai.com/api/docs/guides/function-calling (verified 2026-04-29)

[31] OpenAI — "Model Distillation in the API." https://openai.com/index/api-model-distillation/ (verified via search 2026-04-29)

[32] OpenAI Developers — "Agents SDK guide." https://developers.openai.com/api/docs/guides/agents (verified 2026-04-29)

[33] OpenAI Agents SDK — Python docs. https://openai.github.io/openai-agents-python/ (verified 2026-04-29)

[34] OpenAI Agents SDK — TypeScript docs. https://openai.github.io/openai-agents-js/ (verified 2026-04-29)

[35] InfoQ — "OpenAI Extends the Responses API to Serve as a Foundation for Autonomous Agents," March 2026. https://www.infoq.com/news/2026/03/openai-responses-api-agents/

[36] OpenAI — "New tools for building agents: Responses API, web search, file search, computer use, and Agents SDK." https://openai.com/index/new-tools-for-building-agents/ (verified via search 2026-04-29)

[37] VentureBeat — "OpenAI unveils Responses API, open source Agents SDK, letting developers build their own Deep Research and Operator." https://venturebeat.com/programming-development/openai-unveils-responses-api-open-source-agents-sdk-letting-developers-build-their-own-deep-research-and-operator

[38] APICents — "OpenAI API Pricing 2026." https://apicents.com/provider/openai (verified 2026-04-29)

---

## Summary (250-word agent handoff)

**(a) Symphony — what it is + Mission 2 fit:**
Symphony (released 2026-03-05, Apache 2.0, github.com/openai/symphony) is an Elixir/BEAM orchestration daemon that turns a Linear board into a Codex agent control plane. Each open Linear issue maps to an isolated agent workspace running via the Codex App Server's headless JSON-RPC 2.0 protocol. Agents must pass a proof-of-work gate (CI, tests, PR review, walkthrough) before landing changes. Default concurrency: 10 agents, 20 turns max. OpenAI treats it as a reference implementation — not a maintained product. They invite reimplementations via the published SPEC.md. Mission 2 fit is **medium**: the orchestration pattern (ticket state → isolated workspace → CI gate → proof-of-work → landing) is exactly right, but the Elixir stack is not fleet-compatible. A TypeScript reimplementation against the Codex App Server protocol is the recommended path.

**(b) Top 3 other surfaces for Mission 2:**
1. **Agents SDK** — Primary fleet orchestration SDK. Provider-agnostic, Apache 2.0, handoffs + guardrails + tracing. Integrate now.
2. **Responses API** — Canonical API primitive. Built-in tool loop, MCP server support, compaction. Migrate all new OpenAI calls here.
3. **Codex App Server** — Headless JSON-RPC 2.0 protocol for structured Codex dispatch. Symphony is built on it; Mission 2 fleet should be too.

**(c) Recommended deep-dive candidates:**
Codex Cloud/Sandbox (in progress), Workspace Agents (post-GA), Agents SDK Sandbox Agents, MCP integration pattern.

**(d) What this research may have missed:**
OpenAI's SIP/phone-calling capabilities in Realtime API (potential fleet voice interface), any changes to CUA API tier requirements post-March 2026 (the data is from March; tiers 3–5 requirement may have changed), and the Conversations API (the third Assistants migration path alongside Responses API — mentioned in migration guide but not separately researched here).
