---
title: "OpenAI Codex Cloud — Capability Profile"
description: "Cloud-managed Codex agent execution environment from OpenAI. Capability profile across 12 dimensions for Mission 2 BackendAdapter evaluation: spawn, send, read, status, kill, list, persistence, auth, pricing, reach, visibility, portability."
research_date: 2026-04-29
last_verified: 2026-04-29
staleness_warning: "Re-verify after 30 days. Codex Cloud is in GA with active monthly releases; pricing model changed on 2026-04-02 (token-based); usage limits revised 2026-04-09. Model lineup (GPT-5.x series) evolves frequently. Any claim about credit rates, quota windows, or CLI flags may be stale within weeks."
confidence: medium
sources_count: 28
related:
  - ../landscapes/agentic-fleet-platforms.md (forthcoming)
  - ../landscapes/agentic-substrate-distribution.md
status: active
---

# OpenAI Codex Cloud — Capability Profile

> **Research date:** 2026-04-29
> **Last verified:** 2026-04-29
> **Staleness warning:** Re-verify after 30 days. Codex Cloud's pricing, credit model, and CLI flags are updated monthly. The April 2 and April 9, 2026 pricing changes substantially altered quota behavior. GitHub-dependency outages and cloud rate-limit anomalies are documented recurring issues.
> **Confidence summary:** ~40 VERIFIED claims (primary OpenAI docs fetched 2026-04-29, GitHub issues, OpenAI status page), ~12 LIKELY claims (from changelog entries and community posts), 6 UNVERIFIED claims (where primary-source access was blocked or not found), 0 PARAMETRIC claims — every claim traces to a dated source.
>
> **Primary sources (top 8 by load-bearing weight):**
> - Codex Cloud overview: https://developers.openai.com/codex/cloud (2026-04-29)
> - Codex CLI reference: https://developers.openai.com/codex/cli/reference (2026-04-29)
> - Codex cloud environments: https://developers.openai.com/codex/cloud/environments (2026-04-29)
> - Codex pricing: https://developers.openai.com/codex/pricing (2026-04-29)
> - Codex SDK: https://developers.openai.com/codex/sdk (2026-04-29)
> - Agents SDK guide: https://developers.openai.com/codex/guides/agents-sdk (2026-04-29)
> - Codex changelog: https://developers.openai.com/codex/changelog (2026-04-29)
> - Config reference: https://developers.openai.com/codex/config-reference (2026-04-29)

---

## 1. TL;DR

OpenAI Codex Cloud is a managed cloud coding-agent runtime that launched in research preview May 16, 2025 and reached broad availability (ChatGPT Plus+) by mid-2025. [VERIFIED, src 1, 3] It runs isolated per-task containers on OpenAI infrastructure, tightly coupled to GitHub for repository access. [VERIFIED, src 4, 5]

Codex Cloud is **not a REST API you call from your own orchestrator**. There is no `POST /v1/tasks` HTTP endpoint exposed to arbitrary callers. Instead, access surface is: (a) a ChatGPT web UI at `chatgpt.com/codex`, (b) a CLI subcommand (`codex cloud exec`), (c) a TypeScript SDK (`@openai/codex-sdk`), and (d) a Python SDK communicating with a local app-server via JSON-RPC. [VERIFIED, src 6, 7, 8]

For Mission 2's `BackendAdapter` abstraction, Codex Cloud is a **medium-impedance fit**: the six primitives (`spawn`, `send`, `read`, `status`, `kill`, `list`) all have some analogue, but none map to a clean REST endpoint. The CLI's `codex cloud exec` and `codex cloud list --json` give the closest scriptable interface. The MCP server mode (`codex mcp-server`) gives an agent-to-agent integration path via the OpenAI Agents SDK. The strategic blockers are: GitHub hard-dependency (no private-repo or non-GitHub version control support), opaque cloud-side container access, and a recurring pattern of credit metering anomalies and silent failures in the GitHub integration.

**BackendAdapter verdict:** Viable secondary cloud backend. Not a first-class peer of a REST-API cloud backend like Anthropic Managed Agents. Viable for workloads that are GitHub-hosted and tolerant of CLI-mediated dispatch. Block-on-GitHub availability is a documented production risk [VERIFIED, src 16].

---

## 2. Product Identity & Maturity

### What is it?

There are two distinct but related Codex products. The naming causes confusion:

| Product | What it is | Status |
|---|---|---|
| **Codex CLI** (`github.com/openai/codex`) | Open-source local coding agent. Runs on your machine via subprocess or headless `codex exec`. | GA, open-source, MIT license |
| **Codex Cloud / Codex Web** | Managed cloud agent; tasks run in OpenAI-hosted containers preloaded with your GitHub repo. Accessible via `chatgpt.com/codex`, CLI `codex cloud`, and SDK. | GA (Plus+); some features still preview |

