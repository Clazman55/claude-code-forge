---
description: Phase kickoff — read plan/code/memory, present gameplan with tagged questions, wait for approval. Trigger on "kick it off", "start the phase", or "kickoff".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /kickoff...**
>
> *Reading project context. Will not begin work until the gameplan is approved.*

# /kickoff — Phase Kickoff

This is step 1 of the self-verification loop. Run this at the start of every implementation phase.

## Step 1: Read Context

Read the following sources in order:

1. **Project CLAUDE.md** — at the project root. Tells you what the project is, its architecture, coding standards, and quality gate.
2. **Project MEMORY.md** — find this project's session memory folder. Read MEMORY.md and any satellite files (systems.md, patterns.md, etc.). This tells you current status, decisions, and gotchas.
3. **Plan file** — the plan file path is in the home directory MEMORY.md bootstrap entry for this project. Read the plan to understand the full phase breakdown and what this phase covers.
4. **Relevant source code** — read the files you'll be modifying this phase. Don't skim — understand the current state before proposing changes.

## Step 2: Phase 0 Assumption Check

Verify that this phase's scope still fits within the architecture established during Phase 0.

If a `phase0-architecture.md` satellite exists in the project memory folder:
- "Does anything about this phase's deliverables change or strain a Phase 0 assumption?"
- If YES: flag as a [Design] question in the gameplan. Record approved modifications in the phase satellite with `Modifies: Phase 0 decision #N`.
- If NO: note "Phase 0 assumptions hold" and proceed.

If no architecture satellite exists (legacy projects): consider running a lightweight frame challenge -- enumerate the plan's meta-decisions, flag the most familiar one, generate a brief alternative.

## Step 3: Plan Critique (Subagent)

After the assumption check, spawn a Sonnet subagent to adversarially review the phase plan. The subagent receives the plan content, project CLAUDE.md summary, current phase scope, and the critique checklist below. The session does NOT self-critique -- a separate agent provides genuine adversarial separation.

### Subagent Prompt

Pass the subagent:
- The phase description from the plan file
- A summary of the project's current state (from CLAUDE.md and MEMORY.md)
- The list of skills loaded for this project
- The list of files to be modified

Instruct it to evaluate against these five dimensions and return structured findings:

```yaml
critique_dimensions:
  - id: dependencies
    check: "Does this phase need something from a phase that hasn't run yet? Are there implicit ordering assumptions?"
  - id: sizing
    check: "Is this phase too large for one session? Should it split? Count the distinct deliverables."
  - id: assumptions
    check: "Does the plan assume a library, API, pattern, or behavior exists without confirming? Flag anything unverified."
  - id: skill_gaps
    check: "Does the plan require knowledge not covered by the loaded skills? Are any skill files stubs?"
  - id: regression
    check: "Does this change touch something already stabilized or tested? What could break?"
```

### Rules

- Always run the critique. It is not optional, even for small phases.
- If the subagent finds nothing, include "No issues found" in the gameplan. A clean critique is a valid result.
- Critique findings that need operator input become tagged questions in Questions & Suggestions (use [Critic] tag).
- Critique findings that are informational go in the Plan Critique section of the gameplan.

## Step 3: Present the Gameplan

Present a structured gameplan for this phase, incorporating the critique findings:

### Format

```
## Phase [N]: [Phase Name]

### Scope
[1-3 sentences: what this phase delivers]

### Implementation Steps
1. [Step with enough detail to be actionable]
2. [Step]
3. [Step]
...

### Test Plan
**Automated tests** (session writes and runs before presenting for review):
- [Test file or test suite — what it covers]
- [Test file or test suite — what it covers]

**Manual tests** (TestingGuide.html steps for the operator to verify):
- [User-facing behavior to verify]
- [Integration scenario to walk through]

Automated tests verify code correctness — the session's responsibility, gating code
before review. Manual tests verify user-facing behavior — the operator's inspection
after code is reviewed. Both are phase deliverables. If a phase produces source files,
it produces corresponding test files.

### Plan Critique
[Findings from the Sonnet subagent. Organized by dimension. "No issues found" if clean.]

### Questions & Suggestions
1. [Design] [Question about architectural choice]
2. [Clarification] [Something ambiguous in the plan that needs user input]
3. [Suggestion] [Improvement or alternative approach worth considering]
4. [Risk] [Potential problem or edge case that could derail the phase]
5. [Critic] [Issue raised by plan critique that needs operator input]
...
```

### Tag Definitions

- **[Design]** — architectural or structural decision that affects how code is organized
- **[Clarification]** — ambiguity in the plan or requirements that needs user input before proceeding
- **[Suggestion]** — optional improvement or alternative approach; not blocking
- **[Risk]** — potential problem, edge case, or dependency that could cause issues if not addressed
- **[Critic]** — issue raised by the plan critique subagent that requires a decision before proceeding

### Rules

- Every question/suggestion MUST have exactly one tag
- Number all items for easy reference ("approve all except 3 and 5")
- Minimum 2 questions — if you can't think of any, you haven't read deeply enough
- **Pseudocode gate for complex logic:** If a phase involves algorithmic complexity (balance calculations, state machines, cost scaling, multi-step transformations, combat resolution), include the algorithm as pseudocode in the Implementation Steps and tag it `[Approval Required]`. Do not implement until the operator approves the logic. The gameplan already covers straightforward features; this gate catches the cases where "implement X" hides non-obvious edge cases.
- Do NOT start writing code. Wait for the user to approve the gameplan, answer questions, and give the go-ahead.

## Step 5: Record Phase Decisions

After the user approves the gameplan and answers questions, create a phase satellite:

Write `~/.claude/projects/<project-key>/memory/phaseN-decisions.md` with all decisions from the Q&A. Format: Decision + Why + How to apply + Modifies (if applicable).

This satellite is a living document. Append to it during implementation when significant mid-phase decisions happen (API changes, user redirections, structural discoveries). Don't wait for phase-wrap to record decisions.

## Step 6: Wait

Stop. Do not write code until the user responds. They may approve as-is, modify the plan, answer questions, or redirect entirely. The kickoff is a checkpoint, not a formality.
