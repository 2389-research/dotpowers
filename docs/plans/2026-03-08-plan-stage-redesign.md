# Plan Stage Redesign

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace the 3-way parallel plan fan-out with a draft-audit-patch loop that catches spec drift, format errors, and vague instructions before implementation begins.

**Architecture:** GPT-5.2 drafts the plan (fast, cheap, structured), Opus audits it against the design brief for spec coverage, and GPT-5.2 patches any gaps. A format-validation tool node gates entry to audit, preventing wasted Opus tokens on malformed plans. The loop runs up to 3 iterations before escalating to a human.

**Motivation:** Two pipeline runs exposed three failure modes:
1. **dvd-bounce** — plan dropped rendering specs from the design brief, wrote tautological tests, used vague "write a complete implementation" instructions. Result: 80% spec fidelity, physics bug, missing feature.
2. **RemixOS** — 2 of 3 planning agents reported intent but never wrote files. ConsolidatePlan produced numbered lists instead of checkboxes. PickNextTask returned `all_complete` on an empty project. Three reviewers and six cross-critiques approved `assert True`. Result: 0% implementation.

---

## Section 1: Plan Stage Architecture

### Current (remove)

```
CommitBrief → PlanFanOut → PlanOpus   → PlanJoin → ConsolidatePlan → CommitPlan
                         → PlanGPT    →
                         → PlanGemini →
```

5 nodes + fan-out/join. Three models write full plans in parallel, Opus consolidates. Slow, expensive, 2 of 3 plans were useless in practice.

### New (replace with)

```
CommitBrief → DraftPlan (GPT-5.2)
           → ValidatePlanFormat (tool)
           → AuditPlanSpec (Opus)
                ↓ approved          ↓ gaps found
           CommitPlan          PatchPlan (GPT-5.2)
                                    ↓
                               ValidatePlanFormat (loop back, max 3)
```

### Nodes

**DraftPlan** — `class="draft"`, model: `gpt-5.2`
- Reads `docs/plans/design-brief.md`
- Writes `docs/plans/plan.md` with full TDD steps
- Must use `- [ ] task-NNN: Title` checkbox format
- Must include complete code in every step — never "write a complete implementation" or "add validation"
- Each task: failing test (complete code) → run test (exact command) → minimal implementation (complete code) → run test → commit

**ValidatePlanFormat** — `shape=parallelogram`, tool node
- Verifies plan.md exists and has content
- Checks for `- [ ] task-NNN:` checkbox lines
- Detects hand-wave phrases: "write a complete", "add appropriate", "implement the", "add validation", "add logic", "TODO", "TBD"
- Three-way output: `format_ok_N_tasks`, `fail_no_plan`, `fail_no_checkboxes`, `fail_handwaves_N`

**AuditPlanSpec** — default model (Opus)
- Reads `docs/plans/design-brief.md` and extracts every concrete requirement
- Reads `docs/plans/plan.md` and maps each requirement to a task AND a test
- A requirement is "covered" only if there is both implementation code AND a test that would fail if the requirement were missing
- Flags tautological tests (assertions on hardcoded values instead of computed values)
- Flags tasks where tests don't constrain the implementation
- Output to `docs/plans/plan-audit.txt`:
  - `APPROVED` if all requirements covered
  - Structured gap list if not:
    ```
    GAP: design-brief.md line 47: "corner counter in bottom-right"
      MISSING_TEST: no test asserts counter is rendered
      MISSING_TASK: draw() task has no specification for counter widget

    GAP: design-brief.md line 23: "ASCII art DVD logo (12-15 chars)"
      TAUTOLOGICAL_TEST: test asserts logo_width()==16 but doesn't verify against actual content
    ```

**PatchPlan** — `class="draft"`, model: `gpt-5.2`
- Reads gap list from `docs/plans/plan-audit.txt`
- Reads current `docs/plans/plan.md`
- Adds missing tasks with full TDD steps for each gap
- Fixes tautological tests by replacing hardcoded assertions with computed ones
- Assigns new task IDs continuing from the highest existing ID
- Loops back to ValidatePlanFormat (max 3 iterations total)

### Routing

```
CommitBrief → DraftPlan
DraftPlan → ValidatePlanFormat
ValidatePlanFormat → AuditPlanSpec        [condition: format_ok_*]
ValidatePlanFormat → DraftPlan            [condition: fail_*, first attempt]
ValidatePlanFormat → HumanHelp            [condition: fail_*, after 3 attempts]
AuditPlanSpec → CommitPlan                [condition: outcome=success, APPROVED]
AuditPlanSpec → PatchPlan                 [condition: outcome=fail, gaps found]
PatchPlan → ValidatePlanFormat            [loop back]
```

