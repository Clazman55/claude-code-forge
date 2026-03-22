# Ideas Pipeline

A lightweight system for capturing project ideas before they're ready for Phase 0.

## How It Works

The ideas folder (`~/.claude/ideas/`) is a staging area for concepts that aren't ready for implementation. Each idea gets a short markdown file with:

- **Origin** -- where the idea came from (a project, a conversation, a pain point)
- **Problem** -- what it solves
- **Concept** -- rough shape of the solution
- **Scope** -- what's in and out
- **Open Questions** -- what needs answering before Phase 0

## Lifecycle

```
Observation → Idea file → Brainstorm session → Phase 0 → Project
                              ↓
                          Struck (not worth building)
```

Ideas can be:
- **Created** at any time, from any session
- **Discussed** during brainstorming sessions (home directory, `/brainstorm` command)
- **Promoted** to a project (Phase 0 creates the plan file, idea file moves to `archive/`)
- **Struck** if analysis shows the idea isn't worth pursuing (moved to `archive/` with note)

## Brainstorming Sessions

Open a home directory session, run `/brainstorm`, and Claude will:

1. Read all current idea files
2. Survey completed and active projects for context
3. Enter open-ended exploration mode

The brainstorming session is for thinking, not building. It produces refined idea files, not code.

## Template

```markdown
# Idea: [Name]

## Origin
[Where this came from]

## Problem
[What pain point this addresses]

## Concept
[Rough shape -- 5-10 lines max]

## Scope
- [In scope]
- [Not in scope]

## Open Questions
- [What needs answering before this becomes a project?]
```

## Tips

- Keep idea files short -- they're seeds, not specs
- One idea per file -- don't combine unrelated concepts
- The brainstorming session is where ideas get developed, not the idea file itself
- Don't prematurely promote -- an idea that isn't ready for Phase 0 isn't ready for Phase 0
- Archive rather than delete -- struck ideas still have value as a record of what was considered
