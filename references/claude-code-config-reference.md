---
title: "Claude Code Official Configuration Reference"
description: "Comprehensive reference for Claude Code config artifacts: CLAUDE.md, rules, agents, skills, memory, hooks, settings.json. 22 verified claims from official Anthropic docs (2026-04-12)."
tags: [claude-code, configuration, anthropic, agents, skills, hooks, rules]
modified: 2026-04-13
status: active
confidence: high
staleness: 14
related:
  - ./claude-code-config-version-control.md
---

# Claude Code Official Configuration Reference

> **Research date:** 2026-04-12
> **Last verified:** 2026-04-12
> **Staleness warning:** Re-verify weekly. Claude Code ships 1–3 updates per week — config surface can change with any release.
> **Confidence summary:** 22 verified, 2 likely, 0 unverified claims

## TL;DR

Official Anthropic docs define precise schemas and placement rules for every Claude Code config artifact. Key findings: CLAUDE.md should be under 200 lines (shorter = better adherence), rules support path-scoped frontmatter, agent frontmatter has 15+ fields, skills have a 13-field frontmatter schema, auto memory is loaded up to 200 lines/25KB per session, hooks support 24+ event types across 4 hook types (command, HTTP, prompt, agent). No official guidance on `skills/` having a special SKILL.md format beyond what the skills page documents — the format described in this repo's skills is **community/project-level convention, not an Anthropic standard**.

---

## How It Works

### 1. CLAUDE.md Files

**Source:** https://code.claude.com/docs/en/memory (accessed 2026-04-12)

#### Placement / Scope Table

**VERIFIED** — Four scopes, most-specific wins:

| Scope | Location | Purpose | Shared? |
|---|---|---|---|
| Managed policy | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`; Linux/WSL: `/etc/claude-code/CLAUDE.md`; Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | Org-wide, IT-managed | All users, cannot be excluded |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team-shared project instructions | Yes, via version control |
| User | `~/.claude/CLAUDE.md` | Personal, applies to all projects | Just you |
| Local | `./CLAUDE.local.md` | Personal, project-specific; add to `.gitignore` | Just you (current project) |

**VERIFIED** — Loading behavior:
- Files in the directory hierarchy *above* the working directory are loaded in full at launch
- Files in *subdirectories* load on demand when Claude reads files in those directories
- `CLAUDE.local.md` is appended after `CLAUDE.md` at each level (so personal notes take precedence at that level)
- All discovered files are concatenated, not overriding each other
- Block-level HTML comments (`<!-- ... -->`) are stripped before injection into context

#### Size and Structure Guidance

**VERIFIED** (explicit Anthropic recommendation):
- **Target under 200 lines per CLAUDE.md file.** "Longer files consume more context and reduce adherence."
- Use markdown headers and bullets to group related instructions
- Instructions are concrete if verifiable: "Use 2-space indentation" not "Format code properly"
- "The more specific and concise your instructions, the more consistently Claude follows them"

**VERIFIED** — Anti-patterns from official best practices page:
- If CLAUDE.md is too long, Claude ignores half of it because important rules get lost in the noise
- "Ruthlessly prune. If Claude already does something correctly without the instruction, delete it"
- Do NOT include: anything Claude can figure out by reading code, standard language conventions, detailed API docs (link instead), information that changes frequently, file-by-file descriptions, self-evident practices

**VERIFIED** — What to include:
- Bash commands Claude can't guess
- Code style rules that differ from defaults
- Testing instructions and preferred test runners
- Repository etiquette (branch naming, PR conventions)
- Architectural decisions specific to your project
- Developer environment quirks (required env vars)
- Common gotchas or non-obvious behaviors

#### Import Syntax

**VERIFIED** — `@path/to/import` syntax imports additional files into context:
```
See @README for project overview and @package.json for available npm commands.

