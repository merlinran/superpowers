# Spec-Driven Development — Implementation Plan

**Source spec:** docs/superpowers/specs/2026-06-08-spec-driven-development-design.md

> **For agentic workers:** Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

---

### Task 1: Create spec-to-plan skill

**Files:**
- `skills/spec-to-plan/SKILL.md`

**Source:** docs/superpowers/specs/2026-06-08-spec-driven-development-design.md > "spec-to-plan"

- [ ] **Step 1: Write the spec-to-plan skill file**

```markdown
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
```

- [ ] **Step 2: Verify the SKILL.md has valid frontmatter**

Run: `head -5 skills/spec-to-plan/SKILL.md`
Expected: frontmatter block with `name: spec-to-plan` and a `description:` starting with "Use when..."

- [ ] **Step 3: Commit**

```bash
git add skills/spec-to-plan/SKILL.md
git commit -m "feat: add spec-to-plan skill"
```

---

### Task 2: Create remaining-work skill

**Files:**
- `skills/remaining-work/SKILL.md`

**Source:** docs/superpowers/specs/2026-06-08-spec-driven-development-design.md > "remaining-work"

- [ ] **Step 1: Write the remaining-work skill file**

```markdown
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
```

- [ ] **Step 2: Verify the SKILL.md has valid frontmatter**

Run: `head -5 skills/remaining-work/SKILL.md`
Expected: frontmatter block with `name: remaining-work` and a `description:` starting with "Use when..."

- [ ] **Step 3: Commit**

```bash
git add skills/remaining-work/SKILL.md
git commit -m "feat: add remaining-work skill"
```

---

### Task 3: Modify brainstorming skill

**Files:**
- `skills/brainstorming/SKILL.md`

**Source:** docs/superpowers/specs/2026-06-08-spec-driven-development-design.md > "brainstorming"

- [ ] **Step 1: Change the spec output path in the Checklist section**

Find the line:
```
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
```

Replace with:
```
6. **Write design doc** — save to `specs/<feature-name>.md` and `specs/arch/<system-part>.md` as appropriate. If a spec already exists at the target path, refine it rather than creating a new file. Commit.
```

- [ ] **Step 2: Change the transition step in the Checklist**

Find the line:
```
9. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

Replace with:
```
9. **Transition to implementation** — invoke spec-to-plan skill to create implementation plan
```

- [ ] **Step 3: Update the dot graph**

Find:
```
"Invoke writing-plans skill" [shape=doublecircle];
```

Replace with:
```
"Invoke spec-to-plan skill" [shape=doublecircle];
```

- [ ] **Step 4: Update the terminal state rule**

Find:
```
**The terminal state is invoking writing-plans.** Do NOT invoke frontend-design, mcp-builder, or any other implementation skill. The ONLY skill you invoke after brainstorming is writing-plans.
```

Replace with:
```
**The terminal state is invoking spec-to-plan.** Do NOT invoke frontend-design, mcp-builder, or any other implementation skill. The ONLY skill you invoke after brainstorming is spec-to-plan.
```

- [ ] **Step 5: Update the Documentation section**

Find:
```
- Write the validated design (spec) to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
  - (User preferences for spec location override this default)
```

Replace with:
```
- Write the validated design to `specs/<feature-name>.md` for behavioral specs, and update or create `specs/arch/<system-part>.md` for architecture decisions that emerged.
  - If a spec or arch doc already exists at the target path, refine it in place rather than creating a new file.
  - If CLAUDE.md sets `docs_repo`, specs are saved relative to that repo instead of the code repo.
```

- [ ] **Step 6: Update the Implementation section**

Find:
```
- Invoke the writing-plans skill to create a detailed implementation plan
- Do NOT invoke any other skill. writing-plans is the next step.
```

Replace with:
```
- Invoke the spec-to-plan skill to create a detailed implementation plan
- Do NOT invoke any other skill. spec-to-plan is the next step.
```

- [ ] **Step 7: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: update brainstorming to output living specs and transition to spec-to-plan"
```

---

### Task 4: Modify executing-plans skill

**Files:**
- `skills/executing-plans/SKILL.md`

**Source:** docs/superpowers/specs/2026-06-08-spec-driven-development-design.md > "executing-plans"

- [ ] **Step 1: Add remaining-work subroutine before execution**

Find the line `### Step 1: Load and Review Plan`. Insert a new step before it:

