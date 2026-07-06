# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a best practices repository for Claude Code configuration, demonstrating patterns for skills, subagents, hooks, and commands. It is a reference implementation and documentation collection, not an application codebase — there is no build system, package manager, or test suite. Content is Markdown docs plus live `.claude/` configuration that actually runs in sessions here.

## Answering Best Practice Questions

When the user asks a Claude Code best practice question, **always search this repo first** (`best-practice/`, `reports/`, `tips/`, `implementation/`, and `README.md`) before relying on training knowledge or external sources. This repo is the authoritative source — only fall back to external docs or web search if the answer is not found here.

## Repository Map

| Directory | Purpose |
|-----------|---------|
| `best-practice/` | Canonical reference docs: subagents, commands, skills, settings, memory, MCP, CLI flags, power-ups |
| `implementation/` | How each feature was implemented in this repo |
| `reports/` | Deep-dive research reports (agent memory, tool use, rate limits, …) |
| `tips/` | Dated tip collections from the Claude Code team (Boris, Thariq) |
| `orchestration-workflow/` | Weather system flow diagram and generated outputs |
| `development-workflows/` | Cross-model (Claude + Codex) and RPI workflow docs |
| `agent-teams/` | Agent teams prompt and outputs |
| `changelog/` | Drift-tracking state used by the `/workflows:*` commands |
| `tutorial/`, `videos/`, `presentation/` | Learning material |
| `.claude/` | Live examples: agents, commands, skills, hooks, rules, settings, agent-memory |

## Key Components

### Weather System (Example Workflow)
A demonstration of two distinct skill patterns via the **Command → Agent → Skill** architecture:
- `/weather-orchestrator` command (`.claude/commands/weather-orchestrator.md`): Entry point — asks user for C/F, invokes agent, then invokes SVG skill
- `weather-agent` agent (`.claude/agents/weather-agent.md`): Fetches temperature using its preloaded `weather-fetcher` skill (agent skill pattern); persists learnings via the `memory` frontmatter field to `.claude/agent-memory/weather-agent/MEMORY.md`
- `weather-fetcher` skill (`.claude/skills/weather-fetcher/SKILL.md`): Preloaded into agent — instructions for fetching temperature from Open-Meteo
- `weather-svg-creator` skill (`.claude/skills/weather-svg-creator/SKILL.md`): Invoked via the `Skill` tool — creates SVG weather card, writes `orchestration-workflow/weather.svg` and `orchestration-workflow/output.md`

Two skill patterns: agent skills (preloaded via `skills:` field) vs skills (invoked via `Skill` tool). See `orchestration-workflow/orchestration-workflow.md` for the complete flow diagram. The time system (`/time-command` → `time-agent` → `time-skill`) is a minimal second example of the same pattern.

### Maintenance Workflows
Six `/workflows:*` commands (`.claude/commands/workflows/`) keep the docs in sync with Claude Code releases. Each command is a coordinator that spawns its matching research agents from `.claude/agents/workflows/` **in parallel**, fetches official docs/changelog, compares against the local reports, and presents a unified drift report. These are read-then-report workflows — only take action if the user approves. Drift state lives in `changelog/`.

### Presentation System
See `.claude/rules/presentation.md` — all presentation work is delegated to the `presentation-curator` agent. Its three preloaded skills live in `.claude/skills/presentation/`.

### Hooks System
Cross-platform sound notification system in `.claude/hooks/`:
- `scripts/hooks.py`: Single handler for all Claude Code hook events
- `config/hooks-config.json`: Shared team configuration; `config/hooks-config.local.json`: personal overrides (git-ignored)
- `sounds/`: Audio files organized by hook event (generated via ElevenLabs TTS)

The authoritative list of wired hook events is the `hooks` key in `.claude/settings.json` (27 events as of v2.1.101) — check it there rather than relying on any enumerated list. Special handling: git commits trigger the `pretooluse-git-committing` sound.

## Critical Patterns

### Subagent Orchestration
Subagents **cannot** invoke other subagents via bash commands. Use the Agent tool (renamed from Task in v2.1.63; `Task(...)` still works as an alias):
```
Agent(subagent_type="agent-name", description="...", prompt="...", model="haiku")
```

Be explicit about tool usage in subagent definitions. Avoid vague terms like "launch" that could be misinterpreted as bash commands.

### Frontmatter Reference (Skills, Agents, Commands)
Do not rely on memorized field lists — frontmatter options change between Claude Code versions. The canonical, actively-maintained references are in this repo:
- Skills: `best-practice/claude-skills.md` (`context: fork`, `agent`, `allowed-tools`, preloading, hooks, …)
- Subagents: `best-practice/claude-subagents.md` (`tools`, `model`, `memory`, `permissionMode`, `effort`, `isolation`, …)
- Commands: `best-practice/claude-commands.md`

These docs are kept current by the `/workflows:*` commands; consult them before writing or editing any `.claude/` definition.

### Configuration Hierarchy
1. **Managed** (`managed-settings.json` / MDM plist / Registry): Organization-enforced, cannot be overridden
2. Command line arguments: Single-session overrides
3. `.claude/settings.local.json`: Personal project settings (git-ignored)
4. `.claude/settings.json`: Team-shared settings
5. `~/.claude/settings.json`: Global personal defaults
6. `hooks-config.local.json` overrides `hooks-config.json`

To disable hooks: set `"disableAllHooks": true` in `.claude/settings.local.json`, or disable individual hooks in `hooks-config.json`.

## Workflow Best Practices

From experience with this repository:

- Keep CLAUDE.md under 200 lines per file for reliable adherence — prefer pointers to docs over duplicated content, and repo-specific rules over generic advice
- Use commands for workflows instead of standalone agents
- Create feature-specific subagents with skills (progressive disclosure) rather than general-purpose agents
- Perform manual `/compact` at ~50% context usage
- Start with plan mode for complex tasks
- Use human-gated task list workflow for multi-step tasks
- Break subtasks small enough to complete in under 50% context

### Debugging Tips

- Use `/doctor` for diagnostics
- Run long-running terminal commands as background tasks for better log visibility
- Use browser automation MCPs (Claude in Chrome, Playwright, Chrome DevTools) for Claude to inspect console logs
- Provide screenshots when reporting visual issues

## Git Commit Rules

When committing changes, **create separate commits per file**. Do NOT bundle multiple file changes into a single commit. Each file gets its own commit with a descriptive message specific to that file's changes.

For example, if `README.md`, `best-practice/claude-subagents.md`, and a skill file all changed:
- Commit 1: `git add README.md` → commit with README-specific message
- Commit 2: `git add best-practice/claude-subagents.md` → commit with subagents-doc-specific message
- Commit 3: `git add .claude/skills/weather-fetcher/SKILL.md` → commit with skill-specific message

This makes the git history cleaner and easier to review, revert, or cherry-pick individual changes.

## Documentation

See `.claude/rules/markdown-docs.md` for documentation standards (structure, linking, README table updates). Key docs:
- `best-practice/claude-subagents.md`: Subagent frontmatter, hooks, and repository agents
- `best-practice/claude-commands.md`: Slash command patterns and built-in command reference
- `orchestration-workflow/orchestration-workflow.md`: Weather system flow diagram
