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
> Report findings as a numbered list. If nothing found, say "No reuse issues found."

### Agent 2 -- Quality Check
> Review the changed code for quality issues. Look for:
> - Missing or inadequate error handling
> - Security concerns (injection, hardcoded secrets, unsafe operations)
> - Violations of language-specific best practices
> - Dead code, unused variables, unreachable branches
> - New source files missing a module-level header comment (purpose, dependencies, consumers)
> - Functions exceeding ~50 lines that could be split by responsibility
> Report findings as a numbered list. If nothing found, say "No quality issues found."

### Agent 3 -- Efficiency Check
> Review the changed code for efficiency issues. Look for:
> - Unnecessary allocations or copies
> - O(n^2) patterns where O(n) is possible
> - Redundant operations (repeated lookups, unnecessary re-computation)
> - Overly complex logic that could be simplified
> Report findings as a numbered list. If nothing found, say "No efficiency issues found."

## Step 3: Synthesize and Act

After all three agents return:

1. Present a consolidated summary of all findings, grouped by category (Reuse / Quality / Efficiency)
2. If issues were found, fix them -- edit the files directly
3. After fixes, briefly list what was changed

Do NOT ask for permission to fix -- just fix. The user invoked /forge-review expecting action, not a report.
