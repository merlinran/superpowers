---
name: remaining-work
description: Use when you have an implementation plan and want to check which plan items still need code — based on git diff, not full-codebase reads
---

# Remaining Work

## Overview

Report which plan items have code and test evidence in the current diff, without reading the full codebase. Uses `git diff` and commit logs as the narrowing mechanism.

**Announce at start:** "I'm using the remaining-work skill to check plan progress."

## Inputs

- Plan file path (required).
- Diff base branch: if already cached in this session, reuse it. Otherwise present a dynamic list ordered by likelihood:
  1. Working tree (if relevant paths have uncommitted changes)
  2. `@{upstream}` (if set)
  3. Parent branch (from `git reflog`: the branch HEAD was checked out from)
  4. Defaults that exist in the repo: `main`, `master`, `dev`
  5. Manual entry
- Cache the choice in session memory — do not write it to the plan file.

## Process

1. Parse the plan → extract all item numbers, descriptions, and file paths (both source and test files).
2. `git diff <range> --name-only` → set of changed files.
3. `git log <range> --oneline` → commit messages.
4. For each plan item:
   - Does any changed file match this item's source file paths? → code evidence.
   - Does any changed file match this item's test file paths? → test evidence.
   - Does any commit message reference this item number (e.g., "item-3", "item 3")? → commit evidence.
5. Items with no code evidence → missing.
6. Items with code evidence but no test evidence → untested.
7. Items with both code and test evidence → covered.

## Output

```markdown
## Remaining Work

Based on `<range>`:

### Items missing (no code evidence):
- **Item 3: Password hashing** — `src/auth/hasher.py` not found in diff
- **Item 5: Session timeout** — `src/auth/session.py` not found in diff

### Items untested (code present, test evidence missing):
- **Item 4: Login endpoint** — `src/auth/login.py` found, but `tests/auth/test_login.py` not found in diff

### Items covered:
- Item 1: User model — code + tests matched via `src/auth/models.py`, `tests/auth/test_models.py`
- Item 2: Registration endpoint — code + tests matched via commit "feat: implement user-auth item-2"
```

## Constraints

- Single context window operation. Never read the full codebase for gap analysis.
- Does not modify anything — purely a reporter.
- Coverage checking is about *presence*, not *fidelity*. Fidelity is a code review concern (`requesting-code-review` / `code-review`).

## Integration

- **REQUIRED SUB-SKILL:** None — this skill is a reporter, not an executor.
- **Used by:** `superpowers:executing-plans` and `superpowers:subagent-driven-development` call this internally before starting execution.
- **Related:** `superpowers:spec-to-plan` — produces the plans this skill checks.