[VERIFIED, src 1, 2] This evaluation focuses on **Codex Cloud / Codex Web** — the managed cloud runtime — which is the distinct surface the existing research at `terminal-agent-orchestration-landscape.md §2` identified as "Codex Cloud (experimental)." That description is now partially stale: Codex Cloud is no longer experimental for core task execution, though some features (Codex Security, some multi-agent primitives) remain in beta or research preview.

The CLI and cloud surfaces are the same binary. `codex cloud exec` is a CLI subcommand that submits tasks to the cloud, `codex exec` runs locally. They share config and auth.

### Timeline

| Date | Event |
|---|---|
| May 16, 2025 | Research preview of Codex Cloud announced, powered by codex-1 model [VERIFIED, src 3] |
| June 2025 | ChatGPT Plus users gain access [VERIFIED, src 3] |
| February 2, 2026 | Codex desktop app launched (macOS) with multi-agent orchestration UI [VERIFIED, src 17] |
| February 5, 2026 | GPT-5.3-Codex released [VERIFIED, src 3] |
| February 9, 2026 | GitHub dependency outage takes down Codex Cloud [VERIFIED, src 16] |
| March 2026 | Codex Security (vulnerability patching) enters beta [VERIFIED, src 3] |
| March 5, 2026 | GPT-5.4 for Codex released [VERIFIED, src 3] |
| April 2, 2026 | Pricing model changes from message-based to token/reasoning-time-based [VERIFIED, src 9] |
| April 9, 2026 | Credit limit system revised; community reports 90% productivity drop on Business tier [VERIFIED, src 18] |
| April 2026 | Windows sandbox proxy networking, dynamic bearer tokens, prompt-plus-stdin added [VERIFIED, src 2] |

### Model lineup (as of April 2026)

[VERIFIED, src 10]

| Model | Available in cloud | Notes |
|---|---|---|
| `gpt-5.5` | Yes | Requires ChatGPT auth (not API-key auth) |
| `gpt-5.4` | Yes | Flagship; API-key usable |
| `gpt-5.4-mini` | Yes | Fast/cheap, good for subagents |
| `gpt-5.3-codex` | Yes | Strong coder |
| `gpt-5.3-codex-spark` | ChatGPT Pro only | Research preview, text-only |

---

## 3. The 12 Capability Dimensions

### 3.1 Spawn Semantics

[VERIFIED, src 1, 6, 8]

Codex Cloud tasks are spawned through four surfaces, none of which is a raw HTTP API endpoint exposed to arbitrary callers:

**Surface A — Web UI:**
Navigate to `chatgpt.com/codex`. Connect GitHub. Type prompt. Codex spins up an isolated container preloaded with your repository and begins work.

**Surface B — CLI (`codex cloud exec`):**
```bash
codex cloud exec --env ENV_ID --attempts 1 "Fix the failing unit test in auth module"
```
- `--env ENV_ID`: required; identifies the cloud environment (repository + config bundle)
- `--attempts 1-4`: best-of-N executions OpenAI runs in parallel (best result returned)
- Prompt can be piped via stdin
- Exits non-zero on submission failure — CI-safe

**Surface C — TypeScript SDK:**
```typescript
import { Codex } from '@openai/codex-sdk';
const codex = new Codex();
const thread = await codex.startThread();
const result = await thread.run("Fix the failing unit test in auth module");
// Continue: await thread.run("Now add a test for the edge case");
// Or resume: await codex.resumeThread(threadId).run("follow-up");
```
[VERIFIED, src 7] Threads are persistent; `run()` is sequential on a thread. Parallel threads require multiple `startThread()` calls.

**Surface D — MCP server → OpenAI Agents SDK:**
```bash
codex mcp-server   # starts Codex as a local MCP stdio server
```
Exposed MCP tools: `codex` (initiate session) and `codex-reply` (continue thread by `threadId`). The Agents SDK orchestrates agents that call these tools. [VERIFIED, src 8]

**Key finding:** There is no `POST /v1/codex/cloud/tasks` HTTP endpoint documented in the official developer docs. The `beta /v1/codex/cloud/tasks` claim found in one third-party search result [UNVERIFIED] could not be confirmed against primary OpenAI docs. The canonical programmatic path is CLI or SDK.

**Multi-agent spawn config:**
```toml
[agents]
max_threads = 6          # concurrent agent threads (default: 6)
max_depth = 1            # nesting depth (root = 0, default allows one spawned layer)
job_max_runtime_seconds = 1800  # per-worker timeout (default 30min)
[features]
multi_agent = true       # enables spawn_agent, send_input, resume_agent, wait_agent, close_agent tools
```
[VERIFIED, src 11]

