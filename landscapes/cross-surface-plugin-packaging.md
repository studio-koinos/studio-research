---
research_date: 2026-04-30
last_verified: 2026-04-30
staleness_warning: Re-verify after 14 days. The OpenAI Apps SDK + ChatGPT MCP support shipped in late 2025 / early 2026; cross-surface tool patterns are actively shifting. Re-verify Codex App Server, Apps SDK, and modelcontextprotocol.io/clients before any implementation work.
confidence_summary: 18 verified, 4 likely, 3 unverified, 1 conflicting
session: S-40 AGENTOPS
linear: AO-183, AO-184
---

# Cross-Surface Plugin Packaging Landscape — April 2026

> **TL;DR:** MCP (Model Context Protocol, spec 2025-06-18) is the de-facto cross-vendor LLM-tool integration layer in April 2026. **All four major LLM-coding-agent surfaces (Claude Code, ChatGPT/OpenAI Codex, Cursor, Continue) consume MCP servers natively from a single source codebase.** OpenAI's **Apps SDK** ships MCP-native — ChatGPT users install MCP-backed apps from a marketplace. Linear is the canonical cross-surface example: one hosted MCP server (`https://mcp.linear.app/sse`) runs in Claude, Cursor, Windsurf, VS Code, Zed, and ChatGPT. **Recommended packaging shape for `koinos-context` and similar fleet tools: build a single MCP server (HTTP transport with OAuth), with thin per-surface install instructions; do NOT build per-surface adapters.** [VERIFIED across multiple authoritative sources]
>
> **Primary sources:**
> - OpenAI Codex App Server docs: https://developers.openai.com/codex/app-server (fetched 2026-04-30) [VERIFIED]
> - OpenAI Apps SDK landing: https://developers.openai.com/apps-sdk/ (fetched 2026-04-30) [VERIFIED]
> - OpenAI Responses API reference: https://platform.openai.com/docs/api-reference/responses (fetched 2026-04-30 via Exa) [VERIFIED]
> - Anthropic Claude Code plugins announcement: https://claude.com/blog/claude-code-plugins (published 2025-10-09, fetched 2026-04-30) [VERIFIED]
> - Anthropic Claude Code plugins docs: https://code.claude.com/docs/en/plugins (fetched 2026-04-30; redirected from docs.anthropic.com) [VERIFIED]
> - MCP spec 2025-06-18: https://modelcontextprotocol.io/specification/2025-06-18 (fetched 2026-04-30) [VERIFIED]
> - MCP clients catalog: https://modelcontextprotocol.io/clients (fetched 2026-04-30) [VERIFIED]
> - Cursor MCP docs: https://cursor.com/docs/context/mcp (fetched 2026-04-30) [VERIFIED]
> - Continue MCP docs: https://docs.continue.dev/customize/deep-dives/mcp (fetched 2026-04-30) [VERIFIED]
> - Zed AI MCP docs: https://zed.dev/docs/ai/mcp (fetched 2026-04-30) [VERIFIED]
> - Linear MCP launch: https://linear.app/changelog/2025-05-01-mcp (fetched 2026-04-30) [VERIFIED]
> - Anthropic landscape (internal, S-32): `studio-research/landscapes/anthropic-agent-product-spectrum.md` v2026-04-29 [VERIFIED — internal]
> - OpenAI landscape (internal, S-32): `studio-research/landscapes/openai-agent-product-spectrum.md` v2026-04-29 [VERIFIED — internal]

## What it is

The question this doc answers: **how do you ship a single LLM tool (e.g., a knowledge-graph reader, a project-management connector, a substrate-dispatch tool) such that ChatGPT users, Claude Code users, Cursor users, and Continue users can all use it without per-platform reimplementation?**

The April 2026 answer is **MCP-server-as-source-of-truth + per-surface install manifests**. There is no all-in-one cross-vendor plugin package — but the per-surface manifests are thin (a few lines of YAML/JSON each) because every major LLM agent host has converged on consuming the same underlying MCP server protocol.

