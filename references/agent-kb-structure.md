---
title: "AI Agent Knowledge Base Structure"
description: "How Claude Code, Cursor, and Codex discover and consume repo context. Covers three-tier architecture, Karpathy llm-wiki pattern, AGENTS.md, and file naming conventions for agent-first repos."
tags: [claude-code, knowledge-base, agents, context-engineering, karpathy, agents-md, cursor]
modified: 2026-04-13
status: active
confidence: high
staleness: 30
related:
  - ./obsidian-knowledge-management.md
  - ./claude-code-config-reference.md
---

# AI Agent Knowledge Base Structure

> **Research date:** 2026-04-13
> **Last verified:** 2026-04-13
> **Staleness warning:** Re-verify after 30 days. Cursor rule format, CLAUDE.md features, and AGENTS.md ecosystem all saw major changes in 2025. Anything older than a month may be stale.
> **Confidence summary:** 12 VERIFIED claims, 6 LIKELY claims, 2 UNVERIFIED claims

## TL;DR

AI coding agents discover context through specific auto-loaded files (CLAUDE.md, AGENTS.md, `.cursor/rules/`) and on-demand file reading tools. The dominant pattern that has emerged by 2026 is a **three-tier architecture**: hot memory (always loaded, ~200 lines), path-scoped rules (loaded when matching files are touched), and cold memory (on-demand documents agents retrieve with Grep/Read). Karpathy's llm-wiki pattern — a `raw/` → `wiki/` compiler with `index.md` + `log.md` as agent navigation aids — is the most-adopted community template for pure knowledge repos. AGENTS.md is now an industry standard backed by the Linux Foundation, with 60,000+ repos adopting it. Key practical insight: **the primary reader being an AI agent changes nothing about file format (still Markdown) but changes everything about file organization** — agents navigate by grep/glob, not by browsing, so naming conventions and index files matter more than visual structure.

---

## How It Works

### Auto-Load vs. On-Demand Access

There are two distinct access patterns for every coding agent:

1. **Auto-loaded at session start**: specific files the tool reads unconditionally before the user prompt
2. **On-demand**: files the agent discovers via tool calls (Glob, Grep, Read) during a session

These have very different implications for repo design.

### Claude Code — CLAUDE.md System

**VERIFIED** — from official docs at https://code.claude.com/docs/en/memory (2026-04-13)

**Auto-loaded files (before first user message):**
- `~/.claude/CLAUDE.md` — user-global, all projects
- `/Library/Application Support/ClaudeCode/CLAUDE.md` — org-managed policy (macOS)
- `./CLAUDE.md` or `./.claude/CLAUDE.md` — project root (team-shared)
- `./CLAUDE.local.md` — personal project prefs (gitignored)
- CLAUDE.md files in all **parent directories** up the tree
- `.claude/rules/*.md` — unconditional rules loaded at launch

**On-demand (loaded when Claude reads files in that directory):**
- CLAUDE.md files in **subdirectories** — only loaded when Claude touches files in that subdirectory
- `.claude/rules/*.md` files with `paths:` frontmatter — loaded when matching files are opened
- `~/.claude/projects/<repo>/memory/MEMORY.md` — first 200 lines or 25KB auto-loaded; topic files read on demand

**Key mechanics:**
- `@path/to/file` import syntax in CLAUDE.md expands and loads referenced files at launch (up to 5 hops deep)
- HTML comments `<!-- like this -->` are stripped before context injection — free metadata for human maintainers
- **Target: under 200 lines per CLAUDE.md** — files over 200 lines reduce adherence
- All CLAUDE.md files concat into context (not override) — local takes precedence within a directory
- `claudeMdExcludes` setting can block specific patterns (useful in monorepos)

### Cursor — `.cursor/rules/` System

**VERIFIED** — from official docs at https://cursor.com/docs/rules (2026-04-13)

**File formats supported:**
- `.mdc` — MDC (Markdown with YAML frontmatter for metadata) — current recommended format
- `.md` — plain markdown — also supported
- `.cursorrules` (project root) — legacy format, still supported, not recommended for new projects