### 3.2 Send Semantics

[VERIFIED, src 7, 8]

**Single-shot (cloud default):** A single prompt is submitted; the cloud agent works to completion. This is the dominant interaction model for background tasks.

**Conversational (thread-based):** Both the TypeScript and Python SDKs support multi-turn via thread IDs. `thread.run(prompt)` sends the next message to an existing thread. The MCP server exposes `codex-reply` for the same purpose. [VERIFIED, src 7, 8]

**Cloud-side state:** Thread state, tool results, and environment state are maintained cloud-side between turns on the same thread. The config's `agents.max_threads = 6` limits concurrent threads per session. [VERIFIED, src 11]

**Stdin support:** `codex cloud exec` accepts prompt via `-` (stdin). Example: `echo "Fix tests" | codex cloud exec --env ENV_ID -` [VERIFIED, src 12]

### 3.3 Read / Output Semantics

[VERIFIED, src 6, 12, 13]

**CLI local mode (`codex exec --json`):**
Produces newline-delimited JSON (NDJSON) events on stdout — one event per state change. This is the structured output path for local (non-cloud) runs. Events include: type, content, tool calls, tool results, cost, completion signal.

**Cloud mode output:**
- Results are applied as diffs/patches in the GitHub PR or diff review UI.
- `codex cloud list --json` returns task status as a JSON payload (see §3.5 below for schema).
- There is no documented streaming NDJSON endpoint for cloud task events from the outside. Cloud tasks are fire-and-forget from the CLI perspective; you check status via `list`.

**SDK output:**
`thread.run()` returns the assistant's final message as a string (TypeScript SDK). Python SDK via JSON-RPC over local app-server returns structured response objects.

**Result delivery:**
For GitHub-connected tasks, Codex opens a PR or proposes a diff reviewable at the task URL. For CLI-launched tasks, the diff can be applied locally via the interactive picker.

**Key gap:** No cloud-side SSE/streaming event feed analogous to Anthropic Managed Agents' `GET /v1/sessions/{id}/events`. Cloud task progress is opaque until completion. [VERIFIED by absence in primary docs; src 1, 6]

### 3.4 Status / Kill / List

[VERIFIED, src 12]

**List:**
```bash
codex cloud list [--env ENV_ID] [--json] [--limit 1-20] [--cursor CURSOR]
```
JSON output schema per task:
```json
{
  "id": "task_...",
  "url": "https://chatgpt.com/codex/task/...",
  "title": "Fix the failing unit test",
  "status": "running|completed|failed|cancelled",
  "updated_at": "ISO-8601",
  "environment_id": "env_...",
  "environment_label": "my-repo/main",
  "summary": "...",
  "is_review": false,
  "attempt_total": 1
}
```
Supports pagination via `cursor`. Machine-readable; CI-safe.

**Status:**
No dedicated `codex cloud status <task_id>` command found in primary docs. [UNVERIFIED] Status is readable via `codex cloud list --json` (filter client-side by `id`) or via the web dashboard. The `--json` output includes `status` field and `updated_at`, sufficient for polling.

**Kill:**
No documented `codex cloud cancel <task_id>` CLI command. [UNVERIFIED] Cancellation may be available via the web dashboard only. This is a BackendAdapter gap: no programmatic kill path confirmed.

**Interactive picker:**
Default `codex cloud` (no subcommand) opens an interactive TUI picker showing active/finished tasks, with Ctrl+O to switch environments.

### 3.5 Persistence Model

[VERIFIED, src 4, 5]

**Per-task ephemeral containers with caching:**
- Each task runs in its own isolated cloud container
- Containers are cached for **up to 12 hours** to speed up follow-up tasks and new submissions in the same environment
- Cache is invalidated when: setup scripts change, environment variables change, or secrets change
- Business/Enterprise workspaces share cache state across workspace members

**Thread persistence (SDK-level):**
Thread IDs persist across SDK calls, enabling multi-turn conversations. `codex.resumeThread(threadId)` reconnects. Thread duration limits are not published. [LIKELY, based on `job_max_runtime_seconds = 1800` (30min) default in config, src 11]

**Session files (local CLI):**
Local runs write session files to `~/.codex/sessions/` with run IDs. `codex exec resume [SESSION_ID]` continues local sessions. This is orthogonal to cloud threads.

**Contrast with persistent agents:** Codex Cloud is task/thread-oriented, not a "persistent agent session" that stays running between interactions. Each `codex cloud exec` creates a new container (or hits cache). This differs from a daemon model.

### 3.6 Reach

