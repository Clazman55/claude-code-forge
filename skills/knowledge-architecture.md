# Knowledge Architecture — Session Continuity for Amnesiac Agents

Patterns for maintaining project knowledge across sessions where each agent instance starts
from cold storage with no prior context. Addresses the fundamental constraint: useful context
exceeds what fits in a single auto-loaded file.

Companion to `project-lifecycle.md` (workflow) and project-specific CLAUDE.md files (instructions).
This skill covers the *knowledge layer* that bridges sessions — what to store, where to store it,
and how content migrates between layers as a project matures.

---

## 1. The Three-Layer Model

### Layer 1: Skills (Portable, Generalized)

Reusable patterns extracted from implementation experience. No project-specific names,
state, or configuration values. Useful across projects of the same type.

```
Location:  ~/.claude/skills/*.md          (canonical)
           E:\ProjectName\docs/*.md       (project copies)
Audience:  The craft — any future project of this type
Lifecycle: Accumulates across phases. Project copies updated per-phase.
           Canonical copies synced at project wrap or on major pattern discovery.
```

**Content examples:**
- "Conditional Render Guard (Upstream Change Flag)" with generic code
- "Tier-Separated Multiplier Chains" with `TIER1_IDS`, `TIER2_IDS` placeholders
- Anti-patterns with explanations of *why* they fail

**Format rule:** When a skill section is pure reference data (lookup tables, checklists, phrase lists), use a YAML code block within the markdown file. Prose explanations stay as prose.

**Not skills content:**
- `INFRA_IDS` caches patron building keys (too specific)
- Save version 11 adds engineer defaults (too specific)

### Layer 2: Memory Satellites (Project-Specific Reference)

Implementation details for a specific project. Variable names, function signatures,
save version tables, subsystem architecture, per-phase notes.

```
Location:  ~/.claude/projects/[project-key]/memory/*.md
           (e.g., systems.md, patterns.md, decisions.md)
Audience:  The project — future sessions working on this codebase
Lifecycle: Grows with each phase. Organized by topic, not chronologically.
           Consulted on-demand when working on specific subsystems.
```

**Content examples:**
- "`getInfraMultiplier()` = `1 + bonus * getEngineerEffectMult()`"
- Save version migration table with phase-by-phase additions
- Production chain order with actual variable names
- Per-subsystem DOM cache patterns (`civilCardRefs[]`, `lastCivil{}`, `civilInitialized`)

**Satellite file organization:**
- `systems.md` — Per-phase subsystem details, function inventory, state shapes
- `patterns.md` — Project-verified coding patterns and bug patterns to avoid
- `decisions.md` — Architectural decisions with rationale (optional, for complex projects)
- `testing-patterns.md` — Root cause analysis from `/test-audit`, failure categories, test coverage gaps
- Topic files as needed — whatever logical grouping serves the project

### Layer 3: MEMORY.md (Live Index / Routing Table)

The only file auto-loaded into every session (200-line hard limit). Must orient a fresh
session in seconds: where are we, what's the plan, where's the detail.

```
Location:  ~/.claude/projects/[project-key]/memory/MEMORY.md
Audience:  The session — what does a fresh session need immediately?
Lifecycle: Updated every phase. Actively trimmed to stay under 200 lines.
           Detailed content migrates to satellites when space runs low.
```

**Content examples:**
- "Phase 7c complete. Next: Phase 7d (troop XP/campaigns)"
- "See [systems.md](systems.md) for subsystem details"
- "Current save version: 11"
- Quick-reference summaries (production chain, key ID caches, DOM patterns)

---

## 2. Content Flow Between Layers

```
Canonical Skills ──copy──► Project Skills (docs/)
                              │
                    Phase 1..N builds knowledge
                              │
                         ┌────┴────┐
                         │ Project │
                         │ Skills  │  ← patterns extracted from fresh work
                         ├─────────┤
                         │Satellites│  ← project-specific detail
                         ├─────────┤
                         │MEMORY.md │  ← routing table
                         └────┬────┘
                              │
                    Wrap: audit + sync
                              │
                    ┌─────────┴──────────┐
                    │                    │
              Canonical Skills     Project "brain"
              (updated)            (preserved)
```

