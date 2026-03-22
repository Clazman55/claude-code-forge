---
name: phase0
description: >
  Phase 0 — project research, planning, and scaffolding in the home directory session.
  Run before any project session begins. Produces a plan file, scaffolds the project folder,
  creates project CLAUDE.md, identifies needed skills, and writes the bootstrap memory entry.
  Trigger on "phase 0", "phase zero", "run phase 0".
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /phase0 -- Project Setup...**
>
> *Phase 0 runs before a project session exists. This step produces the plan, scaffold, CLAUDE.md, and bootstrap entry that the project session will need. Nothing exists yet -- that is the correct state. What I produce here is the continuity everything else depends on. I will not rush it.*

# /phase0 — Phase 0 -- Project Setup

## What This Skill Does

Phase 0 runs in the HOME DIRECTORY session (~) before a project session is opened.
It takes a concept from the ideas folder to a fully scaffolded, ready-to-build project -- OR
produces a plan for a deliverable-only idea (document, artifact, no project folder needed).

---

## Step 0 — Confirm Context

Verify this is a home directory session. If running inside a project folder, stop and warn:
"Phase 0 must run in the home directory session, not a project session."

If the operator named the idea in the invocation, use that name directly.
Otherwise ask: "Which idea are we working on? Provide the idea file name or project name."

Read the specified idea file from `~/.claude/ideas/`.
Read `~/.claude/projects/<home-key>/memory/MEMORY.md` for active projects, pipeline context, and dev environment.
Read `~/.claude/CLAUDE.md` for global standards and workflow expectations.

**Classify the idea type** from the idea file's `Project type:` field:
- `Project type: full` -- standard path, all steps apply
- `Project type: deliverable` -- document/artifact output only; skip Steps 3 and 4 (no folder scaffold, no project skills); adjust Step 5 and 7 accordingly
- If no field present, infer from context and confirm with the operator before proceeding.

---

## Step 1 — Research

**For `full` project types:** Read `~/.claude/skills/code-architecture-for-ai.md` before planning the architecture. Apply its principles when making decisions about file structure, module boundaries, and documentation in Steps 2-3.

Check `~/.claude/skills/components-index.md` for existing implementations relevant to this project's requirements. Reference specific components in the plan file (e.g., "Registry scanning: adapt from existing Get-InstalledSoftware implementation") rather than planning to build from scratch.

Based on the idea file, identify what needs to be researched before planning:
- Unknown APIs, libraries, or tools referenced in the idea
- Technical unknowns flagged in Open Questions
- Prior art in existing projects (check Relationship to Existing Projects section)

**Use web search for technical unknowns.** If an approach, file path, API, or tool invocation is uncertain, search for it -- do not guess, do not rigidly follow a suggestion that may be wrong, and do not treat the operator's description of a method as authoritative when the actual implementation details are unknown. The operator describes what they want to accomplish; it is your job to find the correct way to accomplish it. If web search reveals a simpler or better approach than what was described, use it and note the difference.

Perform targeted web searches and code reads focused on unknowns that would change the plan -- not general reading.

Summarize findings before proceeding. Flag anything that changes the idea's assumptions.

---

## Step 2 — Plan File

Generate a plan file at `~/.claude/plans/<codename>.md`.

Codename format: three random common words (e.g. stalwart-marching-centurion).
Do not use project name as codename — keeps plan files neutral and portable.

Plan file must include:
- Project name and one-line description
- Location: `E:\ProjectName`
- Tech stack and runtime targets
- Phase breakdown (Phase 0 through wrap) with deliverables per phase — each phase lists both source files AND their corresponding test files
- Known risks and open questions resolved from research
- Relationship to existing projects (reuse vs reference)
- Skills needed (existing canonical skills + any new ones to create)

Present the plan to the operator for review before proceeding. Wait for approval.

---

## Step 3 — Scaffold Project Folder

**Skip this step for `deliverable` type ideas.**

Once plan is approved:

1. Create project folder at `E:\ProjectName`
2. Create standard structure per project type (PowerShell, HTML5, etc.)
   - **PowerShell projects must include:** `tests/` directory with `Invoke-QualityCheck.ps1` stub. The quality gate framework exists from commit zero.
   - See `project-lifecycle.md` Section 4 for directory templates.
3. Create `E:\ProjectName\CLAUDE.md` with:
   - Project name, description, location
   - Tech stack
   - Phase status table (all phases Pending)
   - Key file paths
   - Skills in use
   - Self-verification loop reminder
4. Initialize git repo: `git init E:\ProjectName`
5. Create `.gitignore` appropriate to project type
6. Initial commit: "Phase 0 scaffold"

---

## Step 4 — Skills

**Skip this step for `deliverable` type ideas** (no project folder to put them in).

Review the plan's skills list:
- Existing canonical skills at `~/.claude/skills/` -- confirm they cover what's needed
- If new skills are required, create stub skill files now or note them as Phase 0 deliverables
- If a project-specific skill file is needed in `E:\ProjectName\docs\`, create it
- **Mandatory skills by tech stack:**
  - PowerShell projects: `pester-testing-patterns` is required. If missing from the idea file's skill list, add it.
  - C# / .NET projects: `xunit-testing-patterns` is required.
  - The automated testing skill for the project's stack is never optional.

---

## Step 5 — Bootstrap Memory Entry

**For `full` projects** -- add to the Active Projects section of `~/.claude/projects/<home-key>/memory/MEMORY.md`:

```
### ProjectName (Phase 0 Complete)
- One-line description
- Location: `E:\ProjectName`
- Plan file: `~/.claude/plans/<codename>.md`
- Skills: list-of-skills
- Key notes from Phase 0
```

**For `deliverable` type** -- add to the Ideas Pipeline section (not Active Projects) to reflect in-progress status:

```
- `idea-file.md` -- PROMOTED: DeliverableName. Plan: `~/.claude/plans/<codename>.md`. Output: [format and destination].
```

In both cases, update the Ideas Pipeline entry for the idea file to mark it as promoted.

---

## Step 6 — Commit

Commit all Phase 0 artifacts to the knowledge base repo:
```
git add -p
git commit -m "Phase 0: <ProjectName> - plan, scaffold, bootstrap entry"
```

---

## Step 7 — Handoff Brief

Present a handoff summary to the operator:

**For `full` projects:**
- Plan file location
- Project folder location
- Phase 1 starting point (what to build first)
- Session prompt to use when opening the project session (reference `~/.claude/prompts.md`)
- Any open questions that Phase 1 must resolve

**For `deliverable` type:**
- Plan file location
- Where output will be written (e.g., `~/.claude/doctrine/share/` or Desktop)
- Phase 1 starting point (first document or section to draft)
- Any open questions that Phase 1 must resolve
- Note: no project session to open -- work happens here in the home directory session

Phase 0 is complete. Await the operator's signal to proceed.

---

## Trigger Phrases
- `/phase0`
- `phase 0`
- `phase zero`
- `run phase 0`
