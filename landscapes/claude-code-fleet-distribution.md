---
research_date: 2026-05-04
last_verified: 2026-05-04
staleness_warning: Re-verify after 14 days. Claude Code plugin system is actively evolving; open issue #21163 (rules distribution gap) may be resolved without notice. Check release notes on each version bump.
confidence_summary: 19 verified, 4 likely, 2 unverified, 0 conflicting
session: S-63 AGENTOPS
linear: AO-244 (trigger), AO-XXX (fleet audit follow-up)
primary_sources:
  - "Claude Code Plugin Marketplaces: https://code.claude.com/docs/en/plugin-marketplaces"
  - "Claude Code Server-Managed Settings: https://code.claude.com/docs/en/server-managed-settings"
  - "Claude Code Endpoint-Managed Settings: https://code.claude.com/docs/en/settings#settings-files"
  - "Claude Code Devcontainer: https://code.claude.com/docs/en/devcontainer"
  - "Rules distribution gap (open issue #21163): https://github.com/anthropics/claude-code/issues/21163"
  - "Perplexity sonar-deep-research (50 queries): 2026-05-04"
  - "Web research (18 live fetches): 2026-05-04"
---

# Claude Code Fleet Distribution — Research

> **Research date:** 2026-05-04
> **Last verified:** 2026-05-04
> **Staleness warning:** Re-verify after 14 days. Plugin system is actively evolving. Open issue #21163 (rules gap) may close without notice.
> **Confidence summary:** 19 verified, 4 likely, 2 unverified
>
> **Primary sources:** Perplexity sonar-deep-research (50 queries), live web fetches of official Anthropic docs (18 fetches), studio-agent-ops local context (Decision 001, Decision 007)

## TL;DR

The Claude Code plugin marketplace is the native first-class distribution mechanism for agents, skills, and hooks — and works for 2-10 users with zero infrastructure beyond a private GitHub repo. **There is one critical blocker: plugins cannot distribute rules files** (`~/.claude/rules/`). This is open issue #21163 with no workaround in the plugin system. For the Enocta deployment, a devcontainer is the only reliable way to deliver rules to all users in one step. The recommended stack: plugin marketplace (agents/skills/hooks) + devcontainer with baked `managed-settings.json` + rules (controlled delivery) + server-managed settings (enforcement, if on Teams/Enterprise plan).

## The Critical Gap First