# Additional Instructions
- git workflow @docs/git-instructions.md
```
- Relative paths resolve from the containing file, not working directory
- Max import depth: 5 hops
- First time external imports are encountered: approval dialog shown

#### `claudeMdExcludes` Setting

**VERIFIED** — Skip specific files by path or glob pattern in settings:
```json
{
  "claudeMdExcludes": [
    "**/monorepo/CLAUDE.md",
    "/home/user/monorepo/other-team/.claude/rules/**"
  ]
}
```
- Matches against absolute file paths using glob syntax
- Arrays merge across settings layers
- Managed policy CLAUDE.md files **cannot** be excluded

#### Managed vs Settings Separation

**VERIFIED** — Official guidance on what belongs where:

| Concern | Configure in |
|---|---|
| Block tools/commands/file paths | Managed settings: `permissions.deny` |
| Enforce sandbox isolation | Managed settings: `sandbox.enabled` |
| Env vars and API routing | Managed settings: `env` |
| Auth method / org lock | Managed settings: `forceLoginMethod` |
| Code style, quality guidelines | Managed CLAUDE.md |
| Data handling / compliance reminders | Managed CLAUDE.md |
| Behavioral instructions | Managed CLAUDE.md |

> "Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer."

### 2. Rules Files (`.claude/rules/` and `~/.claude/rules/`)

**Source:** https://code.claude.com/docs/en/memory (accessed 2026-04-12)

#### Placement

**VERIFIED:**
- **Project rules:** `.claude/rules/` — loaded at launch with same priority as `.claude/CLAUDE.md`
- **User-level rules:** `~/.claude/rules/` — apply to every project; loaded *before* project rules (so project rules have higher priority)
- File naming: descriptive filenames like `testing.md`, `api-design.md`
- All `.md` files discovered recursively — can organize into subdirectories like `frontend/`, `backend/`

#### Loading Behavior

**VERIFIED:**
- Rules without `paths` frontmatter: loaded at launch unconditionally
- Rules with `paths` frontmatter: loaded only when Claude reads files matching the pattern
- User-level rules load before project rules; project rules take higher precedence

#### Path-Scoped Frontmatter

**VERIFIED** — Rules can be scoped to specific files using YAML frontmatter:
```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules
...
```

Supported glob patterns:

| Pattern | Matches |
|---|---|
| `**/*.ts` | All TypeScript files in any directory |
| `src/**/*` | All files under `src/` |
| `*.md` | Markdown files in project root |
| `src/components/*.tsx` | React components in specific directory |

Brace expansion supported: `"src/**/*.{ts,tsx}"`.

#### Symlink Support

**VERIFIED** — `.claude/rules/` supports symlinks:
- Resolved and loaded normally
- Circular symlinks detected and handled gracefully
- Official example: `ln -s ~/shared-claude-rules .claude/rules/shared`

#### Important Distinction: Rules vs Skills

**VERIFIED** — Official guidance:
> "Rules load into context every session or when matching files are opened. For task-specific instructions that don't need to be in context all the time, use skills instead, which only load when you invoke them or when Claude determines they're relevant to your prompt."

### 3. Agents (`.claude/agents/` and `~/.claude/agents/`)

**Source:** https://code.claude.com/docs/en/sub-agents (accessed 2026-04-12)

#### Placement and Scope

**VERIFIED:**

| Location | Scope | Priority (1=highest) |
|---|---|---|
| CLI `--agents` flag | Current session only | 1 |
| Managed settings `.claude/agents/` | All users | 2 |
| `.claude/agents/` | Current project | 3 |
| `~/.claude/agents/` | All your projects | 4 |
| Plugin `agents/` dir | Where plugin enabled | 5 |

Higher-priority location wins when names conflict.

#### File Format

**VERIFIED** — Markdown files with YAML frontmatter:
```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a senior code reviewer...
```

#### Supported Frontmatter Fields

**VERIFIED** (complete list from official docs):

