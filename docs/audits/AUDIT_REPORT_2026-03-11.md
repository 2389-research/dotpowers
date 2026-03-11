# Documentation Audit Report
Generated: 2026-03-11 | Commit: a3817d9

## Executive Summary

| Metric | Count |
|--------|-------|
| Documents scanned | 1 (README.md) |
| Claims verified | ~40 |
| Verified TRUE | 37 (~93%) |
| **Verified FALSE** | **3 (7%)** |

Plans in `docs/plans/` were excluded per audit policy (internal design docs, not user-facing).

## False Claims Requiring Fixes

### README.md

| Line | Claim | Reality | Fix |
|------|-------|---------|-----|
| 3, 106 | "~1150 lines" | Actual: **1273 lines** (`wc -l dotpowers.dot`) — 10% drift since last update | Update to "~1250 lines" or "~1300 lines" |
| 11 | "**Setup** -- git init, create a `feature/<project-name-slug>` branch" | `SetupProject` node does NOT run `git init`. It only runs `git checkout -b "feature/$(basename $(pwd))"`. The Usage section correctly shows `git init` as a **user** step before running the pipeline. | Remove "git init," from the Setup description, or note it assumes git is already initialized |
| 106 | "the pipeline (~1150 lines of DOT)" | Same line-count issue as line 3 | Update to match actual count |

## Verified TRUE Claims (highlights)

| Line | Claim | Evidence |
|------|-------|----------|
| 5 | 6 phases: Brainstorm -> Plan -> Setup -> Implement -> Final Review -> Finish | DOT phases 0-6 match (phases 0+1 = Brainstorm) |
| 9 | "asks you questions one at a time (YAGNI enforced)" | ExploreIdea prompt: "SINGLE most important open question" (L48); AskBrainstormQuestion: "one at a time!" (L92); YAGNI referenced 6 times |
| 9 | "proposes 2-3 approaches with trade-offs" | WriteDesignBrief prompt: "present 2-3 different architectural approaches" (L138) |
| 10 | "GPT-5.2 drafts a TDD plan" | DraftPlan has `class="draft"` -> `.draft { llm_model: gpt-5.2 }` |
| 10 | "shell script rejects vague hand-wavy steps" | ValidatePlanFormat (L293) greps for handwave patterns |
| 10 | "Opus audits every requirement" | AuditPlanSpec (L296) has no class -> defaults to Opus |
| 10 | "Up to 5 iterations" | ValidatePlanFormat counter: `> 5` -> exhausts on 6th attempt = 5 allowed |
| 11 | "create a `feature/<project-name-slug>` branch" | L414: `branch_name="feature/$(basename $(pwd))"` |
| 12 | "Opus checks spec compliance, GPT-5.4 checks code quality" | SpecReview: no class -> Opus; QualityReview: `class="implement"` -> GPT-5.4 |
| 12 | "Every 3rd completed task triggers a batch checkpoint" | CheckBatchComplete (L685): `count % 3 == 0` -> batch_complete |
| 13 | "three models review independently" | FinalReviewOpus, FinalReviewGPT, FinalReviewGemini (3 nodes in parallel fan-out) |
| 13 | "each model critiques the other two reviews" | 6 critique nodes: CritiqueOpusOnGPT, CritiqueOpusOnGemini, CritiqueGPTOnOpus, CritiqueGPTOnGemini, CritiqueGeminiOnOpus, CritiqueGeminiOnGPT |
| 13 | "Opus reads everything and decides: ship, rework, or give up" | ReviewConsensus: no class -> Opus; edges to VerifyTestsFinal (ship), CheckReworkBudget (rework), FailureSummary (give up) |
| 14 | "merge locally, push a PR, keep the branch, or throw it away" | FinishGate edges: MergeLocal, CreatePR, KeepBranch, DiscardConfirm -> DiscardWork |
| 72-77 | Escalation: retry -> debug -> replan -> ask you | `default_max_retry=2`; ImplementTask -> DebugInvestigate -> ReplanTask -> HumanHelp |
| 84 | "Plan validation: 5 iterations" | ValidatePlanFormat counter `> 5` |
| 85 | "Review -> re-implement: 5 per task" | CheckImplementBudget counter `> 5` |
| 86 | "Final review rework: 2 full cycles" | CheckReworkBudget counter `> 2` |
| 86 | "Writes a failure summary and stops" | FailureSummary -> Exit |
| 89 | "Counters reset on each new run" | VerifyBaseline (L446) runs `rm -f .tracker/plan_validate_count .tracker/rework_count .tracker/impl_count_* .tracker/batch_count` |
| 93-100 | Artifact file names | All 5 files (brainstorm.md, design-brief.md, plan.md, plan-audit.txt, current_task_id.txt) confirmed in DOT prompts |
| 101 | "Pipeline state lives in `.tracker/`. Gitignored." | `.tracker/` used for counters; `.gitignore` setup includes `.tracker/` (L431) |
| 63-68 | Model assignment table | `model_stylesheet` confirms: `*` -> Opus 4.6, `.implement` -> GPT-5.4, `.draft` -> GPT-5.2, `.opinion` -> Gemini 3.5 Flash |

## Pattern Summary

| Pattern | Count | Root Cause |
|---------|-------|------------|
| Stale line count | 2 occurrences | Line count grew with recent commits, README not updated |
| Setup phase overclaim | 1 occurrence | Usage section correctly shows user does `git init`, but phase description claims pipeline does it |

## Human Review Queue

- [ ] Line 33: "the pipeline auto-detects" language toolchain — the pipeline reads the tech stack from `plan.md` (which was generated from the idea), and `VerifyBaseline` detects language for verification. "Auto-detects" is defensible but slightly misleading. Consider "the plan specifies the stack and the pipeline sets it up."
