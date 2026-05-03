---
title: "Anthropic Agent Product Spectrum — Vendor Landscape"
description: "Full mapping of Anthropic's agent-related products as candidates for Mission 2's fleet substrate. Covers 11 product surfaces from Messages API to Managed Agents."
tags: [anthropic, managed-agents, claude-code, mission-2, vendor-landscape]
modified: 2026-04-29
status: active
confidence: high
staleness: 30
related:
  - ../evaluations/anthropic-agent-skills.md
  - ../references/claude-code-config-reference.md
  - ./agentic-substrate-distribution.md
---

# Anthropic Agent Product Spectrum — Vendor Landscape

> **Research date:** 2026-04-29
> **Last verified:** 2026-04-29
> **Staleness warning:** Re-verify after 30 days. Anthropic ships 1-3 Claude Code updates per week, Managed Agents is in public beta with active API changes, and MCP spec updates ~monthly.
> **Confidence summary:** ~80% VERIFIED claims from direct docs fetch; ~15% LIKELY from community sources; ~5% PARAMETRIC.
>
> **Primary sources:**
> - Anthropic API Docs (models, pricing, APIs): https://platform.claude.com/docs (verified 2026-04-29)
> - Claude Code plugin docs: https://code.claude.com/docs (verified 2026-04-29)
> - Managed Agents quickstart: https://platform.claude.com/docs/en/managed-agents/quickstart (verified 2026-04-29)
> - MCP Architecture: https://modelcontextprotocol.io/docs/learn/architecture (verified 2026-04-29)
> - Project Glasswing / Claude Mythos: https://www.anthropic.com/glasswing + red.anthropic.com (verified 2026-04-29)
> - Pricing page: https://platform.claude.com/docs/en/about-claude/pricing (verified 2026-04-29)
> - Agent Skills overview: https://platform.claude.com/docs/en/docs/agents-and-tools/agent-skills/overview (verified 2026-04-29)
> - Terminal-agent-orchestration landscape (internal, S-19, 2026-04-19): covers Claude Code CLI invocation in depth

---

## 1. TL;DR — Top 3 Mission-2 Surfaces

Mission 2 of studio-agent-ops needs a fleet substrate: something that dispatches work to agents, manages context across sessions, observes what happened, and is extensible without baking in a single vendor. The Anthropic surface that matters most is three-layered:

**Surface 1 — Claude Managed Agents (beta, high fit)**
The closest thing Anthropic ships to a "managed fleet backend." Agents API + Sessions API + Environments API provide versioned agent configs, cloud-run containers, real-time SSE event streams, and tool sets (bash, read/write, web search, custom). This is what the fleet would run if we wanted Anthropic to manage container lifecycle. Verdict: evaluate for high-volume headless dispatches where we want to offload container management. Current gating factor: beta-only, no batch discount, Managed Agents only available via direct Claude API (not Bedrock/Vertex).

**Surface 2 — Claude Code CLI + Plugin Marketplace (GA, high fit)**
Already the fleet's primary runtime substrate. Claude Code's headless mode (`claude -p`), hooks, skills, plugin marketplace, and MCP config are the environment we version-control in studio-agent-ops. The plugin marketplace (4,200+ skills, 770+ MCP servers) is the extensibility surface. Every new capability lands here first. Verdict: canonical choice — not a candidate, already in production.

**Surface 3 — Messages API + Prompt Caching (GA, high fit as backend for lighter dispatches)**
The direct API is the model-access backbone. Prompt caching at 0.1x cache-read pricing makes multi-turn agentic loops economically tractable. The Batch API adds 50% discount for async work. This is the right surface for custom orchestration where we own the loop and don't need managed containers. Verdict: keep as the raw-inference backend for custom dispatch patterns.

---

## 2. Product Spectrum Table

| Product | Maturity | Primary Use Case | Mission-2 Fit | Deep-Dive Status |
|---|---|---|---|---|
| Claude API (Messages + Batch + Files) | GA | Model inference, batch evals, file reuse | High | n/a (well-understood) |
| Claude Code CLI + desktop + web | GA | Terminal-native agent runtime | High | Existing — claude-code-config-reference.md |
| Claude Agent SDK (Python/TS + 6 more) | GA | Programmatic agent dispatch | High | Needed |
| Claude Managed Agents | Beta | Managed cloud agent execution | High | Needed |
| MCP (Model Context Protocol) | GA (spec), active development | Context-extension protocol | High | n/a (cross-vendor standard) |
| Agent Skills | Beta (API/claude.ai) | Modular capability packages | Med | Existing — evaluations/anthropic-agent-skills.md |
| Computer Use | Beta | Desktop automation | Low | n/a |
| Plugin Marketplace | GA | Claude Code extensibility | High | n/a (covered in agentic-substrate-distribution.md) |
| Anthropic Console / Workbench | GA | Prompt testing, evals, key management | Low | n/a |
| Skill Creator + Evals | GA (2026 update) | Skill authoring + eval loop | Low-Med | n/a |
| Claude Mythos / Project Glasswing | Private preview | Defensive cybersecurity | n/a | n/a |

---

## 3. Per-Product Summaries

---

### 3.1 Claude API — Messages, Batch, Files, Prompt Caching

