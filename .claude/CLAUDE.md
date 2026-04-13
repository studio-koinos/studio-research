# studio-research

Centralized knowledge base for the studio ecosystem. Stores deep research outputs — tool evaluations, architecture analysis, tech comparisons, integration guides — consumed primarily by AI agents during coding sessions.

**This is a knowledge base, not a tool.** No code, no services, no deployments. Plain markdown in git.

**Owner:** Ali Argun, Koinos Studio founder.

## How agents use this repo

Agents in other studio repos discover research here via `Glob` and `Grep` on `../studio-research/`. The workflow:

1. **Before starting new research**, check if it already exists here
2. **Read `index.md`** — flat catalog with one-line summaries, organized by category
3. **Grep for keywords** — frontmatter `tags` and `description` fields are designed for this
4. **Read the relevant doc** — each doc is self-contained with dated sources

## Structure

```
studio-research/
├── CLAUDE.md              # this file — schema and conventions
├── RESOLVER.md            # agent decision tree: where does a new doc go?
├── index.md               # flat catalog: every doc, one-line summary, by category
├── log.md                 # append-only audit trail
├── evaluations/           # tool/product evaluations
├── references/            # architecture research, tech deep dives
├── guides/                # integration guides, how-tos
├── landscapes/            # market/tool landscape surveys
└── archive/               # obsolete research, preserved for provenance
```

## Page types and required sections

Section headings are conceptual, not literal. A doc satisfies "How It Works" with any heading that explains the mechanism (e.g. "Architecture", "How Claude Discovers Files"). Match headings to the doc's natural phrasing.

### Evaluations (evaluations/)
Tool or product viability assessments. Typically Scout outputs.

Required sections: **TL;DR**, **What It Is**, **Viability Assessment** (adopt / defer / reject with reasoning), **Tradeoffs**, **Key Sources**
Optional: **Conflicts & Gaps** — tensions between sources, contradictions, or open questions worth flagging

### References (references/)
Architecture research, technology deep dives, tech comparisons.

Required sections: **TL;DR**, **How It Works**, **Current State**, **Applicability to Our Stack**, **Key Sources**
Optional: **Conflicts & Gaps** — tensions between sources, contradictions, or open questions worth flagging

### Guides (guides/)
Step-by-step integration or implementation guides.

Required sections: **Prerequisites**, **Steps**, **Verification**, **Troubleshooting**, **Key Sources**

### Landscapes (landscapes/)
Market surveys, tool landscape analyses, category overviews.

Required sections: **TL;DR**, **Landscape Map**, **Comparison Table**, **Recommendation**, **Key Sources**

## Frontmatter schema

Every doc starts with this YAML frontmatter:

```yaml
---
title: "Document Title"
description: "One-line summary for agent relevance screening"
tags: [tag1, tag2, tag3]
modified: 2026-04-13
status: active            # active | draft | deprecated | archived
confidence: high          # high | medium | low
staleness: 30             # days before re-verify (per-doc)
related:
  - ../references/other-doc.md
---
```

Field rules:
- `description` is how agents decide whether to read the full doc — make it specific
- `modified` is the last substantive edit date (not typo fixes)
- `staleness` varies by domain: fast-moving tech (AI APIs) = 14 days, stable standards = 90 days
- `related` uses relative file paths, not wikilinks
- `confidence`: high = multiple independent sources, medium = single credible source, low = unverified

## Provenance block

Every doc includes a provenance block immediately after the frontmatter, before the first heading:

```markdown
> **Research date:** YYYY-MM-DD
> **Last verified:** YYYY-MM-DD
> **Staleness warning:** Re-verify after N days. [Why this area changes fast.]
> **Confidence summary:** X VERIFIED claims, Y LIKELY claims, Z UNVERIFIED claims
```

This block is mandatory. It makes staleness and verification status visible to agents without reading the full doc.

## Cross-referencing

Use standard relative markdown links: `[Other Doc](../references/other-doc.md)`

**Do not use `[[wikilinks]]`.** Agents cannot resolve them without semantic inference. Relative paths are directly readable.

## Filing workflow

When adding a new doc:

1. Read `RESOLVER.md` — determine which directory
2. Read that directory's `README.md` — confirm the doc belongs there
3. Write the doc with correct frontmatter + required sections for the page type
4. Update `index.md` — add entry under the correct category with one-line summary
5. Append to `log.md` — `## [YYYY-MM-DD] add | Document Title`

## Key Sources section (mandatory on every doc)

Every doc ends with a **Key Sources** section. Every URL must have an access/publication date:

```markdown
## Key Sources

1. Source Name: URL (YYYY-MM-DD)
2. Source Name: URL (YYYY-MM-DD)
```

## What does NOT belong here

- Code, scripts, or tools (those go in their respective repos)
- Session logs (those stay in each repo's sessions/)
- ADRs (those go in koinos-pm/adrs/)
- Ephemeral task state (that goes in .context/)
- Memory files (those stay in ~/.claude/projects/*/memory/)

## Linear

- Team: RESEARCH (RES)
- Project: Research — centralized knowledge base
