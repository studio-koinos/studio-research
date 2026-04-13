# studio-research S-02 — RES-4, RES-7

## RES-4: Integration test — agent discoverability from cross-repo session
**Linear:** https://linear.app/koinos-studio/issue/RES-4

### What to verify
Open a session in `../studio-agent-ops/` (or simulate from this repo), then confirm an agent can:
1. Glob for docs: `../studio-research/**/*.md`
2. Grep for content: e.g. `grep "knowledge base" ../studio-research/references/agent-kb-structure.md`
3. Read the full doc via relative path
4. Cite a specific finding from the doc with file path + quote

Run each step, show the output. This is runtime verification — not "the files exist so it should work."

### Acceptance criteria
- [ ] Glob returns all 6 docs (5 references + 1 evaluation)
- [ ] Grep finds relevant content by keyword across the repo
- [ ] Read works on a full doc via relative path from a sibling repo working directory
- [ ] Citation pattern confirmed (file path + quoted excerpt)
- [ ] Log findings in RES-4 Linear comment, close as Done

---

## RES-7: Schema refinement — formalize S-01 observations
**Linear:** https://linear.app/koinos-studio/issue/RES-7

### What to update in CLAUDE.md
Three observations from S-01 real usage:

1. **"Conflicts & Gaps" section** — appeared organically in all migrated docs. Add as an optional-but-recommended section to the References and Evaluations page type definitions (not required, so existing docs stay valid).

2. **Research provenance blockquote** — the `> Research date / Last verified / Staleness warning / Confidence summary` block at doc top is useful. Make it explicit in the schema (currently implied but not stated).

3. **Required section headings are conceptual, not literal** — clarify in CLAUDE.md that "How It Works" means "explain how it works," not that the heading must literally say "How It Works." Agents adapting content to a different heading still satisfy the intent.

### Acceptance criteria
- [ ] CLAUDE.md updated with the three clarifications
- [ ] No existing docs invalidated (no new *required* sections)
- [ ] Append to log.md: `## [2026-04-13] update | Schema refinements (S-01 observations)`
- [ ] Close RES-7 as Done
