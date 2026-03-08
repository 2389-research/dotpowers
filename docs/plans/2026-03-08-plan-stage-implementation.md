# Plan Stage Redesign Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace the 3-way parallel plan fan-out with a draft-audit-patch loop, fix PickNextTask silent failure, and add empty-project guardrails to reviewers.

**Architecture:** Edit `dotpowers.dot` to remove 6 plan nodes (PlanFanOut, PlanOpus, PlanGPT, PlanGemini, PlanJoin, ConsolidatePlan), add 4 new nodes (DraftPlan, ValidatePlanFormat, AuditPlanSpec, PatchPlan), fix PickNextTask tool_command, update reviewer prompts, and add `.draft` to model_stylesheet. Validate with `mammoth -validate` after each change.

**Tech Stack:** Graphviz DOT (mammoth pipeline format), shell scripts (tool_command), `mammoth -validate` for validation.

---

### Task 1: Add `.draft` class to model_stylesheet

**Files:**
- Modify: `dotpowers.dot:10-14`

**Step 1: Write the failing test**

Run mammoth validate to establish baseline:
```bash
cd /Users/harper/Public/src/2389/dotpowers && mammoth -validate dotpowers.dot
```
Expected: PASS (current state is valid)

**Step 2: Add the `.draft` class to model_stylesheet**

In `dotpowers.dot`, change lines 10-14 from:
```dot
    model_stylesheet="
      * { llm_model: claude-opus-4-6; llm_provider: anthropic; reasoning_effort: high; }
      .implement { llm_model: gpt-5.4; llm_provider: openai; reasoning_effort: high; }
      .opinion { llm_model: gemini-3.5-flash; llm_provider: gemini; reasoning_effort: high; }
    "
```

To:
```dot
    model_stylesheet="
      * { llm_model: claude-opus-4-6; llm_provider: anthropic; reasoning_effort: high; }
      .implement { llm_model: gpt-5.4; llm_provider: openai; reasoning_effort: high; }
      .draft { llm_model: gpt-5.2; llm_provider: openai; reasoning_effort: high; }
      .opinion { llm_model: gemini-3.5-flash; llm_provider: gemini; reasoning_effort: high; }
    "
```

**Step 3: Validate**

```bash
mammoth -validate dotpowers.dot
```
Expected: PASS

**Step 4: Commit**

```bash
git add dotpowers.dot
git commit -m 'feat(dotpowers): add .draft model class for GPT-5.2'
```

---

### Task 2: Remove old plan nodes and edges

**Files:**
- Modify: `dotpowers.dot:206-393` (Phase 2 node definitions)
- Modify: `dotpowers.dot:1046-1054` (Phase 2 edges)

**Step 1: Identify all nodes and edges to remove**

Nodes to delete (entire node definitions including all attributes):
- `PlanFanOut` (line ~210)
- `PlanOpus` (lines ~212-248)
- `PlanGPT` (lines ~251-288)
- `PlanGemini` (lines ~291-328)
- `PlanJoin` (line ~330)
- `ConsolidatePlan` (lines ~332-377)

Edges to delete:
- `CommitBrief -> PlanFanOut;`
- `PlanFanOut -> PlanOpus;`
- `PlanFanOut -> PlanGPT;`
- `PlanFanOut -> PlanGemini;`
- `PlanOpus -> PlanJoin;`
- `PlanGPT -> PlanJoin;`
- `PlanGemini -> PlanJoin;`
- `PlanJoin -> ConsolidatePlan;`
- `ConsolidatePlan -> CommitPlan;`

Keep `CommitPlan` node and `CommitBrief` node — they stay.

**Step 2: Delete the nodes and edges**

Remove the 6 node definitions and 9 edges listed above. Update the Phase 2 section comment to say:
```dot
  // ============================================================
  // PHASE 2 — PLAN  (draft-audit-patch loop)
  // ============================================================
```

**Step 3: Validate**

```bash
mammoth -validate dotpowers.dot
```
Expected: FAIL — CommitPlan is now orphaned (no incoming edge). That's expected; we'll connect it in Task 3.

**Step 4: Commit**

```bash
git add dotpowers.dot
git commit -m 'refactor(dotpowers): remove 3-way plan fan-out nodes and edges'
```

---

### Task 3: Add DraftPlan node

**Files:**
- Modify: `dotpowers.dot` (Phase 2 section)

**Step 1: Add the DraftPlan node definition**

Insert after the Phase 2 section comment:

```dot
  DraftPlan [
    shape=box,
    class="draft",
    label="Draft Plan (GPT-5.2)",
    goal_gate=true,
    retry_target="DraftPlan",
    fidelity="full",
    prompt="You are working in `run.working_dir`.

## Role
You are a senior architect creating a detailed implementation plan using strict TDD.

## Context
Read docs/plans/design-brief.md for the project design.

## Task
Create a step-by-step implementation plan. Write it assuming the implementing engineer has ZERO context and questionable taste. Document everything they need.

## Plan Header (MANDATORY)
Start the plan with:
- Goal (one sentence)
- Architecture (2-3 sentences)
- Tech stack (language, framework, key deps, test framework, linter)

## Task Format (MANDATORY — follow EXACTLY)
Every task MUST be a markdown checkbox with a task ID:

- [ ] task-001: Short descriptive title
  **Files:** exact paths to create/modify/test
  **Step 1: Write the failing test**
  (complete test code — not pseudocode, not 'add appropriate tests')
  **Step 2: Run test to verify it fails**
  (exact command + expected failure output)
  **Step 3: Write minimal implementation**
  (complete implementation code — not 'implement the logic' or 'add validation')
  **Step 4: Run test to verify it passes**
  (exact command + expected output)
  **Step 5: Commit**
  (exact git command with conventional commit message)

- [ ] task-002: Next task title
  ...

## IRON RULES
- NEVER write 'implement the logic', 'add appropriate', 'write a complete implementation', 'add validation', or 'TODO'. Every function body must be written out in full.
- NEVER write tautological tests. A test that asserts `width() == 16` is WRONG if width should be computed from content. Test the BEHAVIOR, not hardcoded values.
- Every requirement in the design brief MUST have a corresponding task AND test. If the design brief says 'render X in the bottom-right', there must be a test that asserts X is rendered.
- Each step is ONE action (2-5 minutes). 'Write test' and 'run test' are separate steps.
- Tag each task as PARALLEL or SEQUENTIAL based on dependencies.
- Include setup tasks (project init, deps, tooling) before implementation tasks.

## Output
Write your plan to docs/plans/plan.md
"
  ];
```

**Step 2: Add the edge from CommitBrief to DraftPlan**

In the edges section, replace the old `CommitBrief -> PlanFanOut;` (already deleted) with:
```dot
  CommitBrief -> DraftPlan;
```

**Step 3: Validate**

```bash
mammoth -validate dotpowers.dot
```
Expected: May still have warnings about disconnected nodes, but DraftPlan should parse correctly.

**Step 4: Commit**

```bash
git add dotpowers.dot
git commit -m 'feat(dotpowers): add DraftPlan node with strict format requirements'
```

---

### Task 4: Add ValidatePlanFormat tool node

**Files:**
- Modify: `dotpowers.dot` (Phase 2 section, after DraftPlan)

**Step 1: Add the ValidatePlanFormat node definition**

```dot
  ValidatePlanFormat [
    shape=parallelogram,
    label="Validate Plan Format",
    tool_command="#!/bin/sh\nset -eu\nPLAN='docs/plans/plan.md'\nif [ ! -s \"$PLAN\" ]; then\n  printf 'fail_no_plan'\n  exit 0\nfi\ncount=$(grep -c '^- \\[ \\] task-[0-9]\\+:' \"$PLAN\" || true)\nif [ \"$count\" -eq 0 ]; then\n  printf 'fail_no_checkboxes'\n  exit 0\nfi\nhandwaves=$(grep -ciE '(write a complete|add appropriate|implement the|add validation|add logic|TODO|TBD)' \"$PLAN\" || true)\nif [ \"$handwaves\" -gt 0 ]; then\n  printf 'fail_handwaves_%s' \"$handwaves\"\n  exit 0\nfi\nprintf 'format_ok_%s_tasks' \"$count\""
  ];
```

**Step 2: Add edges**

```dot
  DraftPlan -> ValidatePlanFormat;
  ValidatePlanFormat -> AuditPlanSpec [label="format ok", condition="context.tool_stdout~=format_ok_", weight=2];
  ValidatePlanFormat -> DraftPlan [label="format fail", condition="context.tool_stdout~=fail_"];
```

Note: Using `~=` for prefix matching if mammoth supports it. If not, use specific conditions:
```dot
  ValidatePlanFormat -> AuditPlanSpec [label="format ok", condition="context.tool_stdout!=fail_no_plan,context.tool_stdout!=fail_no_checkboxes,context.tool_stdout!=fail_handwaves_", weight=2];
```

Check mammoth docs for condition syntax — may need to use `startswith` or exact match patterns.

**Step 3: Validate**

```bash
mammoth -validate dotpowers.dot
```