**VERIFIED** — Claude Code plugins **cannot distribute rule files** (`~/.claude/rules/*.md`). The plugin system supports agents, skills, hooks, MCP configs, LSP configs, monitors, and some `settings.json` keys — but not rules. This is tracked as [GitHub issue #21163](https://github.com/anthropics/claude-code/issues/21163) and confirmed by the `everything-claude-code` project documentation. No workaround exists within the plugin system itself.

This means studio-agent-ops's current rule distribution (symlinks to `~/.claude/rules/`) has **no plugin-native replacement today**. Rules require either:
- Manual copy to `~/.claude/rules/` on each machine (current approach)
- Devcontainer with rules baked into the image (best multi-user option)
- Dotfiles manager (chezmoi, yadm) with the rules in the managed dotfiles repo (single-user, multi-machine)
- Custom install script that clones and symlinks (current studio-agent-ops approach, manual)

## Distribution Layer Architecture

For a 2-10 user fleet (Enocta), three layers compose into a complete solution:

```
Layer 3 — Enforcement (who can install what, which settings are locked)
  → Server-managed settings (Teams/Enterprise, polled hourly)
  → Endpoint-managed settings / MDM (any plan, baked into managed-settings.json)

Layer 2 — Distribution (getting config onto each machine)
  → Plugin marketplace (agents, skills, hooks, MCP — not rules)
  → Devcontainer (rules, managed-settings.json, CLAUDE.md — everything in one image)
  → Dotfiles manager (chezmoi, yadm — personal multi-machine sync, not team enforcement)

Layer 1 — Version pinning (ensuring all users run the same version)
  → Plugin git tags (e.g. studio-fleet--v2.1.0 → `version: "2.1.0"` in plugin.json)
  → Devcontainer image tag (docker pull org/fleet:v2.1.0)
  → GitHub Actions fleet check on PR
```

## Option Analysis

### Option A — Plugin Marketplace (RECOMMENDED for agents/skills/hooks)

**VERIFIED.** The plugin system is the first-class, Anthropic-native distribution mechanism. Setup:

1. Add a `marketplace.json` to studio-agent-ops (or a dedicated `studio-fleet` repo)
2. Each user runs: `/plugin marketplace add studio-koinos/studio-agent-ops`
3. Each user runs: `/plugin install studio-fleet@studio-koinos`
4. Updates: user runs `/plugin marketplace update` or Claude Code auto-updates (since v2.0.70)

**Version pinning:** Set `version: "2.1.0"` in `plugin.json` + create git tag `studio-fleet--v2.1.0`. Users get updates only when you push a new version. Pin exact commit via `sha` field for reproducible installs.

**What it distributes:** agents (all 8), skills (all 13), hooks (all 7), MCP configs. **NOT rules.**

**Multi-user installation:** Each user runs one command. On Teams/Enterprise, admin can set plugins as installed-by-default or required (locked on). Admin can also set `strictKnownMarketplaces` to prevent users from installing from other sources.

**Private repo support:** Works with private GitHub repos. Users need GitHub access. On Teams/Enterprise, Cowork admin dashboard handles distribution without requiring each user to have repo access.

**Operational overhead (2-10 users):** Low. One setup command per user. Updates are automatic or one-command. The `extraKnownMarketplaces` key in managed-settings.json pre-registers the marketplace so users don't need to run the add command.

**Fit with subscription model:** VERIFIED. Designed for Claude Code CLI subscription users. No API key required.

### Option B — Devcontainer (RECOMMENDED for rules + full reproducibility)

**VERIFIED.** Official Anthropic devcontainer feature exists at `ghcr.io/anthropics/devcontainer-features/claude-code:1.0`.

Rules baked into devcontainer:
```dockerfile
RUN mkdir -p /etc/claude-code
COPY managed-settings.json /etc/claude-code/managed-settings.json
COPY rules/ /etc/claude-code/rules/  # if rules are baked at OS level
```

Or: CLAUDE.md at project root includes rule content inline (workaround until #21163 resolves).

**Version pinning:** Pin claude-code version in Dockerfile: `npm install -g @anthropic-ai/claude-code@X.Y.Z` + `DISABLE_AUTOUPDATER=1`. Pin image tag in `devcontainer.json`.

**What it solves that plugins don't:** Rules delivery, fully reproducible environment, zero manual setup per user (clone repo → open in devcontainer → done).

**What it doesn't solve:** Users who work outside the container environment (local install). Devcontainer requires VS Code, GitHub Codespaces, or JetBrains — terminal-only usage is unsupported.

**Operational overhead (2-10 users):** Medium. Requires Docker familiarity. Image rebuild on config changes. Auth persistence requires named volume mount at `~/.claude`.

### Option C — Server-Managed Settings (RECOMMENDED for enforcement)

**VERIFIED.** Requires Teams or Enterprise Claude plan.

Admin configures settings at claude.ai → Admin Settings → Claude Code → Managed Settings. All users pick up on next startup (hourly poll). Can enforce:
- Which plugin marketplaces are allowed (`strictKnownMarketplaces`)
- Which tools are allowed/denied
- Minimum Claude Code version (`minimumVersion`)
- Auto-mode settings
- `forceRemoteSettingsRefresh: true` = startup blocks until fresh settings arrive (fail-closed)

**Does NOT distribute:** rules, MCP configs (separate limitation).

**Precedence:** Server-managed settings override all local settings. Cannot be bypassed by user without admin access to claude.ai.

**Fit with subscription model:** VERIFIED. But requires Teams ($25/user/mo) or Enterprise plan. If Ali is the only admin and fleet users are client users, this requires purchasing seats for the Enocta client team.

### Option D — Endpoint-Managed Settings / MDM (NO plan requirement)

**VERIFIED.** Works on any Claude Code plan.

File at `/etc/claude-code/managed-settings.json` (Linux/container) or `/Library/Application Support/ClaudeCode/managed-settings.json` (macOS). Delivered via MDM (Jamf, Kandji, Intune) or baked into devcontainer Dockerfile.

The `managed-settings.d/` directory pattern allows multiple files merged alphabetically — enables separate owners for security policy vs. permissions vs. marketplace config without editing a single file.

**Key use for studio context:** Bake into devcontainer Dockerfile = zero-config enforcement without MDM.

**Limitation:** Requires devcontainer or MDM to distribute. If users have local installs outside of managed environments, this provides no coverage.

### Option E — Dotfiles Manager (chezmoi, yadm) — personal multi-machine, NOT team

**LIKELY** (based on community patterns, not official Anthropic docs).

Chezmoi is the dominant community approach for syncing `~/.claude/` across multiple machines for a single user. Supports templates (role/user-conditional content), Age encryption for API keys, `chezmoi apply` for idempotent sync.

**Does NOT scale to team distribution.** Each user needs the Age private key — manual key distribution is still required. Chezmoi is fundamentally personal dotfiles management; it has no team enforcement mechanism.

**Use case in studio context:** Ali's personal multi-machine setup. Not the answer for Enocta client users.

### Option F — npm Package Distribution

**LIKELY** (based on `everything-claude-code` example, not direct Anthropic testing).

Publish `@studio-koinos/fleet-config` to npm (private or public). Post-install script symlinks or copies files to `~/.claude/`. Version managed via npm semver.

**Limitation:** npm install is a one-time pull. No enforcement, no auto-update, no rules gap solution. Plugin marketplace is strictly better for Claude Code users — it's native, auto-updating, and plan-aware.

**Recommendation:** Skip; use plugin marketplace instead.

### Option G — Git Submodule / Subtree

**VERIFIED** (standard git feature, well-understood).

Embed studio-agent-ops as a git submodule in a user-level dotfiles repo. Gives version pinning at the commit level. `git submodule update --remote` pulls the latest.

**What it solves over current symlinks:** Explicit version pinning, multi-machine via dotfiles repo.

**What it doesn't solve:** Auto-update for team members, enforcement, rules gap.

**Recommendation:** Use as the underlying structure for the plugin marketplace (studio-agent-ops is already a git repo; adding `marketplace.json` makes it a plugin marketplace too). Skip as a standalone distribution mechanism.

## Recommended Architecture for Enocta Deployment

**Phased approach — 3 phases over ~4 weeks:**

### Phase 1 (this week): Make studio-agent-ops a plugin marketplace

Add `marketplace.json` + `plugin.json` to studio-agent-ops root. This converts the existing repo into an installable plugin with one command:
```bash
/plugin marketplace add studio-koinos/studio-agent-ops
/plugin install studio-fleet@studio-koinos
```

Distribute agents, skills, hooks to 2-10 users. Rules still via separate step (copy to `~/.claude/rules/`).

Version pin: tag `studio-fleet--v1.0.0` on current master.

### Phase 2 (next 2 weeks): Devcontainer for rules + managed settings

Add `.devcontainer/` to each Enocta repo with:
- `devcontainer.json` referencing the Claude Code feature
- `managed-settings.json` baked into Dockerfile
- `CLAUDE.md` at project root (rules inline if #21163 still open)
- Volume mount for `~/.claude` auth persistence

This gives all Enocta users: consistent rules, permissions locked, same fleet version, zero manual setup beyond opening in devcontainer.

### Phase 3 (if Teams/Enterprise plan): Server-managed settings

If Enocta client is on Teams plan, configure admin-managed settings at claude.ai to:
- `strictKnownMarketplaces` → only studio-agent-ops marketplace
- `minimumVersion` → pin to known-good Claude Code version
- `forceRemoteSettingsRefresh: true` → fail-closed on startup

## Fleet Version Validation (CI pattern)

**LIKELY** — no production examples found, but the pattern is straightforward.

GitHub Actions job in each fleet repo:
```yaml
- name: Validate fleet plugin version
  run: |
    PINNED=$(cat .claude/fleet-version.txt)
    INSTALLED=$(grep version .claude/settings.json | ...)
    [ "$PINNED" = "$INSTALLED" ] || exit 1
```

Or: require a `.claude/fleet-version.txt` checked into each repo, updated whenever the plugin is updated. PRs from repos with stale versions fail the check.

## Summary Decision Matrix

| Need | Best mechanism | Rules? | Multi-user? | Plan required |
|------|---------------|--------|-------------|---------------|
| Distribute agents/skills/hooks | Plugin marketplace | ✗ | ✓ | None |
| Distribute rules | Devcontainer (only reliable option) | ✓ | ✓ | None |
| Enforce org-wide settings | Server-managed settings | ✗ | ✓ | Teams/Enterprise |
| Lock marketplace to approved only | `strictKnownMarketplaces` | ✗ | ✓ | Teams/Enterprise (server) OR none (endpoint) |
| Reproducible environment | Devcontainer | ✓ | ✓ | None |
| Personal multi-machine sync | chezmoi + Age | ✓ | ✗ | None |
| Version pinning | Plugin git tags + devcontainer image tag | Both paths | Both | None |

## Conflicts & Gaps

No conflicts between sources. Key gap: whether `#21163` (rules via plugins) has a target milestone or ETA. None found. The issue is open with community +1s but no Anthropic response with timeline.

## Related but Out of Scope

- **CDK/Terraform for managed-settings delivery**: For large fleet (50+ users), infrastructure-as-code managed-settings.json delivery via Terraform. Not relevant at 2-10 users.
- **AgentOps / LangSmith / Langfuse fleet monitoring**: Runtime observability for what agent versions are executing. Different layer from config distribution.
- **Cloudflare Artifacts for substrate distribution** (AO-160): Binary/substrate distribution, not agent config files. Different problem.
- **Custom MCP server serving agent specs dynamically**: Theoretically possible — MCP tool returns agent spec as a string. Not yet implemented anywhere and would require Claude Code to support runtime agent spec injection (unconfirmed).

## Key Sources

1. Claude Code Plugin Marketplaces: https://code.claude.com/docs/en/plugin-marketplaces (2026-05-04)
2. Claude Code Plugin Dependencies: https://code.claude.com/docs/en/plugin-dependencies (2026-05-04)
3. Rules distribution gap — open issue: https://github.com/anthropics/claude-code/issues/21163 (2026-05-04)
4. Claude Code Server-Managed Settings: https://code.claude.com/docs/en/server-managed-settings (2026-05-04)
5. Claude Code Endpoint-Managed Settings: https://code.claude.com/docs/en/settings#settings-files (2026-05-04)
6. Claude Code Devcontainer: https://code.claude.com/docs/en/devcontainer (2026-05-04)
7. Claude Cowork Plugin Admin: https://support.claude.com/en/articles/13837433-manage-claude-cowork-plugins-for-your-organization (2026-05-04)
8. chezmoi for Claude Code sync: https://www.arun.blog/sync-claude-code-with-chezmoi-and-age/ (2026-05-04)
9. everything-claude-code (community reference): https://github.com/affaan-m/everything-claude-code (2026-05-04)
10. Trail of Bits devcontainer: https://github.com/trailofbits/claude-code-devcontainer (2026-05-04)
11. Complete settings.json reference (v2.1.104): https://gist.github.com/mculp/c082bd1e5a439410158974de90c89db7 (2026-05-04)
12. Perplexity sonar-deep-research (50 queries, $0.76): 2026-05-04
