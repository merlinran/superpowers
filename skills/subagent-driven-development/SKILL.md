---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute plan by dispatching a fresh implementer subagent per task, a task review (spec compliance + code quality) after each, and a broad whole-branch review at the end.

**Why subagents:** You delegate tasks to specialized agents with isolated context. By precisely crafting their instructions and context, you ensure they stay focused and succeed at their task. They should never inherit your session's context or history — you construct exactly what they need. This also preserves your own context for coordination work.

**Core principle:** Fresh subagent per task + task review (spec + quality) + broad final review = high quality, fast iteration

**Narration:** between tool calls, narrate at most one short line — the
ledger and the tool results carry the record.

**Continuous execution:** Do not pause to check in with your human partner between tasks. Execute all tasks from the plan without stopping. The only reasons to stop are: BLOCKED status you cannot resolve, ambiguity that genuinely prevents progress, or all tasks complete. "Should I continue?" prompts and progress summaries waste their time — they asked you to execute the plan, so execute it.

## When to Use

```dot
digraph when_to_use {
    "Have implementation plan?" [shape=diamond];
    "Tasks mostly independent?" [shape=diamond];
    "Stay in this session?" [shape=diamond];
    "subagent-driven-development" [shape=box];
    "executing-plans" [shape=box];
    "Manual execution or brainstorm first" [shape=box];

    "Have implementation plan?" -> "Tasks mostly independent?" [label="yes"];
    "Have implementation plan?" -> "Manual execution or brainstorm first" [label="no"];
    "Tasks mostly independent?" -> "Stay in this session?" [label="yes"];
    "Tasks mostly independent?" -> "Manual execution or brainstorm first" [label="no - tightly coupled"];
    "Stay in this session?" -> "subagent-driven-development" [label="yes"];
    "Stay in this session?" -> "executing-plans" [label="no - parallel session"];
}
```

**vs. Executing Plans (parallel session):**
- Same session (no context switch)
- Fresh subagent per task (no context pollution)
- Review after each task (spec compliance + code quality), broad review at the end
- Faster iteration (no human-in-loop between tasks)

## The Process

### Startup and Dispatch Checks

At skill start:

1. Record the code repository root from the current worktree. Run `scripts/sdd-workspace` and every later SDD helper (`scripts/task-brief`, `scripts/review-package`) from that root, and retain the returned workspace path. Do not change the helpers' working directory when the plan lives in `docs_repo`.
2. Resolve the plan. If only a name is given, resolve `docs_repo` from CLAUDE.md when configured and search that repository's existing `docs/plans/` or `plans/`; otherwise search the code repository.
3. Resolve the plan's source and architecture context through its `Source repository` header.
4. Compute the plan revision with `git hash-object <plan-file>` and validate the Durable Progress ledger.
5. Run `remaining-work` once and create todos from its result:
   - Missing → dispatch the full task.
   - Untested → preserve existing behavior and dispatch only missing test and verification work unless the test exposes a defect.
   - In progress → resume and verify the existing work.
   - Uncertain → stop and ask.
   - Covered → if its evidence is committed, ensure the ledger records `Task N: covered (verified at <head7>, remaining-work verification passed)`, then skip. If evidence is uncommitted, create a verify-and-commit todo: preserve the implementation, rerun its declared verification, and commit only the task-owned changes. If ownership or scope is uncertain, stop and ask. Never dispatch final review while uncommitted covered evidence remains unsettled.

Before every implementer, fixer, task-reviewer, or final-reviewer dispatch, recompute the plan revision. If it changed, invalidate the ledger and all plan-derived dispatch artifacts, rerun `remaining-work`, and regenerate those artifacts before dispatch.

When the plan revision is unchanged, do not rerun `remaining-work` before fixer, task-reviewer, re-reviewer, or final-reviewer dispatches. A task is settled only when its ledger entry records either an approved-review `<head7>` (clean or Minor findings pending) or a committed-evidence `verified at <head7>` baseline. Before deciding whether another implementation task remains, refresh (a) every unsettled task and (b) each settled task whose declared source, test, or interaction paths appear in `git diff <baseline>..HEAD --name-only`, staged, unstaged, or untracked changes. Retain every other settled task's prior classification and ledger entry, and do not rerun its verification command. After refreshing a selected task, replace its prior task entry with one current classification and baseline; committed covered evidence advances to `verified at <current-head7>`, while uncommitted evidence leaves it unsettled. This catches shared-file effects without repeatedly verifying unrelated completed work.

