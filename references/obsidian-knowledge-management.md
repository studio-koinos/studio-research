---
title: "Obsidian-Style Knowledge Management for Git Repos"
description: "Deep research on Obsidian-compatible markdown patterns (frontmatter, wikilinks, MOC, citations, tools) for git-based knowledge repos. Evaluated and rejected for studio-research in favor of llm-wiki pattern."
tags: [knowledge-management, obsidian, markdown, git, wikilinks, moc]
modified: 2026-04-13
status: active
confidence: high
staleness: 60
related:
  - ./agent-kb-structure.md
---

# Obsidian-Style Knowledge Management for Git Repos

> **Research date:** 2026-04-12
> **Last verified:** 2026-04-12
> **Staleness warning:** Re-verify tool compatibility (Quartz, Foam) after 60 days. Frontmatter conventions are stable.
> **Confidence summary:** 9 verified claims, 4 likely, 1 unverified (Anthropic/Ethereum team specifics from Perplexity — not independently confirmed)

## TL;DR

**VERIFIED**: Obsidian-compatible markdown uses YAML frontmatter between `---` delimiters, `[[wikilinks]]` for cross-references, and a `meta/index.md` or `MOC-*.md` file as the navigational hub. The format is pure markdown — any text editor reads it. **VERIFIED**: Two tools make it work in a plain git repo without the app: Foam (VS Code extension) for editing with wikilinks/backlinks, and Quartz for publishing to a searchable static site. **LIKELY**: The hybrid folder structure (numbered stages + topic subtrees) works best for a repo that accumulates research over time. **VERIFIED**: The recommended citation pattern is BibTeX keys in frontmatter `sources:` + inline `[Author Year]` references + a `references/bibliography.md` master list.

---

## How It Works

### 1. Frontmatter YAML Schema

#### Standard Fields (recognized by Obsidian natively)

**VERIFIED** — these fields have special meaning in Obsidian and are preserved by Quartz and Foam:

```yaml
---
title: "Zero-Knowledge Proofs in Practice"
aliases:
  - "ZKP"
  - "zero-knowledge"
tags:
  - cryptography
  - zero-knowledge
  - research
date: 2026-04-12
modified: 2026-04-12
cssclasses:
  - wide
---
```

Field semantics:
- `title` — display name; can differ from filename. Quartz uses it as the page `<title>`.
- `aliases` — alternative names that wikilinks can target. `[[ZKP]]` finds a file with `aliases: [ZKP]`.
- `tags` — list; supports hierarchy with `/` (e.g., `crypto/zk`). Searchable in Obsidian, Foam, Quartz.
- `date` — creation date; ISO 8601 (`YYYY-MM-DD`). Sort and filter key.
- `modified` — last edited; update manually or via a git hook.
- `cssclasses` — rendering hints (wide layout, card view). Ignored by non-Obsidian tools; no harm leaving it.

#### Research-Specific Custom Fields (recommended schema)

**LIKELY** — synthesized from multiple vault templates and community patterns:

```yaml
---
title: "Raft Consensus Algorithm"
date: 2026-03-15
modified: 2026-04-10
status: "review-pending"       # draft | in-progress | review-pending | published | archived
confidence: "high"             # low | medium | high — applies to the claims in this note
type: "research-note"          # research-note | literature-review | experiment-log | project-log | moc
tags:
  - distributed-systems
  - consensus
  - raft
sources:
  - Ongaro2014                  # BibTeX-style keys referencing bibliography.md
  - Lamport1978
reviewed_by: []
reviewed_date: null
related_topics:
  - "[[Paxos Protocol]]"
  - "[[Byzantine Fault Tolerance]]"
project: "consensus-research"  # optional — ties to a project folder
---
```

Field guide:
- `status` tracks the note's readiness. Searchable via `grep` or Dataview.
- `confidence` is epistemological — marks how much to trust the claims in this note. Useful for research that includes early-stage hypotheses.
- `type` enables MOC queries: `type: moc` finds all index notes.
- `sources` holds citation keys; the actual citation lives in `references/bibliography.md`.
- `related_topics` uses wikilinks — these become backlinks automatically in Obsidian/Foam/Quartz.

#### YAML Syntax Rules

```yaml
# Strings with colons or special chars need quotes
title: "Git: A Distributed Model"

# Tags are always a list — never a bare string
tags:
  - distributed-systems
  - consensus
# or inline:
tags: [distributed-systems, consensus]

# Dates — always ISO 8601, no quotes needed
date: 2026-04-12

# Booleans
draft: false
under_review: true

# Null / empty
reviewed_by: []
reviewed_date: null
```