## How it works

### MCP as the portable tool layer

[VERIFIED] MCP (Model Context Protocol) spec 2025-06-18 is the current authoritative version. Architecture: **Hosts → Clients → Servers**, JSON-RPC 2.0 over stdio or HTTP (with optional Server-Sent Events for streaming). Three server primitives: **Resources, Prompts, Tools.** Three client primitives: **Sampling, Roots, Elicitation.** Extension surfaces in active rollout: **MCP Apps** (interactive HTML UIs), **Tasks** (long-running ops), **DCR + CIMD + Enterprise-Managed Authorization** for OAuth.

Source: https://modelcontextprotocol.io/specification/2025-06-18

[VERIFIED] **Transport options as of April 2026:** stdio (local subprocesses), Streamable HTTP (with SSE for streaming, CDN-compatible), and pure SSE (deprecated as of MCP spec 2025-06-18). Per Cursor's implementation: stdio + SSE + Streamable HTTP all supported. Same for Continue. Streamable HTTP is the recommended path for hosted/remote MCP servers because it works through CDNs and supports OAuth flows.

Source: https://cursor.com/docs/context/mcp, https://docs.continue.dev/customize/deep-dives/mcp

[VERIFIED] **HTTP MCP authentication:** OAuth 2.1 with PKCE is the spec-mandated path for remote servers. Bearer-token-via-Authorization-header is also supported. The OAuth flow triggers when a Claude Code (or Cursor, Continue, etc.) session attempts to invoke a tool from an unauthenticated MCP server — the host opens a browser to the authorization endpoint, callback lands at a local redirect URI, token persists in the host's credential store. **[UNVERIFIED on specifics]: We have not confirmed exact callback-port or credential-store details for Claude Code; this is parametric knowledge from MCP spec extrapolation.**

### Claude Code plugin system (Anthropic's distribution layer above MCP)

[VERIFIED] **Plugins shipped 2025-10-09** as a public beta. Source: https://claude.com/blog/claude-code-plugins.

Verbatim from the announcement: *"Claude Code now supports plugins: custom collections of slash commands, agents, MCP servers, and hooks that install with a single command."*