```dot
digraph process {
    rankdir=TB;

    subgraph cluster_per_task {
        label="Per Task";
        "Dispatch implementer subagent (./implementer-prompt.md)" [shape=box];
        "Implementer subagent asks questions?" [shape=diamond];
        "Answer questions, provide context" [shape=box];
        "Implementer subagent implements, tests, commits, self-reviews" [shape=box];
        "Write diff file, dispatch task reviewer subagent (./task-reviewer-prompt.md)" [shape=box];
        "Record every review finding in progress ledger" [shape=box];
        "Exactly one valid review verdict that matches findings?" [shape=diamond];
        "Re-dispatch reviewer for corrected verdict" [shape=box];
        "Spec requirements resolved and no Critical/Important findings?" [shape=diamond];
        "Dispatch fix subagent for spec gaps and Critical/Important findings" [shape=box];
        "Mark task complete in todo list and progress ledger" [shape=box];
    }

    "Read plan, validate ledger revision, classify tasks, create todos" [shape=box];
    "Refresh affected and unsettled tasks; rebuild task todos" [shape=box];
    "More tasks remain?" [shape=diamond];
    "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" [shape=box];
    "All pending findings explicitly dispositioned?" [shape=diamond];
    "Re-dispatch final reviewer for explicit dispositions" [shape=box];
    "Final review clean?" [shape=diamond];
    "Dispatch one final fix subagent; re-review" [shape=box];
    "Use superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];

    "Read plan, validate ledger revision, classify tasks, create todos" -> "More tasks remain?";
    "Dispatch implementer subagent (./implementer-prompt.md)" -> "Implementer subagent asks questions?";
    "Implementer subagent asks questions?" -> "Answer questions, provide context" [label="yes"];
    "Answer questions, provide context" -> "Dispatch implementer subagent (./implementer-prompt.md)";
    "Implementer subagent asks questions?" -> "Implementer subagent implements, tests, commits, self-reviews" [label="no"];
    "Implementer subagent implements, tests, commits, self-reviews" -> "Write diff file, dispatch task reviewer subagent (./task-reviewer-prompt.md)";
    "Write diff file, dispatch task reviewer subagent (./task-reviewer-prompt.md)" -> "Record every review finding in progress ledger";
    "Record every review finding in progress ledger" -> "Exactly one valid review verdict that matches findings?";
    "Exactly one valid review verdict that matches findings?" -> "Re-dispatch reviewer for corrected verdict" [label="no"];
    "Re-dispatch reviewer for corrected verdict" -> "Record every review finding in progress ledger";
    "Exactly one valid review verdict that matches findings?" -> "Spec requirements resolved and no Critical/Important findings?" [label="yes"];
    "Spec requirements resolved and no Critical/Important findings?" -> "Dispatch fix subagent for spec gaps and Critical/Important findings" [label="no"];
    "Dispatch fix subagent for spec gaps and Critical/Important findings" -> "Write diff file, dispatch task reviewer subagent (./task-reviewer-prompt.md)" [label="re-review"];
    "Spec requirements resolved and no Critical/Important findings?" -> "Mark task complete in todo list and progress ledger" [label="yes"];
    "Mark task complete in todo list and progress ledger" -> "Refresh affected and unsettled tasks; rebuild task todos";
    "Refresh affected and unsettled tasks; rebuild task todos" -> "More tasks remain?";
    "More tasks remain?" -> "Dispatch implementer subagent (./implementer-prompt.md)" [label="yes"];
    "More tasks remain?" -> "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" [label="no"];
    "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" -> "All pending findings explicitly dispositioned?";
    "All pending findings explicitly dispositioned?" -> "Re-dispatch final reviewer for explicit dispositions" [label="no"];
    "Re-dispatch final reviewer for explicit dispositions" -> "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)";
    "All pending findings explicitly dispositioned?" -> "Final review clean?" [label="yes"];
    "Final review clean?" -> "Dispatch one final fix subagent; re-review" [label="no"];
    "Dispatch one final fix subagent; re-review" -> "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)";
    "Final review clean?" -> "Use superpowers:finishing-a-development-branch" [label="yes"];
}
```

## Pre-Flight Plan Review

Before dispatching Task 1, scan the plan once for conflicts:

