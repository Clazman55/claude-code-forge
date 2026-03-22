---
description: Project wrap — satellite audit, canonical skill sync, cold storage move, final commit. Trigger on "project wrap", "wrap the project", or "close out".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /wrap...**
>
> *Project complete. Committing to cold storage.*

# /wrap — Project Wrap

Use when the project is truly complete — no more planned phases. This permanently closes the project and moves it to cold storage.

**Key difference from /stabilize:** Wrap closes permanently and moves to cold storage. Stabilize shelves for later reactivation.

**When to wrap vs stabilize:**
- **Wrap** — project is done. No more planned work.
- **Stabilize** — more work is planned (future tiers, features, ports).

## Steps

### 1. Final /forge-review Pass
Run `/forge-review` on the last phase's changes. Fix all real findings.

### 2. Satellite Audit
Read ALL satellite files (patterns.md, systems.md, etc.) cover to cover. For each entry, ask: "Now that the full project is complete, is this generalizable?" Patterns that looked project-specific early on may be clearly universal in hindsight. Promote generalizable patterns to project skills (`docs/`).

### 3. Components Audit
Check `~/.claude/skills/components-index.md`. Did this project adapt any indexed component? If the adaptation is more general or more robust than the version the index currently points to, update the index pointer to this project's version. Did this project create any new reusable component not yet indexed? If so, add it. Direction is always upward -- projects improve implementations, the index follows.

### 4. Canonical Sync
Sync project skills (`docs/`) to canonical skills (`~/.claude/skills/`). This is the ONE time canonical skills get updated for a completed project.

Rules for syncing:
- Strip project-specific names and paths
- Generalize code examples
- Add `[verified in Phase N]` tags
- Do NOT remove existing content from canonical skills — only add or update

### 5. Create Meta-Skills If Discovered
If the project revealed new workflow patterns (like how project-lifecycle.md was created from the Snake project), create new skill files.

### 6. Update Skills Index
Add any new skills to `~/.claude/skills/index.md` with descriptions and dependencies.

### 7. Update Global CLAUDE.md
If workflow improvements were discovered during this project, update `~/.claude/CLAUDE.md`.

### 8. Update Home Directory Memory — Cold Storage Move
In `~/.claude/projects/<home-key>/memory/MEMORY.md`:
- Move the full project entry from the Active Projects section to `completed-projects.md` in the same memory folder
- Replace with a one-line index entry under Completed Projects:
  ```
  - **ProjectName** (v1.0.0) — One-line description. `E:\ProjectName`
  ```

### 9. Update Project MEMORY.md and Satellites
Mark all phases complete. Record:
- Final line count
- Feature summary
- Test count

Satellite files (systems.md, patterns.md) are preserved — they're needed for bug fixes if the project is ever reactivated.

### 10. Final Commit
Commit all changes with message: `Project wrap: [project name]`

Run `git status` after commit to verify clean state.