**Step 4: Commit**

```bash
git add dotpowers.dot
git commit -m 'feat(dotpowers): add ValidatePlanFormat tool node with hand-wave detection'
```

---

### Task 5: Add AuditPlanSpec node

**Files:**
- Modify: `dotpowers.dot` (Phase 2 section)

**Step 1: Add the AuditPlanSpec node definition**

```dot
  AuditPlanSpec [
    shape=box,
    label="Audit Plan vs Spec (Opus)",
    goal_gate=true,
    retry_target="PatchPlan",
    prompt="You are working in `run.working_dir`.

## Role
You are a meticulous spec compliance auditor for implementation plans.

## Task
Verify that every requirement in the design brief has a corresponding task AND test in the plan.

## Process
1. Read docs/plans/design-brief.md — extract every concrete requirement:
   - Features and behaviors (e.g. 'render corner counter in bottom-right')
   - Data types and constraints (e.g. 'State.x as i16')
   - Error handling requirements
   - UI/rendering specifications
   - API contracts

2. Read docs/plans/plan.md — for each requirement, verify:
   a. There is a task that implements it (with complete code, not hand-waves)
   b. There is a test that would FAIL if the requirement were missing
   c. The test is NOT tautological (does not assert hardcoded values that could drift from reality)

3. Flag each gap with structured output:
   - MISSING_TASK: requirement has no corresponding task
   - MISSING_TEST: requirement has a task but no test
   - TAUTOLOGICAL_TEST: test asserts a hardcoded value instead of computing from content
   - VAGUE_IMPLEMENTATION: task says 'implement X' without complete code

## Output
Write results to docs/plans/plan-audit.txt:
- If all requirements covered: write the single word APPROVED
- If gaps found: write structured gap list, one GAP block per issue:

GAP: design-brief.md requirement: 'quoted requirement text'
  MISSING_TEST: description of what test is needed
  SUGGESTED_FIX: what the test should assert

End your response with APPROVED or GAPS_FOUND.

outcome=success if APPROVED
outcome=fail if GAPS_FOUND
"
  ];
```

**Step 2: Add edge to CommitPlan**

```dot
  AuditPlanSpec -> CommitPlan [label="plan approved", condition="outcome=success", weight=2];
  AuditPlanSpec -> PatchPlan [label="gaps found", condition="outcome=fail"];
```

**Step 3: Validate**

```bash
mammoth -validate dotpowers.dot
```

**Step 4: Commit**

```bash
git add dotpowers.dot
git commit -m 'feat(dotpowers): add AuditPlanSpec node for spec coverage verification'
```

---

### Task 6: Add PatchPlan node

**Files:**
- Modify: `dotpowers.dot` (Phase 2 section)

**Step 1: Add the PatchPlan node definition**

```dot
  PatchPlan [
    shape=box,
    class="draft",
    label="Patch Plan (GPT-5.2)",
    goal_gate=true,
    retry_target="HumanHelp",
    max_retries=2,
    prompt="You are working in `run.working_dir`.

## Role
You are patching an implementation plan to address gaps identified by the spec auditor.

## Context
1. Read docs/plans/plan-audit.txt for the list of gaps
2. Read docs/plans/plan.md for the current plan
3. Read docs/plans/design-brief.md for the original requirements

## Task
For each GAP in the audit:
- MISSING_TASK: Add a new task with full TDD steps (failing test with complete code, exact run command, minimal implementation with complete code, exact run command, commit command)
- MISSING_TEST: Add a test to the existing task that would fail if the requirement were missing
- TAUTOLOGICAL_TEST: Replace the hardcoded assertion with one that computes the expected value from the actual content
- VAGUE_IMPLEMENTATION: Replace the hand-wave with complete code

## Rules
- New task IDs continue from the highest existing ID (e.g., if plan has task-012, new tasks start at task-013)
- Use the same checkbox format: - [ ] task-NNN: Title
- Every function body must be complete code — NEVER write 'implement the logic'
- Every test must verify behavior, not hardcoded values
- Do NOT remove or modify existing passing tasks — only add or fix flagged ones

## Output
Update docs/plans/plan.md in place with the fixes.
"
  ];
```

**Step 2: Add edge back to ValidatePlanFormat**

```dot
  PatchPlan -> ValidatePlanFormat;
```

**Step 3: Validate**

```bash
mammoth -validate dotpowers.dot
```
Expected: PASS — all plan-stage nodes are now connected.

**Step 4: Commit**

```bash
git add dotpowers.dot
git commit -m 'feat(dotpowers): add PatchPlan node completing the draft-audit-patch loop'
```

