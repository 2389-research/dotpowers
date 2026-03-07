# dotpowers Design

## Overview

dotpowers is a mammoth DOT pipeline that encodes the full [superpowers](https://github.com/obra/superpowers) software development methodology as an autonomous LLM workflow. Given a project spec as input, it autonomously brainstorms, plans, implements with TDD, reviews via multi-model consensus, and ships — all headlessly via the [attractor](https://github.com/strongdm/attractor) DOT runner.

The core insight: superpowers is a set of skills (prompt documents) that guide an interactive agent through a structured dev workflow. dotpowers translates those skills into pipeline nodes where each node's prompt embeds the discipline, iron laws, and anti-rationalization patterns from the corresponding skill.

## Architecture

The pipeline has 6 phases, mirroring the superpowers skill chain:

```
Brainstorm → Plan → Setup → Implement (TDD loop) → Final Review → Finish
```

Each phase maps to one or more superpowers skills. Cross-cutting concerns (TDD discipline, verification, debugging) are embedded in node prompts rather than existing as separate nodes.

### Phase 1: Brainstorm (conditional human interaction)

Maps to: `superpowers:brainstorming`

- **RefineBrief** (Opus 4.6) — Autonomously refines the input spec. Identifies gaps, proposes architecture, asks itself clarifying questions. Produces a structured design brief.
- **NeedsClarification** (Opus 4.6, auto_status) — Evaluates whether the brief has critical ambiguities. Routes to human gate if yes, straight to planning if no.
- **HumanClarify** (hexagon, mode=freeform) — Only reached conditionally. Human provides answers, loops back to RefineBrief.

Key prompt content from brainstorming skill:
- Explore project context
- Propose 2-3 approaches with trade-offs
- YAGNI ruthlessly
- Scale design sections to complexity

### Phase 2: Plan (multi-model consensus)

Maps to: `superpowers:writing-plans`

- **PlanOpus** / **PlanGPT** / **PlanGemini** — Parallel fan-out. Each model independently produces a task breakdown with TDD steps, exact file paths, exact commands, and expected outputs. Plans assume the implementer has zero context and questionable taste.
- **ConsolidatePlan** (Opus 4.6) — Merges three plans into one. Tags tasks as parallel vs sequential. Ensures each task is 2-5 minutes of work (one action: write test, run test, implement, run test, commit).
- **WritePlanDoc** (tool node) — Writes the consolidated plan to `docs/plans/` and creates a task ledger (TSV) tracking completion status.

Key prompt content from writing-plans skill:
- Each step is ONE action (2-5 minutes)
- Exact file paths always
- Complete code in plan (not "add validation")
- Exact commands with expected output
- Mandatory plan header with goal, architecture, tech stack

### Phase 3: Setup

Maps to: `superpowers:using-git-worktrees`

- **SetupWorktree** (tool node) — Creates git worktree, detects project type, runs setup commands (npm install / uv sync / cargo build / etc).
- **VerifyBaseline** (tool node) — Runs full test suite. Confirms clean starting point. If tests fail, pipeline stops — no implementation on a broken baseline.

### Phase 4: Implement (task loop with TDD)

Maps to: `superpowers:subagent-driven-development`, `superpowers:test-driven-development`

This is the core loop. For each task in the plan:

- **PickNextTask** (tool node) — Reads task ledger, selects next incomplete task, marks it in_progress.
- **ImplementTask** (GPT-5.4) — The TDD node. Prompt embeds the full TDD iron law:
  - Write ONE failing test
  - Run it, verify it fails for the right reason
  - Write minimal code to pass
  - Run it, verify green
  - Refactor if needed
  - Commit with descriptive message
  - Self-review against spec
  - "NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST. Write code before the test? Delete it. Start over."
- **SpecReview** (Opus 4.6) — Spec compliance review. Prompt embeds the spec-reviewer pattern: "The implementer finished suspiciously quickly. Verify everything independently. Compare implementation to requirements LINE BY LINE."
- **QualityReview** (GPT-5.4) — Code quality review. Only runs after spec review passes. Checks patterns, naming, maintainability, security.
- **MarkTaskComplete** (tool node) — Updates ledger, loops back to PickNextTask.

Conditional paths:
- If SpecReview fails → back to ImplementTask with fix instructions (max 2 retries)
- If QualityReview fails → back to ImplementTask with fix instructions (max 2 retries)
- If implementation fails after 2 retries → **DebugInvestigate** (Opus 4.6) — root cause analysis per `superpowers:systematic-debugging`: "NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST"
- If debug fails after 2 more attempts → **ReplanTask** (Opus 4.6) — rethinks the approach
- If replan fails → **HumanHelp** (hexagon) — escalate to human

Loop termination: **PickNextTask** tool returns "all_complete" → exits to Phase 5.

### Phase 4b: Parallel Implementation

Maps to: `superpowers:dispatching-parallel-agents`

When the plan tags tasks as parallelizable:
- **ParallelFanOut** (component) → multiple ImplementTask pipelines run concurrently
- **ParallelFanIn** (tripleoctagon) → **IntegrationValidate** (tool node) runs full test suite to catch conflicts
- If integration fails → sequential re-implementation of conflicting tasks

### Phase 5: Final Review (multi-model consensus with cross-critique)

Maps to: `superpowers:verification-before-completion`, `superpowers:requesting-code-review`

- **ValidateBuild** (tool node) — Runs full test suite, linters, type checkers. Must pass before review.
- **FinalReviewOpus** / **FinalReviewGPT** / **FinalReviewGemini** — Parallel fan-out. Each reviews the complete implementation against the plan.
- **CritiqueOpusOnGPT** / **CritiqueOpusOnGemini** / **CritiqueGPTOnOpus** / **CritiqueGPTOnGemini** / **CritiqueGeminiOnOpus** / **CritiqueGeminiOnGPT** — Parallel cross-critiques. Each model critiques the other two reviews.
- **ReviewConsensus** (Opus 4.6, goal_gate=true) — Synthesizes all reviews and critiques. Returns success/retry/fail.
  - Success → Phase 6
  - Retry → back to ImplementTask with specific fixes (retry_target)
  - Fail → **HumanEscalation** (hexagon) — blocked, needs human

Key prompt content from verification skill:
- "NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE"
- Run the command. Read the output. Check exit code. Count failures. THEN claim.
- Forbidden words: "should", "probably", "seems to"

### Phase 6: Finish (human gate with 4 choices)

Maps to: `superpowers:finishing-a-development-branch`

- **VerifyTestsFinal** (tool node) — One last test run. Must pass.
- **FinishGate** (hexagon) — Presents exactly 4 options:
  1. [M] Merge back to base branch locally
  2. [P] Push and create a Pull Request
  3. [K] Keep the branch as-is
  4. [D] Discard this work

Conditional routing per choice:
- **MergeLocal** — checkout base, pull, merge, verify tests on merged result, delete feature branch, cleanup worktree
- **CreatePR** — push branch, create PR via gh with summary + test plan
- **KeepBranch** — report location, preserve worktree
- **DiscardWork** — requires typed confirmation, then force-delete branch, cleanup worktree
- **CleanupWorktree** (tool node) — removes worktree for options M, P, D

## Model Allocation

| Role | Model | Provider | Rationale |
|------|-------|----------|-----------|
| Deep reasoning (brainstorm, consolidate, consensus) | claude-opus-4-6 | anthropic | Strongest reasoning for architectural decisions |
| Fast implementation (TDD cycles) | gpt-5.4 | openai | Fast code generation with strong tool use |
| Second opinion (parallel planning, cross-critique) | gemini-3.5-flash | google | Cost-effective third perspective |
| Spec review | claude-opus-4-6 | anthropic | Meticulous line-by-line comparison |
| Code quality review | gpt-5.4 | openai | Different perspective from implementer's own model |

## Retry Strategy

Graduated escalation:

1. **Node-level retry** (max 2) — retry the failing node with error context
2. **Phase escalation** — if implementation node exhausts retries, escalate to debug investigation
3. **Re-planning** — if debug exhausts retries, escalate to re-plan the task
4. **Human gate** — if re-planning fails, escalate to human

Graph-level defaults:
- `default_max_retry=2`
- `retry_target=ImplementTask` (most failures are implementation failures)
- `fallback_retry_target=RefineBrief` (nuclear option: start over)

## State Management

Uses the tool-driven ledger pattern from mammoth examples:

- `.ai/ledger.tsv` — tracks task status (planned/in_progress/completed/failed)
- `.ai/current_task_id.txt` — current task pointer
- `.ai/plan.md` — the consolidated implementation plan
- `.ai/design-brief.md` — the brainstorm output

Tool nodes read/update the ledger, enabling idempotency and crash recovery via mammoth checkpoints.

## Anti-Rationalization (embedded in prompts)

Every implementation and review node embeds guards against common LLM failure modes:

**TDD nodes:**
- "Write code before the test? Delete it. Start over. No exceptions."
- "Don't keep it as reference. Don't adapt it. Delete means delete."
- Forbidden: "Too simple to test", "I'll test after", "Already manually tested"

**Review nodes:**
- "The implementer finished suspiciously quickly. Their report may be incomplete."
- "DO NOT take their word. Read actual code. Compare LINE BY LINE."

**Verification nodes:**
- "NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE"
- "Run the command. Read the output. THEN claim."

**Debug nodes:**
- "NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST"
- ">= 3 attempts: STOP and question the architecture"

## Pipeline Configuration

- **Phases:** brainstorm, plan, setup, implement_loop, final_review, finish
- **Tech Stack:** Language-agnostic (plan node determines test framework)
- **Testing:** Determined by plan node based on project type
- **Quality Gates:** ValidateBuild tool node runs tests/linters/type-checks
- **Human Gates:** Conditional — only when agent needs clarification, is stuck, or at finish
- **Retry Strategy:** max 2 per node, escalate to debug → replan → human
- **Models:** Opus 4.6 (reasoning), GPT-5.4 (implementation), Gemini 3.5 Flash (third opinion)
- **Parallelism:** Plan node tags independent tasks; parallel fan-out for implementation, always parallel for reviews/critiques
