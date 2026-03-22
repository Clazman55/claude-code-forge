# Slash Commands

Custom slash commands that automate the repetitive parts of the development workflow. These are Claude Code skill files that trigger on specific phrases.

## How Slash Commands Work

Claude Code supports custom slash commands via skill files placed in `~/.claude/skills/`. Each command is a `SKILL.md` file inside a named subdirectory:

```
~/.claude/skills/
├── kickoff/
│   └── SKILL.md
├── phase-wrap/
│   └── SKILL.md
├── orient/
│   └── SKILL.md
└── ...
```

When you type `/kickoff` in a Claude Code session, it loads the corresponding `SKILL.md` and follows its instructions. The skill file defines what the command reads, what it does, and what it outputs.

## Installation

1. Copy the command directories from this repo's `commands/` folder into your `~/.claude/skills/` directory
2. Each directory contains a single `SKILL.md` file
3. The commands are immediately available in new Claude Code sessions

```bash
# Example: copy all commands
cp -r commands/* ~/.claude/skills/
```

Or copy individual commands:

```bash
# Just the essentials
cp -r commands/orient ~/.claude/skills/
cp -r commands/kickoff ~/.claude/skills/
cp -r commands/phase-wrap ~/.claude/skills/
```

## Command Reference

### Lifecycle Commands

| Command | When to Use | What It Does |
|---------|-------------|-------------|
| `/phase0` | Starting a new project | Research, plan, scaffold project directory, write CLAUDE.md, create plan file |
| `/kickoff` | Beginning a phase | Reads plan + code + memory, presents gameplan with tagged questions, waits for approval |
| `/phase-wrap` | Ending a phase | Updates CLAUDE.md, syncs project skills, updates memory/satellites, commits |
| `/stabilize` | Shelving a project | Full knowledge capture, skill sync, memory audit at a natural pause point |
| `/wrap` | Closing a project | Final satellite audit, canonical skill sync, cold storage move, final commit |

### Session Commands

| Command | When to Use | What It Does |
|---------|-------------|-------------|
| `/orient` | Starting any session | Detects session type (home/project), reads context files, presents current state |
| `/snapshot` | Mid-session preservation | Writes structured session state to disk for continuity across compactions |

### Quality Commands

| Command | When to Use | What It Does |
|---------|-------------|-------------|
| `/forge-review` | After writing code | Runs 3 parallel review subagents (reuse, quality, efficiency) then fixes issues found. This is a customized version of Claude Code's built-in `/simplify` that specifies three focused Sonnet subagents instead of a single general review. |
| `/test-audit` | After non-trivial test failures | Extracts root causes and patterns from test failures into the testing satellite |
| `/audit-code` | Project health check | Full audit: architecture, infrastructure, docs, testing, memory health. Report only -- no changes. |

### Exploration Commands

| Command | When to Use | What It Does |
|---------|-------------|-------------|
| `/brainstorm` | Open-ended thinking | Reads ideas folder, surveys accumulated work, enters exploration mode |

## Customization

These commands are designed to be modified. Common customizations:

- **Adjust the `/forge-review` subagents** -- change the review focus areas or add additional subagents
- **Add project-specific `/kickoff` steps** -- extra checks for your specific workflow
- **Create new commands** -- follow the same `SKILL.md` pattern for your own workflow automation

The `SKILL.md` format is straightforward: markdown instructions that Claude follows step-by-step. Read any existing command to see the pattern.

## Note on `/forge-review`

This command is a customization of Claude Code's built-in `/simplify` command. Both launch 3 parallel review agents for reuse, quality, and efficiency. The key differences in `/forge-review`:

1. **Explicit Sonnet model** -- each subagent is pinned to `model: "sonnet"`, keeping review costs predictable rather than inheriting the session model (which may be Opus)
2. **AI architecture checks** -- the quality agent includes checks from `code-architecture-for-ai.md`: module-level header comments on new files, and flagging functions exceeding ~50 lines for splitting
3. **Lifecycle integration** -- wired into the self-verification loop at step 7, with findings tracked in memory satellites per phase

The subagents run concurrently and their findings are consolidated before fixes are applied.
