# RES-2: Migrate research docs from agent-ops to studio-research

## Acceptance criteria

- [ ] All 5 source docs migrated into studio-research with correct frontmatter + required sections for page type
- [ ] `index.md` populated with one-line entry per migrated doc under correct category heading
- [ ] `log.md` has one `migrate` entry per doc
- [ ] No broken cross-references (relative paths only, no wikilinks)
- [ ] Source files cleaned up: remove originals from agent-ops/research/ and studio-pm/research/, or replace with a one-line pointer to the new location

## Doc migration list

### 1. obsidian-knowledge-management.md — ALREADY DONE (untracked, needs commit)
- Source: `studio-agent-ops/research/obsidian-style-knowledge-management.md`
- Target: `references/obsidian-knowledge-management.md` ← file already exists, untracked
- Action: commit as-is, update index.md, append log.md, clean up source

### 2. claude-code-config.md
- Source: `../studio-agent-ops/research/claude-code-official-config-reference.md`
- Target: `references/claude-code-config.md`
- Type: reference (deep dive into Claude Code config system)
- Action: read source, write with correct frontmatter + sections (TL;DR, How It Works, Current State, Applicability to Our Stack, Key Sources), update index.md, append log.md, clean up source

### 3. agent-kb-structure.md
- Source: `../../studio-pm/research/agent-knowledge-base-structure.md`
- Target: `references/agent-kb-structure.md`
- Type: reference (architecture research on agent knowledge base structures — llm-wiki vs gbrain vs Obsidian)
- Action: read source, write with correct frontmatter + sections, update index.md, append log.md, clean up source

### 4. claude-code-config-version-control.md
- Source: `../studio-agent-ops/research/claude-code-config-version-control.md`
- Target: determine via RESOLVER.md — likely `references/` (tech deep dive) or `guides/` (step-by-step). Read source first to decide.
- Action: route via RESOLVER, write with correct frontmatter + sections for chosen type, update index.md, append log.md, clean up source

### 5. continuous-agent-ops-patterns.md
- Source: `../studio-agent-ops/research/continuous-agent-ops-patterns.md`
- Target: determine via RESOLVER.md — likely `references/` (patterns = architecture research). Read source first to confirm.
- Action: route via RESOLVER, write with correct frontmatter + sections, update index.md, append log.md, clean up source

## Frontmatter schema (from CLAUDE.md)

```yaml
---
title: "Document Title"
description: "One-line summary for agent relevance screening"
tags: [tag1, tag2, tag3]
modified: 2026-04-13
status: active
confidence: high          # high | medium | low
staleness: 30             # days — fast-moving AI tech = 14, stable standards = 90
related:
  - ../references/other-doc.md
---
```

## Schema rules
- `description` is how agents decide whether to read — make it specific
- `confidence`: high = multiple independent sources, medium = single credible source, low = unverified
- `staleness`: Claude Code configs = 14 days (fast-moving), patterns/architecture = 60 days
- `related` uses relative file paths, not wikilinks

## log.md format

```
## [2026-04-13] migrate | Title (from studio-agent-ops)
```

## index.md format

```
- [Title](path/to/doc.md) — one-line summary
```
Under the correct category heading (## References, ## Guides, etc.)

## Source cleanup rule

After migrating each doc, either:
- Delete the original from the source repo (preferred if no other references)
- Or replace with: `# [Title]\n\nMigrated to studio-research. See [references/doc.md](../studio-research/references/doc.md).`
