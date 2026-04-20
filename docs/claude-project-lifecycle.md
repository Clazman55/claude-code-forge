# Claude Project Lifecycle Reference

*A structured workflow for Claude-augmented software development. Single-operator, quality-gated, phase-based.*

---

## Overview

Every project built with Claude Code follows the same lifecycle, regardless of language or domain. The structure exists because Claude has no memory between sessions -- without it, each conversation starts from scratch and accumulated knowledge is lost. This lifecycle solves that problem by externalizing everything into files that persist across sessions.

The workflow has been validated across fifteen projects totalling approximately 150,000 lines of source code and 2,500+ automated tests, spanning PowerShell modules, HTML5 Canvas games, C#/.NET desktop applications (WPF, MonoGame), system utilities, incremental games with esbuild bundling, and a public methodology repository.

```
Phase 0 (Home Session)       Research, plan, scaffold, create skill files
        |
        v
Session Handoff              Open project session, Claude reads bootstrap info
        |
        v
Phase 1..N (Project)         Implement, test, review, verify (loop per phase)
        |
        v
Stabilize (optional)         Satellite audit, canonical sync, shelf for reactivation
        |
        v
Project Wrap                 Final review, skills sync, cold storage transfer
```

**Phase Outputs:**

| Stage | Deliverables |
|-------|-------------|
| Phase 0 | Skill files, CLAUDE.md, TestingGuide skeleton, plan file, directory structure, architecture satellite |
| Session Handoff | Project memory created, context loaded from bootstrap entry |
| Each Phase 1..N | Working code, updated TestingGuide, updated CLAUDE.md, skills tags, updated MEMORY.md |
| Stabilize | Canonical skills synced, project memory marked "Stabilized", stays in active memory |
| Project Wrap | Tagged canonical skills, updated index, home memory moved to cold storage |

> **Note:** Phase 0 is scaffolding only -- no application code. The self-verification loop runs after every phase without exception. The plan file is write-once; CLAUDE.md is the living document. Projects may be stabilized (shelved with knowledge captured) or wrapped (completed and archived to cold storage).

---

## 1. Phase 0: Research & Planning

Phase 0 runs in the **home directory session** at `~/.claude/projects/[home-session]/`. This session is scoped for cross-project work: skills research, planning, and scaffolding. No application code is written here.

> **Do Not Write Application Code in Phase 0.** The deliverables are: skill files, CLAUDE.md, TestingGuide skeleton, plan file, directory structure, and architecture satellite. Nothing else.

### Steps

1. **Assess existing codebase** (if rewriting) -- Read the original code. Document what works and what must change. Create an assessment document for large codebases.
2. **Check skills index** -- Read `~/.claude/skills/index.md`. Cross-reference the plan's technical requirements against the full catalog. Do not trust an idea file's skill list as complete -- the index is the authoritative source.
3. **Check components index** -- Read `~/.claude/skills/components-index.md`. Identify reusable implementations for the project's tech stack. Reference specific components in the plan file so the implementation instance knows what has already been built and tested.
4. **Research and create new skill files** -- Web search and documentation for the tech stack. Follow the canonical format: introduction with provenance, table of contents, numbered sections, code examples, anti-patterns, gotchas table, checklist. **If entering an unfamiliar domain** (new language, framework, or API), prototype the riskiest technical assumption before plan mode -- under 50 lines, answering one feasibility question. If the prototype fails, raise it before the architecture commits to the assumption.
5. **Update skills index** -- Add new skill files to `index.md` with descriptions and dependency mappings.
6. **Set dependency policy** -- Does the built artifact need to be self-contained (no runtime prerequisites beyond the target machine)? Record the decision in the project CLAUDE.md.
7. **Use plan mode for architectural decisions** -- When Phase 0 reaches meaningful decision points (tech stack, runtime target, module structure, key tradeoffs), enter plan mode so the operator gets the interactive UI for review. Plain Q&A is a fallback, not the default. The plan file must include test deliverables per phase.
8. **Pattern fit check** (before writing the plan) -- Ask explicitly: "What prior project am I reaching for? Why?" / "What is different about THIS project that might break that pattern?" / "Which architectural defaults am I carrying? Are they justified here?" Record answers in the plan file's Frame Audit section.
9. **Plan critique agent** -- After drafting the plan but BEFORE presenting to the operator, run a plan critique agent against the draft (dependencies, sizing, assumptions, skill gaps, regression). Include findings in the plan presentation.
10. **Create project directory** -- Standard structure matching the project type (see Directory Structures section). Include `.gitignore` (exclude `.claude/`, runtime artifacts, credentials). Run `git init` and make an initial commit.
11. **Copy relevant skill files to `docs/`** -- These become the project's working copies that accumulate `[verified in Phase N]` tags during implementation.
12. **Write CLAUDE.md** from the template (see CLAUDE.md Template section).
13. **Write TestingGuide.html** from the template (see Testing Methodology section).
14. **Add bootstrap entry** to home directory memory -- project name, location, plan file path, applicable skills, starting phase.
15. **Create plan file** at `~/.claude/plans/[plan-name].md` -- Phase 0 summary plus subsequent phase outlines. This file is write-once. Each phase outline must include test deliverables (automated framework, coverage expectations).
16. **Create project memory folder and architecture satellite** -- Create `~/.claude/projects/<project-key>/memory/` and write `phase0-architecture.md` with structured decisions. Each decision records Default, Break condition, Pattern source, and Why chosen. See `code-architecture-for-ai.md` Phase 0 Decision Points for the standard questions.
17. **Commit Phase 0** -- `git add` all scaffolding files, commit "Phase 0: Project infrastructure". Global files (`~/.claude/`) are not tracked in the project repo; the commit captures the project-side artifacts.

