---
title: "Continuous Agent/Config Operations Patterns"
description: "Patterns for managing AI agent fleets: label-based prompt promotion, agent manifests, toil reviews, DORA metrics for config, registry/catalog patterns, drift detection. Directly actionable for studio-agent-ops."
tags: [agent-ops, configuration, devops, toil, drift-detection, registry]
modified: 2026-04-13
status: active
confidence: high
staleness: 30
related:
  - ./claude-code-config-reference.md
  - ./claude-code-config-version-control.md
---

# Continuous Agent/Config Operations Patterns

> **Research date:** 2026-04-13
> **Last verified:** 2026-04-13
> **Staleness warning:** Re-verify after 30 days — this space is moving fast in 2025-2026
> **Confidence summary:** 18 verified, 9 likely, 4 unverified claims

## TL;DR

Four mature patterns map directly to solo-developer agent fleet management: (1) label-based prompt promotion (dev→staging→production) with no-code-deploy rollback, (2) an agent-manifest-per-repo registry using YAML with owner/version/tier metadata, (3) a monthly 3-metric toil review with a 15-20% toil budget target, and (4) git-diff-based drift scoring that catches config staleness at commit time without API calls. None of these require enterprise tooling — all have lightweight solo-developer implementations.

---

## How It Works

### 1. Agent/Config Management as Code

#### Label-Based Prompt Promotion (VERIFIED)

Prompts (and by extension, agent system configs) are versioned artifacts. Environment assignment is done via labels, not by changing the artifact. The labels `dev`, `staging`, `production` point to version numbers. Promoting to production = move the label.

**Why it works:** Code never needs to change to deploy a prompt update. Rollback is instant — move the `production` label back to the previous version. The agent references the label, not the version number.

**Concrete implementation (Langfuse pattern):**
```
prompts/
├── builder-agent/
│   ├── v1.yaml    # system prompt, model, tools, temperature
│   ├── v2.yaml
│   └── labels.yaml  # production: v2, staging: v3-draft
├── reviewer-agent/
│   └── ...
```
Source: https://langfuse.com/docs/prompt-management/features/prompt-version-control

#### Agent Manifest (agent.yaml) as First-Class Artifact (VERIFIED)

Every agent gets a minimal `agent.yaml` manifest alongside its behavioral files. The manifest declares identity, not behavior — it's the registry card, not the instruction set.

**Minimum viable manifest (GitAgent spec):**
```yaml
spec_version: "0.1.0"
name: builder
version: 0.3.0
model:
  preferred: claude-sonnet-4-6
compliance:
  risk_tier: medium          # low / medium / high
  supervision: human-in-loop
owner: aliargun
last_reviewed: 2026-04-13
repos:
  - aliargun/*               # which repos this agent operates in
```

**Why it works:** The manifest is queryable without reading the full behavioral spec. A script can `ls agents/*/agent.yaml` and build a fleet inventory in seconds.

Source: https://github.com/open-gitagent/gitagent

#### Monorepo Inheritance with Per-Agent Overrides (VERIFIED)

Shared capabilities (tool lists, base rules) live at the repo root. Individual agents only declare what's different. GitAgent calls this "root-level `context.md`, `skills/`, `tools/` are automatically shared across every agent in the monorepo."

**Why it works for fleet management:** A rule change in `rules/core-philosophy.md` propagates to all agents without touching each agent definition. Per-agent overrides are explicit and auditable.

#### Semantic Versioning for Prompts (VERIFIED)

Apply semver (Major.Minor.Patch) to prompt versions. Major = structural change in purpose or contract (breaks downstream). Minor = additive (new tool, expanded context). Patch = corrections (typos, formatting).

**Why it works:** Removes ambiguity about what changed and whether it's safe to roll out. v1.5.3 → v1.5.4 is safe to auto-promote; v1.5.4 → v2.0.0 requires validation gates.

#### Prompt Version Metadata Fields That Matter (VERIFIED)

