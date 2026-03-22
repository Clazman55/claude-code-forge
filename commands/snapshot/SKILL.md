---
description: Session snapshot — write structured state file to disk, then present compact message. Belt and suspenders for session continuity. Trigger on "snapshot", "save state", or "freeze session".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /snapshot...**
>
> *Committing session state to persistent storage before compaction.*

# /snapshot -- Session State Capture

Writes a structured session-state.md to the project's memory folder, then presents a compact message. The state file persists on disk regardless of compaction quality. If the compact message loses detail, the next session reads the file directly.

## Step 1: Detect Memory Folder

- **Home session** (`~` or `~/.claude`): write to `~/.claude/projects/<home-key>/memory/session-state.md`
- **Project session** (`E:\ProjectName`): find this project's memory folder in `~/.claude/projects/` and write `session-state.md` there

## Step 2: Gather State

Collect the following from the current session. Do not re-read files -- use what is already in context.

```yaml
state_fields:
  - id: identity
    content: "Session type (home / project / brainstorming) and current role"
  - id: completed
    content: "Work completed this session with commit hashes. Be specific -- file names, not categories."
  - id: next_task
    content: "Current or next task with enough detail to resume without re-reading everything"
  - id: decisions
    content: "Active decisions, pending questions, unresolved issues. Omit if none."
  - id: key_files
    content: "File paths relevant to the current work -- what the next session should read first"
  - id: repo_state
    content: "Clean or dirty. If dirty, which files are modified and why."
```

## Step 3: Write State File

Write `session-state.md` with this format:

```markdown
# Session State -- [date]

## Identity
[Session type and role]

## Completed
- [Work item with commit hash]
- [Work item with commit hash]

## Next Task
[What to do next, with enough context to start immediately]

## Open Issues
[Decisions, questions, blockers. "None" if clean.]

## Key Files
- [path] -- [why it matters right now]
- [path] -- [why it matters right now]

## Repo State
[Clean/dirty, modified files, reason]
```

### Rules

- Overwrite the previous session-state.md. This is current state, not a log.
- Be terse. This is a handoff brief, not a narrative. Each field should be 1-3 lines.
- Commit hashes are mandatory for completed work. If work is uncommitted, say so.
- The next task field is the most important. A fresh instance should be able to read it and start working without running /orient.

## Step 4: Present Compact Message

After writing the file, generate and present the compact message following the format in CLAUDE.md:

- Session identity and current role
- Work completed with commit hashes
- Current/next task with detail to resume
- Active decisions, pending questions, unresolved issues
- Key file paths
- Repo state

Then tell the user: "State file written. Run `/compact` when ready."

Do NOT automatically trigger compaction. The operator decides when to compact.