[VERIFIED, src 1, 4, 5]

**Cloud vs local boundary:**
Codex Cloud is cloud-only infrastructure. The CLI's `codex exec` subcommand (local) is distinct: it runs the agent locally, against your filesystem, with a local sandbox. `codex cloud exec` submits to OpenAI-managed containers.

**Remote TUI mode:**
A hybrid pattern exists: `codex app-server --listen ws://ADDRESS:PORT` starts the Codex app-server on one machine; `codex --remote ws://host:port` connects the TUI from another machine. This lets you run the agent on a remote host (where code + credentials live) while operating from a different terminal. [VERIFIED, src 13] This is closer to a "local-but-remote" pattern than true cloud-managed execution.

**GitHub hard-dependency:**
Codex Cloud requires a connected GitHub account to access repositories. There is no documented support for GitLab, Bitbucket, or non-Git VCS. [VERIFIED, src 1, 5] Private repos with data-residency requirements or non-GitHub hosting cannot use Codex Cloud. This is the most significant reach constraint.

**Bedrock / Vertex:**
Amazon Bedrock now offers Codex as part of its managed-agents preview (announced April 2026 [VERIFIED, src 19]), but this is the Bedrock Managed Agents product wrapping OpenAI models — distinct from Codex Cloud. For BackendAdapter purposes, treat these as separate backends.

### 3.7 Visibility

[VERIFIED, src 14, 15]

**Web dashboard:**
`chatgpt.com/codex` shows all cloud tasks with status, diffs, PR links, and task history. No programmatic read of per-task execution traces from outside the UI.

**CLI:**
`codex cloud list --json` exposes the task list. `codex cloud` (interactive) shows a TUI with task details.

**OpenTelemetry (local runs only):**
```toml
[otel]
environment = "staging"
exporter = { otlp-http = {
  endpoint = "https://otel.example.com/v1/logs",
  headers = { "x-otlp-api-key" = "${OTLP_TOKEN}" }
}}
log_user_prompt = false   # prompts redacted by default
```
[VERIFIED, src 15] OTel captures: `codex.conversation_starts`, `codex.api_request`, `codex.sse_event`, `codex.user_prompt` (length only, content redacted), `codex.tool_decision`, `codex.tool_result`. Supported exporters: `otlp-http`, `otlp-grpc`. Disabled by default. Works with SigNoz, Grafana Cloud (prebuilt dashboard available [VERIFIED, src 15]).

**Key gap:** OTel is documented for **local CLI runs only**. There is no documented OTel export path for cloud-task execution. Cloud tasks are visible in the OpenAI Traces dashboard (for Agents SDK runs [VERIFIED, src 8]) but this requires using the Agents SDK integration pattern, not the CLI. [LIKELY]

**Audit logs:**
Enterprise/Edu tiers include audit log access per the pricing page. [VERIFIED, src 9] Specifics (what events, retention period, API or UI only) not found in primary docs. [UNVERIFIED]

### 3.8 Authentication

[VERIFIED, src 1, 9, 12]

**For ChatGPT UI and CLI:**
Standard ChatGPT account authentication. `codex login` initiates OAuth device-code flow. `CODEX_API_KEY` env var available for CI headless use.

**GitHub OAuth:**
A GitHub account connection is required for Codex Cloud repository access. This is a separate OAuth grant from the ChatGPT account. Without it, Codex Cloud cannot load repositories.

**For SDK (`@openai/codex-sdk`):**
The TypeScript SDK wraps local app-server communication (Node.js 18+, server-side). Uses the same credentials as the CLI.

**For API-key access (Business/Enterprise):**
`OPENAI_API_KEY` or project-scoped tokens. `gpt-5.5` is unavailable with API-key auth (requires ChatGPT auth). [VERIFIED, src 10]

**Enterprise:**
SSO/MFA available on Business tier. SCIM, EKM (encryption key management), and RBAC on Enterprise/Edu. [VERIFIED, src 9]

**MCP OAuth:**
When Codex acts as MCP server calling external MCP tools, OAuth credentials are stored per `mcp_oauth_credentials_store` config (file or OS keyring). Dynamic bearer token support added April 2026. [VERIFIED, src 2]

**Secrets in cloud environments:**
"Secrets configured for cloud environments are available only during setup and are removed before the agent phase starts." [VERIFIED, src 5] No secret leakage into the active agent phase.

### 3.9 Pricing / Quota

[VERIFIED, src 9, 18]

**Subscription tiers (individual):**

| Plan | Monthly | Notes |
|---|---|---|
| Free | $0 | Basic exploration only |
| Go | $8 | Lightweight tasks |
| Plus | $20 | Several coding sessions/week; cloud integration included |
| Pro | $100 | 5x or 20x higher limits; 2x multiplier until May 31, 2026 |
| API Key | Pay-as-you-go | Token-based, no usage cap |

