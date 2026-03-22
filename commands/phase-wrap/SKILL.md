---
description: End-of-phase wrap — update project CLAUDE.md, project skills, memory/satellites, commit. Trigger on "phase wrap", "wrap the phase", or "end of phase".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /phase-wrap...**
>
> *Phase complete. Updating documentation and committing.*

# /phase-wrap — End of Phase Wrap-Up

Run this after implementation and testing are complete for a phase. This covers steps 9-11 of the self-verification loop, plus the commit.

Assumes:
- Code is written and working (step 2)
- TestingGuide is updated and user has tested (steps 3-5)
- Failures are fixed (step 6)
- `/forge-review` has been run and findings fixed (steps 7-8)

## Pre-Check: Testing Audit

Before starting the wrap, check: were there non-trivial test failures this phase (automated or manual) that required investigation beyond a straightforward fix? If yes, run `/test-audit` first. If everything passed cleanly or failures were trivial, proceed directly to step 1.

## Steps

### 1. Update Project CLAUDE.md
- Mark the current phase as complete in the Implementation Phases checklist: `- [x] Phase N: Description`
- Add any new architectural decisions discovered during the phase
- Update the directory structure if new files were added
- Update quality gate items if new testable requirements emerged

### 2. Update Project Skills (docs/)
- Review code written this phase for patterns worth capturing
- Add new patterns to the relevant skill file in `docs/` with `[verified in Phase N]` tag
- Update existing patterns if the phase revealed better approaches
- Do NOT sync to canonical skills (`~/.claude/skills/`) — that happens at stabilize or project wrap only

### 3. Update MEMORY.md and Satellites
- Update project MEMORY.md status: current phase complete, next phase identified
- Add any decisions, gotchas, or key facts discovered during the phase
- If MEMORY.md is getting long (approaching 200 lines), split detail into satellite files (systems.md, patterns.md)
- Update `/forge-review` findings section with what was caught and fixed this phase

### 4. Commit
Stage all changed files and commit with message: `Phase [N]: [Phase description summary]`

Run `git status` after commit to verify clean state.