**Phase 0 Deliverables:** Skill files, CLAUDE.md, TestingGuide skeleton, plan file, directory structure, architecture satellite. Nothing else. Skills can be refined during implementation -- do not over-engineer the scaffolding.

**Skills System:** Canonical copies live at `~/.claude/skills/` -- the master library, shared across all projects. Project copies live at `[ProjectRoot]/docs/` -- working copies that ship with the project. Verification tags flow in one direction during implementation: project copies get tagged. Canonical copies get tagged at wrap or stabilize.

---

## 2. CLAUDE.md Template

The CLAUDE.md is the contract between Claude and the project. It lives at the project root and is read at every session start. It must be complete enough to bootstrap a session with no other context -- if Claude reads this file and nothing else, it should be able to orient itself and begin work.

```markdown
# [Project Name]

[One-line description]

## Target Runtime
- **Runtime:** [Browser versions / PowerShell 7+ / etc.]
- **Dependency policy:** [Self-contained (no runtime prerequisites) / framework packages permitted / etc. -- set during Phase 0]
- [File structure: single file / module architecture]

## Directory Structure
[ASCII tree showing all files and folders]

## Key Architectural Decisions
[Summary entries. Full decisions with defaults, break conditions, and pattern sources
live in `phase0-architecture.md` in the project memory folder.]
- [Decision 1 with rationale]
- [Decision 2 with rationale]

## Code Organization
[Numbered list of logical sections/modules]

## [Domain-Specific Rules]
[Rules that must be preserved exactly]

## Coding Standards
All code must comply with docs/[lang]-skill.md and docs/skill.md. Key rules:
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

## Self-Verification Loop (per phase)
1. Phase kickoff -- `/kickoff` skill: read plan/code/memory, run Phase 0 assumption check, present gameplan (incl. test plan) + numbered questions/suggestions tagged [Design] [Clarification] [Suggestion] [Risk] [Critic], wait for user. Pseudocode gate for complex algorithms. Creates phase satellite (`phaseN-decisions.md`) from Q&A.
2. Write code + automated tests (Pester/xUnit/inline) -- run tests, fix failures. Code is NOT presented for review until automated tests pass. Record significant mid-phase decisions in the phase satellite.
3. Update TestingGuide.html -- manual test steps for user-facing behavior (separate from automated tests)
4. User follows guide
5. User reports results
6. Fix failures
7. /simplify -- three-agent review. Categorize findings as FIX/DEFER/DISMISS. Deferred and dismissed items go to `deferred.md` satellite.
8. Fix FIX-category issues
8a. /test-audit (conditional) -- if non-trivial failures this phase, extract testing patterns to satellite
8b. Scope reduction check -- compare plan requirements against deliverables, verify new components are wired not just present
9. Update CLAUDE.md -- `/phase-wrap` skill handles steps 8a-11 + commit. Reads phase satellite as source of truth. Checks `deferred.md` for resolved items.
10. Update project skills (docs/) with verified patterns -- canonical sync at wrap only
11. Update MEMORY.md + satellites, then next phase

## Memory Management
- MEMORY.md has a 200-line context limit -- keep it index-level
- Use satellite files (e.g., systems.md, patterns.md) for detailed notes, linked from MEMORY.md
- Proactively split before hitting the limit, not after
- See knowledge-architecture.md for the three-layer model: Skills (portable) -> Satellites (project-specific) -> MEMORY.md (routing table)

## Implementation Phases
- [x] Phase 0: Project infrastructure
- [ ] Phase 1: [Description]
- [ ] Phase 2: [Description]
```

**Section Purposes:**

| Section | Purpose |
|---------|---------|
| Target Runtime | Pins the deployment target. Eliminates ambiguous "should work on..." guidance. |
| Directory Structure | ASCII tree. Claude knows the file layout without exploring. |
| Key Architectural Decisions | Non-obvious choices with rationale. Prevents re-litigation each session. |
| Code Organization | Section/module list. Claude navigates by number, not by searching. |
| Domain-Specific Rules | Behavioral invariants -- collision rules, business logic. Must not change. |
| AI Architecture Notes | File sizing, complexity hotspots, intentional patterns, tight couplings. Prevents Claude from "fixing" deliberate choices. |
| Coding Standards | References to skill files plus project-specific rules. |
| Quality Gate | Testable checklist. Every item maps to a TestingGuide step. |
| Self-Verification Loop | Quick-reference for the review cycle. Same across all projects. |
| Implementation Phases | Phase registry. `[x]` done, `[ ]` pending, `~~text~~` dropped. |