**What it is + maturity + auth + pricing**

The Claude API is a RESTful service at `https://api.anthropic.com` [VERIFIED, source 1]. Auth: `x-api-key` header + `anthropic-version: 2023-06-01` header required on every request. GA since 2023, continuously updated. Available on AWS Bedrock, Google Vertex AI, and Azure AI Foundry in addition to the direct API; third-party platforms have their own billing and may be 1-2 major features behind [VERIFIED, source 1].

Current models as of 2026-04-29 [VERIFIED, source 2]:
- **Claude Opus 4.7** (`claude-opus-4-7`): $5/MTok in, $25/MTok out. 1M context, 128k max output. Adaptive thinking only (extended thinking returns 400 on 4.7+). New tokenizer — up to 35% more tokens for same text.
- **Claude Sonnet 4.6** (`claude-sonnet-4-6`): $3/MTok in, $15/MTok out. 1M context, 64k max output. Extended + adaptive thinking supported.
- **Claude Haiku 4.5** (`claude-haiku-4-5`): $1/MTok in, $5/MTok out. 200k context, 64k max output. Fastest; extended thinking supported.
- Legacy available (Opus 4.6, Sonnet 4.5, Opus 4.5, Opus 4.1). Sonnet 4 and Opus 4 deprecated, retire 2026-06-15.

Key APIs [VERIFIED, source 1]:
- **Messages API** (`POST /v1/messages`, GA): single-turn + multi-turn inference, streaming via SSE, up to 32MB request. Tool use, vision, extended/adaptive thinking, prompt caching all work here.
- **Batch API** (`POST /v1/messages/batches`, GA): async processing of large request volumes, 50% discount on all models, 256MB max request, up to 300k output tokens per item with `output-300k-2026-03-24` beta header. Does not work with Managed Agents or Fast mode.
- **Files API** (`POST /v1/files`, Beta): upload-once/use-many. 500MB per file, 500GB per org. Free to upload/download/list; file content billed as input tokens when used in Messages requests. Requires `files-api-2025-04-14` beta header. Not on Bedrock or Vertex. Not ZDR-eligible [VERIFIED, source 5].
- **Token Counting API** (`POST /v1/messages/count_tokens`, GA): count tokens before sending; free.
- **Models API** (`GET /v1/models`, GA): list available models programmatically; includes `max_input_tokens`, `max_tokens`, `capabilities` object.

Prompt caching [VERIFIED, source 6]:
- Cache writes (5-min TTL): 1.25x base input. Cache writes (1-hour TTL): 2x. Cache reads: 0.1x. Compatible with ZDR. Available on Claude API and Azure AI Foundry (preview); Bedrock/Vertex coming soon.
- Break-even: one cache-read pays off a 5-min write; two reads pay off a 1-hour write.

Fast mode (beta, Opus 4.6 only): $30/MTok in, $150/MTok out (6x standard). Significantly faster output. Not available with Batch API.

Data residency: `inference_geo` parameter on Opus 4.7, 4.6+ for US-only inference; 1.1x multiplier on all token categories.

Service tiers [VERIFIED, source 7]: Standard self-service Tiers 1-4 (spend ladder: $5 → $400+/month, rates increase); Priority Tier (committed spend, 99.5% uptime target, prioritized compute, overflow falls back to standard); Enterprise SLA (99.99% uptime, custom pricing, ~$50K+/year).

**12-dim capability tag (compressed)**

| Dim | Status |
|---|---|
| Streaming | VERIFIED — SSE on Messages API |
| Tool use (custom) | VERIFIED — `tools` array, `tool_use` / `tool_result` blocks |
| Tool use (server-side) | VERIFIED — web search ($10/1k), web fetch (free), code execution ($0.05/hr after 1,550 free hr/mo) |
| Multi-modal (vision) | VERIFIED — all Claude 3+ models |
| Extended thinking | VERIFIED — Sonnet 4.6, Haiku 4.5, legacy 4.x; Opus 4.7+ uses adaptive only |
| Adaptive thinking | VERIFIED — Opus 4.7+, Sonnet 4.6, Opus 4.6 |
| Prompt caching | VERIFIED — all active models |
| Batch async | VERIFIED — 50% discount |
| Files | VERIFIED (beta) — upload/reuse |
| Context window | VERIFIED — 1M (Opus 4.7, Opus 4.6, Sonnet 4.6); 200k (Haiku 4.5) |
| Third-party deployment | VERIFIED — Bedrock, Vertex, Azure |
| Priority tier | VERIFIED — committed spend SLA |

**Mission-2 fit**

The raw API is the model-access layer for any orchestration pattern the fleet builds. Prompt caching is critical for multi-turn agent loops: 90% cost reduction on cached context makes large system prompts (rules, agents, CLAUDE.md) economically viable across many turns. Batch API is the right surface for non-interactive fleet sweeps (audits, lint passes, post-session hygiene). Files API reduces repeated re-uploads when the same reference docs are used across many sessions. The direct API is Mission-2's foundational backend — it just needs the right orchestration layer on top.

**Cross-references**

- `/use-claude-api` skill covers SDK patterns + prompt caching implementation
- See §3.4 for Managed Agents (stateful sessions on top of this API)

---

### 3.2 Claude Code — CLI, IDE, Desktop, Web