---

### Task 7: Update CommitPlan prompt

**Files:**
- Modify: `dotpowers.dot` (CommitPlan node, lines ~379-393)

**Step 1: Update the git add command to reference new artifacts**

Change CommitPlan's prompt from:
```
git add docs/plans/plan.md docs/plans/plan-opus.md docs/plans/plan-gpt.md docs/plans/plan-gemini.md
git commit -m 'docs(plan): add consolidated implementation plan'
```

To:
```
git add docs/plans/plan.md docs/plans/plan-audit.txt
git commit -m 'docs(plan): add implementation plan'
```

**Step 2: Validate**

```bash
mammoth -validate dotpowers.dot
```
Expected: PASS

**Step 3: Commit**

```bash
git add dotpowers.dot
git commit -m 'fix(dotpowers): update CommitPlan to reference new plan artifacts'
```

---

### Task 8: Fix PickNextTask with three-way output

**Files:**
- Modify: `dotpowers.dot` (PickNextTask node, line ~447-451)

**Step 1: Replace the tool_command**

Change PickNextTask's `tool_command` to:

```
#!/bin/sh\nset -eu\nPLAN='docs/plans/plan.md'\ntotal=$(grep -c '^- \\[.\\] task-[0-9]' \"$PLAN\" 2>/dev/null || true)\nif [ \"$total\" -eq 0 ]; then\n  printf 'no_tasks_found'\n  exit 0\nfi\ntarget=$(grep -oE 'task-[0-9]+' \"$PLAN\" | while read -r tid; do\n  if grep -q \"^- \\[ \\] $tid\" \"$PLAN\"; then\n    echo \"$tid\"\n    break\n  fi\ndone)\nif [ -z \"$target\" ]; then\n  printf 'all_complete'\n  exit 0\nfi\nprintf '%s' \"$target\" > docs/plans/current_task_id.txt\nprintf 'next_task-%s' \"$target\"
```

**Step 2: Add the new edge for `no_tasks_found`**

Add:
```dot
  PickNextTask -> HumanHelp [label="no tasks in plan", condition="context.tool_stdout=no_tasks_found"];
```

**Step 3: Validate**

```bash
mammoth -validate dotpowers.dot
```
Expected: PASS

**Step 4: Commit**

```bash
git add dotpowers.dot
git commit -m 'fix(dotpowers): PickNextTask distinguishes done from broken plan'
```

---

### Task 9: Add empty-project guardrail to final reviewers

**Files:**
- Modify: `dotpowers.dot` (FinalReviewOpus, FinalReviewGPT, FinalReviewGemini nodes)

**Step 1: Add pre-check paragraph to each reviewer prompt**

Add this text immediately after the `## Role` line in all three final review prompts:

```
## Pre-check (MANDATORY — do this FIRST)
Before reviewing code quality, verify substantive application code exists beyond scaffolding.
If the only tests are trivial (assert True, assert 1==1, or similar no-op assertions),
or if no application source files exist beyond __init__.py, config, and tooling,
immediately return FAIL with 'NO_APPLICATION_CODE — project contains only scaffolding, no
substantive implementation exists to review.' Do NOT proceed with quality review.
```

**Step 2: Validate**

```bash
mammoth -validate dotpowers.dot
```
Expected: PASS

**Step 3: Commit**

```bash
git add dotpowers.dot
git commit -m 'fix(dotpowers): add empty-project guardrail to final reviewers'
```

---

### Task 10: Final validation and integration commit

**Files:**
- Verify: `dotpowers.dot` (full pipeline)

**Step 1: Run full validation**

```bash
mammoth -validate dotpowers.dot
```
Expected: PASS with no warnings about orphaned nodes or missing edges.

**Step 2: Verify edge connectivity**

Manually trace the plan stage path:
```
CommitBrief → DraftPlan → ValidatePlanFormat → AuditPlanSpec → CommitPlan
                ↑                                    ↓
                └──── ValidatePlanFormat ← PatchPlan ←┘
```

Also verify:
- ValidatePlanFormat has fail edges back to DraftPlan
- PatchPlan has max_retries=2 with retry_target=HumanHelp
- PickNextTask has three output edges (next_task, all_complete, no_tasks_found)
- All three final reviewers have the pre-check paragraph

**Step 3: Count nodes**

Verify net change: 6 removed, 4 added = net -2 nodes from previous pipeline.

**Step 4: Commit (if any final adjustments needed)**

```bash
git add dotpowers.dot
git commit -m 'feat(dotpowers): complete plan stage redesign — draft-audit-patch loop'
```