---

## 3. Directory Structures

### Single-File Application (HTML5)

```
[ProjectRoot]/
+-- .gitignore              # Git ignore rules
+-- ProjectName.html        # The entire application
+-- CLAUDE.md               # Project instructions
+-- docs/
|   +-- skill.md            # Domain skill (project copy)
|   +-- js-skill.md         # Language skill (project copy)
|   +-- existing/           # Reference material (old code, specs)
+-- tests/
    +-- TestingGuide.html   # Manual testing checklist
    +-- [test files]        # Automated tests if applicable
```

### Multi-File Module (PowerShell)

```
[ProjectRoot]/
+-- ProjectName.ps1         # Entry point / launcher
+-- Build-Release.ps1       # Build and packaging script
+-- CLAUDE.md               # Project instructions
+-- config/
|   +-- defaults.json       # Default configuration
+-- src/
|   +-- ProjectName.psd1    # Module manifest
|   +-- ProjectName.psm1    # Root module (dot-sourcing)
|   +-- Core/               # Core functions
|   +-- Feature1/           # Feature module
|   +-- GUI/                # Optional GUI layer
+-- tests/
|   +-- Invoke-QualityCheck.ps1
|   +-- Core.Tests.ps1
|   +-- Feature1.Tests.ps1
+-- docs/
|   +-- skill.md
|   +-- ps-skill.md
+-- logs/
```

### Multi-File Application (HTML5 + esbuild)

When a single-file application outgrows its architecture (typically 5,000+ lines), the project splits into ES6 modules with esbuild bundling.

```
[ProjectRoot]/
+-- .gitignore              # Git ignore rules
+-- index.html              # Dev entry point (ES6 module imports)
+-- build.ps1               # esbuild bundle -> inlines CSS+JS -> dist/
+-- serve.ps1               # esbuild --serve for dev (localhost:8000)
+-- CLAUDE.md               # Project instructions
+-- archive/                # Pre-split single-file backup
|   +-- ProjectName-v1-single-file.html
+-- src/
|   +-- balance.js          # Constants, config, cost tables
|   +-- util.js             # Shared helpers
|   +-- state.js            # All mutable state (G namespace)
|   +-- [subsystem].js      # Feature modules
|   +-- engine.js           # Game loop / tick driver
|   +-- render/             # Render modules (one per tab/view)
|   |   +-- render-[tab].js
|   +-- ui.js               # DOM event handlers
|   +-- main.js             # Wiring, init, entry point
|   +-- styles.css          # Extracted CSS (inlined by build)
+-- tools/
|   +-- esbuild.exe         # Standalone binary (no npm)
+-- dist/
|   +-- ProjectName.html    # Production: single file, file:// compatible
+-- docs/
|   +-- skill.md            # Domain skill (project copy)
|   +-- js-skill.md         # Language skill (project copy)
+-- tests/
    +-- TestingGuide.html   # Manual testing checklist
```

**Key conventions:** Shared mutable state lives on a `G` namespace object. Circular dependencies are resolved via three strategies: shared functions in state.js, `G.fn()` wiring in main.js, and `setXDeps()` injection for render modules. Development uses ES6 modules via `serve.ps1`; production bundles into a single distributable HTML via `build.ps1`.

The `docs/existing/` folder holds reference material -- original code, specs, assessment documents. It is never modified during implementation. The `tests/` folder receives `TestingGuide.html` at minimum; automated test files are added if the tech stack supports a runner.

---

## 4. Session Handoff

Phase 0 runs in the home directory session. Implementation runs in a dedicated project session. The context window of each session stays scoped to its purpose -- home for cross-project work, project for focused implementation.

### Handoff Sequence

1. **Phase 0 completes** in the home directory session. Bootstrap info is recorded in home directory memory.
2. **The operator opens a new session** at `[ProjectRoot]`.
3. **The operator runs `/orient`** -- the skill detects the project context, reads CLAUDE.md, locates the bootstrap entry in home directory memory, reads the plan file, and presents a status report. This is the standard path.
4. **Claude creates project memory** at `~/.claude/projects/[project-key]/memory/MEMORY.md` with initial status.
5. **Implementation begins** at Phase 1.

### Manual Handoff (Fallback)

If `/orient` is not available, the Phase 0 instance provides a handoff prompt at session end for the operator to paste into the new session:

```
We're working on [ProjectName] out of [ProjectRoot]. [One-line description].
Check the global CLAUDE.md for workflow context, then read:

1. Home directory memory for the bootstrap entry
   (`~/.claude/projects/[home-session]/memory/MEMORY.md`)
2. Plan file at `~/.claude/plans/[plan-name].md`
3. Project CLAUDE.md at `[ProjectRoot]/CLAUDE.md`
4. `git log --oneline` to confirm project state

Phase 0 is complete and tested. Start Phase [N].
```