### Model Stylesheet Addition

```css
.draft { llm_model: gpt-5.2; llm_provider: openai; reasoning_effort: high; }
```

Added alongside existing `.implement` (gpt-5.4) and `.opinion` (gemini-3.5-flash).

---

## Section 2: ValidatePlanFormat Tool Command

```sh
#!/bin/sh
set -eu
PLAN="docs/plans/plan.md"

# File must exist and have content
if [ ! -s "$PLAN" ]; then
  printf 'fail_no_plan'
  exit 0
fi

# Must have at least one checkbox task
count=$(grep -c '^- \[ \] task-[0-9]\+:' "$PLAN" || true)
if [ "$count" -eq 0 ]; then
  printf 'fail_no_checkboxes'
  exit 0
fi

# Check for hand-wave phrases that signal vague plans
handwaves=$(grep -ciE '(write a complete|add appropriate|implement the|add validation|add logic|TODO|TBD)' "$PLAN" || true)
if [ "$handwaves" -gt 0 ]; then
  printf 'fail_handwaves_%s' "$handwaves"
  exit 0
fi

printf 'format_ok_%s_tasks' "$count"
```

---

## Section 3: PickNextTask Fix

Current behavior: no checkboxes found → empty target → `all_complete`. This caused the entire RemixOS implement loop to be skipped.

Fix: three-way output distinguishing "done" from "broken":

```sh
#!/bin/sh
set -eu
PLAN="docs/plans/plan.md"

# First: do ANY task checkboxes exist at all?
total=$(grep -c '^- \[.\] task-[0-9]' "$PLAN" 2>/dev/null || true)
if [ "$total" -eq 0 ]; then
  printf 'no_tasks_found'
  exit 0
fi

# Find first unchecked task
target=$(grep -oE 'task-[0-9]+' "$PLAN" | while read -r tid; do
  if grep -q "^- \[ \] $tid" "$PLAN"; then
    echo "$tid"
    break
  fi
done)

if [ -z "$target" ]; then
  printf 'all_complete'
  exit 0
fi

printf '%s' "$target" > docs/plans/current_task_id.txt
printf 'next_task-%s' "$target"
```

**New routing:**
- `next_task-*` → ImplementTask
- `all_complete` → ValidateBuild
- `no_tasks_found` → HumanHelp (plan has no task checkboxes — something is wrong)

---

## Section 4: Reviewer Empty-Project Guardrail

Add to all three final review prompts (FinalReviewOpus, FinalReviewGPT, FinalReviewGemini):

> **Pre-check:** Before reviewing code quality, verify substantive application code exists. If the only test is `assert True` or equivalent, or if no application code files exist beyond `__init__.py` and config, immediately return FAIL with "NO_APPLICATION_CODE" — do not proceed with review.

No new nodes needed. Prevents the "three reviewers approved an empty project" failure mode.

---

## Nodes Removed

- PlanFanOut (component)
- PlanOpus (box)
- PlanGPT (box)
- PlanGemini (box)
- PlanJoin (tripleoctagon)
- ConsolidatePlan (box)

Total: 6 nodes removed.

## Nodes Added

- DraftPlan (box, class="draft")
- ValidatePlanFormat (parallelogram, tool)
- AuditPlanSpec (box, default/Opus)
- PatchPlan (box, class="draft")

Total: 4 nodes added. Net reduction of 2 nodes.

## Nodes Modified

- PickNextTask — new tool_command with three-way output, new edge to HumanHelp
- FinalReviewOpus — add empty-project pre-check to prompt
- FinalReviewGPT — add empty-project pre-check to prompt
- FinalReviewGemini — add empty-project pre-check to prompt
- model_stylesheet — add `.draft` class

## Edges Changed

### Remove
- CommitBrief → PlanFanOut
- PlanFanOut → PlanOpus
- PlanFanOut → PlanGPT
- PlanFanOut → PlanGemini
- PlanOpus → PlanJoin
- PlanGPT → PlanJoin
- PlanGemini → PlanJoin
- PlanJoin → ConsolidatePlan
- ConsolidatePlan → CommitPlan

### Add
- CommitBrief → DraftPlan
- DraftPlan → ValidatePlanFormat
- ValidatePlanFormat → AuditPlanSpec [condition: format_ok_*]
- ValidatePlanFormat → DraftPlan [condition: fail_*, retry]
- ValidatePlanFormat → HumanHelp [condition: fail_*, retries exhausted]
- AuditPlanSpec → CommitPlan [condition: outcome=success]
- AuditPlanSpec → PatchPlan [condition: outcome=fail]
- PatchPlan → ValidatePlanFormat
- PickNextTask → HumanHelp [condition: no_tasks_found]
