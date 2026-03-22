# Project Lifecycle — Skill Reference

Patterns for running multi-phase software projects with Claude Code.
Technology-agnostic workflow validated by two complete projects:
- **A PowerShell backup utility** — 33 exported functions, 239 Pester tests, 10 sub-modules
- **An HTML5 Canvas game** — single file, ~2550 lines, 8 phases, 18 color schemes

This is a meta-skill describing HOW to run a project, not what technology to use. References all other skills as applicable.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Phase 0: Research and Planning](#2-phase-0-research-and-planning)
3. [CLAUDE.md Template](#3-claudemd-template)
4. [Directory Structure Templates](#4-directory-structure-templates)
5. [Session Handoff](#5-session-handoff)
6. [Self-Verification Loop](#6-self-verification-loop)
7. [Quality Gate](#7-quality-gate)
8. [Skills Sync Protocol](#8-skills-sync-protocol)
9. [/forge-review Review Pass](#9-forge-review-review-pass)
10. [MEMORY.md Management](#10-memorymd-management)
11. [Home Directory Memory](#11-home-directory-memory)
12. [Stabilize (Shelf a Project)](#12-stabilize-shelf-a-project)
13. [Project Wrap](#13-project-wrap)
14. [Gotchas](#14-gotchas)
15. [New Project Checklist](#15-new-project-checklist)

---

## 1. Overview

Every project follows this lifecycle:

```
Phase 0 (Home Session)     → Research, plan, scaffold, create skills
Session Handoff            → Open project session, Claude reads bootstrap info
Phase 1..N (Project)       → Implement, test, review, verify (loop per phase)
Stabilize (optional)       → Satellite audit, canonical sync, shelf for later reactivation
Project Wrap               → Final review, skills sync, cold storage
```

Each phase produces:
- **Working code** — feature implemented and user-tested
- **Updated TestingGuide** — manual test steps for the phase
- **Updated CLAUDE.md** — architectural decisions, quality gate items
- **Updated skills** — battle-tested patterns elevated to canonical files
- **Updated MEMORY.md** — project status, decisions, gotchas

---

## 2. Phase 0: Research and Planning

Run from the home directory session: `~/.claude/projects/<home-key>`

### Steps

1. **Assess existing codebase** (if rewriting): Read the original code, document what works and what needs to change. Create an assessment document if the codebase is large.

2. **Check existing skills**: Read `~/.claude/skills/index.md`. Identify which skills apply to the new project's tech stack. Don't create new skills for things that are already covered.

3. **Research and create new skills**: For the project's tech stack, research best practices via web search and documentation review. Create comprehensive skill files following the established format (intro paragraph with provenance, table of contents, numbered sections, code examples, anti-patterns, gotchas table, checklist). **If creating a new skill for an unfamiliar domain** (new language, framework, or API), prototype the riskiest technical assumption before proceeding to plan mode. The prototype should answer one specific feasibility question in under 50 lines — "can the toolchain build and run," "does this API return what we need," "does this library work under our constraints." If the prototype fails, that's a plan-altering finding: raise it before architecture commits to the assumption.

4. **Update skills index**: Add new skills to `~/.claude/skills/index.md` with descriptions and dependency mappings.

5. **Use plan mode for architectural decisions**: When Phase 0 hits meaningful decisions — tech stack, runtime target, module structure, key tradeoffs — enter plan mode so the user gets the interactive UI. Plain Q&A is a fallback, not the default.

6. **Create project directory**:
   ```
   E:\ProjectName\
   ├── .gitignore
   ├── CLAUDE.md
   ├── docs/
   │   ├── skill.md          # Domain skill (copy from canonical)
   │   └── [lang]-skill.md   # Language skill (copy from canonical)
   └── tests/
       └── TestingGuide.html  # Manual testing guide (from template)
   ```

7. **Copy relevant skills** to `E:\ProjectName\docs/` — these are the project's working copies that get `[verified in Phase N]` tags.

8. **Write CLAUDE.md** from template (Section 3).

9. **Write TestingGuide.html** from template (see `manual-testing-methodology.md`).

10. **Add bootstrap entry to home directory memory** (Section 11) with project name, location, plan file, and description.

11. **Create plan file** at `~/.claude/plans/<plan-name>.md` with Phase 0 work summary and subsequent phase outlines.

12. **Commit Phase 0**: `git add` all scaffolding files and commit with "Phase 0: Project infrastructure". Phase 0 touches both project files AND global files (skills, home directory memory, CLAUDE.md) — commit the project repo now so the implementation session starts from a clean baseline. Global files (in `~/.claude/`) are not git-tracked but are updated; the project commit captures the project-side artifacts.

### What Phase 0 is NOT

Phase 0 is research and scaffolding. Don't write application code in Phase 0. The deliverables are: skills, CLAUDE.md, TestingGuide skeleton, plan file, directory structure.

---

## 3. CLAUDE.md Template

Every project gets a CLAUDE.md at its root. This is the contract between Claude and the user.

```markdown
# [Project Name]

[One-line description]

## Target Runtime

- **Runtime:** [Browser versions / PowerShell 7+ / etc.]
- [Constraints: no build tools, no external dependencies, etc.]
- [File structure: single file / module architecture]

## Directory Structure

[ASCII tree showing all files and folders]

## Key Architectural Decisions

- [Decision 1 with rationale]
- [Decision 2 with rationale]
- ...

## Code Organization

[Numbered list of logical sections/modules]

## [Domain-Specific Rules]

[Rules that must be preserved exactly — collision rules, business logic, etc.]

## Coding Standards

All code must comply with `docs/[lang]-skill.md` and `docs/skill.md`. Key rules:
- [Language-specific rules]
- [Project-specific rules]

## AI Architecture Notes

- **File size targets:** [e.g., source files under ~500 lines, functions under ~50 lines]
- **Complexity locations:** [Which files/modules are dense and require careful reading]
- **Intentional patterns:** [Anything that looks wrong but is deliberate -- prevents AI "fixing" it]
- **Tight couplings:** [Which modules depend heavily on each other and why]

See `code-architecture-for-ai.md` for the principles behind these notes.

## Quality Gate

Run after every phase. Check:
- [ ] [Testable item 1]
- [ ] [Testable item 2]
- ...

## Self-Verification Loop (per phase)

1. Phase kickoff — `/kickoff` skill: read plan/code/memory, present gameplan (incl. test plan) + numbered questions/suggestions tagged [Design] [Clarification] [Suggestion] [Risk], wait for user
2. Write code + automated tests (Pester/xUnit/inline) — run tests, fix failures. Code is NOT presented for review until automated tests pass.
3. Update TestingGuide.html — manual test steps for user-facing behavior (the operator's inspection, separate from automated tests)
4. User follows TestingGuide
5. User reports results
6. Fix failures
7. `/forge-review` — three-agent review
8. Fix issues
8a. `/test-audit` (conditional) — if non-trivial failures this phase, extract testing patterns to satellite
9. Update CLAUDE.md — `/phase-wrap` skill handles steps 8a-11 + commit
10. Update project skills (docs/) with verified patterns — canonical sync at wrap only
11. Update MEMORY.md + satellites, then next phase

## Memory Management

- MEMORY.md has a 200-line context limit — keep it index-level
- Use satellite files (e.g., `systems.md`, `patterns.md`) for detailed notes, linked from MEMORY.md
- Proactively split before hitting the limit, not after
- See `knowledge-architecture.md` for the three-layer model: Skills (portable) → Satellites (project-specific) → MEMORY.md (routing table)

## Implementation Phases

- [x] Phase 0: Project infrastructure
- [ ] Phase 1: [Description]
- [ ] Phase 2: [Description]
- ...
```

---

## 4. Directory Structure Templates

### Single-File Application (HTML5)

```
E:\ProjectName\
├── .gitignore              # Git ignore rules
├── ProjectName.html        # The entire application
├── CLAUDE.md               # Project instructions
├── docs/
│   ├── skill.md            # Domain skill (project copy)
│   ├── js-skill.md         # Language skill (project copy)
│   └── existing/           # Reference material (old code, specs)
└── tests/
    ├── TestingGuide.html   # Manual testing checklist
    └── [test files]        # Automated tests if applicable
```

### Multi-File Module (PowerShell)

```
E:\ProjectName\
├── .gitignore              # Git ignore rules
├── ProjectName.ps1         # Entry point / launcher
├── Build-Release.ps1       # Build and packaging script
├── CLAUDE.md               # Project instructions
├── config/
│   └── defaults.json       # Default configuration
├── src/
│   ├── ProjectName.psd1    # Module manifest
│   ├── ProjectName.psm1    # Root module (dot-sourcing)
│   ├── Core/               # Core functions
│   ├── Feature1/           # Feature module
│   └── GUI/                # Optional GUI layer
├── tests/
│   ├── Invoke-QualityCheck.ps1
│   ├── Core.Tests.ps1
│   └── Feature1.Tests.ps1
├── docs/
│   ├── skill.md
│   └── ps-skill.md
└── logs/
```

---

## 5. Session Handoff

Phase 0 runs in the home directory session. Implementation runs in a project-specific session.

### Handoff Steps

1. **Phase 0 completes** in home directory session. Bootstrap info is in home directory memory.

2. **User opens new session** at `E:\ProjectName`.

3. **Claude provides a handoff prompt** at the end of Phase 0 for the user to paste into the new session. This bridges the session gap — the new Claude instance has no memory of Phase 0 work.

4. **User opens new session** at `E:\ProjectName` and pastes the handoff prompt.

5. **Claude reads:**
   - Global `~/.claude/CLAUDE.md` for workflow context
   - Home directory memory for project bootstrap entry
   - Plan file at the path specified in bootstrap
   - Project CLAUDE.md at `E:\ProjectName\CLAUDE.md`
   - `git log --oneline` to understand project commit history

6. **Claude creates project memory** at `~/.claude/projects/<project-key>/memory/MEMORY.md` with initial project status.

7. **Implementation begins** at Phase 1.

### Handoff Prompt Template

At the end of Phase 0, Claude provides this prompt for the user to paste into the new project session:

```
We're working on [ProjectName] out of E:\[ProjectName]. [One-line description].
Check the global CLAUDE.md for workflow context, then read:

1. Home directory memory for the bootstrap entry
   (`~/.claude/projects/<home-key>/memory/MEMORY.md`)
2. Plan file at `~/.claude/plans/<plan-name>.md`
3. Project CLAUDE.md at `E:\[ProjectName]\CLAUDE.md`
4. `git log --oneline` to confirm project state

Phase 0 is complete and tested. Start Phase [N].
```

Fill in the bracketed values. The prompt gives the new session everything it needs to orient itself without guessing.

### Why Session Handoff

- Home directory session (`<home-key>`) is for cross-project work: skills, research, planning
- Project sessions (`<project-key>`) have their own memory folder, scoped to that project
- Claude Code creates the memory folder automatically when a session starts at that path
- Each session's context window stays focused on its project

---

## 6. Self-Verification Loop

Run after every implementation phase. This is the core quality cycle.

```
1. Phase kickoff  ← /kickoff skill handles this step
   └─ Read plan file, CLAUDE.md, current code state, and MEMORY.md/satellites
   └─ Present gameplan: what this phase will build, key implementation approach
   └─ Present a numbered list of questions/suggestions (highest priority first)
   └─ Tag each item: [Design] [Clarification] [Suggestion] [Risk]
   └─ Wait for user response before writing code

2. Write code + automated tests
   └─ Implement the phase's features per the agreed gameplan
   └─ Write automated tests (Pester for PS, xUnit for C#, inline for JS) for implemented features
   └─ Run automated tests and fix failures before proceeding
   └─ Automated tests verify code correctness — this is the session's responsibility
   └─ Code is NOT presented for review until automated tests pass

3. Update TestingGuide.html (manual tests)
   └─ Add test steps for this phase (action/expect/if-fail format)
   └─ Manual tests verify user-facing behavior — this is the operator's inspection
   └─ Automated tests and manual tests are different activities with different owners

4. User follows guide
   └─ User opens TestingGuide, runs through phase steps, checks boxes

5. User reports results
   └─ User clicks "Copy Report" and pastes into conversation

6. Fix failures
   └─ Address any TODO items from the report

7. Run /forge-review  ← /forge-review skill handles steps 7-8
   └─ Three-agent review: code reuse, code quality, efficiency

8. Fix review findings
   └─ Address real findings, skip false positives

8a. Testing audit (conditional)  ← /test-audit skill handles this step
    └─ Only if non-trivial test failures occurred this phase
    └─ Review failures, classify root causes, extract patterns
    └─ Write findings to testing satellite (testing-patterns.md)
    └─ Flag cross-project lessons for feedback memory promotion

9. Update CLAUDE.md  ← /phase-wrap skill handles steps 9-12 (prompts for 8a)
   └─ Mark phase complete, add architectural decisions, update code organization

10. Update project skills (pattern extraction)
    └─ Project copies only (docs/) — canonical skills are NOT updated until project wrap
    └─ Add [verified in Phase N] tags to validated sections
    └─ Extract patterns from fresh work into project skills or satellites

11. Update MEMORY.md and satellites
    └─ Project status, new decisions, review findings, gotchas discovered
    └─ Move detailed subsystem notes to satellite files when MEMORY.md grows past ~150 lines
    └─ MEMORY.md stays as routing table; satellites hold the detail

12. Commit phase
    └─ `git add` changed files, commit with "Phase N: [description]"
    └─ Clean working tree before starting next phase
```

### Equations Reference Document

For projects with interconnected formulas, multipliers, or balance equations (games, financial tools, config-driven pipelines), maintain a living reference document that maps:

- **What** each bonus/effect/multiplier is
- **Who** owns it (which entity, unit, or subsystem)
- **Where** it is consumed (which calculations read it)
- **Scope** restrictions (e.g., "unit X bonus applies to role Z only, not all roles")

This document serves as ground truth for any session working on the project. Without it, a session must infer design intent from code that may have already been corrupted by a prior session. The document should be a project satellite file (e.g., `equations.md`) and updated as part of the phase wrap (step 11 above).

**When to create one:** If the project has more than ~5 interacting multipliers or bonuses, or if any bonus is intentionally scoped to a specific entity rather than applied globally.

---

## 7. Quality Gate

The quality gate is a checklist in CLAUDE.md. Every item must be testable.

### Rules for Quality Gate Items

- **Testable:** Has an observable action and expected result
- **Specific:** Names the exact behavior, not a general quality attribute
- **Negative when needed:** "Wall collision stops movement (NOT death)" specifies what should NOT happen
- **Phase-aware:** Items added as features are implemented
- **Droppable:** Items can be marked as dropped if a feature is cancelled

### Examples

```markdown
## Quality Gate

- [ ] All 18 color schemes render correctly
- [ ] Demo mode runs without spawn collision on all 7 board sizes
- [ ] Screensaver board (384x216) maintains acceptable framerate
- [ ] Settings persist across page reload
- [ ] Food coverage dropdown works (20%/40%/60%/80%), respects cap and floor
- [ ] Wall=stop, body=stop, tail(len>5)=death
- [ ] No `setInterval` in codebase
- [ ] No `var` declarations
- [ ] All public functions have JSDoc
- [ ] Performance overlay toggles on/off (F3 or button)
- [ ] ~~Touch controls~~ (Phase 6 dropped)
```

---

## 8. Skills Sync Protocol

Skills sync has two distinct operations at different times:

### Per-Phase: Pattern Extraction (Step 9)

After each phase's `/forge-review` pass, extract patterns from the work just completed:

- **Project skills** (`docs/*.md`) — update with new patterns, add `[verified in Phase N]` tags
- **Satellites** (`memory/patterns.md`) — project-specific patterns, bug patterns, anti-patterns
- **Canonical skills are NOT touched** — the project builds its own knowledge; export happens at wrap

### Project Wrap: Canonical Sync

At project wrap, the accumulated project knowledge gets audited and synced outward. See Section 12 for the full wrap process.

### Severity Rating

When extracting patterns (per-phase or wrap), categorize by severity:

| Severity | Worth Keeping | Example |
|----------|--------------|---------|
| Critical | Always | Bug that would crash (shadow variable, missing null check) |
| High | Always | Performance issue (O(n) where O(1) is possible), logic error |
| Medium | Always | Code smell (magic numbers, leaked abstraction, redundant state) |
| Low | If it's a concise one-liner | Minor style preference |

### Tagging Format

```markdown
### Offscreen Canvas (Pre-rendering) [verified in Phase 1, Phase 7 — grid cache + food cache with dirty flags]
```

- Tag appears inline after the section header
- Multiple phases can be listed (comma-separated)
- Brief description of what was verified follows the em dash
- Tags go on **project copies** (`docs/`) during implementation; canonical copies get tagged at wrap sync

### New Sections

If a phase discovers a pattern not yet in the skill file, add a new section with the `[verified in Phase N]` tag to the project copy.

---

## 9. /forge-review Review Pass

The `/forge-review` skill runs three parallel review agents on changed code.

### What It Checks

| Agent | Focus |
|-------|-------|
| Code Reuse | New code that duplicates existing utilities, inline logic that could use existing helpers |
| Code Quality | Redundant state, copy-paste, leaked abstractions, magic numbers, stringly-typed code |
| Efficiency | Redundant computation, missed concurrency, hot-path bloat, memory leaks, overly broad operations |

### How It Identifies Changes

With local Git repos set up, `/forge-review` uses `git diff` to see exactly what changed:
- `git diff` shows unstaged changes in the working tree
- `git diff --cached` shows staged changes
- `git diff HEAD~1` shows changes since the last commit
- Falls back to reading full files if git is unavailable

### What It Does NOT Do

- Does NOT make feature suggestions or scope changes
- Does NOT argue with findings — real issues get fixed, false positives get noted and skipped

---

## 10. MEMORY.md Management

Project memory lives at `~/.claude/projects/<project-key>/memory/MEMORY.md`.

For the full three-layer knowledge architecture (MEMORY.md → satellites → skills), see `knowledge-architecture.md`. This section covers MEMORY.md specifically.

### Three-Layer Summary

| Layer | Location | Content | Audience |
|-------|----------|---------|----------|
| Skills | `docs/*.md` + `~/.claude/skills/` | Portable, generalized patterns | The craft — any future project |
| Satellites | `memory/systems.md`, `patterns.md`, etc. | Project-specific reference (variables, chains, migrations) | The project — future sessions on this codebase |
| MEMORY.md | `memory/MEMORY.md` | Live index, status, routing pointers | The session — orient a fresh instance in seconds |

### MEMORY.md Structure

See `knowledge-architecture.md` Section 5 for the full template. Key sections:

- **Status** — current phase, next phase, plan file pointer
- **Satellite Files** — links to systems.md, patterns.md, etc.
- **Architecture Overview** — key structural facts, conventions
- **Quick Reference** — ID caches, key function names, current save version
- **Simplify Patterns** — per-phase `/forge-review` findings (what was caught and fixed) — prevents reintroduction
- **Verified Patterns (quick ref)** — top 5-6, with pointer to patterns.md for full list

Target: 60-120 lines active. Under 60 early phases. Approaching 180 only if many subsystems active.

### When to Split

| Project Stage | Knowledge State |
|--------------|----------------|
| Phase 0-2 | MEMORY.md only. Everything fits. |
| Phase 3-5 | MEMORY.md getting long. Create `patterns.md`. |
| Phase 5-7 | Per-subsystem detail consumes MEMORY.md. Create `systems.md`, restructure MEMORY.md as index. |
| Phase 7+ | Three-layer model fully active. Skill sweeps per-phase. |

Don't pre-create satellite files for a 3-phase project. Let the 200-line pressure drive the split organically.

### Update Frequency

- **After every phase** — update status, add decisions, migrate detail to satellites if needed
- **After discovering gotchas** — add immediately so they're not lost
- **On project wrap** — final status update, line count, feature summary

---

## 11. Home Directory Memory

Lives at `~/.claude/projects/<home-key>/memory/MEMORY.md`.

### Standard Sections

- **User profile** — background, communication preferences, working style
- **Completed projects** — one-line index entries with pointer to cold storage
- **Active projects** — full bootstrap detail (location, plan file, skills, status)
- **Dev environment** — OS, tools, versions

### Cold Storage Pattern (Home Directory Memory)

MEMORY.md is auto-loaded into every session's context with a 200-line truncation limit. To stay under that limit:

- **Active projects** get full detail in MEMORY.md (location, plan file, skills, phase status, key decisions)
- **Completed projects** move to `completed-projects.md` in the same memory folder. MEMORY.md keeps only a one-line index entry per project.
- **On project wrap**, move the full entry from MEMORY.md to `completed-projects.md` and replace it with a summary line.

```markdown
## Completed Projects

Full details in `completed-projects.md`. Summary index:

- **ProjectName** (v1.0.0) — One-line description. `E:\ProjectName`
```

### Active Project Entry

```markdown
### [Project Name] (Phase N complete — ready for Phase M)
- [Brief description]
- Location: `E:\ProjectName`
- Plan file: `~/.claude/plans/<plan-name>.md`
- Skills: [list, including automated testing skill for the stack]
- Testing: [automated framework] (every phase) + TestingGuide.html (manual, field verification)
- [Key details: tech stack, scope, notable features]
- [Status: phases completed, current phase]
```

### Bootstrap Entry (for Session Handoff)

When Phase 0 completes, the home directory memory gets an entry that the new session can read. Include:
- Project name and location
- Plan file path (so Claude can pull it in)
- Brief description of the project
- Which skills apply
- What phase to start at

---

## 12. Stabilize (Shelf a Project)

When a project reaches a natural pause point (MVP complete, milestone achieved) but is NOT finished — more phases are planned for later:

1. **Final `/forge-review` pass** — review the last phase's changes
2. **Satellite audit** — read ALL satellite files cover to cover. Ask: "Is this generalizable?" Patterns that looked project-specific early on may be clearly universal in hindsight. Promote generalizable patterns to project skills (`docs/`).
3. **Components audit** — check `components-index.md`. Did this project adapt any indexed component? If the adaptation is better (more general, more robust), update the index pointer. Did this project create new reusable components? Add them. Direction is always upward.
4. **Canonical sync** — sync project skills (`docs/`) → canonical skills (`~/.claude/skills/`). Strip project-specific names, generalize code examples, add `[verified]` tags. See `knowledge-architecture.md` Section 3 for generalization rules.
5. **Create meta-skills if discovered** — if the project revealed new workflow patterns
6. **Update `~/.claude/skills/index.md`** — add any new skills with descriptions and dependencies
7. **Update project MEMORY.md** — mark status as "Stabilized" (not wrapped). Record completed phases, test counts, final line counts. Satellite files preserved for reactivation.
8. **Update home directory memory** — update project entry status to "Stabilized". Do NOT move to cold storage — project stays in active section.
9. **Commit** — "Stabilize: [project name] — [milestone summary]"

**Key difference from Project Wrap:** The project stays in the active section of home directory memory. No cold storage move. The project is shelved, not completed — a future session can reactivate it by reading the preserved MEMORY.md and satellites.

**When to stabilize vs wrap:**
- **Stabilize** — more work is planned (future tiers, features, ports). Knowledge capture now, continue later.
- **Wrap** — project is done. No more planned work. Move to cold storage.

---

## 13. Project Wrap

When the project is truly complete — no more planned phases:

1. **Final `/forge-review` pass** — review the last phase's changes
2. **Satellite audit** — read ALL satellite files (patterns.md, systems.md, etc.) cover to cover. For each entry, ask: "Now that the full project is complete, is this generalizable?" Patterns that looked project-specific early on may be clearly universal in hindsight. Promote generalizable patterns to project skills (`docs/`).
3. **Components audit** — check `components-index.md`. Did this project adapt any indexed component? If the adaptation is better (more general, more robust), update the index pointer. Did this project create new reusable components? Add them. Direction is always upward.
4. **Canonical sync** — sync project skills (`docs/`) → canonical skills (`~/.claude/skills/`). This is the ONE time canonical skills get updated. Strip project-specific names, generalize code examples, add `[verified]` tags. See `knowledge-architecture.md` Section 3 for generalization rules.
5. **Create meta-skills if discovered** — if the project revealed new workflow patterns (like this skill was created from an earlier project)
6. **Update `~/.claude/skills/index.md`** — add any new skills with descriptions and dependencies
7. **Update global `~/.claude/CLAUDE.md`** — if workflow improvements were discovered
8. **Update home directory memory** — move full project entry from MEMORY.md to `completed-projects.md`, replace with one-line index entry (see Section 11, Cold Storage Pattern)
9. **Update project MEMORY.md and satellites** — mark all phases complete, final line count, feature summary. Satellite files (systems.md, patterns.md) are preserved as project reference for reactivation.
10. **Final commit** — commit all wrap-up changes with "Project wrap: [project name]"

---

## 14. Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| `/forge-review` can't find changes | Git not initialized or no commits yet | Run `git init` and make initial commit; `/forge-review` uses `git diff` |
| Session handoff loses context | New Claude doesn't know what happened in Phase 0 | MEMORY.md is the bridge; home directory memory has bootstrap info |
| Plan file treated as living document | Plan file diverges from reality, causes confusion | Plan file is write-once reference. CLAUDE.md is the living document. |
| Phase numbering gaps (5.5) | Progress tracking breaks | Use string-based phase IDs; gaps are fine |
| Phase dropped mid-project | Quality gate has impossible items | Mark items with ~~strikethrough~~ and "(Dropped)" |
| Over-engineered Phase 0 | Weeks spent on skills before writing any code | Phase 0 is scaffolding. Skills can be updated during implementation. |
| Canonical skills not synced | Pattern proven in Phase 3 is missing from `~/.claude/skills/` | Canonical sync happens at stabilize (Section 12) or project wrap (Section 13), not per-phase. Per-phase updates go to project copies (`docs/`) only. |
| TestingGuide format wrong | Copy Report shows IDs instead of step names | Use `id="check-N-M"` + `<label for="check-N-M">` (see `manual-testing-methodology.md`) |
| Memory file too large | MEMORY.md exceeds 200-line context limit | Move completed projects to `completed-projects.md` (Cold Storage Pattern, Section 11). For project memory, split into satellite files (Section 10, `knowledge-architecture.md`). |
| Project memory deleted on wrap | Completed project reactivated but no memory files exist | Preserve project MEMORY.md and satellites on wrap — they're needed for bug fixes. Cold storage is for *home directory* memory, not project memory. |
| Satellite files created too early | Empty satellite files for a 3-phase project | Let the 200-line pressure drive the split. Phases 0-2 usually fit in MEMORY.md alone. |
| Field surprises lost | Tool deployed, bugs found in field, no record | After first real-world deployment, add a `## Field Notes` section to project MEMORY.md or a satellite. Record what surprised you -- hardware-specific failures, silent flags that didn't work, environment differences. Lightweight, optional, but invaluable for the next version. |

---

## 15. New Project Checklist

Step-by-step from zero to Phase 1:

### Phase 0 (Home Directory Session)

- [ ] Open home directory session
- [ ] Define project scope (what it does, tech stack, constraints)
- [ ] Read `~/.claude/skills/index.md` — identify applicable existing skills. Cross-reference plan technical requirements against the full index (registry → cim-wmi, testing → pester, GUI → wpf, etc.). Do not trust the idea file's skill list as complete.
- [ ] Check `~/.claude/skills/components-index.md` — identify reusable implementations; reference specific components in plan file
- [ ] Research and create any missing skills for the project's tech stack
- [ ] Add new skills to `index.md` with descriptions and dependencies
- [ ] **Use plan mode for architectural decisions** — tech stack, runtime target, module structure, key tradeoffs
- [ ] Create `E:\ProjectName\` directory with structure (Section 4)
- [ ] Create `.gitignore` (exclude `.claude/`, runtime artifacts, credentials)
- [ ] `git init` and initial commit
- [ ] Copy relevant skills to `E:\ProjectName\docs/`
- [ ] Write `E:\ProjectName\CLAUDE.md` from template (Section 3)
- [ ] Write `E:\ProjectName\tests\TestingGuide.html` from template (`manual-testing-methodology.md`)
- [ ] Add bootstrap entry to home directory memory (Section 11)
- [ ] Create plan file at `~/.claude/plans/[plan-name].md`
- [ ] **Commit Phase 0** — `git add` all project scaffolding, commit "Phase 0: Project infrastructure"

### Session Handoff

- [ ] Claude provides handoff prompt for user to paste into new session (Section 5)
- [ ] Open new session at `E:\ProjectName`
- [ ] Paste handoff prompt — Claude reads bootstrap, plan file, project CLAUDE.md
- [ ] Claude creates project memory at `~/.claude/projects/<project-key>/memory/MEMORY.md`

### Phase 1+ (Self-Verification Loop)

- [ ] Implement phase features
- [ ] Write automated tests (Pester/xUnit) for implemented features — run and pass before review
- [ ] Update TestingGuide.html with manual test steps for user-facing behavior
- [ ] User tests and reports results
- [ ] Fix failures
- [ ] Run `/forge-review`, fix findings
- [ ] Update CLAUDE.md (mark phase complete)
- [ ] Sync skills (project + canonical) with `[verified in Phase N]` tags
- [ ] Update MEMORY.md
- [ ] Commit phase (`git add` changed files + `git commit` with phase summary)
- [ ] Proceed to next phase
