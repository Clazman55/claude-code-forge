---
description: Stabilize workflow — shelf a project at a natural pause point with full knowledge capture. Trigger on "stabilize" or "shelf".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /stabilize...**
>
> *Shelving the project. Capturing session state before suspension.*

# /stabilize — Shelf a Project

Use when a project reaches a natural pause point (MVP complete, milestone achieved) but is NOT finished — more phases are planned for later. This is NOT a project wrap. The project stays in the active section of home directory memory.

**Key difference from /wrap:** Stabilize shelves for later reactivation. Wrap closes permanently and moves to cold storage.

**When to stabilize vs wrap:**
- **Stabilize** — more work is planned (future tiers, features, ports). Knowledge capture now, continue later.
- **Wrap** — project is done. No more planned work.

## Steps

### 1. Final /forge-review Pass
Run `/forge-review` on the last phase's changes. Fix all real findings.

### 2. Satellite Audit
Read ALL satellite files (patterns.md, systems.md, etc.) cover to cover. For each entry, ask: "Is this generalizable?" Patterns that looked project-specific early on may be clearly universal in hindsight. Promote generalizable patterns to project skills (`docs/`).

### 3. Components Audit
Check `~/.claude/skills/components-index.md`. Did this project adapt any indexed component? If the adaptation is more general or more robust than the version the index currently points to, update the index pointer to this project's version. Did this project create any new reusable component not yet indexed? If so, add it. Direction is always upward -- projects improve implementations, the index follows.

### 4. Canonical Sync
Sync project skills (`docs/`) to canonical skills (`~/.claude/skills/`). This is one of the two times canonical skills get updated (the other is project wrap).

Rules for syncing:
- Strip project-specific names and paths
- Generalize code examples
- Add `[verified in Phase N]` tags
- Do NOT remove existing content from canonical skills — only add or update

### 5. Create Meta-Skills If Discovered
If the project revealed new workflow patterns that aren't captured in existing skills, create new skill files.

### 6. Update Skills Index
Add any new skills to `~/.claude/skills/index.md` with descriptions and dependencies.

### 7. Update Project MEMORY.md
Mark status as "Stabilized" (not wrapped). Record:
- Completed phases
- Test counts
- Final line counts
- What remains for future work

Satellite files are preserved for reactivation — do NOT delete them.

### 8. Update Home Directory Memory
Update the project entry status to "Stabilized" in home directory MEMORY.md. Do NOT move to cold storage — project stays in the active section.

### 9. Commit
Commit all changes with message: `Stabilize: [project name] — [milestone summary]`

## After Stabilize

The project is shelved. A future session can reactivate it by reading the preserved MEMORY.md and satellites. The home directory memory still has the full bootstrap entry.