One-way flow during implementation. Canonical skills are copied INTO the project at start
and not touched again until project wrap. The project builds its own knowledge base
through implementation phases. At wrap, the refined product is exported back to canonical.

### Migration triggers

| Trigger | Action |
|---------|--------|
| MEMORY.md exceeds ~180 lines | Move per-phase/per-subsystem detail to satellites |
| Phase complete (self-verification) | Update MEMORY.md status, add satellite notes, extract patterns to project skills (`docs/`) |
| Bug pattern caught by /forge-review | Add to satellite patterns.md AND project skills anti-pattern section |
| Project wrap — satellite audit | Read all satellites, reassess with full-project hindsight, promote generalizable patterns to project skills |
| Project wrap — canonical sync | Sync project skills (`docs/`) → canonical (`~/.claude/skills/`), generalize, archive project memory |

---

## 3. Knowledge Operations

Two distinct operations at different lifecycle points. Canonical skills are NOT
touched during implementation — the project builds its own knowledge base and
exports refined patterns at wrap.

### 3a. Per-Phase: Pattern Extraction

Part of the self-verification loop (step 9 in `project-lifecycle.md`). After each
phase's `/forge-review` pass, extract patterns from the work just completed.

**Process:**

1. **Identify delta**: What changed in this phase? (`git diff` from last commit)
2. **Read current project skills** (`docs/`): Know what's already documented.
3. **For each new technique in the delta:**
   - Add to **project skills** (`docs/*.md`) with `[verified in Phase N]` tag
   - Project-specific details (variable names, function signatures) go to **satellites**
4. **Anti-patterns are worth recording**: Bug patterns go to satellite `patterns.md`
5. **Do NOT touch canonical skills** (`~/.claude/skills/`) — that's a wrap-time operation

**What goes where:**

| Content | Destination |
|---------|-------------|
| Generalizable pattern (even if not yet generalized) | Project skills (`docs/`) |
| Project-specific implementation detail | Satellites (`memory/patterns.md`, `systems.md`) |
| Bug pattern / anti-pattern | Satellites (`memory/patterns.md`) |
| Status update, phase progress | MEMORY.md |

### 3b. Project Wrap: Satellite Audit + Canonical Sync

At project wrap, the accumulated project knowledge gets a full retrospective review.
This is the ONLY time canonical skills are updated. See `project-lifecycle.md` Section 12.

**Satellite Audit:**

1. **Read ALL satellite files** cover to cover (patterns.md, systems.md, etc.)
2. **For each entry**, ask: "Now that the full project is complete, is this generalizable?"
3. Patterns that looked project-specific in early phases may be clearly universal in hindsight
4. **Promote** generalizable patterns from satellites → project skills (`docs/`)
5. Anti-patterns that would help in a different project → project skills

**Canonical Sync:**

1. **Read project skills** (`docs/`) — these now contain everything battle-tested
2. **Read canonical skills** (`~/.claude/skills/`) — know what already exists
3. **For each project pattern not in canonical**: apply generalization rules (below), add to canonical
4. **For existing canonical patterns**: update `[verified]` tags with this project's evidence
5. **Update `~/.claude/skills/index.md`** if new skill files or major sections were added

### Generalization Rules (for canonical sync)

- Replace project entity names with generic placeholders (`aquaeductus` → `TIER2_IDS`)
- Remove config specifics (keep the pattern shape, not the numbers)
- Write code examples that compile/run in isolation
- Include the "why" — what goes wrong without this pattern

---

## 4. Decision Rules: What Goes Where

| Signal | Layer |
|--------|-------|
| Would help in a different project | Skills |
| Needed to work on this project next session | Satellite |
| Fresh session needs this in first 30 seconds | MEMORY.md |
| Contains a variable/function name from this project | Satellite |
| Describes a reusable technique with generic names | Skills |
| Current status, phase, plan location | MEMORY.md |
| How a subsystem's save migration works | Satellite |
| The *pattern* for save migration chains | Skills |

---

## 5. MEMORY.md Structure Template

