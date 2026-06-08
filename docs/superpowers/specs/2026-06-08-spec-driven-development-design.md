# Spec-Driven Development Workflow

**Date:** 2026-06-08
**Status:** Draft

## Problem

The current superpowers workflow treats specs and plans as one-shot artifacts: brainstorming writes a dated design doc to `docs/superpowers/specs/`, writing-plans derives a dated implementation plan, and executing-plans/subagent-driven-development executes it once. After execution, both documents are dead — the spec is never revisited, the plan is never updated, and there is no mechanism to iterate.

This creates three failure modes:

1. **Spec rot** — the code drifts from the spec over time. The spec documents intent at a point in time, then becomes irrelevant.
2. **Full regeneration** — when a spec changes, the agent has no way to identify which plan items are affected. It either rewrites the whole plan (wasting tokens) or guesses (missing needed changes).
3. **No progress visibility** — after coding across multiple sessions, there is no way to ask "what plan items are still missing?" without re-reading the full codebase.

## Goals

1. **Living specs and architecture docs** — specs and architecture docs are refined iteratively rather than rewritten. They are the source of truth for what the system should do.
2. **Incremental plan updates** — when a spec changes, only affected plan items are regenerated. The plan references which spec sections it derives from.
3. **Diff-scoped gap analysis** — a skill that determines which plan items have no code evidence, using only git diff and commit logs, without full-codebase reads.
4. **Traceability without markers** — plan items list affected file paths. Commit messages reference plan items. No in-code markers required.

## Non-Goals

- Reverse code-to-spec feedback tooling. All behavior changes go through spec → plan → code.
- In-code traceability markers (e.g., `// Plan: foo#item-3`).
- Changes to the brainstorming collaborative dialogue process — only the output targets change.

## File Conventions

```
specs/                         # project root, overridable via CLAUDE.md
  user-auth.md                 # feature/user-journey spec
  checkout-flow.md
  api-rate-limiting.md
  arch/                        # architecture docs, grouped by system part
    data-layer.md
    api-design.md
    auth-model.md

plans/                         # project root
  user-auth.md                 # one plan per spec, stable name
  checkout-flow.md
```

- Specs are grouped by feature or user journey.
- Architecture docs are grouped by system part. No 1:1 relationship with specs — a spec may reference multiple arch docs, and an arch doc may serve multiple specs.
- Plans are one per spec with the same base filename.
- All three are living documents — refined in place, not rewritten. Git preserves history.
- No date prefixes on filenames. Dates are in git history.

### Cross-Repo Setups

If docs and code live in separate repos, point CLAUDE.md at the docs repo:

```markdown
docs_repo: ../docs-repo
```

Specs, plans, and arch docs are expected at `specs/`, `plans/`, and `specs/arch/` within that repo. Omit this setting and everything lives in the code repo.

### Spec Structure

Structured but freeform. Specs should include identifiable requirement sections that plan items can reference in their `Source:` header. No rigid schema is enforced, but reasonable sections include: Problem, Goals, Requirements, Constraints, Acceptance Criteria.

### Arch Doc Structure

Freeform. Covers technical decisions not suitable for specs: data models, API contracts, dependency graphs, infrastructure choices, security models. Architecture docs explain how the system is built; specs explain what the system should do.

## Skill Pipeline

```
brainstorming
  └→ writes/updates specs/*.md + specs/arch/*.md

spec-to-plan               (new, replaces writing-plans)
  └→ reads spec + arch docs → incremental plan update

executing-plans / subagent-driven-development   (modified)
  └→ runs remaining-work internally → executes only unimplemented items

remaining-work              (new, standalone reporter)
  └→ plan ∩ git diff → gap list
```

## New Skills

### spec-to-plan

**Purpose:** Keep the implementation plan current with the spec. If no plan exists yet, generates one from scratch. If a plan already exists, diffs the spec against the plan's source sections and regenerates only affected items. Unchanged items are preserved verbatim; removed spec sections drop their corresponding items.

**How it works:**
- Read the spec and any referenced architecture docs.
- If a plan already exists: diff the spec against the base branch (session-cached, same dynamic list as `remaining-work`) to find changed sections. Regenerate only items whose `Source:` header references those sections.
- If no plan exists: generate the full plan from the entire spec.
- Unchanged items are preserved verbatim. Removed spec sections drop their corresponding items.
- Each item includes a `Source:` header (which spec section it implements) and a `Files:` header (exact paths it touches).
- Save the plan.