**Business:** Pay-as-you-go with standard or usage-based seats, larger VMs, SSO/MFA.
**Enterprise/Edu:** Contact sales; audit logs, data residency, priority processing.

**Usage quota model (5-hour rolling window):**

| Plan | GPT-5.5 | GPT-5.4 | GPT-5.4-mini | GPT-5.3-Codex |
|---|---|---|---|---|
| Plus | 15–80 tasks | 20–100 | 60–350 | 30–150 |
| Pro 5x | 80–400 | 100–500 | 300–1750 | 150–750 |
| Pro 20x | 300–1600 | 400–2000 | 1200–7000 | 600–3000 |

*Ranges reflect variation in task complexity (reasoning time consumed).*

**Token-based credit rates (Business and new Enterprise — credits per 1M tokens):**

| Model | Input | Cached Input | Output |
|---|---|---|---|
| GPT-5.5 | 125 | 12.50 | 750 |
| GPT-5.4 | 62.50 | 6.25 | 375 |
| GPT-5.4-mini | 18.75 | 1.875 | 113 |
| GPT-5.3-Codex | 43.75 | 4.375 | 350 |

**Per-task cost examples (message-based estimate, Plus/Pro/legacy Enterprise):**
- GPT-5.3-Codex local task: ~5 credits
- GPT-5.3-Codex **cloud task**: ~25 credits (5x more than local)
- GitHub code review: ~25 credits per PR

**April 2026 pricing change impact:**
On April 9, 2026, the limit system shifted to reasoning-time-based billing. Community reports document: Business plan users on GPT-5.4 consuming ~8% of quota per minute (vs ~2.5% on Plus), yielding only ~12.5 minutes of reasoning in a 5-hour window. [VERIFIED, src 18] One community member reported a single basic prompt consuming ~$0.88 at Business rates. Multiple users cancelled citing a reported 90% effective productivity drop vs pre-April system.

**Key finding for BackendAdapter:** Cloud tasks cost 5x more credits than local tasks for the same model. For a fleet that dispatches many short tasks, this cost ratio is significant. The reasoning-time billing model means unpredictable per-task costs — longer reasoning = proportionally higher credit drain.

### 3.10 Maturity / Known Limitations

[VERIFIED, src 16, 17, 18, 20, 21, 22, 23, 24]

**GitHub hard-dependency — documented production risk:**
On February 9, 2026, a GitHub outage took down Codex Cloud. [VERIFIED, src 16] Codex Cloud cannot operate if GitHub is unavailable or if the user's repository is not on GitHub. Expert commentary noted: "AI coding agents have become central to professional development workflows... when GitHub is down, these agents lose the ability to complete their end-to-end workflows." [VERIFIED, src 16]

**Credit metering anomalies (recurring, documented in GitHub issues):**
- Quota dash shows available credits, but code review reports limit reached — a UI/backend consistency bug [VERIFIED, src 20]
- Sudden drops of weekly limits to zero without explanation [VERIFIED, src 21]
- Business plan quota drains dramatically faster than Plus for equivalent workloads [VERIFIED, src 18]
- Improper 429 errors with 61% quota remaining [VERIFIED, src 22]
- Very small tasks (one-line config change) consuming ~2% of 5-hour budget [VERIFIED, src 22]
- "Burning tokens very fast" — Issue #14593; "Usage dropping too quickly" — Issue #13568 [VERIFIED, src 23, 24]

**GitHub auto-code-review silent failures:**
Auto-code-review configured correctly but PR receives zero comments, zero error messages. No indication of failure; only resolved by disconnecting and reconnecting GitHub connector. [VERIFIED, src 20]