### What Claude Reads on Handoff

| File | What Claude Gets |
|------|-----------------|
| `~/.claude/CLAUDE.md` | Operator preferences, workflow rules, skills reference, lifecycle protocol |
| Home directory `MEMORY.md` | Bootstrap entry: project location, plan file path, applicable skill files |
| Plan file | Phase outlines, architectural decisions from Phase 0 research |
| `[ProjectRoot]/CLAUDE.md` | Project-specific rules, quality gate, code organization, domain constraints |

### Bootstrap Entry Format

```markdown
### [Project Name] (active)
- [Brief description]
- Location: `[ProjectRoot]`
- Plan file: `~/.claude/plans/[plan-name].md`
- Skills: [which canonical skill files apply]
- Start at: Phase 1
```

---

## 5. Self-Verification Loop

The core quality cycle. This is the mechanism that prevents the project from producing unchecked work. It runs after every implementation phase, without exception. No shortcuts. No "we'll test it later."

```
Phase kickoff ----> Write code ----> Update TestingGuide ----> User tests ----> User reports
                                                                                    |
     <-----------------------------------------------------------------------------|
     v
Fix failures ----> /simplify ----> Fix findings ----> Update CLAUDE.md
                                                          |
     <----------------------------------------------------|
     v
Update project skills ----> Update MEMORY.md + satellites ----> Commit ----> Next phase
```

| Step | What Happens |
|------|-------------|
| 1. Phase kickoff | `/kickoff` skill: Read plan/code/memory. Run Phase 0 assumption check (verifies current phase scope against architecture satellite). Present gameplan (incl. test plan) + numbered questions/suggestions tagged [Design] [Clarification] [Suggestion] [Risk] [Critic]. Pseudocode gate for complex algorithms (tagged [Approval Required]). Create phase satellite (`phaseN-decisions.md`) from Q&A outcomes. Wait for the operator. |
| 2. Write code + automated tests | Implement features per the agreed gameplan. Write automated tests (Pester/xUnit/inline). Run tests, fix failures. Code is NOT presented for review until automated tests pass. Record significant mid-phase decisions in the phase satellite as they happen. |
| 3. Update TestingGuide | Add action / expect / if-fail steps for user-facing behavior. Manual tests are the operator's inspection -- a separate activity from automated tests, with a separate owner. |
| 4. User tests | The operator opens TestingGuide.html, runs through the steps, checks the boxes. |
| 5. User reports | Clicks "Copy Report", pastes formatted results into the conversation. |
| 6. Fix failures | Address every TODO item from the report. |
| 7. Run /simplify | Three-agent review: code reuse, code quality, efficiency. Categorize findings as FIX (fix now), DEFER (address in a later phase), or DISMISS (false positive / acceptable). Deferred and dismissed items are appended to `deferred.md` satellite with rationale. |
| 8. Fix findings | FIX-category issues are resolved. DEFER/DISMISS items are recorded, not acted upon. |
| 8a. Testing audit | `/test-audit` (conditional): If non-trivial failures occurred this phase, review failures, classify root causes, extract patterns to testing satellite (`testing-patterns.md`). |
| 8b. Scope reduction check | Compare plan file requirements against actual deliverables. Flag anything specified in the plan but delivered as stub, placeholder, TODO, or "v1". Verify new components are wired (called, connected, read), not just present. |
| 9. Update CLAUDE.md | `/phase-wrap` handles steps 8a-11 + commit. Reads phase satellite as source of truth for decisions made this phase. Checks `deferred.md` for items resolved during implementation. Promotes permanent decisions to CLAUDE.md. |
| 10. Update project skills | Project copies (`docs/`) only -- add `[verified in Phase N]` tags. Canonical sync happens at stabilize or wrap. |
| 11. Update MEMORY.md + satellites | Status, decisions, review findings, gotchas. Migrate detail to satellites when MEMORY.md grows. |
| 12. Commit phase | `git add` changed files, commit "Phase N: [description]". Clean working tree before next phase. |

> **Note:** Step 1 sets the tone. The phase kickoff ensures alignment before a single line of code is written. Present the gameplan, surface decisions and risks, and wait for operator confirmation. No implementation before alignment.

> **Note:** Step 10 is not optional. Every phase must extract patterns into project skills (`docs/`). Canonical skill files (`~/.claude/skills/`) are updated at wrap only -- the project builds its own knowledge, then exports the refined product at the end.

---

## 6. Quality Gate

The quality gate is a checklist in CLAUDE.md. Every item is testable. Every item maps to one or more TestingGuide steps. It is the contractual definition of "done" for each phase.

**Rules:** Testable (has an observable action and expected result). Specific (names the exact behavior, not a general attribute). Negative when needed ("Wall collision stops movement (NOT death)" specifies what must NOT happen). Phase-aware (items added as features are implemented). Droppable (items can be marked dropped with strikethrough but never silently deleted).