**Plan item format:**
```markdown
### Item 3: Password hashing

**Source:** specs/user-auth.md > "Password Requirements"
**Files:**
- `src/auth/hasher.py`
- `tests/auth/test_hasher.py` (unit)
- `tests/integration/test_auth_flow.py` (integration)
**Interactions:** Session manager (`src/auth/session.py`), User model (`src/auth/models.py`)

- [ ] **Step 1: Write the failing unit test**
...
- [ ] **Step 4: Write integration test for password hashing in registration flow**
...
```

Integration tests are included when a plan item interacts with other parts of the system. The `Interactions:` header names the specific components this item depends on or feeds into, so the implementer knows what to integration-test against. `remaining-work` checks both unit and integration test files for test evidence.

**Plan header:**
```markdown
# User Auth Implementation Plan

**Source spec:** specs/user-auth.md
**Architecture:** specs/arch/data-layer.md, specs/arch/auth-model.md

> **For agentic workers:** Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan item-by-item.

---
```

**Constraints:**
- Follows the same plan-writing discipline as the original writing-plans skill: bite-sized tasks (2-5 min per step), TDD, exact code in steps, no placeholders, self-review checklist after writing.
- For each plan item, identifies interactions with other system components and includes integration tests where those interactions exist.
- Emits an announcement at start: "I'm using the spec-to-plan skill to create/update the implementation plan."

### remaining-work

**Purpose:** Report which plan items have no code or test evidence in the current diff, without full-codebase reads.

**Inputs:**
- Plan file path
- Diff range: asks once per session using a dynamic list (below). Cached in memory for the session — no file churn.

**First run per session**, presents options ordered by likelihood:

1. Working tree (if relevant paths have uncommitted changes)
2. `@{upstream}` (if set)
3. Parent branch (from `git reflog`: the branch HEAD was checked out from)
4. Defaults that exist in the repo: `main`, `master`, `dev`
5. Manual entry

**Process:**
1. Parse the plan → extract all item numbers, descriptions, and file paths (both source and test files).
2. `git diff <range> --name-only` → set of changed files.
3. `git log <range> --oneline` → commit messages.
4. For each plan item:
   - Does any changed file match this item's source file paths? → code evidence.
   - Does any changed file match this item's test file paths? → test evidence.
   - Does any commit message reference this item number? → commit evidence.
5. Items with no code evidence → reported as missing.
6. Items with code evidence but no test evidence → reported as untested.
7. Items with both code and test evidence → reported as covered.

**Output:**
```markdown
## Remaining Work

Based on `main...HEAD`:

### Items missing (no code evidence):
- **Item 3: Password hashing** — `src/auth/hasher.py` not found in diff
- **Item 5: Session timeout** — `src/auth/session.py` not found in diff

### Items untested (code present, test evidence missing):
- **Item 4: Login endpoint** — `src/auth/login.py` found, but `tests/auth/test_login.py` not found in diff

### Items covered:
- Item 1: User model — code + tests matched via `src/auth/models.py`, `tests/auth/test_models.py`
- Item 2: Registration endpoint — code + tests matched via commit "feat: implement user-auth item-2"
```

**Constraints:**
- Single context window operation. No full-codebase reads.
- Default diff range is `main...HEAD`. The user can override.
- Does not modify anything — purely a reporter.
- Emits an announcement at start: "I'm using the remaining-work skill to check plan progress."

**Coverage vs. fidelity:** `remaining-work` checks *presence* — files exist, commits reference items. It does not check *correctness*. Fidelity belongs to code review (`requesting-code-review` / `code-review`), which reads the actual code and verifies it matches the plan's intent. These are complementary steps in the pipeline: remaining-work says "these items still need code," code review says "the code for these items is correct."

### remaining-work as Subroutine

`executing-plans` and `subagent-driven-development` call `remaining-work` internally before starting execution. They read the gap list and execute only the unimplemented items. This means the human does not need to manually run `remaining-work` before each execution session — executing-plans does it automatically.

`remaining-work` remains available as a standalone reporter for ad-hoc progress checks.

### spec-to-plan as Subroutine

`brainstorming` may call `spec-to-plan` after updating a spec, if the human wants the plan regenerated immediately. This is optional — the human can also run `spec-to-plan` manually later.

## Modified Skills

### brainstorming

