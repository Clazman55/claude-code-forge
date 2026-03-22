---
description: Phase testing audit — review what broke, extract root causes and patterns into testing satellite. Conditional on non-trivial failures. Trigger on "test audit", "audit the tests", or "testing audit".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /test-audit...**
>
> *Reviewing the phase's failure record. The failures teach more than the successes.*

# /test-audit -- Phase Testing Audit

Retrospective extraction of testing lessons from the current phase. Run during phase-wrap, after forge-review fixes and before CLAUDE.md update. Not every phase needs this -- only phases where something non-trivial broke.

## When to Run

- Phase had test failures (automated or manual) that required investigation beyond a straightforward fix
- A bug's root cause was surprising or non-obvious
- The same category of failure appeared more than once in the phase
- A test gap was discovered (something broke that should have been caught earlier)

If everything passed first try or failures were trivial typo-level fixes, skip this and proceed with `/phase-wrap`.

## Step 1: Gather the Phase Record

Read the following to reconstruct what happened:

1. **Conversation history** -- scroll back through the phase's work to identify test failures and fixes
2. **Git diff** -- `git diff HEAD~1` (or appropriate range) to see what changed, focusing on fix commits
3. **TestingGuide.html** -- check which manual test steps failed during user testing
4. **Automated test files** -- check for tests that were added or modified to catch discovered issues

## Step 2: Classify Failures

For each non-trivial failure, document:

| Field | Description |
|-------|-------------|
| **What broke** | Observable symptom (test name, user report, error message) |
| **Root cause** | The actual bug -- not the symptom, the underlying defect |
| **Fix applied** | What code change resolved it |
| **Detectable earlier?** | Could the architecture, tests, or review process have caught this before it manifested? How? |
| **Category** | One of: logic error, scope/boundary error, formula misapplication, state management, integration gap, missing validation, race condition, design flaw, other |

## Step 3: Extract Patterns

Look across the classified failures for:

- **Recurring categories** -- if two failures share a category, that's a pattern worth naming
- **Test coverage gaps** -- failures that should have been caught by automated tests but weren't. Recommend specific tests to add.
- **Assumption failures** -- cases where the code was correct for the wrong assumption. These often point to design documentation gaps.
- **Formula/equations errors** -- if the project maintains an equations reference document (see lifecycle skill section 6), cross-reference failures against it. Formula misapplications and scope errors are common failure modes in these systems. Flag any equations doc entries that need clarification.

## Step 4: Write to Testing Satellite

Write findings to the project's testing satellite file (`testing-patterns.md` in the project memory folder). If the satellite doesn't exist yet, create it.

### Satellite Format

```markdown
# Testing Patterns — [Project Name]

## Phase [N] Audit

### Failures
- **[What broke]**: [Root cause]. Fix: [fix summary]. Category: [category].
  Detectable earlier: [yes/no — how]

### Patterns Identified
- [Pattern name]: [description, evidence, recommended prevention]

### Test Coverage Gaps
- [Gap]: [what test should exist, what it should verify]

### Equations Cross-Reference (if applicable)
- [Equations doc entry]: [what was wrong or ambiguous, how it was clarified]
```

Keep entries concise. The satellite is a reference for future sessions, not a narrative.

## Step 5: Flag Cross-Project Lessons

Review each pattern: does this apply only to this project, or would it help future projects?

- **Project-local** -- stays in the testing satellite. Example: "The delegation bonus calculation silently overflows when loyalty exceeds 150%."
- **Cross-project** -- should become a feedback memory in the home directory memory folder. Example: "Off-by-one errors in percentage-based thresholds are a recurring failure mode -- always test at boundary values (0%, 100%, 100%+epsilon)."

Flag cross-project items for the operator to approve. Do not write feedback memories without approval.

## After the Audit

Proceed with `/phase-wrap`. The testing satellite entries from this audit become part of the phase-wrap's step 3 (Update MEMORY.md and Satellites).