**What it is + maturity + auth + pricing**

Claude Code is Anthropic's terminal-native AI development tool, available as a CLI (`claude`), VS Code extension (2M+ installs, verified publisher), desktop app (claude.ai/download), and web interface at claude.ai/code [VERIFIED, sources 8, 9]. Auth: personal Anthropic account or API key. Pricing: included in Claude Pro ($20/mo), Max ($100-200/mo), or Team/Enterprise plans; CLI can also consume API credits directly. GA as of 2025.

Core headless mode for programmatic fleet use [VERIFIED, source 10 — terminal-agent-orchestration-landscape.md]:
- `claude -p "<prompt>"` — headless, non-interactive
- `--output-format json` or `--output-format stream-json` (NDJSON events)
- `--json-schema <schema>` — structured output enforcement
- `--session-id <uuid>` — session continuity
- `--allowedTools "Read,Edit,Bash"` — tool scoping
- `--permission-mode bypassPermissions|dontAsk`
- `--max-budget-usd`, `--max-turns` — budget/turn caps
- `--bare` — skips CLAUDE.md, hooks, skills, MCP (reproducible across machines)
- Exit codes: 0=success, non-zero=error; stderr=progress, stdout=result

Configuration layers [VERIFIED, source 3]:
- CLAUDE.md files (managed policy, project, user, local scopes; most-specific wins)
- `~/.claude/rules/` — global behavioral rules
- `~/.claude/agents/` — agent definitions
- `~/.claude/skills/` — skill directories (SKILL.md + resources)
- `~/.claude/hooks/` + `settings.json` — lifecycle hook configuration (24+ event types)
- `settings.json` — MCP config (3-scope: User/Project/Local), teammate mode, permissions

Plugin marketplace [VERIFIED, source 9]:
- Official Anthropic marketplace (`claude-plugins-official`) auto-loaded in Claude Code
- As of 2026-04-29: 4,200+ skills, 770+ MCP servers, 2,500+ marketplace entries
- Plugin types: skills, agents, hooks, MCP servers, LSP servers (code intelligence)
- Distribution: GitHub repo (`owner/repo`), Git URL, local path, remote URL
- Install scopes: user, project (team-shared via `.claude/settings.json`), local, managed (admin-set)
- Official plugins: GitHub, GitLab, Linear, Notion, Figma, Vercel, Supabase, Slack, Sentry, and 11 LSP language plugins

Teammate mode [VERIFIED, source 10]: `teammateMode: "tmux"` or `"iterm2"` in settings.json — Claude spawns subagents in split panes natively.

IDE extensions: VS Code extension (official, verified publisher, 2M+ installs). JetBrains: [PARAMETRIC — not confirmed in search results; community mentions exist but no official Anthropic JetBrains plugin found as of 2026-04-29].

**12-dim capability tag (compressed)**

| Dim | Status |
|---|---|
| Headless / programmatic | VERIFIED — `-p` flag, structured output |
| Hooks (lifecycle) | VERIFIED — 24+ event types |
| Skills system | VERIFIED — SKILL.md + 3-level progressive loading |
| MCP client | VERIFIED — 3-scope config, Plugin marketplace |
| Plugin marketplace | VERIFIED — 4,200+ skills, 770+ MCP servers |
| Session continuity | VERIFIED — `--session-id` |
| Tool control (allowedTools) | VERIFIED |
| Teammate mode (multi-pane) | VERIFIED |
| VS Code extension | VERIFIED — 2M+ installs |
| JetBrains extension | PARAMETRIC — not confirmed |
| Budget caps | VERIFIED — `--max-budget-usd` |
| LSP code intelligence | VERIFIED — 11 language plugins |

**Mission-2 fit**

Claude Code CLI is the existing fleet runtime. The plugin marketplace is the extensibility primitive: new capabilities (skills, MCP servers, LSP tools, agents) land here without requiring core changes. The hooks system (`settings.json` PostToolUse, SessionStart, SessionEnd) is the observability surface for studio-agent-ops's forensic instrumentation work. Headless `-p` mode is the dispatch primitive used by every agent-ops orchestration pattern. This is not a candidate — it is the fleet's current execution substrate.

**Cross-references**

- `studio-research/references/claude-code-config-reference.md` — comprehensive config schema (verified 2026-04-12)
- `studio-pm/research/terminal-agent-orchestration-landscape.md` §2 — headless invocation API in detail

---

### 3.3 Claude Agent SDK

**What it is + maturity + auth + pricing**

Anthropic provides official SDKs in 8 languages: Python (`anthropic`), TypeScript (`@anthropic-ai/sdk`), Java (`com.anthropic:anthropic-java`), Go (`anthropic-sdk-go`), C# (`Anthropic`), Ruby (`anthropic`), PHP (`anthropic-ai/sdk`), and a CLI tool (`ant`) [VERIFIED, source 11]. All are GA. Auth: `ANTHROPIC_API_KEY` env var or explicit init. No SDK-level pricing premium — pays standard API rates. All SDKs support direct Claude API, Bedrock, Vertex, and Azure.

All SDKs expose the same GA API surface (Messages, Batch, Token Count, Models) plus a `beta` namespace for pre-GA features. The `beta` namespace handles beta header management automatically [VERIFIED, source 11].

