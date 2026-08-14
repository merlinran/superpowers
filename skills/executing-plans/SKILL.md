---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

## Overview

Load plan, review critically, execute all tasks, report when complete.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

**Note:** Tell your human partner that Superpowers works much better with access to subagents (Claude Code, Codex CLI, Codex App, Copilot CLI, and Gemini CLI all qualify; see the per-platform tool refs in `../using-superpowers/references/`). If subagents are available, use superpowers:subagent-driven-development instead of this skill.

## The Process

### Step 0: Load and Review Plan

1. Ensure an isolated workspace: use superpowers:using-git-worktrees to create one or verify the existing one.
2. Read the plan file. If only a plan name is given, resolve `docs_repo` from AGENTS.md or CLAUDE.md when configured, then search that repository's existing `docs/plans/` or `plans/`; otherwise search the code repository.
3. Resolve the plan's **Source repository:** relative to the code repository root. Read **Source spec:** and **Architecture:** paths relative to that repository before reviewing.
4. Review critically - identify any questions or concerns about the plan
5. If concerns: Raise them with your human partner and wait for resolution; then restart Step 0.
6. If no concerns: proceed to remaining-work analysis

### Step 1: Check Remaining Work

After the plan review is clear, run the remaining-work process internally:

1. Parse the plan to extract task numbers, requirements, file paths, and verification commands.
2. Determine the diff base (session-cached, or present the dynamic list: working tree, upstream, parent branch, defaults, manual).
3. Apply the `remaining-work` evidence process, including committed, staged, unstaged, untracked, and targeted current-tree evidence.
4. Create todos for missing, untested, and in-progress tasks. Stop and ask about uncertain tasks. Omit covered tasks.

### Step 2: Execute Tasks

For each todo:
1. Mark as in_progress
2. For a missing task, follow all plan steps exactly.
3. For an untested task, preserve existing behavior and execute the missing test and verification steps; change implementation only if the test exposes a defect.
4. For an in-progress task, resume at the first incomplete or failing step.
5. Run verifications as specified.
6. Commit the verified task with a message that references it: `feat: implement <spec-name> task-N`.
7. Mark as completed.

### Step 3: Complete Development

After all tasks complete and verified:
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
- Follow that skill to verify tests, present options, execute choice

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly

**Ask for clarification rather than guessing.**

## When to Revisit Earlier Steps

**Return to Review (Step 0) when:**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking

**Don't force through blockers** - stop and ask.

## Remember
- Review plan critically first
- Follow plan steps exactly
- Don't skip verifications
- Reference skills when plan says to
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
- **superpowers:spec-to-plan** - Creates the plan this skill executes
- **superpowers:remaining-work** - Identifies which plan tasks still need work
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
