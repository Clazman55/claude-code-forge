# Claude Project Lifecycle Reference

*A structured workflow for Claude-augmented software development. Single-operator, quality-gated, phase-based.*

---

## Overview

Every project built with Claude Code follows the same lifecycle, regardless of language or domain. The structure exists because Claude has no memory between sessions -- without it, each conversation starts from scratch and accumulated knowledge is lost. This lifecycle solves that problem by externalizing everything into files that persist across sessions.

The workflow has been validated across nine projects totalling approximately 44,000 lines of source code, spanning PowerShell modules, HTML5 games, system utilities, an incremental game with esbuild bundling, and a public methodology repository.

```
Phase 0 (Home Session)       Research, plan, scaffold, create skills
        |
        v
Session Handoff              Open project session, Claude reads bootstrap info
        |
        v
Phase 1..N (Project)         Implement, test, review, verify (loop per phase)
        |
        v
Stabilize (optional)         Satellite audit, canonical sync, shelf for later reactivation
        |
        v
Project Wrap                 Final review, skills sync, cold storage
```

**Phase Outputs:**

| Stage | Deliverables |
|-------|-------------|
| Phase 0 | Skill files, CLAUDE.md, TestingGuide skeleton, plan file, directory structure |
| Session Handoff | Project memory created, context loaded from bootstrap entry |
| Each Phase 1..N | Working code, updated TestingGuide, updated CLAUDE.md, skills tags, updated MEMORY.md |
| Stabilize | Canonical skills synced, project memory marked "Stabilized", stays in active memory |
| Project Wrap | Tagged canonical skills, updated index, home memory moved to cold storage |

**Philosophy:** Systematic, phased, quality-gated. Phase 0 is scaffolding only -- no application code. The self-verification loop runs after every phase without exception. The plan file is write-once; CLAUDE.md is the living document. Projects may be stabilized (shelved with knowledge captured) or wrapped (completed and cold-storaged).

---

## 1. Phase 0: Research & Planning

Phase 0 is where you do everything *except* write application code. It runs in the home directory session -- a workspace scoped for cross-project work like skills research, planning, and scaffolding. The reason for this separation is that each Claude session has its own memory folder, and the home session's memory serves as the bridge between projects.

> **Do Not Write Application Code in Phase 0.** Deliverables are: skill files, CLAUDE.md, TestingGuide skeleton, plan file, and directory structure. Nothing else. Skills can be refined during implementation -- don't over-engineer Phase 0.

### Steps

1. **Assess existing codebase** (if rewriting) -- Read original code, document what works and what changes. Create an assessment document for large codebases.
2. **Check skills index** -- Read `~/.claude/skills/index.md`. Cross-reference the plan's technical requirements against the full index (registry -> cim-wmi, testing -> pester, GUI -> wpf, etc.). Don't trust an idea file's skill list as complete.
3. **Check components index** -- Read `~/.claude/skills/components-index.md`. Identify reusable implementations for the project's tech stack. Reference specific components in the plan file so the implementation instance knows what's already been built and tested.
4. **Research and create new skills** -- Web search + docs for the tech stack. Follow the canonical format: intro with provenance, table of contents, numbered sections, code examples, anti-patterns, gotchas table, checklist. **If entering an unfamiliar domain** (new language, framework, or API), prototype the riskiest technical assumption before plan mode -- under 50 lines, answering one feasibility question ("can the toolchain build and run," "does this API return what we need"). If the prototype fails, raise it before the architecture commits to the assumption.
5. **Update skills index** -- Add new skills to `index.md` with descriptions and dependency mappings.
6. **Use plan mode for architectural decisions** -- When Phase 0 hits meaningful decisions (tech stack, runtime target, module structure, key tradeoffs), enter plan mode so the user gets the interactive UI. Plain Q&A is a fallback, not the default. Plan file must include test deliverables per phase (which automated framework, what coverage expectations).
7. **Create project directory** -- Standard structure matching the project type (see Directory Structures section). Include `.gitignore` (exclude `.claude/`, runtime artifacts, credentials). Run `git init` and make an initial commit.
8. **Copy relevant skills to `docs/`** -- These become the project's working copies that accumulate `[verified in Phase N]` tags.
9. **Write CLAUDE.md** from the template (see CLAUDE.md Template section).
10. **Write TestingGuide.html** from the template (see Testing Methodology section).
11. **Add bootstrap entry** to home directory memory -- project name, location, plan file path, applicable skills, starting phase.
12. **Create plan file** at `~/.claude/plans/[plan-name].md` -- Phase 0 summary + subsequent phase outlines. This file is write-once. Each phase outline must include test deliverables (automated framework, coverage expectations).
13. **Commit Phase 0** -- `git add` all scaffolding files and commit "Phase 0: Project infrastructure". Global files (`~/.claude/`) are not git-tracked but are updated; the project commit captures the project-side artifacts.