**Four rule attachment types** (in MDC frontmatter):
1. `alwaysApply: true` — included in every request
2. `globs: ["**/*.py"]` — auto-attached when editing matching files
3. `description: "..."` (no globs, no alwaysApply) — agent-requested; Cursor AI decides when to include based on description text
4. Manual — user explicitly adds via `@rule-name` in chat

**Key constraints:**
- Rules should stay under 500 lines
- Nested `.cursor/rules/` directories in subdirectories supported
- Symlinks in `.cursor/rules/` work — enables sharing rule sets across projects

### Codex CLI — AGENTS.md

**VERIFIED** — from https://agents.md/ and https://developers.openai.com/codex/guides/agents-md (2026-04-13)

AGENTS.md is a Linux Foundation project (Agentic AI Foundation, December 2025). It is a plain Markdown file with no required schema. Discovery rule: the agent reads the **closest** AGENTS.md in the directory tree to the file being edited.

**Supported by:** OpenAI Codex, Cursor, GitHub Copilot, Google Jules, Factory, Aider, goose, VS Code, Devin, Windsurf, Zed, Warp, RooCode, and 60,000+ open-source repos.

**Claude Code + AGENTS.md interop:** Claude Code reads `CLAUDE.md`, not `AGENTS.md`. Official recommendation: create a `CLAUDE.md` that imports `AGENTS.md` with `@AGENTS.md`.

### Karpathy's llm-wiki Pattern

**VERIFIED** — from gist: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f (2026-04-13)

The most widely-adopted community pattern for agent-consumed knowledge repos. The gist hit 5,000+ stars within days of posting.

**Three-layer architecture:**
```
knowledge-repo/
├── CLAUDE.md          # Schema: tells LLM how the wiki works, naming conventions, workflows
├── raw/               # Immutable source documents
└── wiki/
    ├── index.md       # Master catalog — every article with one-line summary, by category
    ├── log.md         # Append-only chronological record with grep-able prefixes
    ├── concepts/
    ├── entities/
    ├── sources/
    └── synthesis/
```

**How agents navigate:**
1. Read `index.md` to orient (one-line summaries per article, organized by category)
2. Follow links into specific wiki pages
3. Use `log.md` as a grep-able audit trail (`## [2026-04-02] ingest | Article Title` format)
4. Synthesize answers from multiple pages, optionally writing results back

**Scale guidance:** Works well at ~100 articles / ~400,000 words. The index + summary pattern allows navigation without loading everything into context.

### Three-Tier Architecture (Production Pattern)

**VERIFIED** — from arxiv paper https://arxiv.org/html/2602.20478v1 (2026-04-13), real-world deployment with 283 sessions / 108,000 lines of C#

**Tier 1 — Hot Memory (Constitution):**
- Single ~660-line Markdown file, auto-loaded every session
- Contains: code standards, naming conventions, build commands, architecture summaries, routing tables

**Tier 2 — Domain Specialist Agents:**
- 19 specialist agent files, ~9,300 lines total (115–1,233 lines each)
- Over half the content is project domain knowledge, not behavioral instructions
- Invoked per-task by the orchestrating agent based on routing tables in Tier 1

**Tier 3 — Cold Memory (Knowledge Base):**
- 34 on-demand specification documents, ~16,250 lines total
- One document per subsystem, written explicitly for AI consumption
- Accessed via keyword substring matching (not vector search) — simpler and sufficient

**Key finding:** Knowledge-to-code ratio of 24.2% in production. The primary failure mode was spec staleness — outdated docs mislead agents worse than no docs.

### File Naming Conventions That Help Agents

**VERIFIED/LIKELY** — synthesized from multiple sources (2026-04-13)

**What actually helps agents:**
- **Grep-able prefixes in log files:** `## [2026-04-02] ingest | Article Title` → enables `grep "ingest"` or `grep "2026-04"`
- **Typed filenames:** `user__.md`, `feedback__.md`, `reference__.md` — prefix signals how the LLM should process the file
- **`index.md` as the single entry point:** agents read this first to orient, then drill down
- **YAML frontmatter for path scoping:** `paths:` frontmatter in rules means agents only receive context when touching matching files
- **Numbered files for sequence:** `01_getting_started.md`, `02_api_reference.md` — communicates ordering