- tasks that contradict each other or the plan's Global Constraints
- anything the plan explicitly mandates that the review rubric treats as a
  defect (a test that asserts nothing, verbatim duplication of a logic block)

Present everything you find to your human partner as one batched question —
each finding beside the plan text that mandates it, asking which governs —
before execution begins, not one interrupt per discovery mid-plan. If the
scan is clean, proceed without comment. The review loop remains the net for
conflicts that only emerge from implementation.

## Model Selection

Use the least powerful model that can handle each role to conserve cost and increase speed.

**Mechanical implementation tasks** (isolated functions, clear specs, 1-2 files): use a fast, cheap model. Most implementation tasks are mechanical when the plan is well-specified.

**Integration and judgment tasks** (multi-file coordination, pattern matching, debugging): use a standard model.

**Architecture and design tasks**: use the most capable available model.
The final whole-branch review is one of these — dispatch it on the most
capable available model, not the session default.

**Review tasks**: choose the model with the same judgment, scaled to the
diff's size, complexity, and risk. A small mechanical diff does not need the
most capable model; a subtle concurrency change does.

**Always specify the model explicitly when dispatching a subagent.** An
omitted model inherits your session's model — often the most capable and
most expensive — which silently defeats this section.

**Turn count beats token price.** Wall-clock and context cost scale with how
many turns a subagent takes, and the cheapest models routinely take 2-3× the
turns on multi-step work — costing more overall. Use a mid-tier model as the
floor for reviewers and for implementers working from prose descriptions.
When the task's plan text contains the complete code to write, the
implementation is transcription plus testing: use the cheapest tier for
that implementer. Single-file mechanical fixes also take the cheapest tier.

**Task complexity signals (implementation tasks):**
- Touches 1-2 files with a complete spec → cheap model
- Touches multiple files with integration concerns → standard model
- Requires design judgment or broad codebase understanding → most capable model

## Handling Implementer Status

Implementer subagents report one of four statuses. Handle each appropriately:

**DONE:** Generate the review package by invoking `<skill-directory>/scripts/review-package BASE HEAD` while the code repository root remains the working directory. It prints the unique file path it wrote; BASE is the commit you recorded before dispatching the implementer — never `HEAD~1`, which silently drops all but the last commit of a multi-commit task. Then dispatch the task reviewer with the printed path.

**DONE_WITH_CONCERNS:** The implementer completed the work but flagged doubts. Read the concerns before proceeding. If the concerns are about correctness or scope, address them before review. If they're observations (e.g., "this file is getting large"), note them and proceed to review.

**NEEDS_CONTEXT:** The implementer needs information that wasn't provided. Provide the missing context and re-dispatch.

**BLOCKED:** The implementer cannot complete the task. Assess the blocker:
1. If it's a context problem, provide more context and re-dispatch with the same model
2. If the task requires more reasoning, re-dispatch with a more capable model
3. If the task is too large, break it into smaller pieces
4. If the plan itself is wrong, escalate to the human

**Never** ignore an escalation or force the same model to retry without changes. If the implementer said it's stuck, something needs to change.

## Handling Reviewer ⚠️ Items

The task reviewer may report "⚠️ Cannot verify from diff" items — requirements
that live in unchanged code or span tasks. These do not block the rest of the
review, but you must resolve each one yourself before marking the task
complete: you hold the plan and cross-task context the reviewer
lacks. If you confirm an item is a real gap, treat it as a failed spec
review — send it back to the implementer and re-review.

## Constructing Reviewer Prompts

Per-task reviews are task-scoped gates. The broad review happens once, at the
final whole-branch review. When you fill a reviewer template:

- Do not add open-ended directives like "check all uses" or "run race tests
  if useful" without a concrete, task-specific reason
- Do not ask a reviewer to re-run tests the implementer already ran on the
  same code — the implementer's report carries the test evidence
- Do not pre-judge findings for the reviewer — never instruct a reviewer to
  ignore or not flag a specific issue. If you believe a finding would be a
  false positive, let the reviewer raise it and adjudicate it in the review
  loop. If the prompt you are writing contains "do not flag," "don't treat X
  as a defect," "at most Minor," or "the plan chose" — stop: you are
  pre-judging, usually to spare yourself a review loop.
- The global-constraints block you hand the reviewer is its attention
  lens. Copy the binding requirements verbatim from the plan's Global
  Constraints section or the spec: exact values, exact formats, and the
  stated relationships between components ("same layout as X", "matches
  Y"). The reviewer's template already carries the process rules (YAGNI,
  test hygiene, review method) — the constraints block is for what THIS
  project's spec demands.
