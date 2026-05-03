---
role: Centralized research knowledge base for the studio fleet — tool evaluations, architecture analysis, integration guides
substrate: capability
linear-team: RES
status: active
owner: aliargun
loop-position: producer
last-reviewed: 2026-05-03
---

# CLAUDE.md — studio-research

## What This Is

Centralized knowledge base for the studio ecosystem. Stores deep research outputs — tool evaluations, architecture analysis, tech comparisons, integration guides — consumed primarily by AI agents during coding sessions. **Knowledge base, not a tool.** No code, no services, no deployments. Plain markdown in git.

## Substrate Class

**Capability substrate. Producer.** Capability-knowledge base — knowledge *about how to build capability*, not per-tenant graph data. Consumers are exclusively Capability-substrate agents (studio-agent-ops, studio-pm, studio-radar, studio-scout). Distinct from Knowledge substrate (koinos-context's per-tenant graph) — vocabulary collision per ADR-037 §Required follow-ups.

## Scope

### In scope (this repo authors)

- `evaluations/` — tool/product viability assessments (typically Scout outputs)
- `references/` — architecture research, technology deep dives, tech comparisons
- `guides/` — step-by-step integration / implementation guides
- `landscapes/` — market surveys, tool landscape analyses
- `archive/` — obsolete research, preserved for provenance
- `index.md` — flat catalog with one-line summaries by category
- `RESOLVER.md` — agent decision tree for filing
- `log.md` — append-only audit trail

### Out of scope (consumed, not authored)

| Out-of-scope content | Authoritative location |
|---|---|
| Code, scripts, tools | respective repos |
| Session logs | each repo's `sessions/` |
| ADRs (cross-substrate architecture) | `koinos-pm/adrs/` |
| Ephemeral task state | `.context/` in respective repo |
| Memory files | `~/.claude/projects/*/memory/` |
| Knowledge graph engine + per-tenant data | `koinos-context` |

### Cross-repo flows

| Flow | Direction | Mechanism | Status |
|---|---|---|---|
| Research consumption | studio-* repos → this repo | `Glob ../studio-research/**/*.md` + `Grep` | live |
| Scout output filing | studio-scout → `evaluations/` | manual file (per RESOLVER.md) | live |
| Capability registry | this repo → studio-agent-ops AO-104 design | manual reference (`landscapes/agentic-fleet-platforms.md`) | live |
| Knowledge graph indexing (aspirational) | this repo → koinos-context | tagged-for-cross-fleet-relevance pipeline | aspirational |

### Anti-patterns (PR review pushes back)

- **Authoring code, scripts, or tools here** — markdown only.
- **Filing without `RESOLVER.md` consultation** — read it first; categories matter.
- **`[[wikilinks]]`** — agents can't resolve them; use relative paths.
- **Undated sources** — every URL must have access/publication date in Key Sources.
- **Bypassing the provenance block** — `Research date / Last verified / Staleness warning / Confidence summary` is mandatory.

### When scope bends

If a piece of work doesn't fit the predicate above, route per ADR-037 §Decision-routing rule:

- **Architectural** → `koinos-pm/adrs/`
- **Operational** (capability fleet runtime) → `studio-pm/decisions.jsonl`
- **Commercial / brand / strategic / equity** → `koinos-os/decisions.md`
- **Cross-cutting** → `koinos-pm/adrs/` canonical + cross-link

## Repo orientation

```
evaluations/    Tool/product viability (Scout outputs)
references/     Architecture, deep dives, comparisons
guides/         Integration / implementation guides
landscapes/     Market + tool landscape surveys
archive/        Obsolete research (preserved for provenance)
RESOLVER.md     Agent decision tree (where does a new doc go?)
index.md        Flat catalog
log.md          Append-only audit trail
```

## Key references

- **Page types + required sections:** see Evaluations / References / Guides / Landscapes sections in this file's predecessor (each page type has required headings)
- **Frontmatter schema:** `title`, `description`, `tags`, `modified`, `status`, `confidence`, `staleness`, `related`
- **Provenance block (mandatory):** `Research date / Last verified / Staleness warning / Confidence summary`
- **Cross-references:** standard relative markdown links, not wikilinks
- **Filing workflow:** RESOLVER.md → directory README → write doc → update index.md → append log.md

## Linear

Team `RES` (RESEARCH, parent: STUDIO).
