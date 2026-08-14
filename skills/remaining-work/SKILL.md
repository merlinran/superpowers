---
name: remaining-work
description: Use when an implementation plan may be partially complete and you need to identify missing, untested, or in-progress tasks
---

# Remaining Work

## Overview

Report which plan tasks have task-specific code and test evidence in the current tree. Use diffs and commit logs to narrow the inspection; never treat changed-path presence as proof of completion.

**Announce at start:** "I'm using the remaining-work skill to check plan progress."

## Inputs

- Plan file path (required). If only a plan name is given, resolve `docs_repo` from AGENTS.md or CLAUDE.md when configured, then search that repository's existing `docs/plans/` or `plans/`; otherwise search the code repository.
- Task subset (optional). When a caller supplies task numbers for an incremental refresh, classify only those tasks and retain the caller's prior classifications for omitted tasks.
- Diff base branch: if already cached for this code repository in this session, reuse it. Otherwise present a dynamic list ordered by likelihood:
  1. Working tree (if relevant paths have uncommitted changes)
  2. `@{upstream}` (if set)
  3. Parent branch (from `git reflog`: the branch HEAD was checked out from)
  4. Defaults that exist in the repo: `main`, `master`, `dev`
  5. Manual entry
- Cache the choice in session memory — do not write it to the plan file.

## Process

1. Parse the plan → extract all task numbers, including legacy `Item N` headings, descriptions, and file paths (both source and test files). Always report the canonical `Task N` label. If the caller supplied a task subset, restrict inspection, verification, and output to that subset.
2. Collect candidate evidence from every current-tree layer:
   - `git diff <base>...HEAD --name-only` and `git log <base>...HEAD --oneline` for committed branch work.
   - `git diff --cached --name-only` for staged work.
   - `git diff --name-only` for unstaged work.
   - `git ls-files --others --exclude-standard` for untracked work.
3. For each task, inspect the current versions of only its declared source files, test files, interactions, and verification commands. A path match or task-numbered commit (`task-N`, or legacy `item-N`) is a pointer to inspect, not completion evidence.
4. Establish evidence tied to this task's behavior:
   - **Code evidence:** the current source implements the behavior described by this task.
   - **Test candidate:** a current test exercises that same task-specific behavior.
   - Shared files count only when the relevant behavior and test can be identified for this task.
5. For a task with code evidence and a test candidate, run its declared verification command. A passing command establishes test evidence. If the command cannot run, classify the task as uncertain; do not classify it as covered.
6. Classify the task:
   - Behavior absent → missing.
   - Behavior present but task-specific test absent → untested.
   - Task-specific verification fails, or relevant working-tree changes are incomplete → in progress. This status applies whether the failing code is committed or uncommitted.
   - Behavior present and task-specific verification passes → covered. Annotate when evidence is uncommitted.
   - Declared paths or requirements are insufficient to decide, the verification command cannot run, or verification fails outside this task's behavior → uncertain.

## Output

```markdown
## Remaining Work

Based on the current tree (narrowed by `<range>` plus working-tree changes):

### Tasks missing (no code evidence):
- **Task 3: Password hashing** — task-specific behavior absent from `src/auth/hasher.py`
- **Task 5: Session timeout** — declared source file does not exist

### Tasks untested (code present, test evidence missing):
- **Task 4: Login endpoint** — behavior present in `src/auth/login.py`; no test exercises its lockout requirement

### Tasks in progress:
- **Task 6: Logout endpoint** — staged implementation present; task-specific test is incomplete

### Tasks uncertain:
- **Task 7: Audit logging** — declared files do not identify where this behavior is implemented

### Tasks covered:
- Task 1: User model — model behavior implemented and verification passed via `tests/auth/test_models.py`
- Task 2: Registration endpoint — registration behavior and its duplicate-email verification passed in the shared auth files
```

## Constraints

- Single context window operation. Inspect only the files declared by each task; do not read the full codebase.
- Does not edit the plan or source files. It runs the declared verification command only for tasks that have both code evidence and a test candidate.
- Coverage requires task-specific behavior and a passing verification command. Full implementation fidelity remains a code review concern (`requesting-code-review` / `code-review`).
- Absence from `<base>...HEAD` is not absence from the current tree. Code merged into the base and dirty working-tree work must still be classified from targeted inspection.

## Integration

- **REQUIRED SUB-SKILL:** None — this skill is a reporter, not an executor.
- **Used by:** `superpowers:executing-plans` and `superpowers:subagent-driven-development` call this internally before starting execution.
- **Related:** `superpowers:spec-to-plan` — produces the plans this skill checks.
