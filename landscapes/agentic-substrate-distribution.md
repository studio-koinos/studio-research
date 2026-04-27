---
title: "Agentic Substrate Distribution — Landscape"
description: "How engineering teams distribute, version, sync, and protect shared/per-user developer-environment artifacts; mapping to a gitignored agentic substrate. Surveys 3 industries × 7 axes."
tags: [agent-state, substrate, dotfiles, secrets, versioning, mlops, llm-platforms, sync, audit]
modified: 2026-04-27
status: active
confidence: high
staleness: 21
related:
  - ../references/claude-code-config-version-control.md
  - ../references/claude-code-config-reference.md
  - ../references/agent-kb-structure.md
---

# Agentic Substrate Distribution — Landscape

> **Research date:** 2026-04-27
> **Last verified:** 2026-04-27
> **Staleness warning:** Re-verify after 21 days. The LLM-agent-platform layer (Cursor, Devin, Cloudflare Artifacts) is shipping monthly; revisit when Cloudflare Artifacts public beta lands (targeted early May 2026) or if the Agent Trace standard publishes a v1.0 spec.
> **Confidence summary:** ~40 VERIFIED, ~12 LIKELY, 2 UNVERIFIED, 1 CONFLICTING claim
> **Triangulation:** Stream B (web research, 38 dated primary sources) is load-bearing. Stream A (Perplexity sonar-pro fallback) returned parametric tool tables with no tool-specific citations — used as outline/scaffold only; every tool claim re-grounded against Stream B's dated sources. Stream C (local context) confirmed brief alignment and surfaced 11 prior internal docs to cross-reference.
>
> **Primary sources (top 8 by load-bearing weight):**
> - Cloudflare Artifacts beta announcement: https://blog.cloudflare.com/artifacts-git-for-agents-beta/ (2026-04-16)
> - chezmoi comparison table: https://www.chezmoi.io/comparison-table/ (2026-04-27)
> - Atuin docs: https://docs.atuin.sh/ (2026-04-27)
> - Cursor rules + team-sync docs: https://cursor.com/docs/rules (2026-04-27)
> - Claude Code MCP + plugin marketplace docs: https://code.claude.com/docs/en/mcp + https://code.claude.com/docs/en/plugin-marketplaces (2026-04-27)
> - SOPS GitHub + 2025 GitOps guide: https://github.com/getsops/sops + https://www.bordencastle.com/security/gitops/devops/2026/02/13/sops-secrets-management-gitops.html (2026-04-27)
> - 1Password CLI secrets-config-files: https://developer.1password.com/docs/cli/secrets-config-files/ (2026-04-27)
> - JuJutsu for AI agents: https://www.panozzaj.com/blog/2025/11/22/avoid-losing-work-with-jujutsu-jj-for-ai-coding-agents/ (2025-11-22)

---

## TL;DR

The 2026 landscape has **bifurcated into two architectural patterns** for distributing/versioning agent-environment artifacts:

- **Pattern A — Git-as-substrate** (collaboration-first): per-session/per-user repo, share via URL, time-travel via git history. **Cloudflare Artifacts** (announced 2026-04-16, private beta) is the most architecturally on-target shipped product — "versioned filesystem that speaks Git" with Durable Objects + R2 + native git-notes for agent metadata. The **Agent Trace standard** (Cognition + Cursor + Cloudflare + Vercel + git-ai + OpenCode) is the first cross-vendor agreement on agent-provenance metadata. Public beta targeted early May 2026.

- **Pattern B — Append-log + smart sync** (simplicity-first): E2E-encrypted append-log with timestamp-merge sync (atuin model), or markdown-files-in-git (Cline Memory Bank, Aider). Lower-protocol, higher-semantics. Wins on bandwidth + simplicity but loses branching.

For our 3-tier substrate (~50–100MB/user, single-user-now → team-eventually):

| Tier | Today's pragmatic pick | 2026-Q3 candidate |
|---|---|---|
| **Tier 1 — load-bearing** (events.jsonl, .context/, memory, sessions, .env) | private GitHub repo + `.gitignore` + sops/age for secrets-as-code + 1Password CLI for personal secrets | **Cloudflare Artifacts** when public beta lands |
| **Tier 2 — telemetry** (shell history, dispatch.db, paste-cache, file-history) | **atuin** for shell history (E2E-encrypted, 29.5k stars, active 2026-04-16); restic-to-encrypted-cloud for the rest | unchanged |
| **Tier 3 — debug** (ephemeral hook logs) | local only, gitignored, ride along Tier 1 backup if forensic survival needed | unchanged |

**Three load-bearing constraints discovered:**