**Compaction failures:**
Chat compaction fails 100% of the time when the service is under high demand, with no graceful degradation. [VERIFIED, src 24 — issue #19009]

**Security issue (patched March 2026):**
Malicious GitHub branch names could inject commands during task setup and retrieve GitHub authentication tokens. Now patched. [VERIFIED, src 3, SecurityWeek citation]

**Context windows:**
Not documented publicly in official model docs. [UNVERIFIED — likely inherits from underlying GPT-5.x models but no cloud-specific context limit published]

**macOS-only desktop app:**
The Codex desktop app (February 2026) is macOS only. [VERIFIED, src 3, 17]

**gpt-5.5 requires ChatGPT auth:**
Not accessible via API key. Automated/CI use cases are limited to `gpt-5.4` and below. [VERIFIED, src 10]

**No kill/cancel API:**
No programmatic task cancellation found in CLI or SDK docs. [VERIFIED by absence]

**Concurrency limit:**
`max_threads = 6` (default config value). [VERIFIED, src 11] Whether this is enforced cloud-side or only in local multi-agent config is not clearly documented.

### 3.11 Vendor Lock-in / Portability

[VERIFIED, src 5, 11, 25, 26]

**GitHub lock-in:**
Codex Cloud requires GitHub. No GitLab, Bitbucket, local git, or non-GitHub alternatives are documented. This is the stiffest form of vendor coupling: it couples you to a third-party (GitHub) in addition to OpenAI.

**AGENTS.md — partial portability mechanism:**
AGENTS.md is a markdown file placed at repo root (or in `~/.codex/` for global scope) that gives Codex persistent per-project instructions. [VERIFIED, src 25] OpenAI references `agents.md` as a site implying a cross-vendor standard, but as of this research date, no concrete multi-vendor adoption of the AGENTS.md format has been confirmed. Claude Code uses `CLAUDE.md`; Gemini CLI uses `.gemini/settings.json`. No documented interoperability. [UNVERIFIED — treat AGENTS.md portability as aspirational, not realized]

**Open-source CLI — escape hatch:**
The Codex CLI is MIT-licensed and open-source (`github.com/openai/codex`). This means the local execution path is fully portable. The lock-in is specifically in the cloud/managed sandbox surface. [VERIFIED, src 26]

**MCP as integration interface:**
`codex mcp-server` exposes Codex as an MCP stdio server. MCP (Model Context Protocol) is an open standard, meaning the integration surface is not OpenAI-proprietary. Any MCP-compatible orchestrator can call Codex as a tool. [VERIFIED, src 8] This is the highest-portability integration path.

**Webhook formats:**
No webhook or callback URL mechanism found for cloud task completion. Cloud tasks are polled via `codex cloud list --json` or monitored via the web dashboard. [UNVERIFIED — may exist for Enterprise tier; not in public docs]

**Data residency:**
Enterprise/Edu tiers include data residency controls. [VERIFIED, src 9] Specifics (which regions, what data) not published. [UNVERIFIED]

**Sandbox runtime portability:**
Cloud containers use OpenAI's universal image (`openai/codex-universal` — open-sourced, pullable and testable locally [VERIFIED, src 4]). This means: (a) local and cloud containers are identical in base image, (b) you can reproduce the cloud environment locally for testing. This is a meaningful portability affordance absent in fully proprietary managed agents.

**Amazon Bedrock path:**
Amazon Bedrock Managed Agents (limited preview, April 2026) runs OpenAI models + agent harness on AWS. [VERIFIED, src 19] This is a different product surface — if portability to AWS is required, the Bedrock path may provide it. Not the same as Codex Cloud.

---

## 4. BackendAdapter Fit Assessment

### Primitive mapping

| Mission 2 primitive | Codex Cloud analogue | Gap |
|---|---|---|
| `spawn(command, cwd, env, metadata)` | `codex cloud exec --env ENV_ID PROMPT` | ENV_ID must be pre-provisioned; no dynamic container config at spawn time |
| `send(session_id, text)` | `thread.run(prompt)` (SDK) or `codex-reply` (MCP) | Thread IDs required; CLI has no `send` to existing task |
| `read(session_id, events)` | No streaming cloud event API; `codex cloud list --json` for status only | No per-event streaming from cloud side |
| `status(session_id)` | `codex cloud list --json` + filter by id | Polling required; no push notification |
| `kill(session_id)` | No documented programmatic cancel | Full gap |
| `list()` | `codex cloud list --json` | Available; paginated; CI-safe |

**Fit score: 3/6 clean primitives.** `send`, `read`, and `kill` have meaningful gaps. `spawn`, `status`, `list` work but with friction (ENV_ID pre-provisioning, polling-only status).

### Integration path recommendation

For Mission 2, the lowest-friction Codex Cloud integration pattern is:

1. **Use `codex mcp-server` as the integration surface** — not the CLI's cloud commands. This gives a stable MCP protocol interface (open standard, type-safe) vs fragile CLI flag parsing.
2. **Wrap the MCP `codex` and `codex-reply` tools** behind the BackendAdapter interface. `threadId` returned by `codex` maps to the substrate's `session_id`. `codex-reply` maps to `send()`.
3. **For `list()` and `status()`**: fall back to `codex cloud list --json` via subprocess until MCP exposes a listing tool (not currently documented).
4. **For `kill()`**: accept as unimplemented; document as a known gap. Raise a follow-up issue.

This MCP-mediated path is lower coupling than CLI subprocess parsing and is closer to the substrate's agent-as-service model.

### Architecture position

```
┌─────────────────────────────────────────────────────┐
│  terminal-ops / substrate BackendAdapter            │
└──────────────┬──────────────────────────────────────┘
               │ MCP stdio (preferred) | CLI subprocess (fallback)
               ▼
┌─────────────────────────────────────────────────────┐
│  codex mcp-server  (local process, OpenAI Agents SDK │
│  or substrate MCP client connects here)              │
└──────────────┬──────────────────────────────────────┘
               │ HTTPS + GitHub OAuth
               ▼
┌─────────────────────────────────────────────────────┐
│  OpenAI Cloud Container                              │
│  - Ephemeral, 12hr cache                            │
│  - GitHub repo preloaded                            │
│  - Universal image (openai/codex-universal)         │
└─────────────────────────────────────────────────────┘
```

### Risk register

| Risk | Severity | Mitigation |
|---|---|---|
| GitHub outage takes down Codex Cloud | High | Local fallback via `codex exec` (no GitHub needed); design BackendAdapter to degrade gracefully |
| Credit metering anomalies — unexpected quota drain | Medium | Monitor via `codex cloud list`; set `--limit 1` and inspect `attempt_total` |
| No programmatic kill | Medium | Design task timeouts at BackendAdapter layer; use `job_max_runtime_seconds` config |
| Business tier 3x faster quota drain than Plus | Medium | Document per-tier cost model; use `gpt-5.4-mini` for fast/cheap subagent tasks |
| gpt-5.5 unavailable with API key | Low | Use gpt-5.4 for automated tasks |
| Non-GitHub repos unsupported | Blocker if applicable | Verify all target repos are on GitHub before adopting Codex Cloud as a backend |

---

## 5. Open Questions

1. **Is there an undocumented REST API for cloud task submission?** The `beta /v1/codex/cloud/tasks` endpoint appeared in a third-party search result but could not be confirmed against primary OpenAI docs. Worth a direct check against the OpenAI Platform API reference.

2. **What are cloud container concurrency limits?** The `max_threads = 6` config appears in local multi-agent config. Whether this is enforced cloud-side, or what the actual per-account cloud concurrency limit is, is not documented.

3. **What is the task retention period?** `codex cloud list` shows recent tasks, but how far back "recent" goes is not specified. Relevant for audit and replay.

4. **Is AGENTS.md actually cross-vendor?** The reference to `agents.md` as a site implies OpenAI is positioning this as a standard. Whether Claude Code, Gemini CLI, or other agents will adopt it is unknown. Worth a dedicated research spike.

5. **Enterprise webhook / event push?** Is there an enterprise-tier webhook for cloud task completion? No public docs found. Directly affects how the BackendAdapter should poll vs subscribe.

6. **Bedrock Managed Agents (OpenAI-powered) — separate evaluation?** AWS Bedrock now wraps OpenAI models in its managed agent surface. This may provide cloud dispatch without the GitHub hard-dependency. Worth a separate evaluation if GitHub lock-in is a firm blocker.

---

## 6. Sources

| # | URL | Date Verified | Used for |
|---|---|---|---|
| 1 | https://developers.openai.com/codex/cloud | 2026-04-29 | Product identity, spawn, reach |
| 2 | https://developers.openai.com/codex/changelog | 2026-04-29 | Timeline, recent features |
| 3 | https://en.wikipedia.org/wiki/OpenAI_Codex_(AI_agent) | 2026-04-29 | Timeline, maturity, security issue |
| 4 | https://developers.openai.com/codex/cloud/environments | 2026-04-29 | Persistence model, container specs |
| 5 | https://developers.openai.com/codex/agent-approvals-security | 2026-04-29 | Security, auth, vendor lock-in |
| 6 | https://developers.openai.com/codex/cli/features | 2026-04-29 | Spawn (CLI), read, remote TUI |
| 7 | https://developers.openai.com/codex/sdk | 2026-04-29 | Spawn (SDK), send, persistence |
| 8 | https://developers.openai.com/codex/guides/agents-sdk | 2026-04-29 | MCP server, send (codex-reply), visibility |
| 9 | https://developers.openai.com/codex/pricing | 2026-04-29 | Pricing, quotas, tiers, auth |
| 10 | https://developers.openai.com/codex/models | 2026-04-29 | Model lineup, availability |
| 11 | https://developers.openai.com/codex/config-reference | 2026-04-29 | Multi-agent config, thread limits |
| 12 | https://developers.openai.com/codex/cli/reference | 2026-04-29 | Status/list/kill CLI, send semantics |
| 13 | https://developers.openai.com/codex/cli | 2026-04-29 | Remote TUI, reach |
| 14 | https://openai.com/index/introducing-codex/ (403) | 2026-04-29 | [Fetch blocked; used search summary instead] |
| 15 | https://developers.openai.com/codex/config-advanced | 2026-04-29 | OTel visibility, audit |
| 16 | https://status.openai.com/incidents/01KH1KTV2VBD62R0MRZFJ13XNE + https://serenitiesai.com/articles/github-down-ai-coding-tools-dependency-2026 | 2026-04-29 | GitHub dependency incident |
| 17 | https://openai.com/index/introducing-the-codex-app/ | 2026-04-29 | Desktop app, multi-agent UI |
| 18 | https://community.openai.com/t/understanding-the-new-codex-limit-system-after-the-april-9-update/1378768 | 2026-04-29 | Pricing change impact, community |
| 19 | https://aws.amazon.com/about-aws/whats-new/2026/04/bedrock-openai-models-codex-managed-agents/ | 2026-04-29 | Bedrock Managed Agents |
| 20 | https://github.com/openai/codex/issues/15477 | 2026-04-29 | Silent code review failures |
| 21 | https://github.com/openai/codex/issues/19536 | 2026-04-29 | Sudden quota drops |
| 22 | https://github.com/openai/codex/issues/9135 | 2026-04-29 | Improper 429 errors |
| 23 | https://github.com/openai/codex/issues/14593 | 2026-04-29 | Token burn rate issues |
| 24 | https://github.com/openai/codex/issues/19009 + https://github.com/openai/codex/issues/13568 | 2026-04-29 | Compaction/usage failures |
| 25 | https://developers.openai.com/codex/guides/agents-md | 2026-04-29 | AGENTS.md portability |
| 26 | https://github.com/openai/codex | 2026-04-29 | Open-source CLI, MIT license |
| 27 | https://chatgpt.com/codex/pricing/ | 2026-04-29 | [Fetch blocked; used pricing page instead] |
| 28 | https://developers.openai.com/codex/config-sample | 2026-04-29 | Multi-agent, remote sandbox config |

---

## 7. Summary

**BackendAdapter fit verdict:** Viable but second-tier cloud backend. Codex Cloud maps 3 of 6 Mission 2 primitives cleanly (`spawn`, `list`, `status`), has a workable path for `send` via MCP, and has documented gaps in `read` (no streaming cloud event feed) and `kill` (no programmatic cancellation). The MCP server mode is the recommended integration surface — it avoids CLI subprocess fragility and uses an open protocol.

**3 most important findings:**

1. **No raw HTTP API for cloud task dispatch.** Codex Cloud is not a "call this endpoint and get a task running" API like Anthropic Managed Agents. The programmatic paths are CLI (`codex cloud exec`), SDK (`@openai/codex-sdk`), and MCP server (`codex mcp-server`). The MCP path is the best fit for BackendAdapter because it exposes typed `codex` and `codex-reply` tools over an open protocol, returns `threadId` as a session handle, and allows the substrate to own orchestration without OpenAI Agents SDK coupling.

2. **GitHub is a hard dependency — and a documented failure point.** Codex Cloud requires a connected GitHub account. On February 9, 2026, a GitHub outage took the entire Codex Cloud service offline. Any Mission 2 workload that uses non-GitHub repositories, or that requires high availability independent of GitHub uptime, cannot use Codex Cloud as a cloud backend. This is not a temporary limitation — it is architectural to the product.

3. **Credit metering is unreliable and recently got dramatically worse.** The April 9, 2026 pricing change introduced reasoning-time billing that causes the Business tier to drain ~3.2x faster than the Plus tier for identical workloads. GitHub issues document silent code-review failures, sudden quota drops to zero, and improper 429 errors with substantial quota remaining. For a fleet substrate that dispatches tasks programmatically, unpredictable quota behavior is a high-severity operational risk.

**Open questions (top 3):**

1. Does an undocumented REST endpoint exist for cloud task submission? (One third-party source claimed `POST /v1/codex/cloud/tasks` — unconfirmed.)
2. What is the actual cloud-side concurrency limit per account? (`max_threads = 6` is in local config; cloud enforcement unclear.)
3. Is there an Enterprise webhook for cloud task completion, avoiding the polling-only status model?

**What I might have missed:** The OpenAI Platform API reference (`platform.openai.com/docs`) may document cloud endpoints not covered in the Codex developer docs (`developers.openai.com/codex`). These are different doc trees and the platform reference was not fully traversed. The Responses API (replacing Chat Completions for Codex) may have additional programmatic agent control surfaces. Also: the Amazon Bedrock Managed Agents (OpenAI-powered) surface may provide a REST API that Codex Cloud proper does not — this would materially change the fit assessment and warrants a follow-up evaluation.
