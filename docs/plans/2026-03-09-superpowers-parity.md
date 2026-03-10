# Superpowers Parity Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Close all 16 gaps between dotpowers and the original superpowers methodology.

**Architecture:** All changes target `dotpowers.dot`. Most are prompt edits. A few add new nodes/edges for worktree isolation, implementer question loop, and branch safety. No changes to tracker itself.

**Tech Stack:** Graphviz DOT, shell scripts (tool_command)

---

- [ ] task-001: Add branch creation to SetupProject
  **Files:** Modify: `dotpowers.dot` (SetupProject node, ~line 379)

  **What to change:**
  Add to SetupProject's prompt, after step 1 (read plan):
  ```
  1b. Create a feature branch for this work:
      git checkout -b feature/<project-name-slug>
      Never implement directly on main/master without explicit human consent.
  ```
  Also add to the prompt: "If already on a feature branch, skip branch creation."

  Remove `CleanupWorktree` edge from `CreatePR` (line ~1147) — keep the branch alive for PRs per the finishing skill spec. Only `MergeLocal` and `DiscardWork` should clean up.

  **Verify:** Read the modified node, confirm branch creation step is present. Confirm `CreatePR -> CleanupWorktree` edge is removed and replaced with `CreatePR -> Exit`.

---

- [ ] task-002: Add "propose 2-3 approaches" as mandatory brainstorm gate
  **Files:** Modify: `dotpowers.dot` (WriteDesignBrief node, ~line 122)

  **What to change:**
  In `WriteDesignBrief` prompt, add before the section list:
  ```
  ## MANDATORY: Propose Approaches First
  Before writing the full brief, present 2-3 different approaches with trade-offs.
  Lead with your recommended approach and explain why.
  Include these approaches as a section in the design brief so the human can evaluate them.
  ```

  **Verify:** Read modified prompt, confirm approaches section is mandatory.

---

- [ ] task-003: Fix design doc naming to use date prefix
  **Files:** Modify: `dotpowers.dot` (WriteDesignBrief ~line 150, CommitBrief ~line 200)

  **What to change:**
  - In WriteDesignBrief: change `docs/plans/design-brief.md` to `docs/plans/design-brief.md` BUT add instruction: "Also write a dated copy: `docs/plans/$(date +%Y-%m-%d)-<topic-slug>-design.md`"

  Actually, simpler: keep `design-brief.md` as the working name (other nodes reference it), but add to CommitBrief: "Also copy design-brief.md to docs/plans/YYYY-MM-DD-<topic>-design.md for archival."

  **Verify:** Read CommitBrief prompt, confirm dated copy instruction.

---

- [ ] task-004: Add verification-before-completion to all success-claiming nodes
  **Files:** Modify: `dotpowers.dot` (ImplementTask ~line 436, SpecReview ~line 507, QualityReview ~line 545, DebugInvestigate ~line 596)

  **What to change:**
  Add this block to each node's prompt (before the success criteria section):
  ```
  ## VERIFICATION GATE
  Before claiming success, you MUST have fresh evidence:
  - Run commands. Read output. Check exit codes. Count failures. THEN claim.
  - Do NOT say 'should work', 'probably passes', 'seems correct'.
  - If you haven't run the test suite in THIS session, you cannot claim tests pass.
  ```

  **Verify:** Grep for "VERIFICATION GATE" — should appear in ImplementTask, SpecReview, QualityReview, DebugInvestigate, plus FinalReviewOpus (already has equivalent).

---

- [ ] task-005: Fix SpecReview to diff full branch, not just HEAD~1
  **Files:** Modify: `dotpowers.dot` (SpecReview ~line 507)

  **What to change:**
  Replace line 523's instruction `git diff HEAD~1 to see what changed` with:
  ```
  2. Find the full diff for this task:
     - Get the branch base: BASE=$(git merge-base HEAD main 2>/dev/null || git rev-list --max-parents=0 HEAD)
     - Full diff: git diff $BASE...HEAD
     - If that's too large, at minimum: git log --oneline $BASE..HEAD to see all commits, then git show <each-sha> for the task's commits
  ```

  **Verify:** Read SpecReview prompt, confirm it references merge-base diffing.

---