| Good (Testable) | Bad (Not Testable) |
|----------------|-------------------|
| All 18 color schemes render correctly | Code quality is good |
| Wall=stop, body=stop, tail(len>5)=death | Performance is acceptable |
| No `var` declarations in codebase | All features work |
| Settings persist across page reload | Error handling is robust |
| F3 toggles performance overlay on/off | UI looks nice |

**Dropped items:**
```markdown
- [ ] ~~Touch controls~~ (Phase 6 dropped)
```

The quality gate grows phase by phase. Do not attempt to write the complete gate in Phase 0 -- add items as the features they test are implemented. Remove items only by marking them dropped with strikethrough. The gate is a historical record of what the project committed to and what it delivered.

---

## 7. /simplify Review Pass

The `/simplify` skill runs three parallel review agents against code changed during the current phase. It is a quality audit, not a feature review -- the agents examine what was built, not what should have been built.

### Three Review Agents

| Agent | Focus |
|-------|-------|
| Code Reuse | New code duplicating existing utilities; inline logic that could use existing helpers |
| Code Quality | Redundant state, copy-paste blocks, leaked abstractions, magic numbers, stringly-typed logic, test coverage gaps |
| Efficiency | Redundant computation, missed O(1) opportunities, hot-path bloat, memory leaks, overly broad operations |

All three agents flag positive patterns (`[Pattern]`) alongside issues -- techniques worth preserving in project skills.

**Findings categorization:** After the three agents report, findings are categorized:
- **FIX** -- real issue, fix now (default for Critical/High severity)
- **DEFER** -- valid finding but out-of-scope for this phase (record in `deferred.md` with target phase)
- **DISMISS** -- false positive or acceptable tradeoff (record in `deferred.md` with rationale)

The categorization is autonomous (Claude decides) with operator override capability.

**How changes are identified:** Claude uses `git diff HEAD~1` to identify files changed in the last commit, or `git diff` for uncommitted work in progress. It reads the full file and audits systematically.

**What /simplify does NOT do:** It does not propose new features or suggest scope changes. It does not redesign architecture. It does not litigate findings -- FIX issues are fixed, DEFER/DISMISS items are recorded and moved on from.

---

## 8. Skills Sync Protocol

The skills sync has two distinct operations at different points in the lifecycle. Mixing them -- updating canonical copies during active implementation, or skipping the wrap sync -- corrupts the knowledge chain.

### Per-Phase: Pattern Extraction (Step 10)

After each phase's `/simplify` pass, extract patterns from the work just completed:

- **Project skills** (`docs/*.md`) -- update with new patterns, add `[verified in Phase N]` tags
- **Satellites** (`memory/patterns.md`) -- project-specific patterns, bug patterns, anti-patterns discovered during implementation
- **Canonical skill files are NOT touched** -- the project builds its own knowledge; the export happens at wrap

### Project Wrap: Canonical Sync

At project wrap (or stabilize), the accumulated knowledge is audited and synced outward to the canonical library. See Sections 11-12 for the full process.

### Severity Classification

When extracting patterns, classify by severity to determine what warrants preservation:

| Severity | Criterion | Worth Keeping? |
|----------|-----------|---------------|
| Critical | Crash-level bug (shadow variable, missing null check) | Always |
| High | Performance issue (O(n) where O(1) possible), logic error | Always |
| Medium | Code smell (magic numbers, leaked abstraction, redundant state) | Always |
| Low | Minor style preference | Only if a concise one-liner |

### Tagging Format

```markdown
### Offscreen Canvas (Pre-rendering) [verified in Phase 1, Phase 7 -- grid cache + food cache with dirty flags]
```

The tag goes inline after the section header, inside square brackets. Multiple phases are comma-separated. Tags go on **project copies** (`docs/`) during implementation; canonical copies receive them at wrap sync.

**New sections:** If a phase discovers a pattern not yet in any skill file, add a new section with the `[verified in Phase N]` tag to the project copy. Do not add speculative sections -- only patterns that were actually built and tested.

---

## 9. Testing Methodology

Manual testing is the primary quality gate when automated test runners are impractical. The TestingGuide.html is a self-contained, single-file checklist that runs from `file://`.

**When manual testing is appropriate:** `file://` protocol (cross-origin blocks iframe/XHR between HTML files). GUI visual verification (canvas rendering, CSS layout, color schemes). Single-file applications (no module system, no build tools to hook into). No build tools (adding a test runner means adding a dependency the project does not need).

### TestingGuide Structure

| Component | Description |
|-----------|-------------|
| Phase sections | One collapsible `<div class="phase">` per implementation phase |
| Phase header | Click to expand/collapse; shows `N/M` progress counter |
| Step structure | Each step: checkbox + label + action/expect/if-fail block |
| Copy Report | Generates pasteable text report from current checkbox state |
| Reset All | Clears all checkboxes + localStorage (with confirm dialog) |
| Persistence | Checkbox state survives tab close via localStorage |

### Critical: Checkbox + Label ID Pattern

Use `<input id="check-N-M">` and `<label for="check-N-M">` as **separate elements**. Do NOT wrap the input inside the label. The `generateReport()` function uses `label[for="${check.id}"]` to look up step names. Wrapped labels have no `for` attribute -- the report shows raw IDs instead of human-readable names.

