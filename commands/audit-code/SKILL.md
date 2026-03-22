---
description: >
  Full project audit -- code architecture, project infrastructure, documentation,
  testing, memory health, skills sync. Produces a compliance report -- no changes
  made. Trigger on "audit", "audit code", "audit the code", or "scan the project".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /audit-code...**
>
> *Scanning the project. Measuring, not cutting. The report shows current state -- the operator decides what changes.*

# /audit-code -- Full Project Audit

Scans an entire project -- code, infrastructure, documentation, testing, memory, skills. Produces a report. Makes no changes.

---

## Step 1: Identify Scope

Read the project's `CLAUDE.md` for directory structure and file organization. Build a file inventory:

- **Source files** -- all code files (exclude build output, dependencies, generated files)
- **Project infrastructure** -- CLAUDE.md, TestingGuide, quality gate, .gitignore
- **Documentation** -- docs/ folder contents, README if present
- **Memory** -- find the project's memory folder in `~/.claude/projects/`, list MEMORY.md and all satellites
- **Skills** -- docs/*.md skill copies

Count the files per category. This determines how many subagent pairs are needed.

---

## Step 2: Plan Subagent Batches

Divide the audit into logical batches. Each batch is a subagent task. **Run subagents 2 at a time maximum** -- this limits API pressure and context window churn.

Typical batches (adjust based on project size):

| Batch | Focus | What to Read |
|-------|-------|-------------|
| Code Architecture | File sizes, module headers, function length, generic names, undocumented decisions | All source files |
| Code Style | Magic numbers, dead code, hardcoded paths, consistency with project coding standards | All source files |
| CLAUDE.md Health | All sections present and current, phase registry accurate, AI Architecture Notes, directory structure matches reality | CLAUDE.md + actual directory |
| Testing Health | TestingGuide exists and covers all phases, quality gate items map to test steps, automated test coverage matches implemented features | TestingGuide, tests/, quality gate |
| Memory Health | MEMORY.md under 200 lines, satellites linked correctly, status current, no stale info | Project memory folder |
| Skills Sync | Project copies exist for all needed skills, verification tags present, not stale vs canonical | docs/*.md vs ~/.claude/skills/*.md |

For small projects (3-5 source files), consolidate -- Code Architecture + Code Style can be one batch, CLAUDE.md + Testing can be another. Use judgment.

For large projects (20+ source files), split Code Architecture and Code Style into per-module batches. The subagent needs to actually read the files, not skim them.

---

## Step 3: Run Subagents

For each pair of subagents:

1. Launch 2 subagents in parallel with clear instructions:
   - Which files to read
   - Which criteria to check (from the Audit Criteria section below)
   - Report format: category, file path, finding, severity (High/Medium/Low)
2. Wait for both to complete
3. Collect their findings and **append to a scratch file** (`audit-scratch.md` in the project root)
   - Header: batch name, files audited, timestamp
   - All findings listed, no filtering, no summarizing
4. Launch the next pair

Repeat until all batches are complete.

**The main agent writes to the scratch file, not the subagents.** Subagents report back to the main agent; the main agent appends to the file. This is the pattern that works reliably.

**If a subagent fails or times out**, note the gap in the scratch file ("Code Style batch: subagent timed out, files X-Y not audited") and continue. The report should show what was and wasn't covered.

---

## Step 4: Audit Criteria

### Code Architecture
- **File size:** Flag source files exceeding ~500 lines. High priority above ~1000 lines.
- **Module headers:** Flag source files missing a module-level header comment (purpose, dependencies, consumers).
- **Generic module names:** Flag files named `utils`, `helpers`, `common`, `misc` -- dumping ground risk.
- **Function length:** Flag functions exceeding ~50 lines. High priority above ~100 lines.
- **Generic function names:** Flag non-descriptive names (`process`, `doWork`, `handle`, `run`).
- **Undocumented decisions:** Flag code patterns that look unusual or non-obvious but have no explanatory comment.

### Code Style
- **Magic numbers/strings:** Flag unexplained literal values that should be named constants.
- **Dead code:** Flag commented-out blocks, unreachable branches, unused functions.
- **Hardcoded paths:** Flag absolute paths that should be config-driven or relative.
- **Coding standards compliance:** Check against project's coding standards section in CLAUDE.md and relevant skill files in docs/.
- **Consistency:** Flag inconsistent patterns within the same codebase (e.g., mixed naming conventions, mixed error handling approaches).

### CLAUDE.md Health
- **Section completeness:** All template sections present (Target Runtime, Directory Structure, Key Architectural Decisions, Code Organization, Coding Standards, AI Architecture Notes, Quality Gate, Self-Verification Loop, Memory Management, Implementation Phases).
- **Directory structure accuracy:** Does the ASCII tree match the actual file layout?
- **Phase registry:** Are completed phases marked `[x]`? Are dropped phases marked with strikethrough?
- **AI Architecture Notes:** Does the section exist? Does it document file size targets, complexity locations, intentional patterns, tight couplings?
- **Quality gate:** Are all items testable? Do they map to TestingGuide steps?

### Testing Health
- **TestingGuide exists:** Is `tests/TestingGuide.html` present?
- **Phase coverage:** Does TestingGuide have sections for all completed phases?
- **Step quality:** Do steps have Action/Expect/If-Fail structure?
- **Checkbox pattern:** Are checkboxes using `id="check-N-M"` + `<label for>` (not wrapped)?
- **Automated test coverage:** Do source modules have corresponding test files? Is the ratio reasonable?
- **Quality gate script:** Does `Invoke-QualityCheck.ps1` (or equivalent) exist and run?

### Memory Health
- **MEMORY.md size:** Under 200 lines? If over, should content migrate to satellites?
- **Satellite links:** Are all satellites referenced from MEMORY.md?
- **Status current:** Does the status section reflect the actual project state?
- **Stale content:** Any references to phases, decisions, or files that no longer exist?
- **Cold storage readiness:** If project is complete, is the entry ready to move to cold storage?

### Skills Sync
- **Coverage:** Are all skills listed in CLAUDE.md present in docs/?
- **Verification tags:** Do project copies have `[verified in Phase N]` tags for completed phases?
- **Staleness:** Compare project copies against canonical (`~/.claude/skills/`). Flag significant drift -- the project copy may have patterns the canonical doesn't, or the canonical may have been updated since the project copy was made.
- **Missing skills:** Are there patterns in the code that should be in a skill file but aren't?

---

## Step 5: Produce Report

Read the complete scratch file. Produce a structured report:

```
## Project Audit Report -- [Project Name]

### Summary
- Files scanned: N source / N infrastructure / N docs
- Subagent batches: N (all completed / N failed)
- Total findings: N (High: N, Medium: N, Low: N)

### Code Architecture
[Findings with file paths, line counts, severity]

### Code Style
[Findings with file paths, specific issues, severity]

### CLAUDE.md Health
[Missing/outdated sections, accuracy issues]

### Testing Health
[Coverage gaps, format issues, missing tests]

### Memory Health
[Size issues, stale content, missing links]

### Skills Sync
[Drift, missing tags, coverage gaps]

### Already Compliant
[What's in good shape -- acknowledge it]
```

**Important:** Report ALL findings in one list. Do not hide items in low-priority buckets or summarize away details. The operator reads the full report and decides what matters.

---

## Step 6: Clean Up and Offer Next Steps

Delete the scratch file (`audit-scratch.md`).

After presenting the report, offer:

> **Next steps (operator directs):**
> 1. Fix selected findings (scope a compliance phase)
> 2. Document intentional items in CLAUDE.md AI Architecture Notes
> 3. Defer -- add to project backlog
> 4. Some combination of the above

Wait for direction. Do not proceed without approval.
