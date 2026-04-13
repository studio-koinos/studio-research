---
title: "Version-Controlling Claude Code User-Level Configuration"
description: "Community patterns for version-controlling ~/.claude/ config: git-in-place, symlinks from dotfiles repo, chezmoi. Includes settings.json symlink bug (#3575) and official symlink support status."
tags: [claude-code, configuration, version-control, dotfiles, symlinks]
modified: 2026-04-13
status: active
confidence: high
staleness: 30
related:
  - ./claude-code-config-reference.md
---

# Version-Controlling Claude Code User-Level Configuration

> **Research date:** 2026-04-12
> **Last verified:** 2026-04-12
> **Staleness warning:** Re-verify after 30 days. Claude Code's config system is evolving rapidly (plugins launched Oct 2025, managed settings added 2026).
> **Confidence summary:** 8 verified, 3 likely, 1 conflicting claim

## TL;DR

The community is split between two dominant patterns: (1) **git init directly in `~/.claude/`** with an allowlist `.gitignore` (simplest, most popular), and (2) **symlinks from a dotfiles repo to `~/.claude/`** (more structured, supports additional content alongside config). Anthropic explicitly supports symlinks for `rules/` directories. **CRITICAL: `settings.json` as a symlink is broken** (bug #3575 — causes permission failures and performance degradation). No single tool has emerged as standard; chezmoi is the most capable but adds a dependency most people skip.

---

## How It Works

### What Anthropic Officially Says

**VERIFIED** — Anthropic's docs define four configuration scopes:

| Scope | Location | Shared? |
|---|---|---|
| Managed (org/IT) | system dir or MDM | Enforced |
| User | `~/.claude/` | Personal only |
| Project | `.claude/` in repo | Committed to git |
| Local | `.claude/settings.local.json` | Gitignored |

**VERIFIED** — Symlinks for rules/ are explicitly supported:
> "The `.claude/rules/` directory supports symlinks, so you can maintain a shared set of rules and link them into multiple projects. Symlinks are resolved and loaded normally, and circular symlinks are detected and handled gracefully."

**VERIFIED** — `settings.json` as a symlink is broken (bug #3575):
> Symlinking `~/.claude/settings.json` causes permission recognition failures for explicitly-allowed commands and severe performance degradation (seconds-long delays on simple operations). Official workaround: "Replace the symlink with an actual copy of the file."

**VERIFIED** — `commands/` directory does NOT support symlinks (feature request #39475 pending).

**LIKELY** — `agents/` and `skills/` directories are not explicitly mentioned in symlink docs but are expected to work (same filesystem resolution as rules/). No bug reports found for these.

**VERIFIED** — The plugin system (`/plugin install`) is the official distribution mechanism for sharing skills/agents/commands with others. It is NOT a version-control mechanism for personal config.

### Community Patterns (ranked by prevalence)

#### Pattern 1: Git init directly in `~/.claude/` (most popular)

**How it works:** `git init ~/.claude/`, add an allowlist `.gitignore` that tracks only config files, push to a remote.

**Examples:**
- `elizabethfuentes12/claude-code-dotfiles` — shell wrapper pulls on open, commits on exit
- `jarrodwatts/claude-code-config` — ~1,000 stars, install via `git clone ... ~/.claude`
- David Haberlah — daily `git pull --ff-only` via `.zshrc`

**Allowlist `.gitignore` pattern (from elizabethfuentes12):**
```gitignore
# Ignore everything
*
# Then allow specific config
!.gitignore
!CLAUDE.md
!settings.json
!commands/
!hooks/
!agents/
!rules/
!skills/
# Still exclude runtime state
projects/
cache/
backups/
history.jsonl
sessions/
*.key
*.token
credentials.json
```

**Pros:** Simplest possible setup. No symlinks, no wiring, no separate repo. Edits are tracked where they happen. New machine: `git clone <remote> ~/.claude/`.

**Cons:** `~/.claude/` is managed by Claude Code — mixing in git can feel fragile. No room for non-config content (decisions, session logs, research) in the same repo. Future Claude Code updates could add directories that show up as untracked noise.

#### Pattern 2: Symlinks from a dotfiles/config repo (second most popular)

**How it works:** Config files live in a separate repo. Symlinks from `~/.claude/<area>` point to the repo directory.

**Examples:**
- Hubert Sablonniere — GNU Stow (`stow claude` creates symlinks)
- Dylan Bochman — custom `install.sh`, per-item symlinks for skills
- Dr. Mowinckel — modular installer with per-tool symlinks
- Rushi — cross-tool `.agents/` directory symlinked into both `.cursor/` and `.claude/`

**Pros:** Clean separation of config (repo) from runtime state (`~/.claude/`). Can include documentation, decisions, research alongside config. Explicit — you know exactly what's symlinked.

**Cons:** Path dependency (symlinks break if repo moves). New machine requires clone + re-link. Extra indirection. Must not symlink `settings.json` (bug #3575).

**Bochman's insight — per-item vs directory symlinks:**
Per-item symlinks for skills allow local-only experiments without git friction. Directory-level symlinks for rules (where everything is shared). This is the most granular approach found.

#### Pattern 3: chezmoi (most feature-rich, least popular for Claude Code)

**How it works:** chezmoi manages `~/.claude/` as a templated dotfile set. Supports machine-specific values, Age encryption for secrets, and `chezmoi update` for cross-machine sync.

**Examples:** Arun — chezmoi+Age for encrypted commands containing API keys.

**Pros:** Handles machine-specific variation (different model endpoints per machine). Secrets encryption. Cross-machine sync with conflict detection.

**Cons:** Adds a dependency. Encrypted files aren't diffable in git. Over-engineered for single-user single-machine. Only one Claude Code example found.

#### Pattern 4: Dedicated sync tools

**Examples:**
- `onsails/ccsync` — syncs agents, skills, and commands between global and project configs
- `claudectx` — switch entire Claude Code configuration with one command

**LIKELY** — These are early-stage tools with small user bases. Not yet community standards.

### What Should and Should Not Be Version-Controlled

**VERIFIED** — Community consensus across all sources:

| Track | Exclude |
|---|---|
| `CLAUDE.md` | `projects/` |
| `settings.json` | `history.jsonl` |
| `rules/*.md` | `settings.local.json` |
| `agents/*.md` | `~/.claude.json` (OAuth, MCP state) |
| `skills/*/SKILL.md` | `cache/`, `backups/` |
| `hooks/` | `*.key`, `*.token`, `credentials.json` |
| `commands/` | `file-history/` |

`settings.local.json` is auto-gitignored by Claude Code. `~/.claude.json` contains OAuth sessions and per-project MCP state — never commit.

---

## Current State

The space is actively evolving. No single pattern has become the community standard. The plugin system may eventually subsume personal config management but currently serves a different purpose (distribution, not version control).

Key developments to watch:
- Bug #3575 (settings.json symlink) — if fixed, symlink pattern becomes simpler
- Feature request #39475 (commands/ symlink support) — would close a gap in the symlink pattern
- Plugin system evolution — could add personal config sync features

---

## Applicability to Our Stack

Studio-agent-ops uses the symlink pattern (Pattern 2): config files live in `studio-agent-ops/` and are symlinked into `~/.claude/`. This approach was validated by community patterns and explicitly supported by Anthropic for `rules/`.

**Decision factors for our case:**

| Factor | Git in `~/.claude/` | Symlinks to studio-agent-ops |
|---|---|---|
| Setup complexity | None | mv + ln -s per area |
| Ongoing maintenance | None | None (after wiring) |
| Non-config content (decisions, sessions) | Needs separate home | Same repo |
| New machine bootstrap | `git clone <remote> ~/.claude/` | Clone repo + re-symlink |
| Risk of tool conflict | Low but possible | None (separate directory) |
| Symlink fragility | N/A | Low (single machine, stable paths) |

**CONFLICTING:** Perplexity recommends chezmoi as "strongest fit." Web research shows direct git and symlinks are far more popular in practice. For single-user single-machine, chezmoi adds complexity without proportional benefit.

---

## Conflicts & Gaps

1. **agents/ and skills/ symlink support is undocumented.** Only rules/ is explicitly mentioned in the Anthropic docs. LIKELY works (same filesystem mechanics) but not confirmed.
2. **No data on whether Claude Code's plugin system will eventually subsume user-level config management.** If Anthropic adds a "sync my config" feature to plugins, both the symlink and git-in-place approaches become unnecessary.
3. **No community examples of the exact studio-agent-ops pattern** (separate repo with directory-level symlinks for multiple areas). The closest match is Bochman's per-item symlinks.

---

## Key Sources

1. https://code.claude.com/docs/en/memory — symlink support for rules/ (accessed 2026-04-12)
2. https://code.claude.com/docs/en/settings — configuration scope system (accessed 2026-04-12)
3. https://github.com/anthropics/claude-code/issues/3575 — settings.json symlink bug (accessed 2026-04-12)
4. https://github.com/anthropics/claude-code/issues/39475 — commands/ symlink feature request (accessed 2026-04-12)
5. https://github.com/elizabethfuentes12/claude-code-dotfiles — git-in-place pattern (accessed 2026-04-12)
6. https://github.com/jarrodwatts/claude-code-config — popular config repo, ~1K stars (accessed 2026-04-12)
7. https://github.com/trailofbits/claude-code-config — security-focused config template (accessed 2026-04-12)
8. https://github.com/hesreallyhim/awesome-claude-code — community index (accessed 2026-04-12)
9. https://github.com/anthropics/claude-plugins-official — official plugin registry (accessed 2026-04-12)
10. https://www.hsablonniere.com/dotfiles-claude-code-my-tiny-config-workshop--95d5fr/ — GNU Stow pattern (2026)
11. https://dylanbochman.com/blog/2026-01-25-dotfiles-for-ai-assisted-development/ — per-item symlinks (2026-01-25)
12. https://www.arun.blog/sync-claude-code-with-chezmoi-and-age/ — chezmoi+Age (2025/2026)
13. https://medium.com/@haberlah/configure-claude-code-to-power-your-agent-team-90c8d3bca392 — daily sync pattern (2025)
14. https://drmowinckels.io/blog/2026/dotfiles-coding-agents/ — modular installer (2026)
15. https://www.rushis.com/sharing-ai-agent-configs-between-cursor-and-claude-with-symlinks/ — cross-tool symlinks (2026)
16. https://github.com/onsails/ccsync — sync tool (accessed 2026-04-12)
17. https://ai.rundatarun.io/Practical+Applications/syncing-claude-code-configs-across-machines — sync guide (accessed 2026-04-12)
18. https://dev.to/mir_mursalin_ankur/claude-code-configuration-blueprint-the-complete-guide-for-production-teams-557p — team config guide (2026)
19. https://code.claude.com/docs/en/plugin-marketplaces — official plugin distribution (accessed 2026-04-12)
20. https://code.claude.com/docs/en/best-practices — official best practices (accessed 2026-04-12)