Per the infrastructure-as-code pattern, every versioned prompt artifact should include:
- `prompt_version_id` — content hash or sequential ID (not a filename)
- `model` — pin the specific model, not just family
- `temperature` and inference params — changes here silently change behavior
- `tools` — which tools the agent is allowed to call
- `system_prompt` — the full text, not a path reference
- `last_reviewed` — date an engineer last validated this is still accurate
- `owner` — who is responsible when it breaks

### 2. Continuous Improvement Cycles for Operational Tooling

#### Monthly Toil Review (VERIFIED)

Google SRE defines toil as work that is manual, repetitive, automatable, non-tactical, lacks enduring value, and scales with infrastructure. The recommended practice is a monthly review of three metrics:

1. Toil hours per week (rolling 4-week average)
2. Top 5 recurring tasks with frequency + duration + risk scores
3. Pages/interrupts per on-call shift

**Toil budget target:** Push toil under 15-20% of total working time per quarter (more aggressive than Google's 50% ceiling — 50% is the ceiling before SRE value is destroyed, not the target).

**For solo developer:** Replace "on-call shifts" with "unplanned interruptions to flow." Track which agent-ops tasks you keep doing manually that you've meant to automate.

#### The 5-Day Toil Log (VERIFIED)

Before any improvement sprint, spend 5 working days logging every recurring task with: what it was, how long it took, frequency per week, and whether it's automatable. Score each task: frequency × duration × risk. Top scorers become the improvement backlog.

**Why it works:** Intuition about where toil lives is usually wrong. The 5-day log surfaces what actually consumes time, not what you remember complaining about.

#### DORA Applied to Agent Tooling (LIKELY)

Apply the same DORA metrics (deployment frequency, lead time, change failure rate, MTTR) to changes in agent config and tooling, not just product code. A config change that breaks an agent pipeline is a "change failure." Time to notice and roll back is "MTTR."

**Lightweight implementation:** Track in Linear. Each agent-ops commit that causes an unintended behavior change creates an incident issue. Monthly: count incidents, time-to-detect, time-to-resolve. No dashboard required — a monthly Linear query is sufficient for solo scale.

#### Kaizen Sprint (VERIFIED — methodology well established)

A dedicated improvement sprint (1-2 days) focused purely on reducing toil, retiring dead automation, and improving tooling quality. No new features. Cadence: quarterly.

**Kaizen for agent-ops specifically (LIKELY):**
- Review `agents/`, `skills/`, `rules/` for unused or stale entries
- Run audit-config on everything
- Check `last_reviewed` dates on agent manifests; any >90 days gets reviewed
- Retire or archive anything not used in the last quarter (check Linear issue history)

### 3. Registry and Catalog Patterns

#### File-Per-Component with Central Index (VERIFIED)

Backstage's core pattern: every component (agent, skill, rule) owns a `catalog-info.yaml` co-located in its directory. A central index (or script) walks the repo and generates a catalog view. No separate database required.

**Minimum viable metadata (from Backstage descriptor spec and GitAgent):**
```yaml
apiVersion: agent-ops/v1
kind: Agent
metadata:
  name: builder
  owner: aliargun
  version: 0.3.0
  last_reviewed: 2026-04-13
  risk_tier: medium
  repos: ["aliargun/*"]
  status: active   # active | deprecated | experimental
  description: "Implements features in an isolated worktree"
```

**For studio-agent-ops:** A shell script walking `agents/*/agent.yaml` + `skills/*/skill.yaml` and printing a markdown table is a sufficient catalog for solo scale. Backstage is premature infrastructure.

#### Heartbeat/Last-Seen for Staleness Detection (VERIFIED)

For static configs (no runtime heartbeat), the analog is `last_used` metadata derived from git history and Linear issue activity.

**Solo implementation:**
- `last_reviewed:` field in agent manifest (manual, updated on each deliberate review)
- Git log: `git log --since="90 days ago" -- agents/builder/` — if no commits, it may be stale or stable
- Linear query: any issue where this agent was the assigned agent type in the last 90 days

**Retirement criteria (LIKELY):**
- No Linear issues using this agent in 90+ days AND no commits in 90+ days → archive candidate
- Surface at quarterly kaizen sprint for deliberate keep/retire decision

#### Auto-Deprecation by Age (VERIFIED)

Runbooks and catalog entries that go 180 days without review are automatically marked "deprecated." This is not deletion — deprecated means flagged for human review, not removed.

**For agent configs:** Any `agent.yaml` with `last_reviewed` > 90 days → flagged. Any agent with no Linear activity in > 90 days → archive candidate. Both can be surfaced by the pm-hygiene agent.

#### Staleness Scoring (LIKELY)

Assign each catalog entry a freshness score (0-100) based on: required fields populated, days since last review, relationships still valid. Entries below a threshold (score < 50) trigger a notification.

**Solo implementation scoring:**
- `last_reviewed` < 30 days: +30 pts
- All required fields present: +20 pts
- Referenced skills/tools exist on disk: +30 pts
- Agent used in a Linear issue in last 90 days: +20 pts
- Score < 50 → flagged for quarterly kaizen review

### 4. Configuration Drift Detection and Remediation

#### Grounding Score: Filesystem Validation at Commit Time (VERIFIED)

Caliber's approach to drift detection is deterministic (no LLM calls): score config files against the actual filesystem at commit time. Checks:
- Do referenced paths exist on disk? (grounding — 20 pts)
- Are referenced commands/scripts still present? (accuracy — 15 pts)
- Was this config updated more recently than the code it describes? (freshness — 10 pts)

Source: https://github.com/caliber-ai-org/ai-setup

#### Git Diff-Based Config Refresh (VERIFIED)

After any commit, diff the changed files against the config files that reference them. If `agents/builder/agent.md` references a skill that was modified in this commit, the agent config may need updating.

**For studio-agent-ops:** A pre-commit hook that:
1. Reads `git diff --cached --name-only`
2. For each changed file, checks which agent manifests or rules reference it
3. Warns if a rule was changed without updating its `last_modified` date

#### The agent-guard "Non-Blocking Warning" Pattern (VERIFIED)

agent-guard's pre-commit hook never blocks commits — it warns and exits clean. The philosophy: enforcement fatigue from blocking hooks causes developers to bypass or disable them. Warning hooks preserve visibility without friction.

**Recommendation for hooks:** Warn, don't block, for staleness checks. Block only for structural violations (missing required fields, invalid syntax).

---

## Current State

The agent/config operations space is nascent but converging. Key trends as of 2026-04-13:

- **Agent registries** are emerging (TrueFoundry, AWS AI Agent Registry launched April 2026) but target enterprise multi-team scenarios
- **GitOps for prompts** is well-established via Langfuse and similar tools
- **Drift detection** for markdown-heavy configs (CLAUDE.md, rules/*.md) is less tooled than YAML/JSON config drift — Caliber addresses CLAUDE.md specifically but the space is nascent
- **No source found** on specific cadence patterns for reviewing AI agent behavior configs (as opposed to infrastructure configs) — the toil review and DORA patterns are borrowed from SRE/platform engineering

---

## Applicability to Our Stack

### What already exists (no work needed)
- Monorepo inheritance model — `rules/` is shared, agents override selectively. This is the Monorepo Inheritance pattern.
- Git-based audit trail — every config change is a commit.
- Linear as source of truth for usage — agent activity is tied to issues. Usage queries are possible today.

### What's missing (directly actionable)
1. **Agent manifests** — no `agent.yaml` per agent today. Adding one file per agent with 8 fields enables the catalog and staleness detection.
2. **Label files** — no explicit `production` pointer for agent definitions. Currently implicit (latest commit = production).
3. **Quarterly kaizen sprint** — not scheduled. Should be a recurring Linear milestone.
4. **Grounding check hook** — Caliber's rubric (grounding + accuracy + freshness scoring) is the spec for drift detection hooks.
5. **Toil tracking** — no current toil log. A simple running Linear label or comment convention would suffice.

### What's deliberately out of scope for solo scale
- Langfuse / PromptLayer (runtime prompt serving) — adds a network dependency; git + files is sufficient until 50+ prompts
- Backstage catalog — full platform for 1 developer is premature
- OPA/Rego — `yq` + shell validation achieves the same structural check

---

## Conflicts and Gaps

**Conflict:** Caliber's "LLM-free deterministic scoring" vs. agent-guard's "LLM-powered automatic refresh." Both valid but different philosophies. For studio-agent-ops, Caliber's approach (deterministic, no external calls) is better suited to a pre-commit hook.

**Gap:** No source found on specific cadence patterns for reviewing AI agent behavior configs. The toil review and DORA patterns are borrowed from SRE/platform engineering and extrapolated.

**Gap:** Drift detection for markdown-heavy configs is less tooled than YAML/JSON config drift. The space is nascent.

---

## Key Sources

1. Langfuse prompt version control: https://langfuse.com/docs/prompt-management/features/prompt-version-control (2026-04-13)
2. GitAgent framework (YAML schema): https://github.com/open-gitagent/gitagent (2026-04-13)
3. Caliber AI config sync: https://github.com/caliber-ai-org/ai-setup (2026-04-13)
4. TrueFoundry AI agent registry guide: https://www.truefoundry.com/blog/ai-agent-registry (2026-04-13)
5. Google SRE Workbook — eliminating toil: https://sre.google/workbook/eliminating-toil/ (2026-04-13)
6. Prompt management as infrastructure (DEV.to): https://dev.to/astronaut27/prompt-management-is-infrastructure-requirements-tools-and-patterns-32nn (2026-04-13)
7. Roadie Backstage catalog completeness: https://roadie.io/blog/3-strategies-for-a-complete-software-catalog/ (2026-04-13)
8. agent-guard pre-commit hook: https://dev.to/mossrussell/your-ai-agent-is-coding-against-fiction-how-i-fixed-doc-drift-with-a-pre-commit-hook-1acn (2026-04-13)
9. OPA pre-commit hooks: https://github.com/anderseknert/pre-commit-opa (2026-04-13)
10. SRE toil elimination playbook: https://oneuptime.com/blog/post/2025-10-01-what-is-toil-and-how-to-eliminate-it/view (2026-04-13)
11. AgentOps 2026 guide: https://medium.com/@Intellibytes/what-is-agentops-the-ultimate-2026-guide-to-ai-agent-operations-544876848ddd (2026-04-13)
12. DORA metrics guide: https://dora.dev/guides/dora-metrics/ (2026-04-13)
13. Backstage descriptor format: https://backstage.io/docs/features/software-catalog/descriptor-format/ (2026-04-13)
14. AgentOps Python SDK: https://github.com/AgentOps-AI/agentops (2026-04-13)
15. Agent Definition Language (ADL): https://www.nextmoca.com/blogs/agent-definition-language-adl-the-open-source-standard-for-defining-ai-agents (2026-04-13)
16. AWS AI Agent Registry: https://www.theregister.com/2026/04/09/aws_ai_agent_registry/ (2026-04-13)
17. LaunchDarkly prompt versioning: https://launchdarkly.com/blog/prompt-versioning-and-management/ (2026-04-13)
18. GitOps promotion pipeline: https://oneuptime.com/blog/post/2026-02-09-gitops-promotion-pipeline-dev-staging-prod/view (2026-04-13)
19. OpsLevel platform catalog patterns: https://www.opslevel.com/resources/2025-ultimate-guide-to-building-a-high-performance-developer-portal (2026-04-13)
20. Git as compliance audit trail (Kosli): https://www.kosli.com/blog/using-git-for-a-compliance-audit-trail/ (2026-04-13)