- Hand the reviewer its diff as a file: run this skill's
  `scripts/review-package BASE HEAD` and pass the reviewer the file path
  it prints (or, without bash: `git log --oneline`, `git diff --stat`,
  and `git diff -U10` for the range, redirected to one uniquely named
  file). The output never enters your own context, and the reviewer sees
  the commit list, stat summary, and full diff with context in one Read
  call. Use the BASE you recorded before dispatching the implementer —
  never `HEAD~1`, which silently truncates multi-commit tasks.
- A dispatch prompt describes one task, not the session's history. Do not
  paste accumulated prior-task summaries ("state after Tasks 1-3") into
  later dispatches — a real session's dispatch hit 42k chars of which 99%
  was pasted history. A fresh subagent needs its task, the interfaces it
  touches, and the global constraints. Nothing else.
- In the same controller turn that receives a task review, record every finding
  in the progress ledger before any fixer, reviewer, or other dispatch. Require
  exactly one `Task quality` line with an allowed value that matches the spec
  verdict and finding severities. If it is missing, duplicated, malformed, or
  inconsistent, re-dispatch the reviewer for a corrected verdict before deciding task status.
  Dispatch fix subagents for spec gaps and Critical or Important findings.
  Leave Minor findings open for final whole-branch triage. Use the Durable
  Progress finding format.
- A finding labeled plan-mandated — or any finding that conflicts with
  what the plan's text requires — is the human's decision, like any plan
  contradiction: present the finding and the plan text, ask which governs.
  Do not dismiss the finding because the plan mandates it, and do not
  dispatch a fix that contradicts the plan without asking.
- The final whole-branch review gets a package too: run
  `scripts/review-package MERGE_BASE HEAD` (MERGE_BASE = the commit the
  branch started from, e.g. `git merge-base main HEAD`) and include the
  printed path in the final review dispatch, so the final reviewer reads
  one file instead of re-deriving the branch diff with git commands.
- Every fix dispatch carries the implementer contract: the fix subagent
  re-runs the tests covering its change and reports the results. Name the
  covering test files in the dispatch — a one-line fix does not need the
  whole suite. Before re-dispatching the reviewer, confirm the fix report
  contains the covering tests, the command run, and the output; dispatch
  the re-review once all three are present.
- If the final whole-branch review returns findings, dispatch ONE fix
  subagent with the complete findings list — not one fixer per finding.
  Per-finding fixers each rebuild context and re-run suites; a real
  session's final-review fix wave cost more than all its tasks combined.

## File Handoffs

Everything you paste into a dispatch prompt — and everything a subagent
prints back — stays resident in your context for the rest of the session
and is re-read on every later turn. Hand artifacts over as files:

- **Task brief:** before dispatching an implementer, run this skill's
  `scripts/task-brief PLAN_FILE N` — it extracts the task's full text to a
  uniquely named file and prints the path. Compose the dispatch so the
  brief stays the single source of requirements. Your dispatch should
  contain: (1) one line on where this task fits in the project; (2) the
  brief path, introduced as "read this first — it is your requirements,
  with the exact values to use verbatim"; (3) interfaces and decisions
  from earlier tasks that the brief cannot know; (4) your resolution of
  any ambiguity you noticed in the brief; (5) the report-file path and
  report contract. Exact values (numbers, magic strings, signatures, test
  cases) appear only in the brief.
- **Report file:** name the implementer's report file after the brief
  (brief `…/task-N-brief.md` → report `…/task-N-report.md`) and put it in
  the dispatch prompt. The implementer writes the full report there and
  returns only status, commits, a one-line test summary, and concerns.
- **Covered task after invalidation:** no implementer report is required when
  `remaining-work` re-establishes coverage. If a later review needs execution
  evidence, rerun the task's current verification command and write a fresh
  revision-scoped report containing that command and output.
- **Reviewer inputs:** the task reviewer gets three paths — the same brief
  file, the report file, and the review package — plus the global
  constraints that bind the task.
- Fix dispatches append their fix report (with test results) to the same
  report file and return a short summary; re-reviews read the updated file.

## Durable Progress

Conversation memory does not survive compaction. In real sessions,
controllers that lost their place have re-dispatched entire completed task
sequences — the single most expensive failure observed. Track progress in
a ledger file, not only in todos.