```markdown
### Step 0: Check Remaining Work

Before executing any items, run the remaining-work skill internally to identify which plan items still need implementation:

1. Read the plan file to find the **Source spec:** and **Architecture:** references in the header.
2. Use the remaining-work process (diff intersection) to produce a gap list — items missing or untested.
3. Execute only the gap items. Skip items already covered.
```

Step 0 runs before the existing Step 1. No renumbering needed — the existing Steps 1-3 stay as-is.

- [ ] **Step 2: Add commit message convention**

Find the line `### Step 2: Execute Tasks`. After the bullet "For each task:", add a new bullet:

```markdown
   - After committing each item, ensure the commit message references the plan item: `feat: implement <spec-name> item-N`
```

- [ ] **Step 3: Add spec/arch context loading to review step**

Find `### Step 1: Load and Review Plan` and add as step 3 within it:

```markdown
3. If the plan header includes **Source spec:** and **Architecture:** references, read those files for context before reviewing.
```

- [ ] **Step 4: Update Integration section**

Find the line `- **superpowers:writing-plans** - Creates the plan this skill executes` and replace with:

```markdown
- **superpowers:spec-to-plan** - Creates the plan this skill executes
- **superpowers:remaining-work** - Identifies which plan items still need work
```

- [ ] **Step 5: Commit**

```bash
git add skills/executing-plans/SKILL.md
git commit -m "feat: add remaining-work subroutine and commit convention to executing-plans"
```

---

### Task 5: Modify subagent-driven-development skill

**Files:**
- `skills/subagent-driven-development/SKILL.md`
- `skills/subagent-driven-development/implementer-prompt.md`

**Source:** docs/superpowers/specs/2026-06-08-spec-driven-development-design.md > "subagent-driven-development"

- [ ] **Step 1: Add remaining-work subroutine to subagent-driven-development**

In `skills/subagent-driven-development/SKILL.md`, find the line "Read plan, extract all tasks with full text, note context, create TodoWrite". Insert a new step before it in the dot graph and process description:

After the section `## The Process`, before the dot graph, add:

```markdown
**Before dispatching any implementers**, run the remaining-work process internally: parse the plan, intersect with `git diff`, and identify which items are missing or untested. Dispatch implementers only for gap items. Skip items already covered.
```

- [ ] **Step 2: Add commit message convention**

In `skills/subagent-driven-development/SKILL.md`, find the section `## Red Flags`. Add to the "Never:" list:

```markdown
- Commit without referencing the plan item number (e.g., "feat: implement <spec> item-N")
```

- [ ] **Step 3: Update Integration section**

Find the line `- **superpowers:writing-plans** - Creates the plan this skill executes` and replace with:

```markdown
- **superpowers:spec-to-plan** - Creates the plan this skill executes
- **superpowers:remaining-work** - Identifies which plan items still need work
```

- [ ] **Step 4: Update implementer prompt template**

In `skills/subagent-driven-development/implementer-prompt.md`, find the commit message convention and update it. If no explicit commit convention exists, add to the prompt's instructions:

```
- Commit messages MUST reference the plan item: `feat: implement <spec-name> item-N`
```

Let me read the implementer prompt first to find the exact location.

- [ ] **Step 5: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md skills/subagent-driven-development/implementer-prompt.md
git commit -m "feat: add remaining-work subroutine and commit convention to subagent-driven-development"
```

---

### Task 6: Remove writing-plans skill

**Files:**
- `skills/writing-plans/SKILL.md` (remove)
- `skills/writing-plans/plan-document-reviewer-prompt.md` (remove)

**Source:** docs/superpowers/specs/2026-06-08-spec-driven-development-design.md > "Removed Skills: writing-plans"

- [ ] **Step 1: Remove writing-plans skill directory**

```bash
git rm -r skills/writing-plans/
```

Expected: removes `SKILL.md` and `plan-document-reviewer-prompt.md`

- [ ] **Step 2: Verify no dangling references to writing-plans**

```bash
grep -r "writing-plans" skills/ --include="*.md" | grep -v "replaces writing-plans" | grep -v "superseded by" || echo "No dangling references found"
```

Expected: Only references in the new spec-to-plan and remaining-work skills (which mention it in historical context), or no output at all in other skills.

- [ ] **Step 3: Commit**

```bash
git commit -m "feat: remove writing-plans skill, superseded by spec-to-plan"
```

---
