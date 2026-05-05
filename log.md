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

## [2026-05-05] add | Orchestration Layer Internals — Empirically Grounded Synthesis (landscape, S-70 / AO-263 / AO-270)
Synthesis output of the AO-263 expertise phase: 7 singleton deep-dives (cmux 0.63.2, Claude Code CLI/SDK 2.1.128, hooks, MCP, auth/connectors, tmux/iTerm2, Codex CLI 0.128.0) composed under "official sources first + paste-the-output" methodology per `feedback_charter_before_ratify.md`. Output: 14 architectural findings + 9 known gaps + 3 falsifications including a self-referential audit-document falsification (decisions/008 §2 premise falsified by AO-266 expertise-phase finding). Supersedes landscapes/cmux-features.md (which contained 2 falsified claims — Feed CLI surface, workstream.jsonl audit log). Drives orchestrator v2 design discussion.

## [2026-05-05] update | Orchestration Layer Internals — frontmatter + filing-workflow hygiene + Cat-2 architect defensibility patches
Two-pronged review: codex CLI review surfaced 3 hygiene findings (missing YAML frontmatter; missing index/log updates; broken pipe in §9 table at L658 `env | grep OPENAI_API_KEY` parsed as 2 columns); architect-tier review (Tier 1 opus + max) verdict = DEFENSIBLE-WITH-PATCHES with 0 Cat-1, 4 Cat-2, 3 Cat-3 (deferred). All hygiene + Cat-2 patches landed in this update:
- (hygiene) Frontmatter prepended matching sibling conventions; index + log entries added; L658 pipe escaped
- (Cat-2.1) §3.4 ANTHROPIC_API_KEY env-strip claim — added inline citations to claude-code.md §"× auth", auth.md §1 Test 5, and memory/project_ao213_scope_gap.md (subprocess vs in-process asymmetry); singletons cover subprocess side, memory covers in-process side
- (Cat-2.2) §6.2 citation pointer fix: [mcp.md §"open questions"] → [mcp.md §"Plaintext bearer-token exposure" + §"open questions" Q3] (load-bearing GitHub PAT evidence is in the Plaintext-bearer subsection, not §open-questions)
- (Cat-2.3) §1 Fact 5 + §7 Q1 path 3 — softened "cmux wait-for" to "hypothetical cmux blocking-call primitive (name unverified)"; architect grep-confirmed cmux.md does NOT enumerate `wait-for`; flagged for fresh empirical probe before reliance
- (Cat-2.4) §3.5 + §1 Fact 1 — verified FALSIFICATION + custom-agent caveat remain in same paragraph block (no change needed; architect noted "currently they are")
Cat-3 follow-ups deferred per peer-review-scope.md: regex format verification for `lin_api_*` extension (Linear capture against AO-273), no-action-required (Q9 already correctly framed), and §3.10 expansion only-if-load-bearing.