```markdown
# [Project] — Project Memory

## Status
- Phase X complete, next: Phase Y
- Test counts per phase
- Plan file location

## Satellite Files
- [systems.md](systems.md) — subsystem details, save versions, chain order
- [patterns.md](patterns.md) — verified patterns and bug avoidance
- [other.md](other.md) — as needed

## Architecture Overview
- Key structural facts (line count, file layout, section numbering)
- Core conventions (dirty flags, render ownership, cache patterns)

## Quick Reference
- ID caches, key function names, current save version
- Production/computation chain summary (one line)

## Active Design Decisions
- Decisions relevant to upcoming phases
- Link to plan file for full detail

## Review Findings (per phase)
- Phase N: [what /forge-review caught and what was fixed]
- Prevents reintroducing bugs across phases

## Verified Patterns (quick ref)
- See patterns.md for full list. Top 5-6 most-used patterns here.

## Bug Patterns (quick ref)
- See patterns.md for full list. Top 5-6 most critical here.
```

Target: 60-120 lines for an active mid-project state. Under 60 for early phases,
approaching 180 only if many subsystems are simultaneously active.

---

## 6. Satellite File Templates

### systems.md — Subsystem Reference

```markdown
# [Project] — Subsystem Reference

## [Subsystem A] (Phase N)
- Key state variables and where they live
- Function inventory (helpers, tick functions, render functions)
- DOM cache pattern for this subsystem
- Save version and migration notes

## [Subsystem B] (Phase M)
- ...

## Save Version Summary
| Version | Phase | Migration |
|---------|-------|-----------|

## [Derived Chain / Computation Order]
1. Step one
2. Step two
...
```

### patterns.md — Verified Patterns

```markdown
# [Project] — Verified Patterns

## Core Architecture
- [pattern]: [why it works, when verified]

## Cost & Economy
- ...

## DOM & Render
- ...

## State Management
- ...

## Bug Patterns to Avoid
- [anti-pattern]: [symptom] → [fix]
```

---

## 7. When the Model Emerges

This architecture is not needed from Phase 0. It emerges naturally:

| Project Stage | Knowledge State |
|--------------|----------------|
| Phase 0-2 | MEMORY.md only. Everything fits in 200 lines. |
| Phase 3-5 | MEMORY.md getting long. Create `patterns.md` for verified patterns. |
| Phase 5-7 | Per-phase subsystem sections consume MEMORY.md. Create `systems.md`, restructure MEMORY.md as index. |
| Phase 7+ | Three-layer model fully active. Skill sweeps happen per-phase. |

Don't pre-create satellite files for a 3-phase project. Let the pressure of the 200-line
limit drive the split organically.

---

## 8. Relationship to Other Skills

- **`project-lifecycle.md`** defines the workflow; this skill defines the knowledge layer
  that persists across workflow phases. Step 9 (skills sync) and Step 10 (MEMORY.md update)
  in the self-verification loop are where this architecture is actively maintained.
- **`CLAUDE.md`** is the project *instruction* file (checked into the repo). It tells the
  agent what rules to follow. Memory files tell the agent what happened and where things are.
- **Home directory memory** (`~/.claude/projects/<home-key>/memory/MEMORY.md`) is the
  *cross-project* index. It bootstraps sessions by pointing to the right project, plan file,
  and skills. It does NOT contain project-specific implementation details. Completed projects
  move to `completed-projects.md` (Cold Storage Pattern — see `project-lifecycle.md` Section 11).
  Project memory and satellites are preserved for reactivation.

---

## 9. Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| MEMORY.md over 200 lines | Content silently truncated, later sections never loaded | Trim aggressively, move detail to satellites |
| Satellite content duplicated in MEMORY.md | Wasted line budget, drift risk | MEMORY.md should contain *pointers* and *quick refs*, not full detail |
| Skills too project-specific | Future project gets confused by entity names | Strip all project names during generalization |
| Satellite organized chronologically | Hard to find info for a specific subsystem | Organize by topic/system, not by phase |
| Skill sweep skipped for multiple phases | Large batch of patterns to extract at once | At minimum, flag "skill sweep needed since Phase X" in MEMORY.md |
| Patterns in satellite but not in skills | Wisdom trapped in one project, not reusable | Sweep should ask "would this help elsewhere?" for each satellite entry |
| MEMORY.md quick-refs drift from satellites | Inconsistent information across layers | Single source of truth is the satellite; MEMORY.md refs are summaries only |