- At skill start and before trusting the ledger for any later dispatch, compute `git hash-object <plan-file>`, then check the ledger:
  `cat "<sdd-workspace>/progress.md"`.
- The ledger begins with both `Plan: <path-from-code-repository-root>` and
  `Plan revision: <git-hash-object-output>`.
- If no ledger exists, first check `git log` for task-numbered commits and inspect the plan's declared files for prior task execution evidence. Create a fresh ledger with those two header lines, in the listed order and one per line, only when neither check finds evidence. If either does, treat the ledger as lost or of unknown provenance: stop and ask your human partner to choose the review restart scope and base.
- Normalize `Plan:` from the code repository root that owns `.superpowers/sdd/progress.md`. An external plan may therefore be recorded as `../docs-repo/plans/auth.md`.
- Record each reviewer finding in the same controller turn that receives it, before another dispatch or any task-completion entry, as one line under `Pending review findings`: `Finding: open; severity=<Critical|Important|Minor|Unrated>; kind=<spec|quality>; task=<N>; commits=<base7>..<head7>; text=<verbatim finding>`. Update that line after fix and re-review; never rely on conversation memory as the only copy. Spec gaps without a reviewer-assigned severity use `Unrated`.
- Trust completed task entries only when both ledger fields exactly match the current plan. A missing or mismatched field invalidates every task entry. Before replacing `progress.md`, copy every `Finding: open` line to `<sdd-workspace>/pending-findings.snapshot` and read the snapshot back. Write the replacement ledger in this order: current `Plan:` header, current `Plan revision:` header, `Pending review findings (revalidate)`, then every snapshotted finding. Read the replacement ledger back before deleting the snapshot. Only then re-evaluate every task with `remaining-work` and rebuild task entries from current evidence.
- After invalidation, do not reuse task briefs, reports, review packages, or copied constraints derived from the old plan revision. Regenerate current revision-scoped dispatch evidence. Revalidate each snapshotted finding with a fresh task brief and constraints from the current plan plus a review package from its recorded `<base7>` through current `HEAD`; this deliberately includes later changes rather than omitting code that may affect the finding. Before treating the affected task as complete, re-review every spec-compliance entry and every Critical or Important code-quality entry; only unresolved Minor code-quality entries may roll forward to final review. Pass those remaining entries to the final reviewer and write each disposition back to the ledger. If any entry is omitted or ambiguous, re-dispatch for an explicit disposition. `Still valid` remains open and enters the appropriate fix/re-review loop; before finishing, every entry is `fixed (commit ...)`, `not applicable (rationale ...)`, or `deferred by human (rationale ...)`.
- When a task review is approved, append its settled baseline in the same
  message as your other bookkeeping. With no open findings, write
  `Task N: complete (commits <base7>..<head7>, review clean)`. With only open
  Minor findings, write `Task N: complete (commits <base7>..<head7>, review approved; Minor findings pending)`.
- The ledger is your recovery map: the commits it names exist in git even
  when your context no longer remembers creating them. After compaction,
  trust the ledger and `git log` over your own recollection.
- `git clean -fdx` will destroy the ledger and its review artifacts (they are
  git-ignored scratch). Treat that as lost durable review state: stop and ask
  your human partner to choose the review restart scope and base. Do not infer
  pending findings from `git log`, automatically reconstruct partial reviews,
  or finish until the human-directed review restart completes.

## Prompt Templates

- [implementer-prompt.md](implementer-prompt.md) - Dispatch implementer subagent
- [task-reviewer-prompt.md](task-reviewer-prompt.md) - Dispatch task reviewer subagent (spec compliance + code quality)
- Final whole-branch review: use superpowers:requesting-code-review's [code-reviewer.md](../requesting-code-review/code-reviewer.md)

## Example Workflow