### 2. Folder Structure

Three patterns exist in the wild. The hybrid works best for a repo accumulating research over time.

#### Pattern A: Numbered Stages (Zettelkasten-inspired)

```
research/
├── 00-inbox/          # raw captures, unprocessed notes
├── 10-sources/        # literature notes, paper summaries
│   ├── papers/
│   ├── talks/
│   └── tools/
├── 20-concepts/       # evergreen ideas and definitions
│   ├── distributed-systems/
│   └── cryptography/
├── 30-projects/       # active research
│   └── 2026-q2-zk-proofs/
├── 40-published/      # finished artifacts
└── meta/
    ├── index.md       # master MOC
    └── bibliography.md
```

**Tradeoffs:** Clear maturity stages; easy to implement a "process inbox weekly" workflow. Risk: inbox backlog if not maintained. Evergreen concepts don't fit the linear pipeline well.

#### Pattern B: Topic-Based

```
research/
├── distributed-systems/
│   ├── consensus/
│   └── fault-tolerance/
├── cryptography/
│   ├── zero-knowledge/
│   └── signatures/
├── meta/
│   ├── 0-inbox.md
│   ├── index.md
│   └── bibliography.md
└── projects/
    └── 2026-post-quantum/
```

**Tradeoffs:** Mirrors the mental model; natural discovery within a domain. Works well once you have 50+ notes per topic. Progress tracking (draft vs. published) requires frontmatter queries, not folder inspection.

#### Pattern C: Hybrid (Recommended by Obsidian community)

**LIKELY** — recommended by Perplexity synthesis and consistent with multiple template repos:

```
research/
├── 0-inbox/                     # everything unprocessed lands here
├── 1-references/                # literature: papers, talks, tools
│   └── bibliography.md          # master citation list
├── 2-concepts/                  # evergreen ideas, stable knowledge
│   ├── distributed-systems/
│   ├── cryptography/
│   └── machine-learning/
├── 3-projects/                  # active, time-bounded research
│   └── 2026-q2-zk-proofs/
├── 4-published/                 # done, archived
└── meta/
    ├── index.md                 # master MOC
    └── tags.md                  # tag taxonomy reference
```

Why this works for a git repo:
- `git log -- 0-inbox/` shows recent captures clearly
- Folder numbers sort correctly in any file browser
- Concepts and references are separated — concepts get updated; references are append-only
- Projects have natural start/end dates and can be archived to `4-published/`

### 3. Map of Content (MOC) Pattern

An MOC is a **manually curated index note** — not a folder `README.md` and not auto-generated. It reflects your current understanding of a topic's structure.

#### Anatomy of an MOC

```markdown
---
title: "Map of Consensus Mechanisms"
type: "moc"
date: 2025-09-01
modified: 2026-03-15
tags:
  - consensus
  - distributed-systems
  - index
---

# Map of Consensus Mechanisms

**Purpose**: Navigate consensus algorithm research. Entry point for this domain.

## Foundational (read first)
- [[Byzantine Fault Tolerance]] — defines the problem space
- [[Proof of Work]] — original production consensus; efficiency tradeoffs
- [[Paxos Protocol]] — theoretical standard

## Intermediate
- [[Practical Byzantine Fault Tolerance (PBFT)]] — production-ready BFT
- [[Raft Consensus]] — Paxos alternative, easier to implement

## Advanced
- [[Proof of Stake]] — PoW alternative for blockchains
- [[Optimistic Rollups]] — L2 consensus

## Problem Space Map

Consensus Problem
├── Safety (nothing bad happens)
├── Liveness (good things eventually happen)
└── Byzantine resilience (requires f < n/3 honest nodes)

## Key Papers

| Year | Authors | Paper | Notes |
|------|---------|-------|-------|
| 1978 | Lamport et al. | Byzantine Generals Problem | Foundational |
| 1999 | Castro & Liskov | PBFT | Production-ready BFT |
| 2014 | Ongaro & Ousterhout | Raft | Understandable consensus |

## Connections to Adjacent Areas
- Distributed Systems: [[Distributed Ledgers]], [[State Machine Replication]]
- Cryptography: [[Digital Signatures]], [[Merkle Trees]]

## Maintenance
- Last reviewed: 2026-03-15
- Update trigger: new consensus models, or when any linked note's status changes
```

#### MOC Design Principles

1. **Organize by depth of understanding**, not alphabetically. Foundational → advanced.
2. **Show the problem space** — a brief decomposition of what the domain is trying to solve.
3. **Include a key papers table** — this is the lightweight literature review.
4. **Link to adjacent domains** — this is what makes backlinks useful.
5. **Add a maintenance block** — `Last reviewed` and `Update trigger` prevent stale indexes.
6. **Name convention**: prefix with `MOC-` or use `type: moc` in frontmatter. Both approaches are common; the frontmatter approach is more searchable.