```html
<!-- CORRECT: separate input and label linked by id/for -->
<input type="checkbox" id="check-3-1" data-phase="3" data-step="1">
<label for="check-3-1">Settings persist across page reload</label>

<!-- WRONG: wrapping label around input -- breaks Copy Report -->
<label><input type="checkbox"> Settings persist across page reload</label>
```

### Action / Expect / If-Fail Structure

Every test step has three parts:

```html
<p><span class="action">Do:</span> Open the app, change a setting, press F3.</p>
<p><span class="expect">Expect:</span> Performance overlay shows FPS > 30.</p>
<p><span class="if-fail">If FPS is low:</span> Open F12 Console, check for warnings.</p>
```

- **Action** (blue): What the tester does -- name buttons, keys, setting values
- **Expect** (green): Observable outcome, not implementation detail
- **If-Fail** (red): Diagnostic steps when the expectation is not met

### Copy Report Output Format

```
[Project] -- Testing Guide Report

Phase 0: Project Infrastructure: 3/3
  [DONE] Open app in browser
  [DONE] Open test file in browser
  [DONE] Verify project file structure

Phase 1: Core Engine: 7/8
  [TODO] Run test suite
  [DONE] Core feature works
  ...
```

---

## 10. MEMORY.md Management

### Three-Layer Knowledge Model

Project knowledge stratifies into three layers as the project grows in complexity. The model emerges organically -- do not pre-create satellite files for a three-phase project. Let the 200-line pressure drive the split.

See `knowledge-architecture.md` for the full specification.

| Layer | Location | Content | Audience |
|-------|----------|---------|----------|
| Skills | `docs/*.md` + `~/.claude/skills/` | Portable, generalized patterns | Any future project of this type |
| Satellites | `memory/systems.md`, `patterns.md`, etc. | Project-specific reference (variables, chains, migrations) | Future sessions on this codebase |
| MEMORY.md | `memory/MEMORY.md` | Live index, status, routing pointers | The session -- orient a fresh instance in seconds |

The stratification emerges by phase: Phases 0-2 fit in MEMORY.md alone. By Phase 3-5, patterns split to a satellite. By Phase 5-7, per-subsystem detail splits to systems.md, and MEMORY.md becomes a routing table pointing to the detail files.

### Project Memory

Lives at `~/.claude/projects/[project-key]/memory/MEMORY.md`. Created during session handoff. Target: 60-120 lines active, under 200 always.

Key sections: Status, Satellite File links, Architecture Overview, Quick Reference, Verified Patterns (top 5-6 with pointer to patterns.md for the full catalog).

Standard satellite files: `phase0-architecture.md` (foundational decisions with defaults, break conditions, pattern sources), `phaseN-decisions.md` (per-phase kickoff Q&A outcomes and mid-phase changes), `deferred.md` (review findings deferred or dismissed, reviewed at stabilize/wrap), `systems.md` (subsystem reference), `patterns.md` (verified patterns and anti-patterns), `testing-patterns.md` (root cause analysis from `/test-audit`).

### Home Directory Memory

Lives at `~/.claude/projects/[home-session]/memory/MEMORY.md`. Persistent across all projects.

| Section | Contents |
|---------|---------|
| User Profile | Background, communication preferences, working style |
| Completed Projects | One-line index entries with pointer to `completed-projects.md` (cold storage) |
| Active Projects | Full bootstrap detail (location, plan file, skills, phase status) |
| Dev Environment | OS, tools, versions |

### Cold Storage Pattern

MEMORY.md is auto-loaded with a 200-line truncation limit. To stay under that limit:

- **Active projects** get full detail in MEMORY.md (location, plan file, skills, phase status, key decisions)
- **Completed projects** move to `completed-projects.md` in the same memory folder. MEMORY.md keeps only a one-line index entry per project.
- **On project wrap**, move the full entry to cold storage and replace it with a summary line.

```markdown
## Completed Projects

Full details in `completed-projects.md`. Summary index:

- **ProjectName** (v1.0.0) -- One-line description. `[ProjectRoot]`
```

### Active Project Entry

```markdown
### [Project Name] (Phase N complete -- ready for Phase M)
- [Brief description]
- Location: `[ProjectRoot]`
- Plan file: `~/.claude/plans/[plan-name].md`
- [Key details: tech stack, scope, notable features]
- [Status: phases completed, current phase]
```

### Update Frequency

| When | What to Update |
|------|---------------|
| After every phase | Status, decisions, review findings |
| On gotcha discovery | Add immediately -- do not wait for phase end |
| On stabilize | Mark "Stabilized", record test counts and line counts. Project stays in active memory. |
| On project wrap | Final status, move to cold storage, update index line |
| MEMORY.md approaching 200 lines | Split detail into satellite files; keep main file index-level |

---

## 11. Stabilize (Shelf a Project)

When a project reaches a natural pause point -- MVP complete, milestone achieved -- but is NOT finished. More phases are planned. The project is being shelved, not decommissioned.