| Field | Required | Description |
|---|---|---|
| `name` | No | Display name. If omitted, uses filename. |
| `description` | **Yes** | When Claude should delegate to this subagent |
| `tools` | No | Allowlist of tools. Inherits all if omitted |
| `disallowedTools` | No | Denylist — removed from inherited or specified list |
| `model` | No | `sonnet`, `opus`, `haiku`, full model ID, or `inherit`. Defaults to `inherit` |
| `permissionMode` | No | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, or `plan` |
| `maxTurns` | No | Maximum agentic turns before subagent stops |
| `skills` | No | Skills to preload into subagent context at startup (full content injected) |
| `mcpServers` | No | MCP servers available to subagent |
| `hooks` | No | Hooks scoped to this subagent's lifecycle |
| `memory` | No | Persistent memory scope: `user`, `project`, or `local` |
| `effort` | No | `low`, `medium`, `high`, `max` (Opus 4.6 only). Inherits from session if omitted |
| `isolation` | No | Set to `worktree` for isolated git worktree copy |
| `background` | No | (UI) background color |
| `color` | No | `red`, `blue`, `green`, `yellow`, `purple`, `orange`, `pink`, or `cyan` |
| `initialPrompt` | No | Auto-submitted first user turn when running as main session agent |

**VERIFIED** — Tools field behavior:
- If both `tools` and `disallowedTools` set: `disallowedTools` applied first, then `tools` resolved against remaining pool
- Tool in both lists: removed
- To allow specific Agent subtypes: `tools: Agent(worker, researcher), Read, Bash`
- To allow all Agent types: `tools: Agent, Read, Bash`

#### Permission Mode Inheritance

**VERIFIED:**
- Subagents inherit permission context from parent conversation
- `bypassPermissions` parent: cannot be overridden by subagent
- `auto` parent: subagent inherits auto mode; `permissionMode` frontmatter ignored

#### Plugin Security Restriction

**VERIFIED:**
> "For security reasons, plugin subagents do not support the `hooks`, `mcpServers`, or `permissionMode` frontmatter fields. These fields are ignored when loading agents from a plugin."

### 4. Skills (`~/.claude/skills/` and `.claude/skills/`)

**Source:** https://code.claude.com/docs/en/skills (accessed 2026-04-12)

#### Important Context

