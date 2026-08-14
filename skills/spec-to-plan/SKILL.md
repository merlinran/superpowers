---
name: spec-to-plan
description: Use when a spec needs a new implementation plan or an existing plan must be reconciled after spec or architecture changes
---

# Spec-to-Plan

## Overview

Read a spec and its referenced architecture docs, then generate or incrementally update a numbered implementation plan. Each plan task maps to a spec section and lists exact file paths.

**Announce at start:** "I'm using the spec-to-plan skill to create/update the implementation plan."

**Spec path lookup:** Search existing `docs/specs/` or `specs/`.

**Save plans to:** existing `docs/plans/` or `plans/`; otherwise create `plans/`.
- If AGENTS.md or CLAUDE.md sets `docs_repo`, plans are saved relative to that repo instead.

## How It Works

1. Resolve the repository containing the spec. A relative `docs_repo` is relative to the code repository root. Run all source revision and diff commands in the resolved docs repository; never reuse a code-repository diff base.
2. Require the spec and referenced architecture files to be tracked and committed. Express every path relative to the docs repository, then run `git -C <docs-repository> ls-files --error-unmatch -- <spec-file> <architecture-files>` and `git -C <docs-repository> status --short -- <spec-file> <architecture-files>`. If either command fails or the status command prints anything, stop and ask your human partner to add and commit those files. Do not commit them yourself or create/update the plan. Then read the committed files and record each file's last-modifying commit from `git -C <docs-repository> log -1 --format=%H -- <source-file>` as its source revision. Do not use repository `HEAD`: a plan-only commit must not invalidate unchanged sources.
3. If a plan already exists there:
   - Read the source revisions recorded in its header.
   - If revisions are unchanged, preserve the plan verbatim.
   - Otherwise, in the docs repository, diff each recorded revision against the current revision for its source file to find changed spec and architecture sections.
   - Map every changed spec or architecture section to affected plan tasks. Existing `Sources:` references are the primary mapping; for a new section, identify the spec requirement or component it affects, then update or create the corresponding tasks and attach the new source reference.
   - Preserve untouched tasks verbatim.
   - Remove tasks whose source sections were deleted.
   - If the plan has no source revisions, reconcile every task against the current spec and architecture once; preserve an existing task only after confirming its sources and requirements are current.
4. If no plan exists: generate the full plan from the entire spec.
5. Save the plan with current source revisions.

## Plan Document Format

**Plan header:**
```markdown
# [Feature Name] Implementation Plan

**Source spec:** specs/<feature>.md
**Architecture:** specs/arch/<system-part>.md, ...
**Source repository:** `<docs_repo value, or .>`
**Source revisions:**
- `specs/<feature>.md`: `<docs-repository-commit-sha>`
- `specs/arch/<system-part>.md`: `<docs-repository-commit-sha>`

> **For agentic workers:** Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

---
```

**Plan task format:**
```markdown
### Task N: [Component Name]

**Sources:**
- Spec: specs/<feature>.md > "Section Name"
- Architecture: specs/arch/<system-part>.md > "API Contract"
**Files:**
- `src/path/to/file.py`
- `tests/path/to/test_file.py` (unit)
- `tests/integration/path/to/test_integration.py` (integration)
**Interactions:** Component A (`src/other/file.py`), Component B (`src/other/file2.py`)

- [ ] **Step 1: Write the failing unit test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test_file.py::test_name -v`
Expected: FAIL

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test_file.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Write integration test**

```python
def test_integration_with_component_a():
    result = system_flow(input)
    assert component_a_behavior(result)
```

- [ ] **Step 6: Run integration tests**

Run: `pytest tests/integration/path/test_integration.py -v`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add src/path/file.py tests/path/test_file.py tests/integration/path/test_integration.py
git commit -m "feat: implement <spec-name> task-N"
```
```

## Plan Writing Discipline

- **Bite-sized tasks:** Each step is one action (2-5 minutes).
- **TDD:** Write the failing test first, then implementation.
- **Exact code:** Every code step shows complete code, not descriptions.
- **No placeholders:** Never write "TBD", "TODO", "implement later", "add error handling", "write tests for the above".
- **Integration awareness:** For each task, identify interacting components and include integration tests where those interactions exist.
- **Exact file paths always.**
- **Exact source references always.** Every task names each applicable spec and architecture section so either kind of change can invalidate it.
- **DRY, YAGNI, frequent commits.**

## Self-Review

After writing the complete plan:

1. **Spec coverage:** Skim each section in the spec. Can you point to a plan task that implements it? List any gaps.
2. **Architecture coverage:** Skim each architecture section with implementation impact. Can you point to every affected plan task through its `Sources:` references? Add missing references or tasks.
3. **Placeholder scan:** Search for red flags — "TBD", "TODO", "implement later", "add error handling", "write tests". Fix them.
4. **Type consistency:** Do the types, signatures, and names in later tasks match earlier tasks? Fix any mismatches.

If you find issues, fix them inline. If you find a spec requirement with no task, add the task.

## Human Review and Commit

After self-review:

1. Save the plan, then tell your human partner: **"Plan drafted at `<plan-path>`. Please review it; I will incorporate your changes and commit the approved plan before offering execution options."**
2. Stop and wait. Do not offer an execution method or start implementation before explicit plan approval.
3. If the human requests changes, update the plan, repeat Self-Review, and ask for review again.
4. After explicit approval, inspect the plan path in the plan's repository. If it is untracked or differs from its committed version, run `git add <plan-file>`, then commit only that path with `git commit --only <plan-file> -m "docs: update <feature> implementation plan"`; verify that new commit contains the approved plan and no unrelated paths. If the plan is already tracked and unchanged, do not create an empty commit; verify it matches the committed copy and use `git log -1 --format=%H -- <plan-file>` to identify its existing commit.

## Execution Handoff

Only after the approved plan commit is verified: **"Plan reviewed and committed as `<commit>`. Two execution options: 1. Subagent-Driven (recommended) or 2. Inline Execution. Which approach?"**

**If Subagent-Driven:** Use superpowers:subagent-driven-development
**If Inline:** Use superpowers:executing-plans

## Integration

- **REQUIRED SUB-SKILL:** None — this skill produces plans for executing-plans or subagent-driven-development to consume.
- **Related:** `superpowers:remaining-work` — checks plan progress after implementation.