1. **Final `/simplify` pass** -- review the last phase's changes.
2. **Deferred findings review** -- read `deferred.md`. For each DEFER item: resolve (implement now), re-defer with updated rationale, or dismiss. For each DISMISS item: confirm the rationale still holds. All items must have a final disposition.
3. **Satellite audit** -- read ALL satellite files cover to cover. For each entry, ask: "Is this generalizable beyond this project?" Promote generalizable patterns to project skills (`docs/`).
4. **Components audit** -- check `components-index.md`. Did this project adapt any indexed component? If the adaptation is superior, update the index pointer. Did this project create new reusable components? Add them.
5. **Canonical sync** -- sync project skills (`docs/`) to canonical skill files (`~/.claude/skills/`). Strip project-specific names, generalize code examples, add `[verified]` tags. See `knowledge-architecture.md` Section 3 for generalization rules.
6. **Create meta-skills if discovered** -- if the project revealed new workflow patterns that do not fit existing skill files.
7. **Update `~/.claude/skills/index.md`** -- add any new skills with descriptions and dependencies.
8. **Update project MEMORY.md** -- mark status as "Stabilized" (not wrapped). Record completed phases, test counts, final line counts. Satellite files are preserved for reactivation.
9. **Update home directory memory** -- update the project entry status to "Stabilized". Do NOT move to cold storage -- the project stays in the active section.
10. **Commit** -- "Stabilize: [project name] -- [milestone summary]"

> **Note:** Stabilize is not wrap. The project stays in active memory. No cold storage transfer. A future instance can reactivate it by reading the preserved MEMORY.md and satellites.

**When to stabilize vs. wrap:**
- **Stabilize** -- more work is planned (future tiers, features, ports). Capture knowledge now, continue later.
- **Wrap** -- the project is done. No more planned work. Archive and close.

---

## 12. Project Wrap

When the project is truly complete -- no more planned phases, no more features to add. The project wrap ensures that accumulated knowledge is preserved and the record is closed cleanly.

1. **Final `/simplify` pass** -- review the last phase's changes.
2. **Deferred findings review** -- read `deferred.md`. ALL items must resolve at wrap -- nothing defers past project close. For each DEFER item: implement now or dismiss with final rationale. For each DISMISS item: confirm rationale. Mark the file as resolved.
3. **Satellite audit** -- read ALL satellite files (patterns.md, systems.md, etc.) cover to cover. For each entry, ask: "Now that the full project is complete, is this generalizable?" Patterns that looked project-specific early on may be clearly universal in hindsight. Promote generalizable patterns to project skills (`docs/`).
4. **Components audit** -- check `components-index.md`. Did this project adapt any indexed component? If the adaptation is better, update the index pointer. Did this project create new reusable components? Add them.
5. **Canonical sync** -- sync project skills (`docs/`) to canonical skill files (`~/.claude/skills/`). This is the ONE time canonical skills are updated during the project's life. Strip project-specific names, generalize code examples, add `[verified]` tags.
6. **Create meta-skills if discovered** -- if the project revealed workflow patterns that do not fit existing skill files.
7. **Update `~/.claude/skills/index.md`** -- add any new skills with descriptions and dependencies.
8. **Update global `~/.claude/CLAUDE.md`** -- if the project revealed workflow improvements.
9. **Update home directory memory** -- move the full project entry from MEMORY.md to `completed-projects.md`. Replace with a one-line index entry (Cold Storage Pattern).
10. **Update project MEMORY.md and satellites** -- all phases complete, final line count, feature summary. Satellite files are preserved for reactivation if needed.

If a project teaches you something that does not fit existing skill files, write a new one. That is how the skill library grows.

---

## 13. Gotchas

Collected from completed projects. These are not theoretical risks -- they are failures that occurred on real projects. Assume they will recur.

### Lifecycle Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| `/simplify` scope too broad | Auditing files not changed in this phase | Pass `git diff HEAD~1 --name-only` to scope the audit to phase-changed files only |
| Session handoff loses context | New instance doesn't know Phase 0 happened | MEMORY.md is the bridge; home memory has bootstrap info |
| Plan file treated as living document | Plan diverges from reality, causes confusion | Plan file is write-once. CLAUDE.md is the living document. |
| Phase numbering gaps (5.5) | Progress tracking breaks | Use string-based phase IDs; gaps are acceptable |
| Phase dropped mid-project | Quality gate has impossible items | Mark with `~~strikethrough~~` and "(Dropped)" |
| Over-engineered Phase 0 | Weeks on scaffolding before writing any code | Phase 0 is scaffolding. Skills refine during implementation. |
| Canonical skills not synced | Pattern from Phase 3 missing from canonical | Canonical sync happens at stabilize (Section 11) or project wrap (Section 12), not per-phase. Per-phase updates go to project copies (`docs/`) only. |
| TestingGuide format wrong | Copy Report shows IDs, not step names | Use `id="check-N-M"` + `<label for>` (not wrapped) |
| Memory file too large | MEMORY.md exceeds 200-line context limit | Move completed projects to `completed-projects.md` (Cold Storage Pattern). For project memory, split into satellite files (Section 10, `knowledge-architecture.md`). |
| Project memory deleted on wrap | Completed project reactivated but no memory files exist | Preserve project MEMORY.md and satellites on wrap -- needed for bug fixes. Cold storage is for *home directory* memory, not project memory. |
| Satellite files created too early | Empty satellite files for a 3-phase project | Let the 200-line pressure drive the split. Phases 0-2 usually fit in MEMORY.md alone. |
| Field surprises lost | Tool deployed, bugs found in field, no record | After first real-world deployment, add a Field Notes section to project MEMORY.md or a satellite. |