**VERIFIED** — Skills are an Anthropic-standardized format, following the [Agent Skills](https://agentskills.io) open standard:
> "Custom commands have been merged into skills. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way."

#### Placement

**VERIFIED:**

| Location | Applies to |
|---|---|
| Enterprise managed | All users in org |
| `~/.claude/skills/<skill-name>/SKILL.md` | All your projects |
| `.claude/skills/<skill-name>/SKILL.md` | This project only |
| Plugin `skills/<skill-name>/SKILL.md` | Where plugin enabled |

When skills share the same name: enterprise > personal > project. Plugin skills use `plugin-name:skill-name` namespace.

#### SKILL.md Format

**VERIFIED** — Every skill directory requires a `SKILL.md` entrypoint:
```
my-skill/
├── SKILL.md           # Main instructions (required)
├── template.md        # Optional supporting files
├── examples/
│   └── sample.md
└── scripts/
    └── validate.sh
```

**VERIFIED** — Tip: "Keep `SKILL.md` under 500 lines. Move detailed reference material to separate files."

#### Frontmatter Reference

**VERIFIED** (complete official list):

| Field | Required | Description |
|---|---|---|
| `name` | No | Lowercase letters, numbers, hyphens only; max 64 characters. If omitted, uses directory name |
| `description` | Recommended | Used by Claude to decide when to apply. **Truncated at 250 characters** in skill listing. Front-load key use case |
| `argument-hint` | No | Hint shown during autocomplete. Example: `[issue-number]` |
| `disable-model-invocation` | No | `true` = only user can invoke (not auto-loaded by Claude). Default: `false` |
| `user-invocable` | No | `false` = hidden from `/` menu. Default: `true` |
| `allowed-tools` | No | Tools Claude can use without permission prompts. Space-separated string or YAML list |
| `model` | No | Model to use when skill is active |
| `effort` | No | `low`, `medium`, `high`, `max` (Opus 4.6 only). Overrides session effort |
| `context` | No | Set to `fork` to run in forked subagent context |
| `agent` | No | Subagent type when `context: fork` set. Built-in: `Explore`, `Plan`, `general-purpose` |
| `hooks` | No | Hooks scoped to skill lifecycle |
| `paths` | No | Glob patterns limiting when skill activates. Comma-separated or YAML list |
| `shell` | No | `bash` (default) or `powershell` for inline commands |

#### `disable-model-invocation` vs `user-invocable`

**VERIFIED** — Invocation control matrix:

| Frontmatter | User can invoke | Claude can invoke | In context |
|---|---|---|---|
| (default) | Yes | Yes | Description always in context |
| `disable-model-invocation: true` | Yes | No | Description NOT in context |
| `user-invocable: false` | No | Yes | Description always in context |

#### String Substitutions

**VERIFIED:**

| Variable | Description |
|---|---|
| `$ARGUMENTS` | All arguments passed on invocation |
| `$ARGUMENTS[N]` | Specific argument by 0-based index |
| `$N` | Shorthand for `$ARGUMENTS[N]` |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Directory containing the skill's SKILL.md |

#### Context Budget

**VERIFIED** — Skill description budget:
- Scales dynamically at 1% of context window, fallback 8,000 characters
- Each description entry capped at 250 characters
- Set `SLASH_COMMAND_TOOL_CHAR_BUDGET` env var to raise limit

#### Shell Execution (Security)

**VERIFIED** — `` !`command` `` syntax runs shell commands before skill content is sent to Claude. To disable: set `"disableSkillShellExecution": true` in settings.

### 5. Memory (Auto Memory)

**Source:** https://code.claude.com/docs/en/memory (accessed 2026-04-12)

#### What It Is

**VERIFIED:**
- Auto memory lets Claude accumulate knowledge across sessions without you writing anything
- Content: build commands, debugging insights, architecture notes, code style preferences, workflow habits
- Requires Claude Code v2.1.59 or later

#### Storage Location

**VERIFIED:**
```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # Concise index, loaded into every session
├── debugging.md       # Detailed notes on debugging patterns
├── api-conventions.md # API design decisions
└── ...                # Other topic files Claude creates
```
- `<project>` path derived from git repository — all worktrees share one directory
- Outside git repo: project root used

#### Loading Behavior

**VERIFIED:**
- **First 200 lines of `MEMORY.md`, or first 25KB, whichever comes first** — loaded at session start
- Content beyond that threshold: NOT loaded at session start
- Topic files (e.g., `debugging.md`): NOT loaded at startup; Claude reads on demand
- CLAUDE.md files are loaded in full regardless of length

#### Configuration

**VERIFIED:**
```json
{
  "autoMemoryEnabled": false,
  "autoMemoryDirectory": "~/my-custom-memory-dir"
}
```
- `autoMemoryDirectory` accepted from policy, local, and user settings only — NOT from project settings (security: prevents redirecting writes to sensitive locations)
- Env var: `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`

### 6. Hooks (settings.json)

**Source:** https://code.claude.com/docs/en/hooks-guide (accessed 2026-04-12), https://code.claude.com/docs/en/hooks (accessed 2026-04-12)

#### What Hooks Are

**VERIFIED:**
> "Hooks are user-defined shell commands that execute at specific points in Claude Code's lifecycle. They provide deterministic control over Claude Code's behavior, ensuring certain actions always happen rather than relying on the LLM to choose to run them."

#### Configuration Location

**VERIFIED** — hooks are configured in settings.json files:

| Location | Scope | Shareable |
|---|---|---|
| `~/.claude/settings.json` | All your projects | No |
| `.claude/settings.json` | Single project | Yes (committable) |
| `.claude/settings.local.json` | Single project | No (gitignored) |
| Managed policy settings | Organization-wide | Yes, admin-controlled |
| Plugin `hooks/hooks.json` | When plugin enabled | Yes |
| Skill/agent frontmatter | While component active | Yes |

#### Complete List of Hook Events (24 events)

**VERIFIED:**

| Event | When it fires | Can block? |
|---|---|---|
| `SessionStart` | Session begins or resumes | No (output added to context) |
| `SessionEnd` | Session terminates | No |
| `UserPromptSubmit` | Prompt submitted, before Claude processes | Yes (exit 2 erases prompt) |
| `PreToolUse` | Before tool call executes | Yes |
| `PermissionRequest` | When permission dialog appears | Yes |
| `PermissionDenied` | Tool call denied by auto mode | No (use JSON `retry: true`) |
| `PostToolUse` | After tool call succeeds | No (shows stderr to Claude) |
| `PostToolUseFailure` | After tool call fails | No |
| `Notification` | Claude Code sends a notification | No (shows to user) |
| `SubagentStart` | Subagent spawned | No |
| `SubagentStop` | Subagent finishes | Yes |
| `TaskCreated` | Task created via TaskCreate | Yes |
| `TaskCompleted` | Task marked as completed | Yes |
| `Stop` | Claude finishes responding | Yes |
| `StopFailure` | Turn ends due to API error | No |
| `TeammateIdle` | Agent team teammate about to go idle | Yes |
| `InstructionsLoaded` | CLAUDE.md or rules/*.md loaded | No |
| `ConfigChange` | Config file changes during session | Yes |
| `CwdChanged` | Working directory changes | No |
| `FileChanged` | Watched file changes on disk | No |
| `WorktreeCreate` | Worktree being created | Yes (non-zero = fails) |
| `WorktreeRemove` | Worktree being removed | No |
| `PreCompact` | Before context compaction | No |
| `PostCompact` | After context compaction completes | No |
| `Elicitation` | MCP server requests user input | Yes |
| `ElicitationResult` | User responds to MCP elicitation | Yes |

#### Hook Configuration Schema

**VERIFIED:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write",
            "async": false,
            "timeout": 600,
            "statusMessage": "Formatting...",
            "once": false
          }
        ]
      }
    ]
  }
}
```

#### Hook Types

**VERIFIED** — Four hook types:

| Type | Description | Fields |
|---|---|---|
| `command` | Shell command | `command`, `async`, `shell` |
| `http` | POST to URL | `url`, `headers`, `allowedEnvVars` |
| `prompt` | Single-turn LLM evaluation | `prompt`, `model` (defaults to Haiku) |
| `agent` | Multi-turn LLM with tool access | `prompt`, `model`, timeout 60s, up to 50 turns |

#### Matcher Patterns

**VERIFIED:**
- Empty string `""` or omitted: fires on every occurrence
- Letters, digits, `_`, `|`: exact string or `|`-separated list (e.g., `Edit|Write`)
- Any other characters: JavaScript regex (e.g., `mcp__.*`)
- **Matchers are case-sensitive**

`if` field (requires v2.1.85+): filter by tool name AND arguments using permission rule syntax: `"Bash(git *)"`, `"Edit(*.ts)"`.

#### Exit Codes

**VERIFIED:**
- **Exit 0**: action proceeds; stdout parsed for JSON
- **Exit 2**: action blocked; stderr fed to Claude as feedback
- **Any other**: non-blocking error; transcript shows `<hook name> hook error`

#### Environment Variables Available to Hooks

**VERIFIED:**

| Variable | Available in | Description |
|---|---|---|
| `CLAUDE_PROJECT_DIR` | All hooks | Project root (quoted for spaces) |
| `CLAUDE_PLUGIN_ROOT` | Plugin hooks | Plugin installation directory |
| `CLAUDE_PLUGIN_DATA` | Plugin hooks | Plugin persistent data directory |
| `CLAUDE_ENV_FILE` | SessionStart, CwdChanged, FileChanged | File path for persisting env vars |
| `CLAUDE_CODE_REMOTE` | All hooks | Set to `"true"` in web, unset locally |

#### Security Note

**VERIFIED:**
- Hook commands run with full file system access
- `allowedEnvVars` required for header env var interpolation in HTTP hooks
- `allowManagedHooksOnly` setting (managed only): blocks user/project hooks

#### Disable All Hooks

**VERIFIED:**
```json
{
  "disableAllHooks": true
}
```

### 7. Settings.json Schema

**Source:** https://code.claude.com/docs/en/settings (accessed 2026-04-12)

#### Scope Precedence (highest to lowest)

**VERIFIED:**
1. Managed settings (server-managed > MDM/OS-level > file-based > HKCU registry)
2. CLI/programmatic options
3. Local project settings (`.claude/settings.local.json`)
4. Project settings (`.claude/settings.json`)
5. User settings (`~/.claude/settings.json`)

#### Settings File Locations

**VERIFIED:**

| Scope | Location |
|---|---|
| Managed | macOS: `/Library/Application Support/ClaudeCode/managed-settings.json`; Linux: `/etc/claude-code/managed-settings.json`; Windows: `C:\Program Files\ClaudeCode\managed-settings.json` |
| User | `~/.claude/settings.json` |
| Project | `.claude/settings.json` |
| Local | `.claude/settings.local.json` (auto-gitignored) |

#### Key Settings Fields

**VERIFIED** — JSON Schema available at `https://json.schemastore.org/claude-code-settings.json` (add `"$schema"` key to get VS Code autocomplete).

| Setting | Description |
|---|---|
| `claudeMdExcludes` | Glob patterns to skip CLAUDE.md files. Arrays merge across layers |
| `autoMemoryEnabled` | Enable/disable auto memory. Default: `true` |
| `autoMemoryDirectory` | Custom auto memory location. Not accepted from project settings |
| `disableAllHooks` | Disable all hooks. Default: `false` |
| `disableSkillShellExecution` | Prevent `` !`command` `` execution in skills. Default: `false` |
| `env` | Environment variables applied to every session |
| `hooks` | Hook configuration (see Hooks section above) |
| `permissions.allow` / `permissions.deny` | Tool permission rules |
| `sandbox.enabled` | OS-level sandbox isolation |

#### Merge Behavior

**VERIFIED:**
- Arrays (e.g., `claudeMdExcludes`, `permissions.allow`): **concatenated and de-duplicated** across layers
- Scalar values: more specific scope overrides broader
- Objects: deep-merged

#### Managed Settings Drop-in Directory

**VERIFIED** — For Linux, a drop-in directory is supported:
```
/etc/claude-code/managed-settings.d/
```
Files sorted alphabetically and merged; numeric prefixes control merge order (e.g., `10-telemetry.json`, `20-security.json`).

---

## Current State

### Official Config Hygiene Recommendations

**Source:** https://code.claude.com/docs/en/best-practices, https://code.claude.com/docs/en/memory (accessed 2026-04-12)

#### What To Use for What

**VERIFIED** — Official guidance from best practices page:

| When you need... | Use |
|---|---|
| Project-wide persistent context | `CLAUDE.md` |
| Personal context across projects | `~/.claude/CLAUDE.md` |
| Personal project-specific notes (not shared) | `CLAUDE.local.md` |
| Topic-specific instructions (project) | `.claude/rules/<topic>.md` |
| Reusable workflows / domain knowledge | Skills (`SKILL.md`) |
| Actions that MUST happen every time (deterministic) | Hooks |
| Isolated task delegation | Sub-agents |
| Auto-learned preferences across sessions | Auto memory |

> "CLAUDE.md is loaded every session, so only include things that apply broadly. For domain knowledge or workflows that are only relevant sometimes, use skills instead."

#### CLAUDE.md Anti-Patterns (Official)

**VERIFIED** — From best practices page (official "what not to include" list):
1. Anything Claude can figure out by reading code
2. Standard language conventions Claude already knows
3. Detailed API documentation (link to docs instead)
4. Information that changes frequently
5. Long explanations or tutorials
6. File-by-file descriptions of the codebase
7. Self-evident practices like "write clean code"

#### Rules vs. CLAUDE.md vs. Skills Decision

**VERIFIED** — Official guidance:
- CLAUDE.md: for facts Claude should hold in every session
- Rules: for path-specific or topical instruction sets — loaded conditionally or always, but designed for instructions not procedures
- Skills: for multi-step procedures, procedures that apply only sometimes, domain knowledge

#### Context and Size Warnings

**VERIFIED** (multiple sources):
- CLAUDE.md under 200 lines: official recommendation
- SKILL.md under 500 lines: official tip
- MEMORY.md: first 200 lines/25KB loaded; beyond that ignored at session start
- Too many skills: descriptions truncated in context (250 char cap per description)
- "Bloated CLAUDE.md files cause Claude to ignore your actual instructions!"

#### Deduplication

**LIKELY** (inferred from multiple official statements, not stated as a single rule):
- Official docs consistently emphasize "one source per instruction" by routing content to the right mechanism
- Conflicting instructions across CLAUDE.md files: "Claude may pick one arbitrarily"
- Recommendation: "Review your CLAUDE.md files, nested CLAUDE.md files in subdirectories, and `.claude/rules/` periodically to remove outdated or conflicting instructions"

---

## Applicability to Our Stack

This reference is directly applicable — Claude Code is the primary development tool across all studio repos. The config-hygiene rule (`~/.claude/rules/config-hygiene.md`) and subagent-routing rule (`~/.claude/rules/subagent-routing.md`) were built from these findings.

Key constraints derived from this research:
- Global `~/.claude/CLAUDE.md` stays under 200 lines — identity only
- Rules split between global (`~/.claude/rules/`) and project (`.claude/rules/`)
- Skills use the official `SKILL.md` format with accurate frontmatter
- Hooks configured in `settings.json`, not in CLAUDE.md

---

## Conflicts & Gaps

1. **No official documentation for `~/.claude/skills/` SKILL.md "frontmatter" for disable-model-invocation globally** — the docs confirm `disable-model-invocation` as a per-skill frontmatter field, not a global config. Our repo's CLAUDE.md routing table ("Suggest-only skills have `disable-model-invocation: true`") is consistent with official docs.

2. **No official documentation for a separate `SKILL.md` as a "routing table" for Claude.** The skill's `description` field is the routing mechanism. The current pattern of using a `CLAUDE.md` table to explain skills Claude can't see (because `disable-model-invocation: true` hides them from model context) is a **community/project-level workaround** with no official equivalent.

3. **Skills vs. commands distinction:** officially merged. `.claude/commands/*.md` files still work identically to `.claude/skills/<name>/SKILL.md`. The skills format is preferred going forward.

4. **`InstructionsLoaded` hook matchers:** load reason values are `session_start`, `nested_traversal`, `path_glob_match`, `include`, `compact`. Useful for auditing which files load and why — underdocumented in the main hooks guide.

---

## Key Sources

1. https://code.claude.com/docs/en/memory — CLAUDE.md, rules, auto memory (accessed 2026-04-12)
2. https://code.claude.com/docs/en/settings — settings.json schema, scopes, precedence (accessed 2026-04-12)
3. https://code.claude.com/docs/en/hooks-guide — hooks guide with all 24 event types (accessed 2026-04-12)
4. https://code.claude.com/docs/en/hooks — full hooks reference (accessed 2026-04-12)
5. https://code.claude.com/docs/en/sub-agents — agent frontmatter schema (accessed 2026-04-12)
6. https://code.claude.com/docs/en/skills — SKILL.md frontmatter schema (accessed 2026-04-12)
7. https://code.claude.com/docs/en/best-practices — config hygiene, what to put where (accessed 2026-04-12)
8. https://code.claude.com/docs/en/overview — overview with all config mechanisms listed (accessed 2026-04-12)
9. https://json.schemastore.org/claude-code-settings.json — official settings JSON schema for autocomplete