### 4. Citation and Source Management

**VERIFIED** — four patterns exist; the hybrid is best for a technical git repo:

#### Pattern 1: BibTeX Keys + Central Bibliography (Recommended by Obsidian community)

Store all citations in `1-references/bibliography.md`:

```markdown
---
title: "Master Bibliography"
type: "reference"
---

# Bibliography

## Distributed Systems

[Ongaro2014]: https://raft.github.io/raft.pdf
> Ongaro, D., & Ousterhout, J. (2014). In Search of an Understandable Consensus Algorithm.
> USENIX Annual Technical Conference. Retrieved: 2026-04-12.

[Castro1999]: https://pmg.csail.mit.edu/papers/osdi99.pdf
> Castro, M., & Liskov, B. (1999). Practical Byzantine Fault Tolerance.
> OSDI '99. Retrieved: 2026-04-12.
```

#### Pattern 2: Markdown Footnotes (Lightweight, GitHub-Readable)

```markdown
Raft achieves sub-second leader election[^1] and is widely deployed in production systems[^2].

[^1]: Ongaro, D., & Ousterhout, J. (2014). In Search of an Understandable Consensus Algorithm. https://raft.github.io/raft.pdf
[^2]: etcd, CockroachDB, TiKV all use Raft. As of 2026-04-12.
```

#### Pattern 3: Inline Author-Year

```markdown
The Byzantine Generals Problem (Lamport et al., 1978) established that f Byzantine faults
require 3f+1 nodes. PBFT (Castro & Liskov, 1999) made this practical.
```

#### Pattern 4: Wikilink to a Source Note

Create a dedicated note per major paper:

```
1-references/papers/Castro1999-PBFT.md
```

#### Recommended Hybrid for a Technical Research Repo

1. Frontmatter `sources:` field lists BibTeX keys for the note.
2. Inline text uses `Author (Year)` format for readability.
3. Each note has a `## References` section at the bottom with full citations + URLs + retrieved dates.
4. `1-references/bibliography.md` aggregates all citations as the authoritative list.
5. For papers that need deep analysis, create a dedicated source note and wikilink to it.

**Retrieved date is mandatory** on every URL — research repos live for years and URLs rot.

### 5. Tools That Work Without the Obsidian App

**VERIFIED** — tested/documented by their respective projects:

| Tool | Wikilinks | Backlinks | Frontmatter | Best For | Cost |
|------|-----------|-----------|-------------|----------|------|
| **Foam** (VS Code) | Native | Auto graph | Full YAML | Primary editing in VS Code | Free |
| **Quartz** (static SSG) | Converts to HTML | Auto panels | Full YAML | Publishing as searchable site | Free |
| **zoni/obsidian-export** (CLI) | Converts to relative links | None | Preserved | One-way export to standard MD | Free |
| **Dendron** (VS Code) | Hierarchical | Auto | Extended YAML | Hierarchical vaults | Free |
| **MkDocs Material** | Plugin needed | Manual | Extracted | Styled docs sites | Free |
| **LogSeq** | Native | Auto | Different format | Outliner UX (not recommended — format drift) | Free |

#### Foam (Primary Recommendation for Editing)

Install: VS Code → Extensions → search "Foam for VS Code"

What it does in a plain git repo:
- Resolves `[[wikilinks]]` to filenames — click to navigate
- Shows a backlinks panel (which notes link to the current file)
- Renders a knowledge graph in the editor
- No lock-in — it's just a VS Code extension reading plain `.md` files
- Works with any git workflow — stage, commit, push normally

#### Quartz (Primary Recommendation for Publishing)

Quartz converts an Obsidian-format vault to a fast static site with full-text search, graph view, and backlink panels. It's the closest to "Obsidian Publish" without the subscription.

Quartz respects these frontmatter fields natively (VERIFIED from docs):
- `title`, `description`, `aliases`, `tags`, `date`, `draft`, `permalink`, `cssclasses`

Wikilinks, callouts, Mermaid diagrams, LaTeX, footnotes, tables — all supported.

---

## Current State

**LIKELY** — from Perplexity synthesis; specific org details not independently verified:

Teams using Obsidian-style markdown in git repos follow these patterns:

**Engineering teams** (Foam + Quartz + GitHub Pages): VS Code as primary editor, Foam for navigation, Quartz deployed via GitHub Actions for public-facing knowledge bases. Research notes stay in `main`; WIP in feature branches.