**Changes:**
- Output target changes from `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` to `specs/<feature-name>.md` and `specs/arch/<system-part>.md`.
- If a spec file already exists at the target path, the skill refines it rather than creating a new file.
- If an architecture doc already exists, the skill updates it when tech decisions emerge.
- Transition to implementation: invoke `spec-to-plan` instead of `writing-plans`.
- Design doc writing step: save to `specs/` instead of `docs/superpowers/specs/`.
- Self-review step: unchanged process, different file location.

**Unchanged:**
- Collaborative dialogue process (one question at a time, explore context, propose approaches).
- Output is still a markdown design document — just saved to a different location.

### executing-plans

**Changes:**
- Before executing any items, run `remaining-work` internally to identify unimplemented items.
- Execute only items with no evidence — skip items that already have code.
- Commit messages reference plan item identifiers (e.g., `feat: implement user-auth item-3`).
- If the plan header specifies `Source spec:` and `Architecture:`, refer to those files for context.

**Unchanged:**
- Bite-sized step execution with TodoWrite tracking.
- Stop conditions (blocker, test failure, unclear instruction, repeated verification failure).
- `superpowers:finishing-a-development-branch` at completion.
- Review checkpoints.

### subagent-driven-development

**Changes:**
- Commit messages reference plan item identifiers (same convention as executing-plans).
- Before dispatching implementers, run `remaining-work` internally to identify which items need work.

**Unchanged:**
- Fresh subagent per task.
- Two-stage review (spec review, code quality review).
- Implementer prompt template.

## Removed Skills

### writing-plans

Superseded by `spec-to-plan`. All planning is now spec-driven. The byte-sized task discipline, TDD, exact code in steps, no-placeholders rule, and self-review checklist are preserved in `spec-to-plan`.

## Design Decisions

### Why diff-scoping instead of full-codebase comparison

`git diff <base>...HEAD --name-only` is free, deterministic, and narrows the problem from "check 30 plan items against 200 files" to "check 2-3 items against maybe 10 changed files." This keeps gap analysis affordable in a single context window. The base branch is detected dynamically per session (upstream, parent branch, defaults, or manual) rather than hardcoded to `main`.

### Why per-session base branch caching

The base branch (what `main...HEAD` compares against) varies per session — different feature branches have different parents and upstreams. Storing it in the plan file causes churn. Asking once and caching in session memory avoids file noise while keeping the ask lightweight (the dynamic list makes it a one-keystroke choice).

### Why no checksums

Checksums would catch the case where a spec changed but the plan wasn't regenerated. But `spec-to-plan` already uses git diff to detect spec changes — the mechanism is built into the update loop, not an external guard. No separate checksum machinery needed.

### Why no in-code traceability markers

Plan items list the file paths they touch. Commit messages reference plan item numbers. The intersection of these two signals is sufficient for gap detection without adding comments to source code. In-code markers would be technically precise but impose a maintenance burden and pollute code with metadata unrelated to its function.

### Why strict spec-first

All behavior changes go through spec → plan → code. This prevents spec-code drift by design. Bug fixes and refactors that don't change behavior don't need a spec. No reverse code-to-spec feedback tooling.

### Why coverage vs. fidelity split

`remaining-work` checks *presence* — do files exist and commits reference plan items? It does not check *correctness* — does the code actually match what the plan item asks for? Fidelity is a code review concern (`requesting-code-review` / `code-review`), which reads the actual code and verifies it against the plan. Splitting these keeps `remaining-work` cheap (single context window, no full-code reads) and keeps code review focused (it only reviews items that claim coverage).

### Why stable plan filenames

Plans are living artifacts derived from specs — one plan per spec, updated in place. Dated filenames made sense for one-shot plans but conflict with incremental updates. Git preserves history; the filename reflects what it builds, not when it was written.

### Why no separate "refine spec" step

The brainstorming skill already produces specs through collaborative dialogue. Changing its output target from `docs/superpowers/specs/` to `specs/` and adding refinement behavior (updating existing files rather than creating new ones) is sufficient. A separate skill for spec refinement would add ceremony without new capability.

### Why docs and code can live in separate repos

Specs, plans, and arch docs form one coherent policy layer. Code is implementation. They may reasonably live in different repos (docs are policy, code is execution). One line in CLAUDE.md (`docs_repo: ../docs-repo`) is enough to configure this. Specs, plans, and arch are always together in the docs repo — there is no reason to split them from each other.