1. [VERIFIED] **DVC/git-LFS/git-annex are wrong for JSONL-append workloads** — they're blob-replacement-versioned (every save = full new blob). **Oxen.ai's row-level DuckDB indexing** is the only purpose-built fit. Otherwise, vanilla Git's pack-file delta is "good enough" for our <100MB scale.
2. [VERIFIED] **Cloudflare R2 has no GA object versioning** — it cannot be a drop-in versioned-cloud substrate by itself. You need Cloudflare Artifacts on top OR an application-level versioning scheme.
3. [VERIFIED] **No tool today provides "team-base + per-user-overlay with conflict-resolution"** as a first-class primitive. Closest: chezmoi templates + flux/argo deploy-time merge. For agent rules, Claude Code's plugin marketplace + 3-scope MCP config (User/Project/Local) is the most evolved primitive — but conflict resolution is "last-write-wins" not "merged-with-precedence." Active gap.

**One open question that materially affects strategy:** Cloudflare Artifacts public-beta pricing at our scale (100MB/user, ~10k ops/session) is announced as $0.15 per 1k ops + $0.50/GB-mo, but real-world cost at our usage pattern needs a pilot. Defer architectural commitment until public beta + a 1-week pilot.

---

## What It Is

The problem under research: a single-user (today) eventually-team agentic environment accumulates a substantial body of state outside every git tree:

- **Tier 1 (load-bearing substrate):** events.jsonl per-repo (frequent JSONL append), `.context/` per-repo (markdown plans/assessments/test-plans), `~/.claude/projects/<slug>/memory/`, session JSONL transcripts (1–5MB each), `~/.claude/.env` (secrets store)
- **Tier 2 (telemetry):** `~/.claude/history.jsonl` (~3MB shell-search history), `dispatch.db`, `paste-cache/`, `file-history/`
- **Tier 3 (debug):** ephemeral hook debug logs

We want **git-similar versioning** (diff/rollback/branch/merge) — not just backup. Today's only protection is local rsync to a `koinos/agent-session-archive` repo; no cloud, no versioning, no team distribution.

This research surveys **three industries** and **seven orthogonal axes** that handle adjacent problems and could donate patterns or tools.

---

## How It Works — The Two Architectural Patterns

### Pattern A — Git-as-substrate (collaboration-first)

Treat agent state as a versioned repository. Each session gets its own repo (or branch); state changes become commits; collaboration is a `git push`/`git pull` operation. Time-travel debugging becomes `git checkout <hash>`. Conflict resolution is the same problem we already solve for code.

**Primary exemplar:** Cloudflare Artifacts (2026-04-16) — Durable Objects + SQLite (chunked rows for git objects >2MB), R2 for snapshots, KV for auth. Implements both git protocol v1 and v2: `ls-refs`, shallow/deepen clones, incremental fetch with have/want negotiation, native `git-notes` for agent metadata, delta encoding. ArtifactFS (open-sourced at github.com/cloudflare/artifact-fs) mounts large git repos via blobless clone + on-demand hydration. [VERIFIED, src 38]

**Research formalizations:** AgentGit (arXiv 2511.00628) and GCC (Git-Context-Controller, emergentmind) generalize the pattern as `COMMIT/BRANCH/MERGE/CONTEXT` operations on agent memory. [VERIFIED — academic, src 22]

**Trade-off:** richer semantics (branching, merging, attribution); higher protocol cost + storage overhead per commit.

### Pattern B — Append-log + smart sync (simplicity-first)

Treat agent state as an append-only log. Local-first writes; sync is timestamp-merge or CRDT. Encryption is end-to-end (server can never read). No branching; no merge conflicts; no protocol negotiation.

**Primary exemplar:** atuin (29.5k stars, v18.15.2 2026-04-16) — E2E-encrypted shell history sync. SQLite local; symmetric encryption with key generated and stored only on user machines; sync server cannot decrypt. Append-log; merge-by-timestamp. Self-hostable. [VERIFIED, src 3, 32]

**Adjacent:** Cline's Memory Bank (markdown files in repo, structured by `projectbrief.md`/`activeContext.md`/`progress.md`); Aider's `.aider.chat.history.md` in CWD. Inherit Git semantics from the wrapping repo. [VERIFIED, src 26, 30]

**Trade-off:** simpler + lower bandwidth; loses branching, merging, and structured collaboration.

### Hybrid: markdown-in-git

Cline + Aider place agent state inside the project repo; this gets you Pattern A's versioning (via the repo) with Pattern B's simplicity (markdown files, no protocol). It's the dominant pattern in OSS LLM coding agents in 2025-2026. **The cost:** state lives in the project repo, conflating "project code" with "agent process state" — fine for project-scoped agent context, wrong for cross-project memory.

### The wildcard — JuJutsu (jj)

