---
description: Session orientation — read context files, detect session type, present current state and next action. Trigger on "orient", "get oriented", "where are we", or "bootup".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /orient...**
>
> *Session orientation. Context is blank -- reconstructing state from skill files and memory.*

# /orient — Session Orientation

Detects what kind of session this is and reads the appropriate context. Replaces copy-pasting prompt templates.

## Step 1: Detect Session Type

Check the current working directory:

- **Home directory** (`~` or `~/.claude`): This is a **home session** — cross-project work, maintenance, brainstorming, Phase 0.
- **Project directory** (e.g., `E:\ProjectName`): This is a **project session** — implementation, testing, phase work on a specific project.

## Step 2: Read Context (Home Session)

If this is a home session:

1. Read `~/.claude/CLAUDE.md` — global instructions and working relationship
2. Read `~/.claude/projects/<home-key>/memory/MEMORY.md` — project history, user profile, ideas pipeline
3. Scan `~/.claude/ideas/` — list what idea files exist with a one-line summary of each
4. Check `~/.claude/skills/index.md` — know what skills are available

## Step 2: Read Context (Project Session)

If this is a project session:

1. Read `~/.claude/CLAUDE.md` — global instructions
2. Read the project's `CLAUDE.md` at the project root — project-specific instructions, architecture, quality gate
3. Find this project's session memory folder in `~/.claude/projects/` — read its MEMORY.md and any satellite files (systems.md, patterns.md, etc.)
4. Read the plan file referenced in the home directory MEMORY.md bootstrap entry for this project

## Step 3: Present Status

After reading all sources, present a brief orientation report:

```
## Session Oriented

**Type:** [Home / Project: ProjectName]
**Current state:** [What phase, what's done, what's next]
**Dirty files:** [Any uncommitted changes — run git status]
**Next action:** [What the user most likely wants to do based on current state]
```

Then wait for instructions. Do not start working until directed.