```
You: I'm using Subagent-Driven Development to execute this plan.

[Read plan file once: docs/plans/feature-plan.md]
[Extract all 5 tasks with full text and context]
[Create todos for all tasks]

Task 1: Hook installation script

[Run task-brief for Task 1; dispatch implementer with brief + report paths + context]

Implementer: "Before I begin - should the hook be installed at user or system level?"

You: "User level (~/.config/superpowers/hooks/)"

Implementer: "Got it. Implementing now..."
[Later] Implementer:
  - Implemented install-hook command
  - Added tests, 5/5 passing
  - Self-review: Found I missed --force flag, added it
  - Committed

[Run review-package, dispatch task reviewer with the printed path]
Task reviewer: Spec ✅ - all requirements met, nothing extra.
  Strengths: Good test coverage, clean. Issues: None. Task quality: Approved.

[Mark Task 1 complete]

Task 2: Recovery modes

[Run task-brief for Task 2; dispatch implementer with brief + report paths + context]

Implementer: [No questions, proceeds]
Implementer:
  - Added verify/repair modes
  - 8/8 tests passing
  - Self-review: All good
  - Committed

[Run review-package, dispatch task reviewer with the printed path]
Task reviewer: Spec ❌:
  - Missing: Progress reporting (spec says "report every 100 items")
  - Extra: Added --json flag (not requested)
  Issues (Important): Magic number (100)

[Dispatch fix subagent with all findings]
Fixer: Removed --json flag, added progress reporting, extracted PROGRESS_INTERVAL constant

[Task reviewer reviews again]
Task reviewer: Spec ✅. Task quality: Approved.

[Mark Task 2 complete]

...

[After all tasks]
[Dispatch final code-reviewer]
Final reviewer: All requirements met, ready to merge

Done!
```

## Advantages

**vs. Manual execution:**
- Subagents follow TDD naturally
- Fresh context per task (no confusion)
- Parallel-safe (subagents don't interfere)
- Subagent can ask questions (before AND during work)

**vs. Executing Plans:**
- Same session (no handoff)
- Continuous progress (no waiting)
- Review checkpoints automatic

**Efficiency gains:**
- Controller curates exactly what context is needed; bulk artifacts move
  as files, not pasted text
- Subagent gets complete information upfront
- Questions surfaced before work begins (not after)

**Quality gates:**
- Self-review catches issues before handoff
- Task review carries two verdicts: spec compliance and code quality
- Review loops ensure fixes actually work
- Spec compliance prevents over/under-building
- Code quality ensures implementation is well-built

**Cost:**
- More subagent invocations (implementer + reviewer per task)
- Controller does more prep work (extracting all tasks upfront)
- Review loops add iterations
- But catches issues early (cheaper than debugging later)

## Red Flags

**Never:**
- Start implementation on main/master branch without explicit user consent
- Commit without referencing the plan task number in the commit message (e.g., "feat: implement <spec-name> task-N")
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed Critical/Important issues or silently discard Minor findings
- Dispatch multiple implementation subagents in parallel (conflicts)
- Make a subagent read the whole plan file (hand it its task brief —
  `scripts/task-brief` — instead)
- Skip scene-setting context (subagent needs to understand where task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance (reviewer found spec issues = not done)
- Skip required review loops (spec gaps or Critical/Important findings = fix
  and re-review; Minor findings = ledger entry and final review)
- Let implementer self-review replace actual review (both are needed)
- Tell a reviewer what not to flag, or pre-rate a finding's severity in the
  dispatch prompt ("treat it as Minor at most") — the plan's example code is
  a starting point, not evidence that its weaknesses were chosen
- Dispatch a task reviewer without a diff file — generate it first
  (`scripts/review-package BASE HEAD`) and name the printed path in the
  prompt
- Move to next task while the review has open Critical/Important issues
- Re-dispatch a task the progress ledger already marks complete — check
  the ledger's plan path and revision first, then confirm its commits with
  `git log` after any compaction or resume

**If subagent asks questions:**
- Answer clearly and completely
- Provide additional context if needed
- Don't rush them into implementation

**If reviewer finds spec gaps or Critical/Important issues:**
- Implementer (same subagent) fixes them
- Reviewer reviews again
- Repeat until spec compliant with no Critical/Important issues
- Don't skip the re-review

**If reviewer finds only Minor issues:**
- Keep their ledger entries open for final review
- Mark the task complete; Minor-only means `Task quality: Approved`

**If subagent fails task:**
- Dispatch fix subagent with specific instructions
- Don't try to fix manually (context pollution)

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
- **superpowers:spec-to-plan** - Creates the plan this skill executes
- **superpowers:remaining-work** - Identifies which plan tasks still need work
- **superpowers:requesting-code-review** - Code review template for reviewer subagents
- **superpowers:finishing-a-development-branch** - Complete development after all tasks

**Subagents should use:**
- **superpowers:test-driven-development** - Subagents follow TDD for each task

**Alternative workflow:**
- **superpowers:executing-plans** - Use for parallel session instead of same-session execution