**What does NOT help agents:**
- Elaborate visual folder hierarchies — agents navigate by grep/glob, not browsing
- README.md as the sole entry point — agents may not know to start there
- Long prose explanations — agents perform better on concise, directive text

---

## Current State

AGENTS.md is now a Linux Foundation standard with 60,000+ repos adopting it. The Karpathy llm-wiki pattern is the dominant template for pure knowledge repos. Claude Code's CLAUDE.md system and Cursor's `.cursor/rules/` system are both actively evolving — symlink support, path-scoped rules, and import syntax all added or stabilized in 2025–2026.

What does NOT survive `/compact` in Claude Code: subdirectory CLAUDE.md files are not re-injected after compaction. Only the project-root CLAUDE.md survives compaction. Design for this: put critical context in the root CLAUDE.md or `@import` it explicitly.

---

## Applicability to Our Stack

This research directly informed the design of studio-research. Key decisions made based on these findings:

- **llm-wiki pattern adopted** (not Obsidian MOC) — agents navigate by grep/glob, not browsing; `index.md` + `log.md` is the right navigation surface
- **Flat `references/` directory** — descriptive filenames, shallow hierarchy, grep-friendly
- **`index.md` as single entry point** — all docs catalogued with one-line summaries
- **YAML frontmatter** with `tags` and `description` fields designed for grep-based discovery
- **`log.md`** with grep-able `## [YYYY-MM-DD] operation | Title` format

See also: the Obsidian pattern was evaluated and rejected — see [obsidian-knowledge-management.md](./obsidian-knowledge-management.md) for why.

---

## Conflicts and Gaps

**Conflicts:**
- CLAUDE.md vs AGENTS.md: some practitioners maintain both (CLAUDE.md imports AGENTS.md for shared content). Official Anthropic recommendation is the import pattern. Risk: drift between the two files.
- File size guidance: CLAUDE.md docs say "under 200 lines" while some practitioners say "under 300 lines." Official doc says 200.
- Context vs enforcement: CLAUDE.md is explicitly NOT enforced configuration — it's context Claude reads and tries to follow.

**Open gaps:**
- How Codex CLI specifically discovers files beyond AGENTS.md during a session is not documented publicly
- What happens to agent context during `/compact` with nested CLAUDE.md files — only project-root CLAUDE.md survives compaction
- Exact token cost curves for different structures are not publicly benchmarked

---

## Key Sources

1. Anthropic Claude Code memory docs: https://code.claude.com/docs/en/memory (2026-04-13)
2. Cursor rules docs: https://cursor.com/docs/rules (2026-04-13)
3. AGENTS.md official spec: https://agents.md/ (2026-04-13)
4. GitHub agentsmd/agents.md: https://github.com/agentsmd/agents.md (2026-04-13)
5. Karpathy llm-wiki gist: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f (2026-04-13)
6. VentureBeat on Karpathy llm-wiki: https://venturebeat.com/data/karpathy-shares-llm-knowledge-base-architecture-that-bypasses-rag-with-an (2026-04-13)
7. Codified Context arxiv paper: https://arxiv.org/html/2602.20478v1 (2026-04-13)
8. context-engineering-intro repo: https://github.com/coleam00/context-engineering-intro (2026-04-13)
9. claude-memory-compiler repo: https://github.com/coleam00/claude-memory-compiler (2026-04-13)
10. aarora79/personal-knowledge-base: https://github.com/aarora79/personal-knowledge-base (2026-04-13)
11. CLAUDE.md for Teams (DEV Community): https://dev.to/dr_hernani_costa/claudemd-for-teams-context-as-infrastructure-3hme (2026-04-13)
12. Augment Code AGENTS.md guide: https://www.augmentcode.com/guides/how-to-build-agents-md (2026-04-13)
13. Context Engineering for AI Agents (arxiv): https://arxiv.org/html/2510.21413v1 (2026-04-13)
14. Thomas Landgraf on context engineering: https://thomaslandgraf.substack.com/p/context-engineering-for-claude-code (2026-04-13)
15. Complete Guide to AI Agent Memory Files: https://medium.com/data-science-collective/the-complete-guide-to-ai-agent-memory-files-claude-md-agents-md-and-beyond-49ea0df5c5a9 (2026-04-13)