[VERIFIED] **Plugin manifest:** A plugin is a directory with a `.claude-plugin/plugin.json` manifest. The plugin can ship:
- `skills/<name>/SKILL.md` — skills (Anthropic's skill primitive)
- `commands/` — custom slash commands
- `agents/` — subagent personas
- `hooks/hooks.json` — lifecycle hooks
- `.mcp.json` — bundled MCP server configuration
- `.lsp.json` — bundled LSP server configuration (code intelligence)
- `monitors/monitors.json` — monitors
- `bin/` — executables
- `settings.json` — settings overrides

Skills are namespaced with the plugin (`/plugin-name:skill`).

Source: https://code.claude.com/docs/en/plugins

[VERIFIED] **Distribution mechanism:** A "marketplace" is **any git repo or URL with a `.claude-plugin/marketplace.json` file**. Install: `/plugin marketplace add user/repo` followed by `/plugin install <plugin-name>`. Local development: `--plugin-dir ./local`. Submission to Anthropic's official marketplace via `claude.ai/settings/plugins/submit`.

[VERIFIED] **A single plugin can bundle MCP server + skills + commands + agents + hooks + LSP + settings.** This is the key architectural property: plugins are not "MCP server wrappers" — they're richer collections where MCP-server-bundling is one of several declarations.

[VERIFIED] **Anthropic operates a live MCP registry** at `https://api.anthropic.com/mcp-registry/v0/servers` with a `worksWith` discriminator (`claude-code`, `claude-api`, `claude-desktop`). This means MCP servers can be discovered, not just hand-installed. Source: https://code.claude.com/docs/en/mcp.

### OpenAI's tool-publishing surfaces (April 2026)

[VERIFIED] **OpenAI Apps SDK is shipped and MCP-native.** Source: https://developers.openai.com/apps-sdk/.

Verbatim: *"Our framework to build apps for ChatGPT."* Core concept titled *"MCP Apps in ChatGPT."* Developers build:
1. An MCP server
2. Conversational UI components (HTML)
3. Hosting infrastructure

Apps appear in the **ChatGPT app store**. This is the canonical path for shipping a tool that ChatGPT users (chat.openai.com) can install.

[VERIFIED] **Codex App Server is producer-side, JSON-RPC 2.0, MCP-adjacent but NOT MCP.** Source: https://developers.openai.com/codex/app-server (fetched directly).

Verbatim: *"Like MCP, `codex app-server` supports bidirectional communication using JSON-RPC 2.0 messages."*

It's how third parties **embed Codex into their own products** (deep integrations, auth, conversation history, approvals, streamed agent events). Open source in the openai/codex GitHub repo. **It is not a tool-publishing path** — it's a host-protocol path. Tool publishing for Codex users still goes through MCP (since Codex CLI is itself an MCP client per modelcontextprotocol.io/clients).

[VERIFIED] **Responses API exposes MCP as a first-class tool type.** Source: https://platform.openai.com/docs/api-reference/responses.

Verbatim: *"Give the model access to additional tools via remote Model Context Protocol (MCP) servers."*

Parameters on the `Mcp` tool type:
- `server_label` — identifier
- `server_url` OR `connector_id` — direct URL or pre-built service connector (Dropbox, Gmail, Google Drive, Microsoft Teams, SharePoint are confirmed connector_ids)
- `allowed_tools` — list or filter
- `headers` — auth headers
- `require_approval` — `always` / `never` / per-tool
- `authorization` — OAuth token

This means: **API consumers can ship MCP-backed tooling at the API layer without touching ChatGPT consumer surface.** The Responses API is the production-ready path.

[VERIFIED] **Original ChatGPT Plugins (March 2023 launch) are deprecated.** `https://platform.openai.com/docs/plugins` returns 404 as of 2026-04-30. They are no longer the recommended integration shape for new work.

[LIKELY] **Custom GPTs + GPT Actions are still functional but legacy.** GPT Actions are documented at `platform.openai.com/docs/actions` (function-calling-based, REST API conversion, no MCP). They're scoped to Custom GPTs. Per internal landscape doc (`openai-agent-product-spectrum.md` §4): *"Custom GPTs (deprecated 2026-08-26)"* — sunset date claimed. **CONFLICTING:** The cross-surface web fetch could not confirm the 2026-08-26 sunset date directly (URL https://help.openai.com/en/articles/8554407 returned 403). Treat the sunset claim as LIKELY but not VERIFIED.

[VERIFIED] **AgentKit + Agent Builder (October 2025 launch)** is OpenAI's higher-level visual platform — Agent Builder canvas, ChatKit (embeddable chat), Connector Registry. **It does NOT provide a tool-publishing path for third parties.** It's a buyer-side toolkit.

### Cross-IDE MCP support (April 2026)

[VERIFIED] **The modelcontextprotocol.io/clients page** lists ~80 MCP clients with feature matrices. Confirmed clients: Claude Code, Claude Desktop, Claude.ai, ChatGPT, OpenAI Codex CLI, Cursor, Continue, Cline, Zed, Augment, Amp, Amazon Q Developer, BoltAI, Chatbox, and others.

ChatGPT entry on that page (verbatim): *"ChatGPT is OpenAI's AI assistant that provides MCP support for remote servers… Support for MCP via connections UI in settings… Support for MCP Apps."*

[VERIFIED] **Cursor:** Three transports (stdio, SSE, Streamable HTTP). Config: `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global). Supports Tools, Prompts, Resources, Roots, Elicitation, MCP Apps. OAuth on remote servers.

[VERIFIED] **Continue:** Same three transports as Cursor. Config: `.continue/mcpServers/*.yaml`. **Critical portability quote:** *"If you're coming from another tool that uses JSON MCP format configuration files (like Claude Desktop, Cursor, or Cline), you can copy those JSON config files directly into your `.continue/mcpServers/` directory and Continue will automatically pick them up."* — explicit cross-client config compatibility.

[VERIFIED] **Zed:** Calls them "context servers" (renamed but same MCP). Supports Tools and Prompts (not yet Discovery, Sampling, Elicitation). Config: `settings.json` under `context_servers`. Local: `command/args/env`. Remote: `url + headers` (OAuth flow auto if no Authorization header).

[UNVERIFIED] **Aider:** Not in modelcontextprotocol.io/clients list as of fetch. URL `aider.chat/docs/usage/tools.html` returned 404. Likely no first-party MCP support yet — the project's tooling shape predates MCP convergence.

### Cross-surface tool examples (April 2026)

[VERIFIED] **Linear is the canonical cross-surface example.** Source: https://linear.app/changelog/2025-05-01-mcp (launched 2025-05-01).

One hosted MCP server at `https://mcp.linear.app/sse` (SSE transport, OAuth). Verbatim from Linear's announcement: *"We're following the authenticated remote MCP spec, so the server is centrally hosted and managed."*

Confirmed working surfaces (per Linear's docs): **Claude, Cursor, Windsurf, VS Code, Zed via `mcp-remote` shim.** This proves the live 2026 pattern: build one remote MCP server, publish per-surface install instructions, ship to all consumer hosts.

[VERIFIED] **Official MCP server reference implementations** at https://github.com/modelcontextprotocol/servers — Everything (TS), Fetch (TS), Filesystem (TS), Git (Python), Memory (TS), Sequential Thinking (TS), Time (TS). Stdio default. These are templates for any team building MCP servers.

[LIKELY] **Slack, Notion, GitHub** — assumed to follow Linear's shape (single hosted MCP server with per-surface install guides) but specific docs not fetched in this round.

## Current state

**Convergence on MCP is the structural finding.** As of April 2026:

| Surface | MCP support | Path for tool publishers |
|---|---|---|
| Claude Code | Native MCP host + plugin marketplace | Build MCP server → wrap in Claude Code plugin (`marketplace.json`) for richer bundling |
| Claude.ai (web) | Native MCP host | Build MCP server (HTTP+OAuth) → users connect via settings UI |
| Claude Desktop | Native MCP host | Build MCP server → users add to settings.json |
| ChatGPT (chat.openai.com) | Native MCP host (since late 2025/early 2026) | Build MCP server + Apps SDK manifest → publish to ChatGPT app store |
| OpenAI Responses API | Native MCP tool type | Build MCP server (HTTP+OAuth) → API consumers wire it via `Mcp` tool param |
| OpenAI Codex CLI | Native MCP client (stdio + HTTP+OAuth) | Build MCP server → users add to Codex config |
| Cursor | Native MCP host (3 transports) | Build MCP server → users add to `.cursor/mcp.json` |
| Continue | Native MCP host (3 transports) | Build MCP server → users add to `.continue/mcpServers/` |
| Zed | Native MCP host (renamed "context servers") | Build MCP server → users add to `settings.json` |
| Cline, Amp, Amazon Q, ~80 others | Native MCP clients | Build MCP server → install per-surface |

**The path that works everywhere: build one MCP server (HTTP transport, OAuth 2.1 auth) and write thin per-surface install guides.** The MCP server is the universal artifact.

**Anthropic's plugin format adds value above MCP** (skills + commands + agents + hooks + LSP) but is Claude-Code-specific. If you want "richer bundling on Claude Code AND MCP-only on every other surface," structure your codebase as: MCP server (the portable layer) + Claude Code plugin (the Claude-specific richer wrapper that bundles the MCP server).

**OpenAI's Apps SDK adds value above MCP** (UI components for ChatGPT app store) but is ChatGPT-specific. Same pattern: MCP server is the portable core; Apps SDK wraps it for ChatGPT.

## Landscape / alternatives

**There is no all-in-one cross-vendor plugin format.** The closest thing is the MCP spec itself, which has been donated to the Agentic AI Foundation (Dec 2025, OpenAI co-founder mentioned per workos.com source [LIKELY but not verified]). MCP plays the role for LLM tools that LSP plays for code intelligence: a vendor-neutral protocol, with vendor-specific extensions (Anthropic's plugin format, OpenAI's Apps SDK) layered on top.

Alternative shapes considered + rejected:

1. **Per-surface native plugins** (Anthropic plugin + ChatGPT Apps SDK package + Cursor extension + Continue extension). Rejected: 4× implementation cost; impossible to maintain identical behavior; no one ships this way.
2. **REST API + per-surface OpenAPI manifests** (the original 2023 ChatGPT Plugins shape). Rejected: ChatGPT Plugins are deprecated; no surface other than legacy GPT Actions consumes this shape; MCP has won.
3. **Vendor-neutral plugin spec via Agentic AI Foundation.** [UNVERIFIED] May exist but if so, hasn't shipped — every surface still uses its own format above MCP.

## Applicability — recommended pilot for `koinos-context` and AO-184

**Recommended packaging shape for AO-184 pilot:**

1. **Source of truth:** the existing `koinos-context` MCP server (HTTP transport, OAuth, hosted at `https://mcp.koinos.studio/mcp`; canonical domain established S-40, AO-187). This is already MCP-compliant per the registration we did in S-40 (`claude mcp add --transport http koinos-context https://mcp.koinos.studio/mcp`).
2. **Claude Code distribution:** wrap the MCP server in a Claude Code plugin with `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` in a public git repo (e.g., `studio-koinos/koinos-context-plugin`). Bundle alongside: any `koinos-context`-specific skills, custom slash commands (e.g., `/ctx-search`, `/ctx-decisions`), and the MCP server config. Installation: `/plugin marketplace add studio-koinos/koinos-context-plugin && /plugin install koinos-context`.
3. **ChatGPT distribution:** build an OpenAI Apps SDK manifest pointing at the same `mcp.koinos.studio/mcp` endpoint. Build the conversational UI components (HTML) for ChatGPT's app store presentation. Submit via OpenAI's app submission flow. The MCP server itself is unchanged; the Apps SDK package is just the ChatGPT-specific wrapper.
4. **Cursor / Continue / Zed distribution:** publish a one-page README in the same repo with config snippets for each. They all consume the same MCP HTTP endpoint with OAuth — copy-paste install.
5. **OpenAI Responses API path:** document for API consumers how to wire `koinos-context` via the `Mcp` tool parameter in their Responses API calls. No additional packaging needed; just docs.

**Pilot scope (smallest meaningful test):**

- Phase 1: Claude Code plugin shipped (lowest friction; we already use Claude Code daily).
- Phase 2: Apps SDK package shipped to ChatGPT app store (highest reach; gates on ChatGPT app submission approval).
- Phase 3: README install guides for Cursor + Continue + Zed.

**Stage advancement criterion (per `decisions/006_verified-design-via-composition.md` engineering method):** Phase 1 ratchets to Phase 2 only after the Claude Code plugin is observed working on a real `koinos-context` query end-to-end (OAuth flow + tool call + response) in a real Claude Code session. Don't speculate-build all three phases in parallel.

**Why not start with Apps SDK first?** Apps SDK has the larger user reach (ChatGPT consumer base) but the longest validation cycle (app store submission, review). Claude Code plugin is internal-controlled, Anthropic's plugin marketplace is open submission, and we use Claude Code daily — fastest validation loop.

## Conflicts & gaps

- **CONFLICTING:** Custom GPTs sunset date. Internal landscape doc claims 2026-08-26; external research could not verify (relevant URL 403'd). Resolution: treat as LIKELY-not-VERIFIED; check OpenAI's official deprecation announcements before relying on it for any decision.
- **UNVERIFIED:** Aider's MCP support. Not in modelcontextprotocol.io/clients list. URL 404'd. Treat as "no first-party MCP support yet" but verify directly if Aider integration becomes load-bearing.
- **UNVERIFIED:** OAuth 2.1 callback-port + credential-store specifics for Claude Code's MCP authentication flow. We registered koinos-context successfully and got the "Needs authentication" signal, but haven't traced the actual OAuth flow end-to-end. Empirical test: invoke a tool from `koinos-context` and observe what happens.
- **GAP:** No hands-on validation of the Apps SDK submission process. Claim is "shipped" per the docs but the submission/review timeline + approval-rate empirics are unknown.
- **GAP:** `agentic-substrate-distribution.md` (internal) does NOT discuss cross-vendor tool packaging — it focuses on agent-state distribution. This research extends rather than overlaps.

## Related but out of scope

- **OpenAI Codex App Server as a substrate backend (AO-183):** consumer-side, NOT producer-side. It's a JSON-RPC protocol for embedding Codex into apps, similar to how `claude -p` is a CLI for embedding Claude. Substrate's Tier 2/3 backend roster might add a `CodexAppServerAdapter` for that purpose. Distinct from THIS research.
- **MCP-over-stdio cloud-default (memory `reference_mcp_over_stdio_cloud_default.md`):** confirmed by this research — cloud agents (Anthropic + OpenAI + Cloudflare) have converged on MCP. THIS research extends with the specific finding that ChatGPT consumer surface ALSO consumes MCP natively as of late 2025 / early 2026 (Apps SDK).
- **WebFetch 403 → Exa fallback pattern:** worth a separate reference memory; the pattern works reliably for vendor-doc retrieval where plain WebFetch is blocked.
- **Anthropic MCP registry at api.anthropic.com/mcp-registry:** worth surfacing to `koinos-context` team — if the MCP server gets registered with `worksWith: claude-code`, Claude Code users may be able to discover it without manual `claude mcp add`.

## Key sources

External (fetched 2026-04-30):
1. https://developers.openai.com/codex/app-server — Codex App Server (producer-side, JSON-RPC 2.0, MCP-adjacent)
2. https://developers.openai.com/apps-sdk/ — OpenAI Apps SDK (MCP-native, ChatGPT app store)
3. https://platform.openai.com/docs/api-reference/responses — Responses API MCP tool type
4. https://platform.openai.com/docs/guides/agents — OpenAI Agents SDK
5. https://claude.com/blog/claude-code-plugins — Claude Code plugins announcement (2025-10-09)
6. https://code.claude.com/docs/en/plugins — Claude Code plugins docs (redirected from docs.anthropic.com)
7. https://code.claude.com/docs/en/mcp — Claude Code MCP setup docs
8. https://modelcontextprotocol.io/specification/2025-06-18 — MCP spec (current authoritative version)
9. https://modelcontextprotocol.io/clients — MCP clients catalog
10. https://cursor.com/docs/context/mcp — Cursor MCP docs
11. https://docs.continue.dev/customize/deep-dives/mcp — Continue MCP docs
12. https://zed.dev/docs/ai/mcp — Zed AI MCP docs
13. https://linear.app/changelog/2025-05-01-mcp — Linear MCP server launch (2025-05-01)
14. https://github.com/modelcontextprotocol/servers — MCP server reference implementations

Internal (verified S-32, latest 2026-04-29):
15. `studio-research/landscapes/anthropic-agent-product-spectrum.md` — covers MCP (§3.5), Plugin Marketplace (§3.8), Agent Skills (§3.6)
16. `studio-research/landscapes/openai-agent-product-spectrum.md` — covers AgentKit (§4.6), Codex App Server (§4.13), Symphony (§3)
17. `studio-research/landscapes/agentic-substrate-distribution.md` — adjacent (substrate state distribution; not cross-vendor tooling)
18. `studio-research/references/claude-code-config-reference.md` — MCP scope semantics (User/Project/Local)
19. `~/.claude/projects/.../memory/reference_mcp_over_stdio_cloud_default.md` — S-27 finding extended by THIS research

## Update 2026-04-30 (S-40, same day) — Anthropic/OpenAI plugin parity is much higher than initially framed

The original synthesis above implied Anthropic has substantially more plugin primitives than OpenAI. **A parity-focused follow-up research run (same session) materially changed this picture.** Key corrections:

### CLI tier is near-parity (Claude Code ≈ Codex CLI)

[VERIFIED] **Codex CLI shipped a plugin system that mirrors Claude Code's almost feature-for-feature**:
- Manifest: `.codex-plugin/plugin.json` (mirrors Claude's `.claude-plugin/plugin.json`). Source: developers.openai.com/codex/plugins/build.
- Marketplace: `codex plugin marketplace add owner/repo` accepts GitHub shorthand, Git URLs, SSH, local dirs, with `--ref` pinning. Source: developers.openai.com/codex/cli/reference.
- Skills: `.agents/skills/SKILL.md` paths — **open Agent Skills standard, explicitly platform-agnostic.** Source: claude.com/docs/skills, developers.openai.com/codex/skills.
- Hooks: `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Stop`, `UserPromptSubmit` events with regex matchers — same shape as Claude Code hooks (behind feature flag in `config.toml`). Source: developers.openai.com/codex/hooks.
- Subagents: `.codex/agents/*.toml`; plugin-bundled agents pending per open issue github.com/openai/codex/issues/18988.

### The portable primitive: Agent Skills standard

[VERIFIED] **Skills are now a cross-vendor open standard.** Both Anthropic and OpenAI reference the "Agent Skills specification" as platform-agnostic. A `SKILL.md` authored once works in both Claude Code (via `.claude/skills/` or plugin `skills/`) and Codex CLI (via `.agents/skills/`). The shared `.agents/skills/` path convention between vendors looks like deliberate alignment.

### Custom slash commands — divergence

[VERIFIED] **Codex CLI deprecated user-defined slash commands** in favor of Skills, with the position that "Skills provide a superset of the functionality" via `$skill` auto-completion. Source: github.com/openai/codex/issues/18857.

This is a **philosophical divergence**: Claude Code keeps `/commands` as a first-class user-invoked primitive; Codex collapses into model-invokable Skills with optional explicit `$skill` invocation. Practical implication: a slash command on Claude Code = a skill with `disable-model-invocation: true`; the equivalent in Codex = a skill with `allow_implicit_invocation: false` invoked via `$skill`.

### Web/consumer tier — gap remains, but Apps SDK auto-cascades to Codex

[VERIFIED] **OpenAI auto-generates a Codex plugin from approved Apps SDK submissions.** Source: developers.openai.com/apps-sdk. Practical implication: one Apps SDK submission to ChatGPT app store also yields a Codex CLI install path — distribution multiplier.

[VERIFIED] **Plugins on Claude.ai web/Desktop are Cowork-only.** Source: claude.com/docs/connectors/building/what-to-build — *"plugins: Works in: Claude Code, Cowork"*. Free or Pro Claude.ai users would only get Custom Connectors (remote MCP, OAuth) + Skills (Pro+); they would not install the plugin. This is a real product gating.

[VERIFIED] **Anthropic ships a "unified directory"** on Claude.ai web/Desktop with Skills + Connectors + Plugins tabs in the `Customize` sidebar. Source: support.claude.com/articles/14328846. Narrows the web-tier UX gap with ChatGPT App Store.

### Web-tier comparison (where the gap actually is)

| Surface | MCP | Skills | Slash Commands | Hooks | Subagents | UI Components |
|---|---|---|---|---|---|---|
| Claude.ai web/Desktop | ✓ Custom Connectors | ✓ Pro+ | ✗ | ✗ | ⚠ Plugin (Cowork only) | ⚠ Apps SDK partner |
| ChatGPT web | ✓ Apps SDK + Connectors | ⚠ Custom GPTs Instructions (not portable Skills) | ✗ | ✗ | ✗ | ✓ **Apps SDK iframe + MCP Apps bridge** |

ChatGPT has the richer **UI-component** story (iframe + JSON-RPC over postMessage); Anthropic has the richer **portable-skills** story (Skills available Pro+ as standalone files). Both lack hooks and user-defined slash commands on the consumer tier. **No cross-vendor parity on the web tier — but the asymmetry is small.**

### Recommended packaging shape — REVISED for AO-184

Replace the earlier "Phase 1 = Claude Code plugin first" sequencing with a tier-aware shape:

```
Single MCP server (mcp.koinos.studio/mcp, HTTP+OAuth) ← universal floor (every surface)
│
├── Tier-1 RICH UX — Anthropic + OpenAI CLI (single source of truth)
│   ├── skills/                         ← shared SKILL.md files (cross-vendor standard)
│   ├── .mcp.json                       ← shared MCP server config
│   ├── .claude-plugin/plugin.json      ← Claude Code manifest
│   ├── .codex-plugin/plugin.json       ← Codex CLI manifest
│   ├── hooks-claude/hooks.json         ← Claude Code hooks (PreToolUse/PostToolUse/etc.)
│   └── hooks-codex/hooks.toml          ← Codex CLI hooks (same events, different schema)
│
├── Tier-1 RICH UX — Anthropic + OpenAI web/consumer
│   ├── Anthropic: Custom Connector (remote MCP + OAuth) + Skills bundled separately for Pro+
│   └── OpenAI: Apps SDK manifest + UI components (iframe + MCP Apps bridge)
│       └── auto-generates Codex CLI plugin on app-store approval
│
└── Tier-2 BARE MCP — everyone else
    ├── README install snippets for Cursor, Continue, Zed, Cline, Aider, etc.
    └── Same hosted MCP server; surface-specific config syntax only
```

### Phase plan — REVISED

**Phase 1 (Tier-1 CLI both vendors, single source of truth):**
- Phase 1a: Claude Code plugin works locally end-to-end (install → OAuth → tool call → uninstall)
- Phase 1b: Add Codex CLI plugin manifest (`.codex-plugin/plugin.json`); same skills + MCP work in Codex CLI
- Phase 1c: Promote to dedicated repo `studio-koinos/koinos-context-plugin`; submit to Anthropic + Codex marketplaces

**Phase 2 (Tier-1 web/consumer):**
- Phase 2a: Anthropic Custom Connector + standalone Skills bundle for Claude.ai Pro+
- Phase 2b: OpenAI Apps SDK package (MCP + UI components) → ChatGPT app store + auto-cascade to Codex CLI

**Phase 3 (Tier-2 mechanical):**
- READMEs for Cursor / Continue / Zed / Cline. One-page install snippets each.

### Cross-references added

- Per-surface capability matrix derived from S-40 parity research (see Linear AO-184 for the verbatim matrix)
- Codex CLI plugin docs: developers.openai.com/codex/plugins/build, developers.openai.com/codex/skills, developers.openai.com/codex/hooks
- Apps SDK → Codex auto-cascade: developers.openai.com/apps-sdk
- Anthropic unified directory: support.claude.com/articles/14328846
- Cowork-only plugin gating: claude.com/docs/connectors/building/what-to-build
- Open Agent Skills standard: claude.com/docs/skills

## Edit history

- 2026-04-30 (S-40 AGENTOPS): Filed by /deep-research per Ali's directive after AO-184 capture (cross-surface plugin packaging) was created. Stream A (Perplexity sonar-deep-research) failed with hallucination; Stream A fallback (sonar-pro) returned thin coverage. Stream B (web-fetch agent with Exa fallback) returned authoritative answers from 14/20 official URLs. Synthesis incorporates Stream B + internal landscape docs.
- 2026-04-30 (S-40, same session, parity-focused follow-up): Added Update section after Ali surfaced parity question — "for Anthropic + OpenAI we need to deliver the most effective UX we can." Web-fetch parity research surfaced that Codex CLI plugin system is near-isomorphic to Claude Code's; Skills are cross-vendor open standard. Original framing partially corrected. Recommended packaging shape revised to tier-aware shape (CLI dual-target + web dual-target + tier-2 MCP-only).