For Managed Agents (beta), all 8 SDKs provide `client.beta.agents`, `client.beta.sessions`, `client.beta.environments`, and `client.beta.sessions.events` namespaces with full CRUD + streaming [VERIFIED, source 12]. The beta header (`managed-agents-2026-04-01`) is set automatically by the SDK.

CLI (`ant`): installed via Homebrew (macOS) or `curl` (Linux). Supports all beta APIs with YAML syntax. Useful for scripting and verification.

GitHub repos [VERIFIED, source 11]:
- `anthropics/anthropic-sdk-python`
- `anthropics/anthropic-sdk-typescript`
- `anthropics/anthropic-sdk-java`
- `anthropics/anthropic-sdk-go`
- `anthropics/anthropic-sdk-ruby`
- `anthropics/anthropic-sdk-csharp`
- `anthropics/anthropic-sdk-php`

**12-dim capability tag (compressed)**

| Dim | Status |
|---|---|
| 8 language SDKs | VERIFIED |
| Managed Agents support | VERIFIED (beta) |
| Streaming helpers | VERIFIED — all SDKs |
| Retry + error handling | VERIFIED — built-in |
| Bedrock/Vertex/Azure | VERIFIED |
| CLI (`ant`) | VERIFIED |
| Type safety | VERIFIED — Pydantic (Python), TypeScript generics, builder pattern (Java/Go/C#) |
| Beta namespace | VERIFIED — auto-handles beta headers |

**Mission-2 fit**

The Python SDK is the standard integration point for any custom orchestration layer. The `client.beta.sessions` surface (Managed Agents SDK) is the programmatic path to Anthropic's managed containers. The CLI (`ant`) is useful for shell-level orchestration scripts and verification. The SDK is not a product in itself — it's the access layer for everything else. Mission-2 should standardize on Python (already used across the fleet) and TypeScript (for terminal-ops substrate).

**Cross-references**

- `/claude-api` skill — SDK usage patterns, caching, model migration
- §3.4 (Managed Agents) — the beta SDK namespace used for cloud agent sessions

---

### 3.4 Claude Managed Agents

**What it is + maturity + auth + pricing**

Claude Managed Agents is Anthropic's hosted agent-execution infrastructure: a set of APIs that let you define reusable agent configs (Agents API), configure container templates (Environments API), run stateful agent sessions in cloud containers (Sessions API), and stream events in real time via SSE (Events API) [VERIFIED, source 12]. Public beta as of 2026-04. Auth: same API key as Messages API. Required beta header: `managed-agents-2026-04-01` (auto-set by SDK). Available only via direct Claude API — not on Bedrock, Vertex, or Azure [VERIFIED, source 1].

Core concepts [VERIFIED, source 12]:
- **Agent**: Versioned config defining model, system prompt, tools, MCP servers, skills. Created once, referenced by ID+version in every session. Supports `agent_toolset_20260401` (full toolset) or selective tool configs.
- **Environment**: Container template — `type: cloud`, networking (`unrestricted` or allowlisted). Created once, reused. Container lifecycle managed by Anthropic.
- **Session**: A running agent instance. Receives user events, emits agent events (text, tool use, status). SSE stream for real-time output.
- **Events**: User messages, tool results, agent messages, tool-use events, `session.status_idle` (done signal), `session.status_running`.

Built-in agent toolset (`agent_toolset_20260401`) [VERIFIED, source 13]:
bash, read, write, edit, glob, grep, web_fetch (free beyond tokens), web_search ($10/1k). All enabled by default; can be individually disabled or switched to opt-in via `default_config.enabled: false`. Custom tools (user-defined, client-executed) also supported.

Pricing [VERIFIED, sources 2, 14]:
- Tokens: identical to standard Messages API rates (Opus 4.7 = $5/$25 MTok, etc.)
- Prompt caching: applies at standard multipliers (cache reads 0.1x)
- Web search inside sessions: $10/1k searches
- **Session runtime: $0.08/session-hour**, metered to millisecond, only while status is `running`. Idle time (waiting for user, tool confirmations, queue) is free.
- Batch API discount: does NOT apply to Managed Agents
- Code Execution container billing: replaced by session runtime (no double-billing)
- Worked example (1 hr, Opus 4.7, 50k in / 15k out): ~$0.705 total; with prompt caching ~$0.525.

**12-dim capability tag (compressed)**

| Dim | Status |
|---|---|
| Managed container lifecycle | VERIFIED — Anthropic-run cloud containers |
| Versioned agent configs | VERIFIED — Agents API, ID+version |
| Real-time SSE event stream | VERIFIED — Sessions Events stream |
| Built-in toolset | VERIFIED — bash, read, write, edit, glob, grep, web_fetch, web_search |
| Custom tools (client-executed) | VERIFIED — custom tool type in agent config |
| MCP server support | VERIFIED — declared in agent config |
| Skills support | VERIFIED — declared in agent config |
| Networking config | VERIFIED — unrestricted or allowlisted |
| Multi-SDK support | VERIFIED — all 8 SDKs |
| Prompt caching | VERIFIED — applies automatically |
| Priority tier | VERIFIED — available |
| Direct API only | VERIFIED — no Bedrock/Vertex |

**Mission-2 fit**

High. Managed Agents is the closest Anthropic offering to a "fleet backend" — versioned agent configs, cloud containers, streaming events, built-in tools. It eliminates the need to manage the container lifecycle ourselves for sessions that warrant cloud execution. The $0.08/session-hour runtime fee is modest; the real cost is tokens (same as raw API). The primary concern is the beta status and the Batch-API exclusion (no 50% discount on managed sessions). For the current fleet pattern (Claude Code CLI + `.context/` dispatch), Managed Agents is complementary rather than a replacement — it suits autonomous longer-running tasks where we want Anthropic to handle the container. Recommend a pilot with one concrete use case (e.g., the verifier agent) before broader adoption.

**Cross-references**

- Parallel deep-dive: `evaluations/anthropic-agent-skills.md` covers the skills surface used inside Managed Agents
- §3.3 (SDK) — all Managed Agents endpoints available via `client.beta.*`

---

### 3.5 MCP — Model Context Protocol

**What it is + maturity + auth + pricing**

MCP is an open-source, Anthropic-led protocol standard for connecting AI applications to external data sources, tools, and workflows [VERIFIED, source 15]. Published spec, open governance via modelcontextprotocol.io. Not an Anthropic product per se — it is an open standard with broad ecosystem adoption: Claude, ChatGPT, VS Code Copilot, Cursor, and many others. No pricing — it is a protocol, not a service.

Architecture [VERIFIED, source 15]:
- **MCP Host**: An AI application (Claude Code, Claude Desktop) that manages one or more MCP clients.
- **MCP Client**: Component inside the host that maintains a dedicated connection to one MCP server.
- **MCP Server**: Program that provides context (tools, resources, prompts) to clients. Can run locally or remotely.

Transport options [VERIFIED, source 15]:
- **stdio** (local only): standard input/output streams, direct process communication, no network overhead. Used for local MCP servers (e.g., filesystem server launched by Claude Desktop).
- **Streamable HTTP** (local or remote): HTTP POST for client-to-server messages, optional SSE for streaming. Supports OAuth and standard HTTP auth. This is the current standard for remote MCP servers. SSE-only transport is deprecated as of MCP spec 2025-06-18.

Protocol: JSON-RPC 2.0. Data layer primitives:
- **Tools**: Executable functions (file operations, API calls, database queries)
- **Resources**: Data sources (file contents, DB records)
- **Prompts**: Reusable interaction templates
- **Sampling** (client primitive): servers can request LLM completions from the host
- **Elicitation** (client primitive): servers can request user input
- **Tasks** (experimental): durable execution wrappers for long-running operations

Claude Code MCP config: 3-scope (User, Project, Local). Project-scope shared via `.claude/settings.json`. Plugin marketplace wraps MCP server setup as installable plugins (e.g., `linear@claude-plugins-official`) [VERIFIED, source 9].

Server registry: No single canonical registry. Community-maintained catalogs exist (claudemarketplaces.com, 770+ servers). Anthropic's official marketplace (`claude-plugins-official`) bundles pre-configured MCP servers for common services (GitHub, Linear, Notion, Figma, Supabase, Slack, Sentry, etc.) [VERIFIED, source 9].

**12-dim capability tag (compressed)**

| Dim | Status |
|---|---|
| Open standard (not vendor lock-in) | VERIFIED |
| stdio transport | VERIFIED |
| Streamable HTTP transport | VERIFIED |
| SSE transport | VERIFIED (deprecated in spec 2025-06-18) |
| OAuth support | VERIFIED (Streamable HTTP) |
| Tools / Resources / Prompts | VERIFIED |
| Sampling (server → client LLM call) | VERIFIED |
| Elicitation | VERIFIED |
| Tasks (durable execution) | VERIFIED (experimental) |
| Cross-vendor support | VERIFIED — Claude, ChatGPT, Cursor, VS Code |
| Plugin marketplace wrap | VERIFIED — claude-plugins-official |
| 3-scope Claude Code config | VERIFIED |

**Mission-2 fit**

High — but MCP is infrastructure, not a product to adopt. It is the extensibility protocol that connects Claude Code and Managed Agents to external services. The fleet's existing MCP servers (Linear, Supabase, etc.) already run on this protocol. The `claude-plugins-official` marketplace wraps the most common MCP servers into one-line installs. For Mission-2, MCP is table stakes, not a differentiator. The relevant strategic questions are: (a) which MCP servers should be fleet-standard, and (b) whether to write custom MCP servers for fleet-specific surfaces (cmux notify, events.jsonl, etc.).

---

### 3.6 Agent Skills

**What it is + maturity + auth + pricing**

Agent Skills are modular, filesystem-based capability packages that extend Claude's functionality. Each Skill is a directory containing a `SKILL.md` file (YAML frontmatter + instructions) plus optional scripts and reference materials [VERIFIED, source 4]. Skills use progressive 3-level loading: only metadata (~100 tokens) loads at startup; SKILL.md body (< 5k tokens) loads when triggered; bundled resources load on demand via bash — meaning large reference libraries have no context cost until accessed.

Maturity [VERIFIED, source 4]:
- Pre-built Agent Skills (PowerPoint, Excel, Word, PDF): GA on claude.ai + API
- Custom Skills on Claude API: Beta — requires 3 beta headers (`code-execution-2025-08-25`, `skills-2025-10-02`, `files-api-2025-04-14`)
- Custom Skills on Claude Code: GA, filesystem-based, no API upload needed
- Custom Skills on claude.ai: Beta — upload as zip; individual user only (not org-wide)

API Skills surface (`/v1/skills`, Beta): Create/manage custom skills programmatically; organization-wide sharing across workspace members.

Pricing: no skill-specific fee. Skills run inside the code execution environment; code execution costs apply when used via API (1,550 free hours/month per org, then $0.05/hr/container). On Claude Code: no container fee.

Security: Skills execute arbitrary code. Audit thoroughly before installing from untrusted sources. API Skills have no network access or runtime package installation. Claude Code Skills have full network access and local package installation.

Cross-surface limitation [VERIFIED, source 4]: Skills do NOT sync across surfaces. API Skills ≠ claude.ai Skills ≠ Claude Code Skills. Must manage separately per surface.

A separate parallel deep-dive covers this product in full: `evaluations/anthropic-agent-skills.md`.

**Mission-2 fit**

Medium. Claude Code Skills are already in production use at studio-agent-ops (the `~/.claude/skills/` directory we version-control). The API Skills surface (beta) enables programmatic skill management across an org workspace — relevant if the fleet needs org-wide capability rollouts via API rather than filesystem. The cross-surface sync limitation is the main friction point for fleet-wide skill standardization.

---

### 3.7 Computer Use

**What it is + maturity + auth + pricing**

Computer use is a beta capability that enables Claude to interact with desktop environments via screenshot capture, mouse control, and keyboard input [VERIFIED, source 16]. Beta — requires `anthropic-beta: computer-use-2025-11-24` (for Opus 4.7, 4.6, Sonnet 4.6, Opus 4.5) or `computer-use-2025-01-24` (earlier models). ZDR-eligible (screenshots not retained after response).

Supported on Opus 4.7, Opus 4.6, Sonnet 4.6, Opus 4.5 [VERIFIED, source 16]. Tool type: `computer_20251124` for newer models; `computer_20250124` for earlier.

Actions supported (latest `computer_20251124`): screenshot, left_click, type, key, mouse_move, scroll, left_click_drag, right_click, middle_click, double_click, triple_click, left_mouse_down/up, hold_key, wait, zoom (with `enable_zoom: true`).

Pricing [VERIFIED, source 14]: Standard tool use pricing. System prompt overhead: 466-499 tokens. Tool definition: 735 tokens per tool (Claude 4.x). Additional: screenshot images billed as vision tokens.

Reference implementation: `anthropics/anthropic-quickstarts/tree/main/computer-use-demo` (Docker container, X11 virtual display, agent loop) [VERIFIED, source 16].

Limitations: High latency relative to human interaction; coordinate accuracy issues; scroll and spreadsheet interactions unreliable; no account creation on social platforms; prompt injection risk from screen content.

**Mission-2 fit**

Low for current fleet architecture. Computer use requires an application managing the screenshot/action loop — the agent itself cannot run it directly. Fleet agents operate headlessly and don't need desktop interaction. Possible niche: automated browser verification of deployed UIs. Not a priority for Mission-2.

---

### 3.8 Plugin Marketplace

**What it is + maturity + auth + pricing**

The Claude Code plugin marketplace is a distribution system for extensions that add skills, agents, hooks, and MCP servers to Claude Code [VERIFIED, source 9]. Official marketplace (`claude-plugins-official`) is auto-loaded at Claude Code startup and visible at `claude.com/plugins`. No pricing — plugins are free to use; they may themselves consume API calls.

Distribution: any GitHub repo (`owner/repo`) with a `.claude-plugin/marketplace.json` file can be a marketplace. Supports Git URLs, local paths, and remote URLs. Team admins can push marketplace config via `extraKnownMarketplaces` in `.claude/settings.json`.

Scopes: user (personal, all projects), project (team-shared via `.claude/settings.json`), local (personal, current project only), managed (admin-pushed, cannot be overridden).

Plugin types [VERIFIED, source 9]:
- **Code intelligence**: LSP server integration for 11 languages (clangd, gopls, pyright, rust-analyzer, typescript-language-server, etc.)
- **External integrations**: Pre-configured MCP servers (GitHub, GitLab, Linear, Figma, Vercel, Supabase, Slack, Sentry, Atlassian, Asana, Notion, Firebase)
- **Development workflows**: Skill and agent bundles (commit-commands, pr-review-toolkit, agent-sdk-dev, plugin-dev)
- **Output styles**: Response style customization

The plugin spec (`marketplace.json`) is Anthropic's proprietary format — not MCP. Plugins bundle MCP servers as part of a richer package (skill + MCP server + hooks in one).

**Mission-2 fit**

High — directly relevant. The official marketplace is how the fleet distributes LSP tools, MCP servers, and skill bundles to all Claude Code sessions without per-user setup. Project-scope plugins in `.claude/settings.json` are the fleet-wide distribution primitive. This is documented separately in `agentic-substrate-distribution.md` §Plugin Marketplaces.

---

### 3.9 Anthropic Console / Workbench

**What it is + maturity + auth + pricing**

The Anthropic Console (`platform.claude.com`) is the web UI for API key management, usage monitoring, workspaces, and rate limit tracking. GA. Auth: Anthropic account. The Workbench tab is an interactive prompt editor with model selection, parameter sliders, and tool testing [LIKELY, sources 17, 18].

Workbench features [LIKELY, source 17]:
- Side-by-side prompt comparison
- Auto-generated test cases (via Claude)
- CSV import/export for test cases
- Prompt version history

Skills section in Console [VERIFIED, source 4]: `platform.claude.com/plugins/submit` for submitting skills to the official marketplace; skill management for org workspace.

**Mission-2 fit**

Low. The Console is for human-facing prompt iteration and key management. Agents use the API directly. Relevant only for: (a) API key provisioning and monitoring, (b) submitting official plugin/skill contributions.

---

### 3.10 Evals / Skill Creator Evals

**What it is + maturity + auth + pricing**

Anthropic's evaluation tooling is split between the Console eval tool and the updated Skill Creator [VERIFIED, sources 17, 18]. The Skill Creator (2026 update) operates in 4 modes: Create, Eval, Improve, Benchmark. Evals can run with multi-agent parallelism. Improve mode analyzes failure logs and suggests skill instruction changes automatically. Benchmark mode enables head-to-head skill comparisons.

The Anthropic Cookbook [VERIFIED, source 19] (`github.com/anthropics/claude-cookbooks`) provides executable recipe notebooks for: tool use, agent patterns, MCP integration, extended thinking, RAG, prompt caching, multimodal, production patterns.

Anthropic also maintains `github.com/anthropics/evals` — a repo of evaluation benchmarks.

**Mission-2 fit**

Low-Medium. The eval loop in Skill Creator is useful for authoring and testing fleet skills (the skills in `~/.claude/skills/`). Not relevant as infrastructure. The Cookbook is a research/reference resource, not a production surface.

---

### 3.11 Claude Mythos / Project Glasswing

**What it is + maturity + auth + pricing**

Claude Mythos Preview is a specialized model for defensive cybersecurity, offered separately under Project Glasswing [VERIFIED, source 20]. Private preview — invitation only, no self-serve sign-up. Accessible only at `red.anthropic.com`. Announced 2026-04-07.

Project Glasswing [VERIFIED, source 20]: Anthropic initiative with 12 launch partners (AWS, Apple, Broadcom, Cisco, CrowdStrike, Google, JPMorganChase, Linux Foundation, Microsoft, NVIDIA, Palo Alto Networks). Partners receive access for vulnerability discovery and remediation. Anthropic committed $100M in model usage credits to the initiative. Post-program pricing: $25/$125 per MTok (in/out), identical to legacy Opus 4.1 rate.

Capability: Mythos Preview autonomously identified and exploited a 17-year-old remote code execution vulnerability in FreeBSD (CVE-2026-4747) with no human involvement after initial request. Found thousands of zero-days across major OSes and browsers.

**Mission-2 fit**

n/a. Invitation-only, single-domain focus (cybersecurity). Not relevant to fleet substrate work.

---

## 4. Recommended Deep-Dive Candidates

Products warranting their own `evaluations/<product>.md` beyond what already exists:

1. **Claude Managed Agents** — `evaluations/anthropic-managed-agents.md` (needed). The beta API is rich and evolving. A structured evaluation should cover: session lifecycle semantics, tool-by-tool behavior, pricing model at fleet scale (vs. raw API), container environment constraints (no runtime package install), SSE stream event taxonomy, and head-to-head comparison against rolling our own loop (cost, ops burden, latency). This is the highest-value gap in the current evaluation coverage.

2. **Claude Agent SDK** — `evaluations/anthropic-agent-sdk.md` (needed, scoped). The SDK surface for Managed Agents is new (8 languages, all beta). A scoped evaluation should cover: Python + TypeScript patterns for the studio fleet, streaming event handling, error semantics, and session state management. Can be short (20-30 lines) — mostly code patterns.

3. **Agent Skills** — already in progress at `evaluations/anthropic-agent-skills.md` (parallel deep-dive).

Not needed:
- Messages API: well-understood, covered by `/claude-api` skill
- Claude Code: covered by `claude-code-config-reference.md` + terminal-agent landscape
- MCP: cross-vendor standard, no Anthropic-specific deep-dive needed
- Computer Use: low Mission-2 relevance
- Console/Workbench: not infrastructure

---

## 5. Anthropic-Specific Risk + Lock-In

**What migrating away from Anthropic costs the fleet:**

| Surface | Lock-in level | Migration cost |
|---|---|---|
| Claude Code CLI (fleet runtime) | High | Replace with Codex CLI, Gemini CLI, or custom runner. Skill format, hook events, plugin marketplace, and settings.json schema are all Anthropic-proprietary. Moderate-to-high effort. |
| Agent Skills (SKILL.md format) | Medium | SKILL.md is a de facto standard in Claude Code but not cross-agent. Migrating to a vendor-neutral format (e.g., MCP Resources) would require rewriting ~50 skills. |
| Managed Agents (Sessions/Agents/Environments API) | High | No equivalent managed offering from OpenAI or Google as of 2026-04-29. Migration = build your own container management layer. |
| MCP servers | Low | MCP is open standard; Claude Code plugins wrap MCP servers. The wrapper format is Anthropic-proprietary but the server logic is portable. |
| Claude models (via Messages API) | Low | SDK abstracts model IDs. Switching to GPT-4o or Gemini requires SDK change + prompt tuning. Comparable capability in early 2026. |
| Prompt caching patterns | Low-Med | Cache-control headers are Anthropic-specific. OpenAI has analogous caching; Gemini context caching is structurally similar. Moderate refactor. |

**Structural risk:**
The fleet's deepest lock-in is in the Claude Code plugin + skill ecosystem. Anthropic controls the marketplace, the SKILL.md format, the plugin manifest schema, and the hook event taxonomy. These are not standardized. If Anthropic changes the plugin model (e.g., removes the `/plugin` command, changes SKILL.md loading semantics), the fleet's skill library breaks without warning. Mitigation: skills in `~/.claude/skills/` are version-controlled in studio-agent-ops and mostly portable to other agents that can read markdown — the format is not cryptographically tied to Claude Code's runtime.

**Financial risk:**
Managed Agents billing introduces a new cost dimension ($0.08/session-hour) that is hard to predict for exploratory sessions. The Batch API discount (50%) doesn't apply to Managed Agents — fleet sweeps that could run async will cost 2x more if routed through Managed Agents instead of the Messages API. Monitor carefully before fleet-wide adoption.

---

## 6. Sources

| # | URL | Type | Verified |
|---|---|---|---|
| 1 | https://platform.claude.com/docs/en/api/getting-started | Official API docs | 2026-04-29 |
| 2 | https://platform.claude.com/docs/en/docs/about-claude/models/overview | Official model reference | 2026-04-29 |
| 3 | /Users/aliargun/Documents/GitHub/studio/studio-research/references/claude-code-config-reference.md | Internal research | 2026-04-12 |
| 4 | https://platform.claude.com/docs/en/docs/agents-and-tools/agent-skills/overview | Official Agent Skills docs | 2026-04-29 |
| 5 | https://platform.claude.com/docs/en/build-with-claude/files | Official Files API docs | 2026-04-29 |
| 6 | https://platform.claude.com/docs/en/build-with-claude/prompt-caching | Official prompt caching docs | 2026-04-29 |
| 7 | https://docs.anthropic.com/en/api/service-tiers | Official service tiers | 2026-04-29 |
| 8 | https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code | VS Code Marketplace | 2026-04-29 |
| 9 | https://code.claude.com/docs/en/discover-plugins | Official plugin marketplace docs | 2026-04-29 |
| 10 | /Users/aliargun/Documents/GitHub/studio/studio-pm/research/terminal-agent-orchestration-landscape.md | Internal research (S-19) | 2026-04-19 |
| 11 | https://platform.claude.com/docs/en/api/client-sdks | Official SDK docs | 2026-04-29 |
| 12 | https://platform.claude.com/docs/en/managed-agents/quickstart | Official Managed Agents quickstart | 2026-04-29 |
| 13 | https://platform.claude.com/docs/en/managed-agents/tools | Official Managed Agents tools | 2026-04-29 |
| 14 | https://platform.claude.com/docs/en/about-claude/pricing | Official pricing | 2026-04-29 |
| 15 | https://modelcontextprotocol.io/docs/learn/architecture | Official MCP architecture | 2026-04-29 |
| 16 | https://platform.claude.com/docs/en/docs/build-with-claude/computer-use | Official Computer Use docs | 2026-04-29 |
| 17 | https://www.anthropic.com/news/evaluate-prompts | Anthropic blog (evals) | 2026-04-29 |
| 18 | https://tessl.io/blog/anthropic-brings-evals-to-skill-creator-heres-why-thats-a-big-deal/ | Community analysis | 2026-04-29 |
| 19 | https://github.com/anthropics/claude-cookbooks | Official GitHub | 2026-04-29 |
| 20 | https://www.anthropic.com/glasswing | Official Glasswing page | 2026-04-29 |
| 21 | https://red.anthropic.com/2026/mythos-preview/ | Official Mythos preview | 2026-04-29 |
| 22 | https://fortune.com/2026/04/07/anthropic-claude-mythos-model-project-glasswing-cybersecurity/ | Fortune (Glasswing launch) | 2026-04-29 |
| 23 | https://wavespeed.ai/blog/posts/claude-managed-agents-pricing-2026/ | Community (Managed Agents pricing) | 2026-04-29 |
| 24 | https://claudemarketplaces.com/ | Community plugin directory | 2026-04-29 |
| 25 | https://github.com/anthropics/claude-plugins-official | Official plugin repo | 2026-04-29 |
| 26 | /Users/aliargun/Documents/GitHub/studio/studio-pm/research/terminal-ops-cao-evaluation.md | Internal research (S-16) | 2026-04-21 |
| 27 | /Users/aliargun/Documents/GitHub/studio/studio-research/landscapes/agentic-substrate-distribution.md | Internal research (S-25) | 2026-04-27 |
| 28 | https://platform.claude.com/docs/en/build-with-claude/extended-thinking | Official thinking docs | 2026-04-29 |
