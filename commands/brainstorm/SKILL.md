---
description: Brainstorming session — read ideas folder, survey accumulated work, enter open-ended exploration mode. Trigger on "brainstorm", "let's think about", "explore an idea", or "cogitate".
user-invocable: true
---

**MANDATORY FIRST OUTPUT -- say this before doing anything else, before any tool calls:**

> **Running /brainstorm...**
>
> *Session orientation. Project sessions build things within a defined scope. This session holds the full picture: every completed project, every idea in the pipeline, every pattern extracted from prior work. When the operator wants to think, this is the right mode. I will read everything before I say anything.*

# /brainstorm — Brainstorming Session

Enter brainstorming mode. This reads all context needed to have an informed, open-ended exploration of a topic.

## Step 1: Read Context

1. Read `~/.claude/CLAUDE.md` — global instructions and working relationship
2. Read `~/.claude/projects/<home-key>/memory/MEMORY.md` — project history, user profile, completed and active projects, ideas pipeline
3. Read `~/.claude/projects/<home-key>/memory/user_domain_knowledge.md` — domain knowledge map, project selection patterns, calibration guidance. Informs how to present options and which domains need explanation vs shorthand.
4. Scan `~/.claude/ideas/` — list all idea files with a one-line summary of each
5. Check `~/.claude/skills/index.md` — know what skills exist and what domains are covered
6. Check `~/.claude/skills/components-index.md` — know what implementations already exist. "We could build X" should become "We have X in [project], adapt or build fresh?" when applicable.

## Step 2: Focus the Session

If the user provided a topic with the command (e.g., `/brainstorm backup tool port`):
- Find any existing idea files related to that topic
- Read them fully
- Identify which skills, completed projects, or active work relates to the topic

If no topic was provided:
- Present the full ideas pipeline with status
- Ask what the user wants to explore

## Step 3: Engage

Once context is loaded and the topic is identified:

1. Confirm what you found — existing ideas, related projects, applicable skills
2. Ask your first substantive question about the topic — not a generic "what would you like to explore?" but a specific question informed by what you just read
3. Brainstorming mode means open-ended thinking. Don't jump to solutions. Explore the problem space. Challenge assumptions. Present options with trade-offs.

## Rules

- This is a home directory session activity. If invoked in a project session, note it but proceed anyway — the user knows what they're doing.
- Do NOT create files unless explicitly asked. Brainstorming is thinking, not building.
- If the brainstorm produces something worth capturing, offer to create or update an idea file at the end.
- Steelman before challenging. Don't be agreeable for the sake of usefulness.
