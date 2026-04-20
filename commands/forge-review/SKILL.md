---
description: Review changed code for reuse, quality, and efficiency using Sonnet subagents, then fix any issues found. Trigger on "forge review", "inspect", "review the code", or "clean it up".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /forge-review...**
>
> *Dispatching review subagents. They will report entropy and redundant logic.*

# /forge-review -- Sonnet Subagent Code Review

## Step 1: Identify Changed Files

Run `git diff --name-only HEAD` and `git diff --name-only --staged` to find all changed files. If no changes exist, check `git diff --name-only HEAD~1` for the most recent commit. Read all changed files to understand the full scope.

## Step 2: Launch Three Sonnet Subagents in Parallel

Use the Agent tool to launch **three subagents simultaneously** in a single message. Each MUST use `model: "sonnet"`. Each agent receives the list of changed files and instructions for its specific review focus.

### Agent 1 -- Reuse Check
> Review the changed code for duplication and reuse opportunities. Look for:
> - Repeated logic that could be consolidated into a shared function
> - Patterns that duplicate existing utilities in the codebase
> - Copy-paste code across files
> Also flag any pattern in this code that seems reusable across projects or worth extracting to a skill file. Report as [Pattern] with file:line.
> Report findings as a numbered list. If nothing found, say "No reuse issues found."

### Agent 2 -- Quality Check
> Review the changed code for quality issues. Look for:
> - Missing or inadequate error handling
> - Security concerns (injection, hardcoded secrets, unsafe operations)
> - Violations of language-specific best practices
> - Dead code, unused variables, unreachable branches
> - New source files missing a module-level header comment (purpose, dependencies, consumers)
> - Functions exceeding ~50 lines that could be split by responsibility
> - **Test coverage gaps:** For each changed source file with logic changes, check whether corresponding test changes exist. Flag files with new/modified logic but no test coverage.
> Report findings as a numbered list. If nothing found, say "No quality issues found."

### Agent 3 -- Efficiency Check
> Review the changed code for efficiency issues. Look for:
> - Unnecessary allocations or copies
> - O(n^2) patterns where O(n) is possible
> - Redundant operations (repeated lookups, unnecessary re-computation)
> - Overly complex logic that could be simplified
> Also flag any clever optimization or performance pattern worth preserving. Report as [Pattern] with file:line.
> Report findings as a numbered list. If nothing found, say "No efficiency issues found."

## Step 3: Categorize Findings

After all three agents return, categorize each finding:

| Category | Meaning | Action |
|----------|---------|--------|
| **FIX** | Real issue, worth fixing now | Edit the code directly |
| **DEFER** | Real issue, not worth fixing this phase | Record in `deferred.md` with reason |
| **DISMISS** | Not a real issue (false positive, intentional pattern) | Record one-line in `deferred.md` so future reviews don't re-raise |

## Step 4: Act

1. Present a consolidated summary with FIX/DEFER/DISMISS tags
2. Fix all FIX-category items directly
3. For DEFER and DISMISS items, append to `~/.claude/projects/<project-key>/memory/deferred.md` (create on first use)
4. After fixes, briefly list what was changed and what was deferred

Do NOT ask for permission to fix FIX items -- just fix. Deferrals are autonomous but visible in the summary -- the user can override.
