---
name: remaining-work
description: Use when an implementation plan may be partially complete and you need to identify missing, untested, or in-progress items
---

# Remaining Work

## Overview

Report which plan items have item-specific code and test evidence in the current tree. Use diffs and commit logs to narrow the inspection; never treat changed-path presence as proof of completion.

**Announce at start:** "I'm using the remaining-work skill to check plan progress."

## Inputs

- Plan file path (required). If only a plan name is given, resolve `docs_repo` from CLAUDE.md when configured, then search that repository's existing `docs/plans/` or `plans/`; otherwise search the code repository.
- Diff base branch: if already cached for this code repository in this session, reuse it. Otherwise present a dynamic list ordered by likelihood:
  1. Working tree (if relevant paths have uncommitted changes)
  2. `@{upstream}` (if set)
  3. Parent branch (from `git reflog`: the branch HEAD was checked out from)
  4. Defaults that exist in the repo: `main`, `master`, `dev`
  5. Manual entry
- Cache the choice in session memory — do not write it to the plan file.

## Process

1. Parse the plan → extract all item/task numbers, descriptions, and file paths (both source and test files).
2. Collect candidate evidence from every current-tree layer:
   - `git diff <base>...HEAD --name-only` and `git log <base>...HEAD --oneline` for committed branch work.
   - `git diff --cached --name-only` for staged work.
   - `git diff --name-only` for unstaged work.
   - `git ls-files --others --exclude-standard` for untracked work.
3. For each item, inspect the current versions of only its declared source files, test files, interactions, and verification commands. A path match or item-numbered commit is a pointer to inspect, not completion evidence.
4. Establish evidence tied to this item's behavior:
   - **Code evidence:** the current source implements the behavior described by this item.
   - **Test evidence:** a current test exercises that same item-specific behavior and the item's declared verification command passes.
   - Shared files count only when the relevant behavior and test can be identified for this item.
5. Classify the item:
   - Behavior absent → missing.
   - Behavior present but item-specific test absent → untested.
   - Item-specific verification fails, or relevant working-tree changes are incomplete → in progress. This status applies whether the failing code is committed or uncommitted.
   - Behavior and item-specific test both present → covered. Annotate when evidence is uncommitted.
   - Declared paths or requirements are insufficient to decide, or verification fails outside this item's behavior → uncertain.

## Output

```markdown
## Remaining Work

Based on the current tree (narrowed by `<range>` plus working-tree changes):

### Items missing (no code evidence):
- **Item 3: Password hashing** — item-specific behavior absent from `src/auth/hasher.py`
- **Item 5: Session timeout** — declared source file does not exist

### Items untested (code present, test evidence missing):
- **Item 4: Login endpoint** — behavior present in `src/auth/login.py`; no test exercises its lockout requirement

### Items in progress:
- **Item 6: Logout endpoint** — staged implementation present; item-specific test is incomplete

### Items uncertain:
- **Item 7: Audit logging** — declared files do not identify where this behavior is implemented

### Items covered:
- Item 1: User model — model behavior implemented and exercised by `tests/auth/test_models.py`
- Item 2: Registration endpoint — registration behavior and its duplicate-email test are present in the shared auth files
```

## Constraints

- Single context window operation. Inspect only the files declared by each item; do not read the full codebase.
- Does not edit the plan or source files. It may run the item's declared verification command to establish test evidence.
- Coverage requires item-specific behavior and test evidence. Full implementation fidelity remains a code review concern (`requesting-code-review` / `code-review`).
- Absence from `<base>...HEAD` is not absence from the current tree. Code merged into the base and dirty working-tree work must still be classified from targeted inspection.

## Integration

- **REQUIRED SUB-SKILL:** None — this skill is a reporter, not an executor.
- **Used by:** `superpowers:executing-plans` and `superpowers:subagent-driven-development` call this internally before starting execution.
- **Related:** `superpowers:spec-to-plan` — produces the plans this skill checks.