### Testing Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| Label wraps input | Copy Report shows IDs, not names | Separate `<input id>` + `<label for>` |
| Duplicate checkbox IDs | Checkboxes toggle in pairs, state corrupted | `check-{phase}-{step}` with unique phase numbers |
| `navigator.clipboard` blocked on `file://` | Copy Report silently fails | Include `execCommand('copy')` fallback |
| localStorage full | Checkbox state lost on reload | Wrap in try/catch, continue without persistence |
| Phase numbering gaps in TestingGuide | `updateProgress()` misses the phase | String-based `data-phase="5.5"` |
| `data-phase` missing on checkbox | Progress counter never updates | Always include `data-phase` on every checkbox |
| Steps too vague | Tester doesn't know what to look for | Every step needs Action + Expect + If-Fail |

---

## 14. Quick Reference

### Phase 0 Checklist (Home Directory Session)

- [ ] Open home directory session
- [ ] Define project scope (what it does, tech stack, constraints)
- [ ] Read `~/.claude/skills/index.md` -- cross-reference plan requirements against full index
- [ ] Check `~/.claude/skills/components-index.md` -- identify reusable implementations
- [ ] Research and create any missing skill files for the project's tech stack
- [ ] Add new skills to `index.md` with descriptions and dependencies
- [ ] Set dependency policy (self-contained vs. framework packages)
- [ ] **Use plan mode for architectural decisions** -- tech stack, runtime target, module structure, key tradeoffs
- [ ] Run pattern fit check before writing the plan
- [ ] Run plan critique agent against draft plan
- [ ] Create project directory with standard structure
- [ ] Create `.gitignore` (exclude `.claude/`, runtime artifacts, credentials)
- [ ] `git init` and initial commit
- [ ] Copy relevant skill files to `[ProjectRoot]/docs/`
- [ ] Write `[ProjectRoot]/CLAUDE.md` from template
- [ ] Write `[ProjectRoot]/tests/TestingGuide.html` from template
- [ ] Add bootstrap entry to home directory memory
- [ ] Create plan file at `~/.claude/plans/[plan-name].md` (include test deliverables per phase)
- [ ] Create project memory folder + architecture satellite (`phase0-architecture.md`)
- [ ] **Commit Phase 0** -- `git add` all project scaffolding, commit "Phase 0: Project infrastructure"

### Session Handoff Checklist

- [ ] Open new session at `[ProjectRoot]`
- [ ] Run `/orient` -- reads bootstrap entry, plan file, project CLAUDE.md automatically
- [ ] Claude creates project memory at `~/.claude/projects/[project-key]/memory/MEMORY.md`

### Per-Phase Loop (Steps 1-12)

| # | Action | Output |
|---|--------|--------|
| 1 | Phase kickoff (`/kickoff`) | Phase 0 assumption check + gameplan (incl. test plan) + numbered questions/suggestions tagged [Design] [Clarification] [Suggestion] [Risk] [Critic]. Pseudocode gate for complex algorithms. Creates phase satellite. |
| 2 | Write code + automated tests | Feature implemented, automated tests pass. Code NOT presented for review until tests pass. Mid-phase decisions recorded in phase satellite. |
| 3 | Update TestingGuide | Manual test steps for user-facing behavior (separate from automated tests) |
| 4 | User tests | Checkboxes filled |
| 5 | User reports | Copy Report pasted to conversation |
| 6 | Fix failures | All TODOs resolved |
| 7 | `/simplify` | 3-agent audit + FIX/DEFER/DISMISS categorization. Deferred items to `deferred.md`. |
| 8 | Fix findings | FIX-category issues resolved |
| 8a | Testing audit (`/test-audit`, conditional) | If non-trivial failures: classify root causes, extract patterns to testing satellite |
| 8b | Scope reduction check | Plan requirements vs. actual deliverables verified; stubs/placeholders flagged |
| 9 | Update CLAUDE.md (`/phase-wrap`) | Phase marked complete. Reads phase satellite, checks deferred.md. Steps 8a-11 + commit. |
| 10 | Update project skills | `[verified in Phase N]` on project copies (`docs/`) only; extract patterns to project skills + satellites |
| 11 | Update MEMORY.md + satellites | Status, decisions, gotchas; migrate detail to satellites when MEMORY.md grows |
| 12 | Commit phase | Clean working tree before next phase |

---

*All processes are documented. All knowledge must be preserved.*
