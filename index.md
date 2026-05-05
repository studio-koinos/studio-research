# Research Index

Flat catalog of all documents. One-line summary per entry. Updated on every add/update/migrate/archive.

**Quick access:** `grep "^- \[" index.md` lists all entries. `grep "tags:.*keyword" evaluations/*.md references/*.md` finds by topic.

## Evaluations

- [Anthropic Agent Skills & Managed Agents](evaluations/anthropic-agent-skills.md) — Cloud-hosted managed agent runtime from Anthropic. Capability profile across 12 dimensions for Mission 2 BackendAdapter evaluation: spawn, send, read, status, kill, list, persistence, auth, pricing, reach, visibility, portability.
- [Cloudflare Agents](evaluations/cloudflare-agents.md) — BackendAdapter capability profile for Cloudflare's agent runtime built on Workers + Durable Objects. Covers spawn/send/read/status/kill semantics, persistence model, pricing, portability, and Mission 2 fit assessment.
- [cmux](evaluations/cmux.md) — Source-level evaluation of manaflow-ai/cmux as a terminal-ops backend candidate for Mission 2's agent fleet platform registry. Covers all 12 capability dimensions, BackendAdapter fit, and empirical findings from Ali's production use.
- [Context7 MCP Server](evaluations/context7-mcp.md) — MCP server for fetching current library/framework docs; verdict: adopt
- [OpenAI Codex Cloud](evaluations/openai-codex-cloud.md) — Cloud-managed Codex agent execution environment from OpenAI. Capability profile across 12 dimensions for Mission 2 BackendAdapter evaluation: spawn, send, read, status, kill, list, persistence, auth, pricing, reach, visibility, portability.

## References

<!-- Architecture research, tech deep dives. Format: - [Title](path) — one-line summary -->

- [AI Agent Knowledge Base Structure](references/agent-kb-structure.md) — How Claude Code, Cursor, and Codex discover and consume repo context; covers three-tier architecture, Karpathy llm-wiki pattern, AGENTS.md, and file naming conventions for agent-first repos.
- [Claude Code Official Configuration Reference](references/claude-code-config-reference.md) — Comprehensive reference for Claude Code config artifacts: CLAUDE.md, rules, agents, skills, memory, hooks, settings.json. 22 verified claims from official Anthropic docs (2026-04-12).
- [Version-Controlling Claude Code User-Level Configuration](references/claude-code-config-version-control.md) — Community patterns for version-controlling ~/.claude/ config: git-in-place, symlinks from dotfiles repo, chezmoi. Includes settings.json symlink bug (#3575) and official symlink support status.
- [Continuous Agent/Config Operations Patterns](references/continuous-agent-ops-patterns.md) — Four mature patterns for solo-developer agent fleet management: label-based prompt promotion, agent manifest registry, monthly toil review, and git-diff-based drift scoring. Directly applicable to studio-agent-ops.
- [Obsidian-Style Knowledge Management for Git Repos](references/obsidian-knowledge-management.md) — Deep research on Obsidian-compatible markdown patterns (frontmatter, wikilinks, MOC, citations, tools) for git-based knowledge repos. Evaluated and rejected for studio-research in favor of llm-wiki pattern.

## Guides

<!-- Integration guides, how-tos. Format: - [Title](path) — one-line summary -->

## Landscapes

<!-- Market surveys, category overviews. Format: - [Title](path) — one-line summary -->

- [Agentic Fleet Platforms — Capability Registry](landscapes/agentic-fleet-platforms.md) — Central registry for Mission 2: 12-dim capability matrix × N backends with verdicts, integration patterns, and BackendAdapter shape implications. Promoted from .context/AO-104/synthesis-progress.md (S-27).
- [Agentic Substrate Distribution](landscapes/agentic-substrate-distribution.md) — How teams distribute, version, sync, and protect shared/per-user developer-environment artifacts. Surveys 3 industries × 7 axes (dotfiles, MLOps, LLM platforms; secrets, sync, history, audit). Headline: Cloudflare Artifacts (2026-04-16 beta) ships exactly the architecture for Tier 1 substrate; today's pragmatic pick is private-GitHub-repo + SOPS+age + 1Password CLI.
- [Anthropic Agent Product Spectrum](landscapes/anthropic-agent-product-spectrum.md) — Full mapping of Anthropic's agent-related products as candidates for Mission 2's fleet substrate. Covers 11 product surfaces from Messages API to Managed Agents.
- [Cloudflare Agent Services Stack](landscapes/cloudflare-agent-services-stack.md) — Full mapping of Cloudflare's agent-related services as composable primitives for Mission 2's fleet substrate. Covers 19 services across compute, storage, inference, observability, and security.
- [Cross-Surface Plugin Packaging](landscapes/cross-surface-plugin-packaging.md) — How to ship one MCP server cross-surface (Claude Code, ChatGPT, Cursor, Continue, Zed). Recommends single-MCP + thin per-surface manifests over per-surface adapters; Linear MCP as canonical example. (S-40, AO-183/AO-184.)
- [OpenAI Agent Product Spectrum](landscapes/openai-agent-product-spectrum.md) — Full mapping of OpenAI's agent-related products as candidates for Mission 2's fleet substrate. Covers API platform, Responses API, Agents SDK, AgentKit, Symphony, Codex CLI, Codex Cloud, Realtime API, Computer Use, Workspace Agents, Batch API, and supporting primitives.
- [Orchestration Layer Internals — Empirically Grounded Synthesis](landscapes/orchestration-layer-internals.md) — Synthesis of 7 surface deep-dives (cmux, Claude Code CLI/SDK, hooks, MCP, auth/connectors, tmux/iTerm2, Codex CLI) under "official sources first + paste-the-output" discipline (S-70, AO-263, AO-270). 14 architectural findings + 9 known gaps + 3 falsifications including a self-referential audit-document falsification. Supersedes cmux-features.md.

## Archive

<!-- Obsolete or superseded research. Format: - [Title](path) — archived YYYY-MM-DD, reason -->