**Research organizations**: Numerated stage folders (inbox → concepts → projects); heavy MOC usage; citations as BibTeX keys; status field tracks paper-submission pipeline. Git PRs used for peer review of research notes.

**Open-source knowledge bases**: Many public GitHub repos use Obsidian-compatible markdown with topic-based folders and `README.md` as MOC per folder. No frontmatter (prioritizes accessibility) but consistent wikilink syntax and a central `index.md`.

**VERIFIED** — common across all observed patterns:
- YAML frontmatter with at minimum `title`, `date`, `tags`
- `[[wikilinks]]` for cross-references (not `[Title](path.md)` — the latter breaks when files move)
- A `meta/` or `_index/` folder that is git-tracked but excluded from any published output
- `bibliography.md` or `references.md` as the citation anchor point
- `type: moc` in frontmatter for index notes, enabling `grep -r "type: moc"` to list all indexes

---

## Applicability to Our Stack

**Evaluated and rejected for studio-research (2026-04-13).** The Obsidian pattern was the initial candidate for structuring the centralized knowledge base. After deep research comparing this approach against alternatives (see [Agent Knowledge Base Structure](./agent-kb-structure.md)), the llm-wiki pattern was chosen instead.

**Why rejected:**
- `[[wikilinks]]` require semantic inference to resolve to file paths — agents navigate by `Glob` and `Grep`, not by browsing a graph view
- Numbered folders (`0-inbox/`, `1-references/`) carry no semantic meaning for agent pattern matching
- MOC pattern assumes human curation and visual navigation, not agent-driven discovery
- The hybrid folder structure optimizes for human browsing, not for `grep -r "tags:.*keyword"` queries

**What was adopted from this research:**
- YAML frontmatter convention (simplified to 8 fields in studio-research schema)
- `confidence` field concept (retained)
- `staleness` concept (adapted: per-doc days-to-recheck instead of Obsidian's Dataview queries)
- Flat `index.md` as a catalog (simplified from the MOC pattern)

**This doc remains active as a reference** — it documents how the Obsidian pattern works, which is useful context for understanding why specific design decisions were made differently for studio-research.

See also: [design-studio-research.md](../../studio-pm/docs/design-studio-research.md) for the full decision rationale.

---

## Conflicts and Gaps

**No conflicts between sources** — all streams agreed on core conventions.

**Open questions:**
- Dendron's dot-notation hierarchy (`domain.subdomain.topic`) is an alternative organizational model that some teams prefer — not evaluated deeply here
- Automated MOC generation (Dataview plugin, obsidian-moc-index-generator) requires Obsidian or a plugin runtime — not available in plain git without a build step
- LogSeq is explicitly not recommended: its metadata format uses `:PROPERTIES:` blocks that differ from YAML frontmatter and require conversion scripts to interoperate

---

## Key Sources

1. Perplexity sonar-deep-research synthesis (10 search queries) — 2026-04-12
2. Quartz authoring content docs: https://quartz.jzhao.xyz/authoring-content — 2026-04-12
3. Quartz Obsidian compatibility: https://quartz.jzhao.xyz/features/Obsidian-compatibility — 2026-04-12
4. Foam VS Code extension: https://foamnotes.com/ — 2026-04-12
5. zoni/obsidian-export (Rust CLI): https://github.com/zoni/obsidian-export — 2026-04-12
6. Obsidian forum — YAML frontmatter discussion: https://forum.obsidian.md/t/how-do-you-put-yaml-to-use-in-your-system/18987 — 2026-04-12
7. kepano/kepano-obsidian vault template: https://github.com/kepano/kepano-obsidian — 2026-04-12
8. voidashi/obsidian-vault-template: https://github.com/voidashi/obsidian-vault-template — 2026-04-12
9. LalieA/obsidian-scientific-research-vault: https://github.com/LalieA/obsidian-scientific-research-vault — 2026-04-12
10. MOC pattern guide (Shuvangkar Das): https://blog.shuvangkardas.com/obsidian-moc-map-of-content/ — 2026-04-12
11. Nested YAML frontmatter blog: https://bbbburns.com/blog/2025/07/nested-yaml-frontmatter-for-obsidian-book-notes/ — 2026-04-12
12. seqis/ObsidianMOC: https://github.com/seqis/ObsidianMOC — 2026-04-12
13. leolaurindo/obsidian-moc-index-generator: https://github.com/leolaurindo/obsidian-moc-index-generator — 2026-04-12
14. Foam GitHub repo: https://github.com/foambubble/foam — 2026-04-12