**Skills System:** Canonical copies live at `~/.claude/skills/` -- the master, shared across all projects. Project copies live at `[ProjectRoot]/docs/` -- working copies that ship with the project. Tags flow both directions: when a project copy gets a `[verified in Phase N]` tag, the canonical copy gets the same tag.

---

## 2. CLAUDE.md Template

The CLAUDE.md is the contract between Claude and the project. It lives at the project root and is the first thing Claude reads at every session start. A well-written CLAUDE.md means a new Claude instance can orient itself and start working without any additional context. A poorly written one means you'll spend the first ten minutes of every session re-explaining your project.

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
1. Phase kickoff -- read plan/code/memory, run plan critique (subagent), present gameplan (incl. test plan) + numbered questions/suggestions tagged [Design] [Clarification] [Suggestion] [Risk] [Critic], wait for user. Pseudocode gate for complex algorithms: include pseudocode tagged [Approval Required], do not implement until approved.
2. Write code + automated tests (Pester/xUnit/inline) -- run tests, fix failures. Code is NOT presented for review until automated tests pass.
3. Update TestingGuide.html -- manual test steps for user-facing behavior (the operator's inspection, separate from automated tests)
4. User follows guide
5. User reports results
6. Fix failures
7. /simplify
8. Fix issues
8a. /test-audit (conditional) -- if non-trivial failures this phase, extract testing patterns to satellite
9. Update CLAUDE.md
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

Every section in the template serves a specific purpose. If a section doesn't pull its weight, it's wasting context tokens:

| Section | Purpose |
|---------|---------|
| Target Runtime | Pins the deployment target. Eliminates ambiguous "should work on..." advice. |
| Directory Structure | ASCII tree. Claude knows the file layout without exploring. |
| Key Architectural Decisions | Non-obvious choices with rationale. Prevents re-litigating each session. |
| Code Organization | Section/module list. Claude navigates by number, not by searching. |
| Domain-Specific Rules | Behavior invariants -- collision rules, business logic. Must not change. |
| AI Architecture Notes | File sizing, complexity hotspots, intentional patterns, tight couplings. Prevents AI from "fixing" deliberate choices. |
| Coding Standards | References to skill files + project-specific rules. |
| Quality Gate | Testable checklist. Every item maps to a TestingGuide step. |
| Self-Verification Loop | Quick-reference for the 12-step cycle. Same across all projects. |
| Implementation Phases | Phase registry. `[x]` done, `[ ]` pending, `~~text~~` dropped. |

---

## 3. Directory Structures

Consistent directory structure means Claude can navigate any project without exploration. These templates cover three common patterns -- choose the one that matches your project's complexity and tech stack.

### Single-File Application (HTML5)

```
[ProjectRoot]/
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
[ProjectRoot]/
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

### Multi-File Application (HTML5 + esbuild)

When a single-file app outgrows its architecture (typically 5,000+ lines), split into ES6 modules with esbuild bundling.

```
[ProjectRoot]/
├── .gitignore              # Git ignore rules
├── index.html              # Dev entry point (ES6 module imports)
├── build.ps1               # esbuild bundle -> inlines CSS+JS -> dist/
├── serve.ps1               # esbuild --serve for dev (localhost:8000)
├── CLAUDE.md               # Project instructions
├── archive/                # Pre-split single-file backup
│   └── ProjectName-v1-single-file.html
├── src/
│   ├── balance.js          # Constants, config, cost tables
│   ├── util.js             # Shared helpers
│   ├── state.js            # All mutable state (G namespace)
│   ├── [subsystem].js      # Feature modules
│   ├── engine.js           # Game loop / tick driver
│   ├── render/             # Render modules (one per tab/view)
│   │   └── render-[tab].js
│   ├── ui.js               # DOM event handlers
│   ├── main.js             # Wiring, init, entry point
│   └── styles.css          # Extracted CSS (inlined by build)
├── tools/
│   └── esbuild.exe         # Standalone binary (no npm)
├── dist/
│   └── ProjectName.html    # Production: single file, file:// compatible
├── docs/
│   ├── skill.md            # Domain skill (project copy)
│   └── js-skill.md         # Language skill (project copy)
└── tests/
    └── TestingGuide.html   # Manual testing checklist
```

**Key conventions:** Shared mutable state on a `G` namespace object. Circular dependencies resolved via three strategies: shared functions in state.js, `G.fn()` wiring in main.js, and `setXDeps()` injection for render modules. Dev uses ES6 modules via `serve.ps1`; production bundles into a single distributable HTML.

The `docs/existing/` folder holds reference material -- original code, specs, assessment documents. It is never modified during implementation. The `tests/` folder gets `TestingGuide.html` at minimum; automated test files are added if the tech stack supports it.

---

## 4. Session Handoff

Phase 0 runs in the home directory session. Implementation runs in a dedicated project session. The reason for the split is scoping: the home session is for cross-project work (skills, research), while the project session focuses on one codebase. Each session gets its own memory folder, keeping context focused.

### Handoff Sequence

1. **Phase 0 completes** in the home directory session. Bootstrap info is in home directory memory.
2. **User opens new session** at `[ProjectRoot]`.
3. **User runs `/orient`** -- the skill detects the project context, reads CLAUDE.md, finds the bootstrap entry in home directory memory, reads the plan file, and presents a status report. This is the standard path.
4. **Claude creates project memory** at `~/.claude/projects/[project-key]/memory/MEMORY.md` with initial status.
5. **Implementation begins** at Phase 1.

### Manual Handoff (Fallback)

If `/orient` is not available, Claude provides a handoff prompt at the end of Phase 0 for the user to paste into the new session:

```
We're working on [ProjectName] out of [ProjectRoot]. [One-line description].
Check the global CLAUDE.md for workflow context, then read:

1. Home directory memory for the bootstrap entry
   (~/.claude/projects/[home-session]/memory/MEMORY.md)
2. Plan file at ~/.claude/plans/[plan-name].md
3. Project CLAUDE.md at [ProjectRoot]/CLAUDE.md
4. `git log --oneline` to confirm project state

Phase 0 is complete and tested. Start Phase [N].
```

### What Claude Reads on Handoff

| File | What Claude Gets |
|------|-----------------|
| `~/.claude/CLAUDE.md` | User preferences, workflow rules, skills reference, lifecycle protocol |
| Home directory `MEMORY.md` | Bootstrap entry: project location, plan file path, applicable skills |
| Plan file | Phase outlines, architectural decisions from Phase 0 research |
| `[ProjectRoot]/CLAUDE.md` | Project-specific rules, quality gate, code organization, domain rules |

### Bootstrap Entry Format

```markdown
### [Project Name] (active)
- [Brief description]
- Location: `[ProjectRoot]`
- Plan file: `~/.claude/plans/[plan-name].md`
- Skills: [which canonical skills apply]
- Start at: Phase 1
```

---

## 5. Self-Verification Loop

This is the core quality cycle. It runs after every implementation phase without exception. The loop exists because Claude can introduce subtle regressions, and without systematic verification, problems compound across phases. Twelve steps, no shortcuts.

Two things to note: automated tests (step 2) and manual tests (step 3) are *separate activities with separate owners*. Claude writes and runs automated tests before presenting code for review. The operator runs manual tests via the TestingGuide to verify user-facing behavior.

```
Phase kickoff --> Write code --> Update TestingGuide --> User tests --> User reports
                                                                            |
     <--------------------------------------------------------------------------
     v
Fix failures --> /simplify --> Fix findings --> /test-audit (conditional)
                                                      |
     <------------------------------------------------
     v
Update CLAUDE.md --> Update project skills --> Update MEMORY.md + satellites --> Commit --> Next phase
```

| Step | What Happens |
|------|-------------|
| 1. Phase kickoff | Read plan/code/memory. Run plan critique (subagent). Present gameplan (incl. test plan) + numbered questions/suggestions tagged [Design] [Clarification] [Suggestion] [Risk] [Critic]. Pseudocode gate for complex algorithms (tagged [Approval Required]). Wait for user. |
| 2. Write code + automated tests | Implement features per agreed gameplan. Write automated tests (Pester/xUnit/inline). Run tests, fix failures. Code is NOT presented for review until automated tests pass. |
| 3. Update TestingGuide | Add action / expect / if-fail steps for user-facing behavior. Manual tests are the operator's inspection -- separate activity from automated tests, separate owner. |
| 4. User tests | Opens TestingGuide.html, runs through steps, checks boxes |
| 5. User reports | Clicks "Copy Report", pastes formatted results into conversation |
| 6. Fix failures | Address every TODO item from the report |
| 7. Run /simplify | Three-agent review: code reuse, code quality, efficiency |
| 8. Fix findings | Real issues get fixed; false positives noted and skipped |
| 8a. Testing audit | `/test-audit` (conditional): If non-trivial failures occurred this phase, review failures, classify root causes, extract patterns to testing satellite (`testing-patterns.md`). |
| 9. Update CLAUDE.md | Mark phase complete, add decisions, update code organization. |
| 10. Update project skills | Project copies (`docs/`) only -- add `[verified in Phase N]` tags. Canonical sync at stabilize/wrap. |
| 11. Update MEMORY.md + satellites | Status, decisions, review findings, gotchas -- migrate detail to satellites when MEMORY.md grows |
| 12. Commit phase | `git add` changed files, commit "Phase N: [description]". Clean working tree before next phase. |

> **Step 1 Sets the Tone.** The phase kickoff ensures alignment before code is written. The plan critique provides genuine adversarial review. Present a gameplan, surface decisions and risks, and wait for user confirmation. No coding before alignment.

> **Step 10 Is Not Optional.** Every phase must extract patterns into project skills (`docs/`). Canonical skills (`~/.claude/skills/`) are updated at project wrap only -- the project builds its own knowledge, then exports the refined product at the end.

---

## 6. Quality Gate

The quality gate is a checklist in CLAUDE.md where every item is testable and maps to one or more TestingGuide steps. Vague items like "code quality is good" don't belong here -- if you can't observe the result, you can't verify it.

**Rules:** Testable (has an observable action and expected result). Specific (names the exact behavior, not a general attribute). Negative when needed ("Wall collision stops movement (NOT death)" specifies what should NOT happen). Phase-aware (items added as features are implemented). Droppable (items can be marked dropped but never silently deleted).

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

The quality gate grows phase by phase. Don't try to write the complete gate in Phase 0. Add items as features are implemented. Remove items only by marking them dropped with strikethrough -- never by deleting them. The gate is a historical record.

---

## 7. /simplify Review Pass

The `/simplify` skill runs three parallel review agents on code changed during the current phase. It catches the kind of issues that are invisible when you're focused on making features work -- redundant computation, missed reuse opportunities, and code smells that compound over time.

### Three Review Agents

| Agent | Focus |
|-------|-------|
| Code Reuse | New code duplicating existing utilities; inline logic that could use existing helpers |
| Code Quality | Redundant state, copy-paste blocks, leaked abstractions, magic numbers, stringly-typed logic |
| Efficiency | Redundant computation, missed O(1) opportunities, hot-path bloat, memory leaks, overly broad operations |

**How changes are identified:** Claude uses `git diff HEAD~1` to identify files changed in the last commit, or `git diff` for uncommitted work in progress. Claude reads the full file and audits systematically.

**What /simplify does NOT do:** Does not make feature suggestions or scope changes. Does not redesign architecture. Does not argue with findings -- real issues get fixed, false positives get noted and skipped.

### Real Examples (Snake Project)

| Phase | Finding |
|-------|---------|
| Phase 1 | Delta cap missing (spiral-of-death on tab return); grid cache not cleared on reset |
| Phase 4 | Hot-path string concat in color comparison -> scalar comparisons; pre-convert hex stops with `.map()` |
| Phase 5 | O(N) `unshift()` at 78K snake length -> path-index tracking (O(1)) |
| Phase 5.5 | `const target` shadow bug in updateDemo; duplicated food-eat blocks -> `consumeFood()` helper |

---

## 8. Skills Sync Protocol

Skills sync ensures that knowledge learned during implementation doesn't stay trapped in one project. It operates at two different lifecycle points with different rules about what gets updated.

### Per-Phase: Pattern Extraction (Step 10)

After each phase's `/simplify` pass, extract patterns from the work just completed:

- **Project skills** (`docs/*.md`) -- update with new patterns, add `[verified in Phase N]` tags
- **Satellites** (`memory/patterns.md`) -- project-specific patterns, bug patterns, anti-patterns
- **Canonical skills are NOT touched** -- the project builds its own knowledge; export happens at wrap

### Project Wrap: Canonical Sync

At project wrap (or stabilize), the accumulated project knowledge gets audited and synced outward. See Sections 11-12 for the full process.

### Severity Classification

When extracting patterns (per-phase or wrap), classify by severity:

| Severity | Criterion | Worth Keeping? |
|----------|-----------|---------------|
| Critical | Crash-level bug (shadow variable, missing null check) | Always |
| High | Performance issue (O(n) where O(1) possible), logic error | Always |
| Medium | Code smell (magic numbers, leaked abstraction, redundant state) | Always |
| Low | Minor style preference | Only if concise one-liner |

### Tagging Format

```markdown
### Offscreen Canvas (Pre-rendering) [verified in Phase 1, Phase 7 -- grid cache + food cache with dirty flags]
```

Tag goes inline after the section header, inside square brackets. Multiple phases are comma-separated. A brief description of what was verified follows. Tags go on **project copies** (`docs/`) during implementation; canonical copies get tagged at wrap sync.

**New sections:** If a phase discovers a pattern not yet in any skill file, add a new section with the `[verified in Phase N]` tag to the project copy. Don't add speculative sections -- only patterns that were actually built and tested.

---

## 9. Testing Methodology

Manual testing via TestingGuide.html is the primary quality gate when automated runners are impractical. This happens more often than you'd think: `file://` protocol blocks cross-origin requests between HTML files, canvas rendering needs visual verification, and single-file apps have no module system to hook a test runner into.

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

Use `<input id="check-N-M">` and `<label for="check-N-M">` as **separate elements**. Do NOT wrap the input inside the label. The `generateReport()` function uses `label[for="${check.id}"]` to look up step names. Wrapped labels have no `for` attribute -- the report shows raw IDs instead of names.

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
<p><span class="action">Do:</span> Open Snake.html, set board to Screensaver, press F3.</p>
<p><span class="expect">Expect:</span> Performance overlay shows FPS > 30.</p>
<p><span class="if-fail">If FPS is low:</span> Open F12 Console, check for [PERF] warnings.</p>
```

- **Action** (blue): What the tester does -- name buttons, keys, setting values
- **Expect** (green): Observable outcome, not implementation detail
- **If-Fail** (red): Diagnostic steps when expectation is not met

### Copy Report Output Format

```
Snake Game -- Testing Guide Report

Phase 0: Project Infrastructure: 3/3
  [DONE] Open Snake.html in browser
  [DONE] Open Snake.test.html in browser
  [DONE] Verify project file structure

Phase 1: Core Engine: 7/8
  [TODO] Run test suite (Snake.test.html)
  [DONE] Snake moves on screen
  ...
```

---

## 10. MEMORY.md Management

Because Claude has no memory between sessions, all project knowledge must be externalized into files. The challenge is that MEMORY.md is auto-loaded with a 200-line truncation limit -- anything past line 200 is silently dropped. This means you need a strategy for what goes where as a project grows.

### Three-Layer Knowledge Model

Project knowledge naturally stratifies into three layers as complexity grows. See the `knowledge-architecture` skill file for the full specification.

| Layer | Location | Content | Audience |
|-------|----------|---------|----------|
| Skills | `docs/*.md` + `~/.claude/skills/` | Portable, generalized patterns | The craft -- any future project |
| Satellites | `memory/systems.md`, `patterns.md`, etc. | Project-specific reference (variables, chains, migrations) | The project -- future sessions on this codebase |
| MEMORY.md | `memory/MEMORY.md` | Live index, status, routing pointers | The session -- orient a fresh instance in seconds |

This model emerges organically: Phases 0-2 fit in MEMORY.md alone. By Phase 3-5, patterns split to a satellite. By Phase 5-7, per-subsystem detail splits to systems.md, and MEMORY.md becomes a routing table. Don't pre-create satellite files for a 3-phase project.

### Project Memory

Lives at `~/.claude/projects/[project-key]/memory/MEMORY.md`. Created during session handoff. Target: 60-120 lines active, under 200 always.

Key sections: Status, Satellite File links, Architecture Overview, Quick Reference, Verified Patterns (top 5-6 with pointer to patterns.md).

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
| On gotcha discovery | Add immediately -- don't wait for phase end |
| On stabilize | Mark "Stabilized", record test counts and line counts. Project stays in active memory. |
| On project wrap | Final status, move to cold storage, update index line |
| MEMORY.md approaching 200 lines | Split detail into satellite files; keep main file index-level |

---

## 11. Stabilize (Shelf a Project)

Stabilize is for projects that have reached a natural pause point -- MVP complete, milestone achieved -- but are NOT finished. More phases are planned for later. The goal is to capture everything a future Claude instance needs to pick up where you left off.

1. **Final `/simplify` pass** -- review the last phase's changes.
2. **Satellite audit** -- read ALL satellite files cover to cover. Ask: "Is this generalizable?" Promote generalizable patterns to project skills (`docs/`).
3. **Components audit** -- check `components-index.md`. Did this project adapt any indexed component? If the adaptation is better (more general, more robust), update the index pointer. Did this project create new reusable components? Add them. Direction is always upward.
4. **Canonical sync** -- sync project skills (`docs/`) -> canonical skills (`~/.claude/skills/`). Strip project-specific names, generalize code examples, add `[verified]` tags. See the `knowledge-architecture` skill file Section 3 for generalization rules.
5. **Create meta-skills if discovered** -- if the project revealed new workflow patterns.
6. **Update `~/.claude/skills/index.md`** -- add any new skills with descriptions and dependencies.
7. **Update project MEMORY.md** -- mark status as "Stabilized" (not wrapped). Record completed phases, test counts, final line counts. Satellite files preserved for reactivation.
8. **Update home directory memory** -- update project entry status to "Stabilized". Do NOT move to cold storage -- the project stays in the active section.
9. **Commit** -- "Stabilize: [project name] -- [milestone summary]"

**Key difference from Project Wrap:** The project stays in the active section of home directory memory. No cold storage move. The project is shelved, not completed -- a future instance can reactivate it by reading the preserved MEMORY.md and satellites.

**When to stabilize vs wrap:**
- **Stabilize** -- more work is planned (future tiers, features, ports). Knowledge capture now, continue later.
- **Wrap** -- project is done. No more planned work. Move to cold storage.

---

## 12. Project Wrap

When the project is truly complete -- no more planned phases. The project wrap ensures patterns are preserved and the project record is closed cleanly. This is the ONE time canonical skills get updated with everything the project learned.

1. **Final `/simplify` pass** -- review the last phase's changes.
2. **Satellite audit** -- read ALL satellite files (patterns.md, systems.md, etc.) cover to cover. For each entry, ask: "Now that the full project is complete, is this generalizable?" Patterns that looked project-specific early on may be clearly universal in hindsight. Promote generalizable patterns to project skills (`docs/`).
3. **Components audit** -- check `components-index.md`. Did this project adapt any indexed component? If the adaptation is better (more general, more robust), update the index pointer. Did this project create new reusable components? Add them. Direction is always upward.
4. **Canonical sync** -- sync project skills (`docs/`) -> canonical skills (`~/.claude/skills/`). This is the ONE time canonical skills get updated. Strip project-specific names, generalize code examples, add `[verified]` tags. See the `knowledge-architecture` skill file Section 3 for generalization rules.
5. **Create meta-skills if discovered** -- if the project revealed new workflow patterns that don't fit existing skills.
6. **Update `~/.claude/skills/index.md`** -- add any new skills with descriptions and dependencies.
7. **Update global `~/.claude/CLAUDE.md`** -- if workflow improvements were discovered.
8. **Update home directory memory** -- move full project entry from MEMORY.md to `completed-projects.md`, replace with one-line index entry (Cold Storage Pattern).
9. **Update project MEMORY.md and satellites** -- all phases complete, final line count, feature summary. Satellite files are preserved for reactivation.

If a project teaches you something that doesn't fit existing skills, write a new one.

---

## 13. Gotchas

These are real problems encountered across completed projects. If you use this workflow, assume they will happen to you too.

### Lifecycle Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| `/simplify` scope too broad | Auditing files not changed in this phase | Pass `git diff HEAD~1 --name-only` to scope the audit to phase-changed files only |
| Session handoff loses context | New Claude doesn't know Phase 0 happened | MEMORY.md is the bridge; home memory has bootstrap info |
| Plan file treated as living document | Plan diverges from reality, causes confusion | Plan file is write-once. CLAUDE.md is the living document. |
| Phase numbering gaps (5.5) | Progress tracking breaks | Use string-based phase IDs; gaps are fine |
| Phase dropped mid-project | Quality gate has impossible items | Mark with `~~strikethrough~~` and "(Dropped)" |
| Over-engineered Phase 0 | Weeks on skills before writing any code | Phase 0 is scaffolding. Skills update during implementation. |
| Canonical skills not synced | Pattern from Phase 3 missing from canonical | Canonical sync happens at stabilize (Section 11) or project wrap (Section 12), not per-phase. Per-phase updates go to project copies (`docs/`) only. |
| TestingGuide format wrong | Copy Report shows IDs, not step names | Use `id="check-N-M"` + `<label for>` (not wrapped) |
| Memory file too large | MEMORY.md exceeds 200-line context limit | Move completed projects to `completed-projects.md` (Cold Storage Pattern). For project memory, split into satellite files (Section 10, `knowledge-architecture` skill file). |
| Project memory deleted on wrap | Completed project reactivated but no memory files exist | Preserve project MEMORY.md and satellites on wrap -- needed for bug fixes. Cold storage is for *home directory* memory, not project memory. |
| Satellite files created too early | Empty satellite files for a 3-phase project | Let the 200-line pressure drive the split. Phases 0-2 usually fit in MEMORY.md alone. |
| Field surprises lost | Tool deployed, bugs found in field, no record | After first real-world deployment, add a `## Field Notes` section to project MEMORY.md or a satellite. Record what surprised you -- hardware-specific failures, silent flags that didn't work, environment differences. Lightweight, optional, but invaluable for the next version. |

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
- [ ] Research and create any missing skills for the project's tech stack
- [ ] Add new skills to `index.md` with descriptions and dependencies
- [ ] **Use plan mode for architectural decisions** -- tech stack, runtime target, module structure, key tradeoffs
- [ ] Create `[ProjectRoot]/` directory with standard structure
- [ ] Create `.gitignore` (exclude `.claude/`, runtime artifacts, credentials)
- [ ] `git init` and initial commit
- [ ] Copy relevant skills to `[ProjectRoot]/docs/`
- [ ] Write `[ProjectRoot]/CLAUDE.md` from template
- [ ] Write `[ProjectRoot]/tests/TestingGuide.html` from template
- [ ] Add bootstrap entry to home directory memory
- [ ] Create plan file at `~/.claude/plans/[plan-name].md` (include test deliverables per phase)
- [ ] **Commit Phase 0** -- `git add` all project scaffolding, commit "Phase 0: Project infrastructure"

### Session Handoff Checklist

- [ ] Open new session at `[ProjectRoot]`
- [ ] Run `/orient` -- reads bootstrap entry, plan file, project CLAUDE.md automatically
- [ ] Claude creates project memory at `~/.claude/projects/[project-key]/memory/MEMORY.md`

### Per-Phase Loop (Steps 1-12)

| # | Action | Output |
|---|--------|--------|
| 1 | Phase kickoff | Plan critique (subagent) + gameplan (incl. test plan) + numbered questions/suggestions tagged [Design] [Clarification] [Suggestion] [Risk] [Critic]. Pseudocode gate for complex algorithms. |
| 2 | Write code + automated tests | Feature implemented, automated tests pass. Code NOT presented for review until tests pass. |
| 3 | Update TestingGuide | Manual test steps for user-facing behavior (separate from automated tests) |
| 4 | User tests | Checkboxes filled |
| 5 | User reports | Copy Report pasted to conversation |
| 6 | Fix failures | All TODOs resolved |
| 7 | `/simplify` | 3-agent audit complete |
| 8 | Fix findings | Real issues fixed, false positives noted |
| 8a | Testing audit (`/test-audit`, conditional) | If non-trivial failures: classify root causes, extract patterns to testing satellite |
| 9 | Update CLAUDE.md | Phase marked complete |
| 10 | Update project skills | `[verified in Phase N]` on project copies (`docs/`) only; extract patterns to project skills + satellites |
| 11 | Update MEMORY.md + satellites | Status, decisions, gotchas; migrate detail to satellites when MEMORY.md grows |
| 12 | Commit phase | Clean working tree before next phase |

---

*All processes are documented. All knowledge must be preserved.*
