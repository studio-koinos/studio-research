# Research Log

Append-only. Never rewrite existing entries.

**Format:** `## [YYYY-MM-DD] operation | Title`

**Operations:** `add`, `update`, `migrate`, `archive`, `lint`

**Quick access:** `grep "^## \[" log.md | tail -10`

---

## [2026-04-13] add | Repository scaffolded
Structure: CLAUDE.md schema, RESOLVER.md, index.md, log.md, 5 directories with README resolvers.
Design: Karpathy llm-wiki pattern + gbrain RESOLVER concept. See studio-pm/docs/design-studio-research.md.

## [2026-04-13] migrate | Obsidian-Style Knowledge Management for Git Repos (from studio-agent-ops)
## [2026-04-13] migrate | Claude Code Official Configuration Reference (from studio-agent-ops)
## [2026-04-13] migrate | Version-Controlling Claude Code User-Level Configuration (from studio-agent-ops)
## [2026-04-13] migrate | Continuous Agent/Config Operations Patterns (from studio-agent-ops)
## [2026-04-13] migrate | AI Agent Knowledge Base Structure (from studio-pm)

## [2026-04-13] add | Context7 MCP Server
Evaluation of Context7 MCP for real-time library docs. Verdict: adopt. First evaluation-type doc — tests the filing workflow.

## [2026-04-13] update | Schema refinements (S-01 observations)
CLAUDE.md: conceptual headings note, Conflicts & Gaps optional section (Evaluations + References), Provenance block made explicit.

## [2026-04-27] add | Agentic Substrate Distribution (landscape)
Multi-source landscape survey for AO-157. 3 industries (dotfiles, MLOps, LLM platforms) × 7 axes (incl. secrets, sync, audit). 38 dated primary sources. Triangulated across Perplexity (sonar-pro fallback after sonar-deep-research refused fabrication), web research subagent, and local context. Headline: Cloudflare Artifacts (private beta 2026-04-16, public early May 2026) ships the exact architecture for Tier 1; pragmatic today is private-GitHub-repo + SOPS+age + 1Password CLI. AO-160 debug log location decision (`.claude/events-debug.log` not `/tmp`) confirmed by research.

## [2026-04-30] add | Agentic Fleet Platforms — Capability Registry (landscape, S-28)
## [2026-04-30] add | Anthropic Agent Product Spectrum — Vendor Landscape (landscape, S-28)
## [2026-04-30] add | Cloudflare Agent Services Stack — Vendor Landscape (landscape, S-28)
## [2026-04-30] add | OpenAI Agent Product Spectrum — Vendor Landscape (landscape, S-28)
## [2026-04-30] add | Cross-Surface Plugin Packaging (landscape, S-40)
## [2026-04-30] add | Anthropic Agent Skills & Managed Agents (evaluation, S-27)
## [2026-04-30] add | Cloudflare Agents (evaluation, S-27)
## [2026-04-30] add | cmux (evaluation, S-27)
## [2026-04-30] add | OpenAI Codex Cloud (evaluation, S-27)
S-44 batch tracking commit per AO-188. 9 research artifacts that were authored across S-27 (4 evaluations), S-28 (4 landscapes), and S-40 (1 landscape) but never landed in git. All have dated provenance frontmatter (research_date, last_verified, staleness_warning) — content final, only tracking gap. Future-discipline: handover-session skill will be extended to grep untracked files in sibling repos at session close.

## [2026-05-03] lint | Frontmatter reconciled to canonical schema (AO-192)
All 9 AO-188 artifacts (5 landscapes + 4 evaluations) updated: added `tags`, `modified`, `staleness` to YAML; removed `research_date`, `last_verified`, `staleness_warning`, `sources_count` from YAML (demoted to provenance block in body). cross-surface-plugin-packaging.md also gained all base canonical fields (title, description, status, confidence, related) and a new provenance block. CLAUDE.md extended with optional `session` + `linear` provenance block lines.