- [ ] task-006: Add receiving-code-review guidance to ImplementTask for rework
  **Files:** Modify: `dotpowers.dot` (ImplementTask ~line 436)

  **What to change:**
  Add after the Self-Review Checklist section:
  ```
  ## When Re-entered After Review Failure
  If this node is reached after a spec or quality review failed:
  1. Read the review feedback from context CAREFULLY
  2. Verify each finding against the actual codebase — reviewers can be wrong
  3. For each valid finding: fix it. For invalid findings: document why it's wrong.
  4. Prioritize: blocking issues first, then simple fixes, then complex fixes
  5. Do NOT blindly implement every suggestion — check YAGNI. Reviewers sometimes suggest additions that aren't in the spec.
  6. After fixes: re-run full test suite to confirm nothing broke
  ```

  **Verify:** Read ImplementTask prompt, confirm rework guidance section exists.

---

- [ ] task-007: Make DebugInvestigate follow TDD for fixes
  **Files:** Modify: `dotpowers.dot` (DebugInvestigate ~line 596)

  **What to change:**
  Replace the Phase 3 hypothesis/testing section with:
  ```
  ## Phase 3: TDD Fix
  1. Write a failing test that reproduces the bug FIRST
  2. Run it — confirm it FAILS, confirm it fails for the RIGHT reason (the actual bug)
  3. Write the SMALLEST fix to make the test pass
  4. Run it — confirm it PASSES
  5. Run ALL tests — confirm nothing else broke
  6. Commit: test first, then fix

  You MUST see the test fail before writing the fix. This is not optional.
  If you wrote a fix before writing the test, delete the fix and start over.
  ```

  **Verify:** Read DebugInvestigate prompt, confirm TDD fix process replaces the old hypothesis approach.

---

- [ ] task-008: Add implementer question loop
  **Files:** Modify: `dotpowers.dot` (add ImplementerQuestion hexagon + CheckImplementerReady tool node, modify edges)

  **What to add (new nodes):**
  ```dot
  ImplementerQuestion [
    shape=hexagon,
    mode="freeform",
    label="Implementer Has a Question",
    prompt="The implementer has a question about the current task before starting work. See the question above and provide your answer."
  ];

  CheckImplementerReady [
    shape=parallelogram,
    label="Implementer Ready?",
    tool_command="#!/bin/sh\nif grep -q 'READY_TO_IMPLEMENT' docs/plans/implementer-status.txt 2>/dev/null; then\n  printf 'ready'\nelse\n  printf 'has_question'\nfi"
  ];
  ```

  **What to add to ImplementTask prompt** (before Process section):
  ```
  ## Before Starting
  If you have ANY questions about the task spec that would change your implementation:
  1. Write your question to docs/plans/implementer-status.txt
  2. End with HAS_QUESTION
  3. outcome=success (the question will be routed to the human)

  If the spec is clear enough to proceed:
  1. Write READY_TO_IMPLEMENT to docs/plans/implementer-status.txt
  2. Proceed with implementation
  ```

  **Edge changes:**
  - ImplementTask -> CheckImplementerReady (new, unconditional — replaces direct ImplementTask -> SpecReview)
  - CheckImplementerReady -> SpecReview [condition: context.tool_stdout=ready]
  - CheckImplementerReady -> ImplementerQuestion [condition: context.tool_stdout=has_question]
  - ImplementerQuestion -> ImplementTask (human answers, implementer retries)
  - Remove: ImplementTask -> SpecReview (replaced by above)

  **Verify:** Trace the path: ImplementTask -> CheckImplementerReady -> SpecReview (if ready) or -> ImplementerQuestion -> ImplementTask (if question). Confirm old direct edge is removed.

---

