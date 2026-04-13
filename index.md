# Research Index

Flat catalog of all documents. One-line summary per entry. Updated on every add/update/migrate/archive.

**Quick access:** `grep "^- \[" index.md` lists all entries. `grep "tags:.*keyword" evaluations/*.md references/*.md` finds by topic.

## Evaluations

- [Context7 MCP Server](evaluations/context7-mcp.md) — MCP server for fetching current library/framework docs; verdict: adopt

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

## Archive

<!-- Obsolete or superseded research. Format: - [Title](path) — archived YYYY-MM-DD, reason -->