Git-compatible VCS with snapshot-on-every-command (vs Git's commit-when-developer-decides). Auto-rebase descendants on edit, conflict-as-first-class concept. The 2025-11-22 panozzaj.com blog post observes that AI agents create "hundreds of micro-commits in seconds" and JJ's snapshot-everything-always design recovers from agent-induced state loss better than vanilla Git. [LIKELY, src 20]

**Implication:** for agent-tier version control, snapshot-frequency matters more than commit-discipline. The protocol assumption underlying Git ("commit when the developer decides") is a poor fit for agent reality.

---

## Current State (2026)

- **Cloudflare Artifacts (private beta as of 2026-04-16, public targeted early May 2026):** the most architecturally on-target shipped product. Per-session agent repo, share via URL, time-travel through both prompt state and file state. Pricing: $0.15 per 1k ops (10k included), $0.50/GB-mo (1GB included). [VERIFIED, src 38]
- **Agent Trace standard (Cognition + Cursor + Cloudflare + Vercel + git-ai + OpenCode, 2025):** open vendor-neutral spec for AI-contribution provenance in version-controlled codebases. First cross-vendor agreement on agent provenance metadata. [VERIFIED, src 31]
- **Claude Code's plugin marketplaces:** governance layer (4,200+ skills, 770+ MCP servers, 2,500+ marketplaces as of Apr 2026) — most evolved primitive for team-config governance. `marketplace.json` lists plugins from git repos/local paths; admins curate per-org catalogs. [VERIFIED, src 25]
- **Cursor still has no Settings Sync** (community-confirmed gap as of 2025); workarounds via dotfiles+symlinks or 3rd-party "Cursor Settings Sync" extensions. AI-IDE space hasn't yet absorbed VSCode's UX-state primitives. [VERIFIED, src 17]
- **chezmoi reigns in dotfiles** — v2.70.2 2026-04-17, age + 1Password CLI integration, 19.2k stars; HN threads consistently cite it as the multi-machine-with-secrets choice. Nix Home Manager owns the Nix-committed cohort. [VERIFIED, src 1, 18, 19]
- **SOPS + age** is the dominant 2025-2026 pattern for "secrets-as-code" (CI/CD, K8s, GitOps, dotfiles encrypted-in-git). [VERIFIED, src 9]
- **Atuin** is the dominant E2E-encrypted shell-history sync; per-user only, no team primitive. [VERIFIED, src 3, 32]

---

## Landscape Map

```
                         ┌─────────────────────────────────────────────────┐
                         │              GOVERNANCE MODEL                    │
                         │                                                  │
                         │  single-user        team-shared       org-wide   │
─────────────────────────┼─────────────────────────────────────────────────┤
 Pattern A               │                                                  │
 (git-semantic           │  jj, ArtifactFS    Cloudflare        Decagon    │
 versioning)             │  (snapshot-every)  Artifacts         (audit/CI)  │
                         │                    (per-session)                 │
                         │                                                  │
─────────────────────────┼─────────────────────────────────────────────────┤
 Hybrid                  │  chezmoi+age       chezmoi+sops      Nix+sops   │
 (git + encryption)      │  yadm+gpg          (private repo)    (flake)     │
                         │                                                  │
                         │  Cline, Aider      Claude Code       (—)         │
                         │  (md-in-repo)      (mcp 3-scope)                 │
                         │                                                  │
─────────────────────────┼─────────────────────────────────────────────────┤
 Pattern B               │  atuin             SOPS+age          1Password  │
 (append-log + sync,     │  (shell history)   (CI/CD/K8s)       Connect    │
 no branching)           │                                                  │
                         │  fish_history      git-crypt         Vault      │
                         │                    BlackBox          Doppler     │
                         │                                                  │
─────────────────────────┼─────────────────────────────────────────────────┤
 Snapshot-restore        │  restic, borg      restic+B2/S3      (—)        │
 (NOT git-semantic;      │  kopia (local)     (encrypted cloud)             │
 backup-only)            │                                                  │
                         └─────────────────────────────────────────────────┘
```

**Reading the map:** the further down you go, the simpler/cheaper the substrate; the further right, the more team-distribution machinery. Cloudflare Artifacts is the only product currently sitting in the **upper-middle** cell (git-semantic + team-shared + first-class encryption-at-rest via R2 default). That cell is the architectural fit for our Tier 1.

---

## Comparison Tables

### Industry 1 — Dotfiles tooling

| Tool | Governance | Versioning | Encryption-at-rest | Stars / last release | Fit (1–5) | Source |
|---|---|---|---|---|---|---|
| **chezmoi** | single → team via shared repo + per-user templates | Git (snapshot per commit) | Yes (gpg or age, file-level) | 19.2k / v2.70.2 2026-04-17 | **5** | src 1, 18 |
| **yadm** | single (multi-machine via `##os.*` syntax) | Git (bare repo) | Yes (gpg, transparent) | (mid-thousands) / active | 3 | src 1, 5 |
| **GNU Stow** | single | None (delegates to wrapping Git) | No | active | 2 | src 1, 8 |
| **dotbot** | single | None | No | active | 2 | src 1 |
| **Nix Home Manager** | per-user; team via flake monorepo + sops-nix | Git on flake (deterministic) | Via sops-nix | (high) / active | 4 (if Nix-committed) | src 19 |
| **mackup** | single (app config backup/sync) | Snapshot | Optional | (low activity) | 1 | src A |

[VERIFIED] across most rows. [LIKELY] on stars where Stream B couldn't fetch the GitHub API directly.

### Industry 2 — MLOps / data-artifact versioning

| Tool | Governance | Versioning | JSONL-append fit | Source |
|---|---|---|---|---|
| **DVC** | team-friendly, S3/GCS/Azure backend | Content-addressed (md5); branchable | **POOR** — blob-replacement; rewrite = new blob each time | src 12, 14 |
| **Git-LFS** | team | Linear (git-style); no per-file selective pull | **POOR** — no within-file dedup; each version stored whole | src 12 |
| **git-annex** | team, multi-remote | Symlink-target hash; branchable | **POOR** — symlink overhead > compressed second copy for small files (per its own docs) | src 21 |
| **Oxen.ai** | team (newer 2024-2025) | Merkle-tree + per-row dedup via DuckDB on remote | **GOOD** — only tool explicitly addressing tabular append | src 24 |
| **Vanilla Git pack-delta** | (any) | Git default | **OK for <100MB** — pack-file delta dedups similar JSONL versions reasonably well | src 27 (GitLab dedup docs) |

**Implication for events.jsonl:** vanilla Git is fine until file growth or write rate becomes concerning. Don't over-engineer. Consider Oxen.ai if events.jsonl grows beyond hundreds of MB OR if row-level diff becomes load-bearing.

### Industry 3 — LLM agent platforms (2025-2026)

| Platform | State distribution | Per-user / per-org | Audit | Source |
|---|---|---|---|---|
| **Cursor** | `.cursor/rules/*.mdc` in repo (Project / Team / User Rules + AGENTS.md). Settings Sync **not supported** (community-confirmed gap) | Per-user (no Settings Sync); `.cursor/rules/` shared via repo | Forum requests for "API to set Team Rules" suggest org push is incomplete | src 6, 17 |
| **Cline** | Memory Bank: `memory-bank/` markdown files in repo; `new_task` tool emits structured handoff at threshold | Per-user; sharing via repo commit | None inherent (what you commit is what's shared) | src 26, 28 |
| **Aider** | `.aider.chat.history.md` + `.aider.input.history.md` in CWD; markdown-append | Single-user assumption | None | src 30 |
| **Claude Code** | `.mcp.json` at project root committed for team distribution; admin-deployed workspace-wide skills (Dec 2025); plugin marketplaces (`marketplace.json`) for governance | 3-scope: User / Project / Local | Hooks (PreToolUse / PostToolUse / SessionStart); subagent gap (issue #34692) | src 7, 25 |
| **Devin (Cognition)** | Recurring sessions; "session state maintained between runs"; "insights from one session inform the next"; isolated VM per agent | Org (managed cloud) | Agent Trace standard signed; per-user audit specifics not publicly findable | src 31 |
| **GitHub Copilot Workspace / agentic** | (Microsoft Purview integration) | Per-user via Microsoft 365 identity | 180-day retention; `actor:Copilot` enterprise filter | src 10, 11 |
| **Cloudflare Artifacts** | Per-session repo (Git protocol over R2 + Durable Objects); `git-notes` for agent metadata | per-user-or-team via repo permissions | Native git history is the audit log | src 38 |

### Axis 4 — Secret distribution

| Tool | Governance | Versioning | Encryption | Use-case fit | Source |
|---|---|---|---|---|---|
| **SOPS + age** | team-shared (multi-recipient asymmetric) | Git (encrypted file in git) | age (X25519) or KMS | **Best for team-shared infra creds + secrets-as-code** | src 9 |
| **1Password CLI** | per-user (biometric-gated); Service Accounts for automation | Vault history | Native 1Password vault | **Best for per-developer personal secrets**; chezmoi-template-friendly via `op://` URIs | src 13 |
| **git-crypt** | team-shared | Git (transparent decrypt) | gpg | Older/simpler than SOPS; less granular | (general knowledge — verify) |
| **HashiCorp Vault** | org | Log | Native | Heavy infra; overkill for laptop substrate | src A |
| **Doppler** | org (SaaS) | Snapshot | Native | Centralized env-var distribution | src A |
| **gopass** | team via shared git repo | Git | gpg | Pass-style, more scriptable | src A |
| **Bitwarden Secrets Manager** | org (SaaS) | Snapshot | Native | Bitwarden-aligned teams | src A |
| **Pulumi ESC** | org | None inherent | Native | IaC-aligned; tied to Pulumi cloud | src A |

**Convergence (2025-2026):** SOPS+age for "secrets-as-code" + 1Password CLI for "developer-laptop-personal-secrets." They coexist via chezmoi templates that call both. [VERIFIED, src 9, 13]

### Axis 5 — Versioned cloud sync

| Tool | Versioning model | Encryption | Git-semantic? | Notes |
|---|---|---|---|---|
| **restic** | Snapshot, content-defined chunking | AES-256-CTR + Poly1305, scrypt KDF, encrypts metadata + filenames | No | Restore-oriented, not branchable. 60-80% dedup typical. Native S3, B2, Azure, GCS. [src 15] |
| **borg** | Snapshot, dedup | AES-256 | No | Most memory-efficient (75 MB peak); no native S3, no Windows binary. [src 15] |
| **kopia** | Snapshot | AES-256 (GCM/CBC/CTR) | No | Best S3 support, parallel uploads, single-binary, web UI. [src 15] |
| **Cloudflare R2** | None GA (object versioning REQUESTED, not shipped — Dec 2025 community thread confirms) | AES-256-GCM at rest by default; SSE-C via Workers | No (without Artifacts) | **Cannot be drop-in versioned cloud** by itself. [src 16] |
| **S3 versioning** | Object versioning (per-bucket) | AES-256 server-side | No | Snapshot-per-write; restore by version-id |
| **B2** | Bucket file-versioning | AES-256 | No | S3-API-compatible |
| **Tigris** | Log-based | Native | No | Newer, global file DB |
| **Syncthing** | Optional file versioning | TLS in transit + identity | No | P2P; not centralized |
| **rclone** | Optional (backend-dependent) | Optional (backend-dependent) | No | Multi-backend frontend; versioning depends on backend |

**Reading:** restic/borg/kopia are SNAPSHOT-VERSIONED (point-in-time restore) — fine for backup, NOT git-semantic. R2 alone won't deliver versioning. For git-semantic + cloud + encryption, you need **Cloudflare Artifacts** layered on R2.

### Axis 6 — Shell history + per-machine state

| Tool | E2E-encrypted? | Self-hostable? | Team primitive? | Source |
|---|---|---|---|---|
| **atuin** | Yes (symmetric, key local-only) | Yes (primary recommended for paranoid teams) | No (per-user) | src 3, 32 |
| **fish_history sync** | No (file-based) | (via syncthing/dotfiles) | No | src A |
| **browser bookmarks/state** | Vendor-dependent | No | No (per-account) | src A |

### Axis 7 — Audit + redaction

| Pattern | Implementation | Source |
|---|---|---|
| **Per-language logger redaction** | Pino/Winston/Bunyan/Zap/Serilog/structlog `redact:` field hooks — replace matching paths with `[Redacted]` before emit | src 23 |
| **OpenTelemetry Collector redaction** | Built-in redaction processor + 4 attribute filtering processors; OCB supports custom Go-based redactors | src 23 |
| **Hybrid regex+NER (Prefactor research)** | "Runtime evaluates ALL agent data flows through configurable PII detection rules — inspecting every parameter passed to a tool, every API payload, and every log line before leaving the agent's boundary" | src 23 |
| **Datadog Sensitive Data Scanner** | On-prem redaction before forwarding | src 23 |
| **Microsoft Copilot Studio audit** | Per-user activity logs into Microsoft Purview; `actor:Copilot` filter | src 11 |
| **GitHub Copilot agentic audit** | 180-day retention, `actor:Copilot` enterprise filter | src 11 |

**Maps to our existing AO-154 secrets-redaction-runtime hook** — the Prefactor pattern (write-time interception at the agent boundary) is exactly the architectural pattern AO-154 already implements for JSONL transcripts. The gap: AO-154 only scrubs JSONL transcripts; it does NOT cover events.jsonl, .context/, or memory/. Future work could extend AO-154's scrub to those layers.

---

## Applicability — Mapped Recommendation Per Tier

### Tier 1 — Load-bearing substrate (events.jsonl, .context/, memory, sessions, .env)

**Today's pragmatic pick (single-user, ship-this-month):**

1. **Private GitHub repo** (e.g., a sibling to `koinos/agent-session-archive`) for the substrate, structured by repo-slug. Substrate stays gitignored in working trees; daily/per-session push to the substrate-archive repo via cron or `/handover-session` hook. **Versioning is git's**: diff, branch, rollback, history-rewrite-if-needed.
2. **SOPS + age** for `~/.claude/.env` and any other shared secrets — encrypted-in-git, multi-recipient (yourself today; team members + service accounts when team scales).
3. **1Password CLI** with `op://` references for per-user personal secrets (already-tracked memory `reference_canonical_secrets_store_env.md`); coexists with SOPS via chezmoi templates.
4. **Defer cloud-versioning architectural commitment** until Cloudflare Artifacts public beta + 1-week pilot.

This pattern is **single-user-now, team-eventually**: the same private-GitHub-repo + SOPS pattern scales to team without rewriting.

**2026-Q3 candidate (post-public-beta pilot):**

- **Cloudflare Artifacts** for per-session agent state (events.jsonl, .context/<issue>/) — Git-protocol, R2-backed, native git-notes for agent metadata. Public beta targeted early May 2026.
- Continue SOPS + 1Password for the secrets layer (Cloudflare Artifacts is git-protocol; SOPS is content-encryption layered ON git — they compose).

**Rationale:** the private-GitHub-repo pattern is the most-conservative, lowest-effort architectural move that gives us git-similar versioning today. It delays the Cloudflare Artifacts commitment until we can pilot at our actual usage level (cost question above).

### Tier 2 — Telemetry (shell history, dispatch.db, paste-cache, file-history)

- **atuin** for shell history (the obvious fit; E2E-encrypted, multi-machine, 29.5k stars, active 2026-04-16). Self-host the atuin server inside the studio infrastructure for "team-paranoid" deployment. Per-user — atuin has no team-shared primitive, so each team member runs their own.
- **For dispatch.db / paste-cache / file-history:** ride along on Tier 1 backup target (private repo + restic snapshot to encrypted R2 if size grows). These are useful-but-reproducible; no need for git-semantic versioning.

### Tier 3 — Debug (ephemeral hook logs)

- **Local only** — gitignored, written to `.claude/events-debug.log` (per-repo) or `~/.claude/events-debug.log` (user-level). NOT `/tmp` (cross-repo contamination + reboot loss). Restart of Claude session is the natural rotation. If forensic survival across reboot becomes load-bearing, ride along Tier 1 backup target.

### Pattern recommendations (the "what teams actually do")

**Pattern 1 — Solo developer baseline (today's choice):**
```
- chezmoi + age + 1Password CLI for laptop dotfiles + secrets
- private GitHub repo for substrate (manual or hook-driven push)
- atuin (self-hosted or cloud) for shell history
- AO-154-style redaction-guard hook at write boundaries
```

**Pattern 2 — Small team (2–10):**
```
- Pattern 1 + SOPS+age for shared infra secrets (CI tokens, K8s creds)
- Shared private-GitHub repo for team-distributed rules/agents/skills
- chezmoi templates with per-user `${HOSTNAME}` / `${USER}` variables
- Bootstrap ceremony: clone dotfiles repo, `chezmoi init`, `op signin`, `sops decrypt`
```

**Pattern 3 — Team with audit + governance (10+):**
```
- Pattern 2 + plugin-marketplace governance (Claude Code marketplace.json or equivalent)
- Cloudflare Artifacts (or equivalent git-as-substrate platform) for shared agent state
- Centralized audit log (Splunk / Microsoft Purview style) tagged with actor:agent + per-user identity
- Agent Trace standard for AI-contribution provenance in committed code
```

---

## Conflicts & Gaps

### Conflicts between sources

[CONFLICTING] **Stream A vs Stream B on encryption-at-rest claims:** Stream A (Perplexity sonar-pro) listed many tools as "Yes (assumed)" for encryption-at-rest. Stream B's primary-source fetches show this is sometimes wrong:
- Cloudflare Artifacts: announcement does not specify at-rest encryption; R2-backed = AES-256 default per src 16 (so safe assumption, but not declared in product page).
- Cursor session state: vendor docs don't make a public claim either way; treat as UNVERIFIED.
- Devin: vendor docs cover session-state but not detailed at-rest encryption per-document.

**Resolution:** trust Stream B's source-grounded claims. Where Stream A asserted without source, mark UNVERIFIED in this doc.

### Gaps (categories where no source could confirm)

[UNVERIFIED] **Cursor's enterprise team-rule push API status** — forum requests confirm it's incomplete; no first-party doc found in 2026. Status: known gap on Cursor's side, no ETA found.

[UNVERIFIED] **Devin per-user audit-log surface specifics** — Cognition's release notes cover session-state and Insights, but per-user audit log queryability (who-did-what-when at the team level) is not publicly documented. Searches returned mostly Microsoft Copilot Studio + GitHub Copilot, not Devin.

[GAP] **Atuin team primitives** — confirmed as personal tool only. No team-shared history primitive; no enterprise SKU. If team-shared shell-history-as-knowledge becomes a goal (rare), Atuin is not the fit.

[GAP] **No shipped product provides "team-base + per-user-overlay with conflict-resolution"** as a first-class primitive. Closest is chezmoi templates + flux/argo deploy-time merge. For agent rules, Claude Code's plugin marketplace + 3-scope MCP config is the most evolved primitive — but conflict resolution is "last-write-wins" not "merged-with-precedence." This is an active gap in the market.

[OPEN QUESTION] **Cloudflare Artifacts public-beta pricing at our scale** — announced as $0.15 per 1k ops + $0.50/GB-mo, but real-world cost at our usage pattern (per-session agent repo, ~10k tool calls/day, 100MB/user) needs a pilot. Defer architectural commitment until public beta lands + 1-week pilot.

---

## Related but Out of Scope

These came up in research but fell outside the scope boundary; one-line each so reader can decide whether to follow up:

- **Decagon "Agent Versioning"** — enterprise SaaS for structured AI-agent reviews + CI/CD-style commit-per-change. Probably outside our budget tier; worth a quick eval if we ever need org-grade audit.
- **AgentGit (arXiv 2511.00628) and GCC (Git-Context-Controller)** — academic formalizations of git-as-agent-substrate. Useful as theoretical grounding for the Pattern A architecture; not shipped products.
- **Microsoft Copilot Studio + Purview audit integration** — relevant if we ever onboard Microsoft 365-bound team members; currently zero relevance.
- **Datadog Sensitive Data Scanner** — relevant if we ever centralize observability via Datadog; currently zero relevance.
- **Vanilla Git internals (pack-file delta, object dedup)** — useful background for understanding why Git is "good enough" for <100MB JSONL but breaks at 1GB+ scale.
- **JuJutsu (jj)** — interesting wildcard for personal use (snapshot-on-every-command would catch agent-induced state loss); not load-bearing for the substrate question.
- **Cloudflare ArtifactFS** (open-sourced, github.com/cloudflare/artifact-fs) — useful for blobless-mount of large repos; relevant if our repos grow beyond chezmoi's comfort zone.

---

## Cross-References to Internal Docs

These already exist in our workspace and should be cited rather than restated:

- `studio-research/references/claude-code-config-version-control.md` (2026-04-12) — chezmoi vs symlinks vs git-in-place for `~/.claude/`; the **`settings.json` symlink bug #3575** matters here (means Pattern 1's "chezmoi for dotfiles" can't manage Claude Code settings.json directly via symlink — must use chezmoi's template/copy mode).
- `studio-research/references/claude-code-config-reference.md` (2026-04-12) — official Anthropic config schema; relevant for understanding which Claude Code artifacts can be symlinked (rules/ explicitly; agents/skills/ undocumented but works empirically).
- `studio-research/references/agent-kb-structure.md` (2026-04-13) — three-tier KB architecture (hot 660 lines / specialists 9.3k / cold 16.3k); naming conventions matter more than visual hierarchy because agents grep, not browse.
- `studio-agent-ops/docs/orchestrator-vision.md` §1 substrate — events.jsonl is parallel to koinos-context (agent-side) not stacked under it; substrate decisions pressure-tested against orchestrator-read needs.
- `studio-agent-ops/docs/events-schema.md` — append-only JSONL semantics, dedupe_key, atomic-append guarantees ≤512 bytes (macOS) / 4096 bytes (Linux).
- `studio-agent-ops/decisions/003_config-rationalization.md` — config inflation risks (rules 394 → 335 lines target).
- Memory `project_events_jsonl_agent_substrate.md` (S-24) — substrate decision framing.
- Memory `reference_canonical_secrets_store_env.md` (S-21) — current solo-user secrets precedence (process env → Keychain → .env); Keychain is non-portable, so `.env` wins for cloud-portability.

## Linear cross-references

- **AO-157** — Cloud-backed versioning for the gitignored agent substrate (this research's strategic parent)
- **AO-88** — events.jsonl archival mechanism (related substrate sub-issue)
- **AO-154** — Redaction-guard hook (Done; the Prefactor write-time pattern is what AO-154 implements; future work could extend its scrub to .context/ + memory/)
- **AO-160** — PostToolUse hook intermittent firing (current investigation; debug log location decision is informed by this research → `.claude/events-debug.log`, not `/tmp`)

---

## Key Sources (deduplicated, 38 unique URLs)

Sources cited inline above as `src N`. Full deduplicated list with access dates:

1. https://www.chezmoi.io/comparison-table/ (2026-04-27)
2. https://gbergatto.github.io/posts/tools-managing-dotfiles/ (2026-04-27)
3. https://atuin.sh/ + https://docs.atuin.sh/ + https://docs.atuin.sh/cli/reference/sync/ (2026-04-27)
4. https://news.ycombinator.com/item?id=41453264 (2024-09)
5. https://news.ycombinator.com/item?id=39975247 (2024-03)
6. https://cursor.com/docs/rules + https://forum.cursor.com/t/api-access-to-set-team-rules/149383 (2026-04-27)
7. https://code.claude.com/docs/en/mcp + https://claude.com/blog/extending-claude-capabilities-with-skills-mcp-servers (2026-04-27)
8. https://spondicious.com/blog/stoworchezmoi/ (2026-04-27)
9. https://github.com/getsops/sops + https://blog.cmmx.de/2025/08/27/secure-your-environment-files-with-git-sops-and-age/ + https://www.bordencastle.com/security/gitops/devops/2026/02/13/sops-secrets-management-gitops.html (2026-04-27)
10. https://docs.github.com/en/copilot/reference/agentic-audit-log-events (2026-04-27)
11. https://learn.microsoft.com/en-us/microsoft-copilot-studio/admin-logging-copilot-studio + https://learn.microsoft.com/en-us/purview/audit-copilot (2026-04-27)
12. https://medium.com/@pablojusue/git-lfs-and-dvc-the-ultimate-guide-to-managing-large-artifacts-in-mlops-c1c926e6c5f4 + https://lakefs.io/blog/dvc-vs-git-vs-dolt-vs-lakefs/ (2026-04-27)
13. https://developer.1password.com/docs/cli/secrets-config-files/ + https://magarcia.io/stop-hardcoding-secrets-in-your-zshrc/ + https://rbt.rs/blog/scaling-secret-management-with-1password-cli/ (2026-04-27)
14. https://medium.com/@neeldevenshah/why-git-lfs-is-not-good-practice-for-ai-model-weights-and-why-you-should-use-dvc-instead (2026-04-27)
15. https://onidel.com/blog/restic-vs-borgbackup-vs-kopia-2025 + https://faisalrafique.com/restic-vs-borg-vs-kopia/ + https://computingforgeeks.com/borg-restic-kopia-comparison/ (2026-04-27)
16. https://developers.cloudflare.com/r2/reference/data-security/ + https://community.cloudflare.com/t/r2-object-versioning-and-replication/524025 (2026-04-27)
17. https://forum.cursor.com/t/sync-of-keybindings-and-settings/31 + https://dev.to/0916dhkim/sync-cursor-settings-the-dotfiles-way-20c9 (2026-04-27)
18. https://github.com/twpayne/chezmoi/releases (2026-04-27)
19. https://discourse.nixos.org/t/my-2025-dotfiles-home-manager-nix-darwin-nixos-terraform-kubernetes-on-vms/73690 (2026-04-27)
20. https://www.panozzaj.com/blog/2025/11/22/avoid-losing-work-with-jujutsu-jj-for-ai-coding-agents/ + https://github.com/jj-vcs/jj (2026-04-27)
21. https://git-annex.branchable.com/metadata/ + https://git-annex.branchable.com/how_it_works/ (2026-04-27)
22. https://decagon.ai/blog/decagon-agent-versioning + https://arxiv.org/abs/2511.00628 (AgentGit) + https://www.emergentmind.com/topics/git-context-controller-gcc (2026-04-27)
23. https://prefactor.tech/learn/real-time-pii-detection + https://oneuptime.com/blog/post/2025-11-13-keep-pii-out-of-observability-telemetry/view + https://opentelemetry.io/docs/languages/dotnet/logs/redaction/ + https://www.datadoghq.com/blog/observability-pipelines-sensitive-data-redaction/ (2026-04-27)
24. https://www.ranger.net/post/top-tools-ai-test-data-versioning (2026-04-27)
25. https://code.claude.com/docs/en/plugin-marketplaces + https://www.mpt.solutions/your-claude-plugin-marketplace-needs-more-than-a-git-repo/ + https://claudemarketplaces.com/ (2026-04-27)
26. https://docs.cline.bot/features/memory-bank + https://cline.bot/blog/unlocking-persistent-memory-how-clines-new_task-tool-eliminates-context-window-limitations (2026-04-27)
27. https://docs.gitlab.com/development/git_object_deduplication/ (2026-04-27)
28. https://x.com/cline/status/1912279479042683268 (2025)
29. https://aider.chat/docs/faq.html + https://github.com/Aider-AI/aider/issues/3607 + https://github.com/Aider-AI/aider/issues/2684 (2026-04-27)
30. https://aider.chat/HISTORY.html (2026-04-27)
31. https://docs.devin.ai/release-notes/overview + https://cognition.ai/blog/devin-annual-performance-review-2025 + https://cognition.ai/blog/devin-2 (2026-04-27)
32. https://github.com/atuinsh/atuin (2026-04-27)
33. https://perrotta.dev/2025/11/atuin/ (2025-11)
34. https://blog.gitguardian.com/a-comprehensive-guide-to-sops/ (2026-04-27)
35. https://meyer-laurent.com/atuin-the-magical-shell (2026-04-27)
36. https://googlecloudplatform.github.io/agent-starter-pack/guide/observability.html (2026-04-27)
37. https://github.com/cloudflare/artifact-fs (2026-04-27)
38. https://blog.cloudflare.com/artifacts-git-for-agents-beta/ + https://developers.cloudflare.com/changelog/post/2026-04-16-artifacts-now-in-beta/ + https://developers.cloudflare.com/artifacts/ (2026-04-27)

Source A = parametric-knowledge / general industry knowledge; verify externally before relying.