- [ ] task-009: Add human checkpoint every 3 tasks
  **Files:** Modify: `dotpowers.dot` (add CheckBatchComplete tool node, modify MarkTaskComplete edges)

  **What to add (new node):**
  ```dot
  CheckBatchComplete [
    shape=parallelogram,
    label="Batch Checkpoint?",
    tool_command="#!/bin/sh\nset -eu\nmkdir -p .tracker\nCOUNTER='.tracker/batch_count'\ncount=0\n[ -f \"$COUNTER\" ] && count=$(cat \"$COUNTER\")\ncount=$((count + 1))\nprintf '%s' \"$count\" > \"$COUNTER\"\nif [ $((count % 3)) -eq 0 ]; then\n  printf 'batch_complete'\nelse\n  printf 'continue'\nfi"
  ];
  ```

  Add new human gate:
  ```dot
  BatchReview [
    shape=hexagon,
    mode="freeform",
    label="Batch Checkpoint (3 tasks done)",
    prompt="3 tasks have been completed. Review progress so far and provide feedback, or say 'continue' to keep going."
  ];
  ```

  **Edge changes:**
  - MarkTaskComplete -> CheckBatchComplete (replaces MarkTaskComplete -> PickNextTask)
  - CheckBatchComplete -> PickNextTask [condition: context.tool_stdout=continue]
  - CheckBatchComplete -> BatchReview [condition: context.tool_stdout=batch_complete]
  - BatchReview -> PickNextTask

  Also add batch_count to the VerifyBaseline counter reset list.

  **Verify:** Trace: MarkTaskComplete -> CheckBatchComplete -> PickNextTask (if not 3rd) or -> BatchReview -> PickNextTask (every 3rd). Confirm VerifyBaseline resets batch_count.

---

- [ ] task-010: Fix DiscardConfirm to show commit list
  **Files:** Modify: `dotpowers.dot` (DiscardConfirm ~line 1014)

  **What to change:**
  Add a prompt attribute to the DiscardConfirm hexagon:
  ```
  prompt="WARNING: You are about to discard all work on this branch.\n\nCommits that will be deleted:\n(See git log above)\n\nType 'discard' to confirm, or anything else to go back to the shipping options."
  ```

  Also add a tool node before DiscardConfirm that runs `git log --oneline` so the commits are in context. Or simpler: add to the FinishGate prompt instructions to show the commit log before presenting options.

  **Verify:** Read DiscardConfirm, confirm prompt warns about commit deletion.

---

- [ ] task-011: Fix CreatePR to not clean up worktree
  **Files:** Modify: `dotpowers.dot` (edges ~line 1147)

  **What to change:**
  - Remove edge: `CreatePR -> CleanupWorktree`
  - Add edge: `CreatePR -> Exit`

  **Verify:** Grep for `CreatePR ->`, confirm it goes to Exit. Confirm CleanupWorktree is only reached from MergeLocal and DiscardWork.

---

- [ ] task-012: Apply YAGNI to brainstorm and plan prompts explicitly
  **Files:** Modify: `dotpowers.dot` (ExploreIdea ~line 28, RefineUnderstanding ~line 73, DraftPlan ~line 208)

  **What to change:**
  Add to ExploreIdea prompt (after Process section):
  ```
  ## YAGNI
  Ruthlessly cut anything not strictly needed for v1. If a feature is 'nice to have', drop it. If there's a simpler approach that covers 80% of the use case, prefer it.
  ```

  Add same YAGNI block to RefineUnderstanding prompt.

  DraftPlan already has YAGNI (line 257) — verify it's there, no change needed.

  **Verify:** Grep for "YAGNI" — should appear in ExploreIdea, RefineUnderstanding, WriteDesignBrief (already has it at line 146), and DraftPlan.

---

- [ ] task-013: Validate DOT file syntax
  **Step 1:** Run `dot -Tsvg dotpowers.dot > /dev/null`
  **Step 2:** Verify all new edges reference nodes that exist
  **Step 3:** Verify no orphaned nodes (every node has at least one incoming or outgoing edge)

  **Verify:** `dot` exits 0.

---

- [ ] task-014: Update README to reflect new capabilities
  **Files:** Modify: `README.md`

  **What to change:**
  - Update "What happens when you run it" section 4 (Implement) to mention question loop and batch checkpoints
  - Update "What it won't do" to remove "will not run unattended" claim — it now has human checkpoints but still requires interaction
  - Remove mention of sequential-only limitation if parallel is added (skip if not)
  - Add note about feature branch isolation in Setup description

  **Verify:** Read README, confirm it matches actual pipeline behavior.

---

- [ ] task-015: Copy updated DOT file to Desktop
  **Step 1:** `cp dotpowers.dot ~/Desktop/dotpowers-2026-03-09.dot`
  **Verify:** File exists on Desktop with today's date.

---

- [ ] task-016: Commit all changes
  **Step 1:** `git add dotpowers.dot README.md`
  **Step 2:** `git commit -m 'feat(dotpowers): close superpowers methodology gaps'`
  **Verify:** `git log --oneline -1` shows the commit.
