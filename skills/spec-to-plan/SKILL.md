---
name: spec-to-plan
description: Use when you have a spec file and need to generate or update an implementation plan — derive numbered plan items from specs, incremental updates on spec changes
---

# Spec-to-Plan

## Overview

Read a spec and its referenced architecture docs, then generate or incrementally update a numbered implementation plan. Each plan item maps to a spec section and lists exact file paths.

**Announce at start:** "I'm using the spec-to-plan skill to create/update the implementation plan."

**Save plans to:** `plans/<spec-base-name>.md`
- If CLAUDE.md sets `docs_repo`, plans are saved relative to that repo instead.

## How It Works

1. Read the spec file and any architecture docs referenced in the spec.
2. If a plan already exists at `plans/<spec-base-name>.md`:
   - Determine the diff base (reuse session-cached branch from `remaining-work`, or present the same dynamic list: working tree, upstream, parent branch, defaults, manual).
   - `git diff <base>...HEAD -- <spec-file> <arch-docs>` to find changed sections.
   - Regenerate only plan items whose `Source:` header references changed sections.
   - Preserve untouched items verbatim.
   - Remove items whose source sections were deleted.
3. If no plan exists: generate the full plan from the entire spec.
4. Save the plan.

## Plan Document Format

**Plan header:**
```markdown
# [Feature Name] Implementation Plan

**Source spec:** specs/<feature>.md
**Architecture:** specs/arch/<system-part>.md, ...

> **For agentic workers:** Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan item-by-item. Steps use checkbox (`- [ ]`) syntax for tracking.

---
```

**Plan item format:**
```markdown
### Item N: [Component Name]

**Source:** specs/<feature>.md > "Section Name"
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
git commit -m "feat: implement <spec> item-N"
```
```

## Plan Writing Discipline

- **Bite-sized tasks:** Each step is one action (2-5 minutes).
- **TDD:** Write the failing test first, then implementation.
- **Exact code:** Every code step shows complete code, not descriptions.
- **No placeholders:** Never write "TBD", "TODO", "implement later", "add error handling", "write tests for the above".
- **Integration awareness:** For each item, identify interacting components and include integration tests where those interactions exist.
- **Exact file paths always.**
- **DRY, YAGNI, frequent commits.**

## Self-Review

After writing the complete plan:

1. **Spec coverage:** Skim each section in the spec. Can you point to a plan item that implements it? List any gaps.
2. **Placeholder scan:** Search for red flags — "TBD", "TODO", "implement later", "add error handling", "write tests". Fix them.
3. **Type consistency:** Do the types, signatures, and names in later items match earlier items? Fix any mismatches.

If you find issues, fix them inline. If you find a spec requirement with no item, add the item.

## Execution Handoff

After saving the plan: **"Plan saved to `plans/<filename>.md`. Two execution options: 1. Subagent-Driven (recommended) or 2. Inline Execution. Which approach?"**

**If Subagent-Driven:** Use superpowers:subagent-driven-development
**If Inline:** Use superpowers:executing-plans

## Integration

- **REQUIRED SUB-SKILL:** None — this skill produces plans for executing-plans or subagent-driven-development to consume.
- **Related:** `superpowers:remaining-work` — checks plan progress after implementation.
