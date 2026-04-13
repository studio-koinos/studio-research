# Notes for RES-2 builder

## Context

studio-research is an agent-first knowledge base (Karpathy llm-wiki pattern + gbrain RESOLVER concept). Plain markdown in git — no code, no services. Agents in other studio repos discover research here via Glob/Grep on `../studio-research/`.

The scaffold (RES-1) is live at `studio-koinos/studio-research`. Schema and conventions are in `.claude/CLAUDE.md`. Directory routing is in `RESOLVER.md`.

## Repo layout relative to studio-research

All paths are relative to `studio-research/`:
- Source docs in agent-ops: `../studio-agent-ops/research/`
- Source doc in studio-pm: `../../studio-pm/research/` (note: two levels up)

## Key decisions already made

- **No wikilinks** — use standard relative markdown links only. Agents can't resolve wikilinks.
- **required sections vary by page type** — check CLAUDE.md before writing each doc
- **All URLs in Key Sources must have access dates** — format: `URL (YYYY-MM-DD)`

## The untracked file

`references/obsidian-knowledge-management.md` already exists and is properly formatted. Just commit it. Read it first to confirm frontmatter is valid, then stage + include in the commit.

## Source doc notes (from PM review)

- `obsidian-style-knowledge-management.md` in agent-ops: content is "evaluated and rejected for studio-research" — keep the doc, it's historical reference, but note that conclusion in description
- `agent-knowledge-base-structure.md` in studio-pm: this is the S-02 deep research output that informed the studio-research design decision. High value. Mark confidence: high.
- `claude-code-official-config-reference.md`: Claude Code config is fast-moving — set staleness: 14
- Use RESOLVER.md to route the two undecided docs (config-version-control, continuous-agent-ops-patterns)

## Commit

Single commit: `RES-2: migrate 5 research docs from agent-ops and studio-pm`

Branch: `feature/res-2-migrate-research-docs-from-agent-ops-to-studio-research`

## Linear

After committing: update RES-2 status to Done in Linear (team: RESEARCH).
