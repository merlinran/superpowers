---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

## Overview

Load plan, review critically, execute all tasks, report when complete.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

**Note:** Tell your human partner that Superpowers works much better with access to subagents. The quality of its work will be significantly higher if run on a platform with subagent support (Claude Code, Codex CLI, Codex App, and Copilot CLI all qualify; see the per-platform tool refs in `../using-superpowers/references/`). If subagents are available, use superpowers:subagent-driven-development instead of this skill.

## The Process

### Step 0: Check Remaining Work

Before executing any items, run the remaining-work process internally to identify which plan items still need implementation:

1. Parse the plan to extract item numbers and file paths.
2. Determine the diff base (session-cached, or present the dynamic list: working tree, upstream, parent branch, defaults, manual).
3. `git diff <base>...HEAD --name-only` and `git log <base>...HEAD --oneline`.
4. Identify items with no code or test evidence — these are the gaps.
5. Execute only the gap items. Skip items already covered.

### Step 1: Load and Review Plan
1. Read plan file. If only a plan name is given, search existing `docs/plans/` or `plans/`.
2. If the plan header includes **Source spec:** and **Architecture:** references, read those files for context before reviewing.
3. Review critically - identify any questions or concerns about the plan
4. If concerns: Raise them with your human partner before starting
5. If no concerns: Create todos for the plan items and proceed

### Step 2: Execute Tasks

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed
5. After committing each item, ensure the commit message references the plan item: `feat: implement <spec-name> item-N`

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

**Return to Review (Step 1) when:**
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
- **superpowers:remaining-work** - Identifies which plan items still need work
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
