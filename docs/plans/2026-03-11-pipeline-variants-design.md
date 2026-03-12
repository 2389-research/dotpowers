# Pipeline Variants Design

## Overview

Three DOT pipeline variants forked from dotpowers.dot, each adding specific capabilities from the test-kitchen and scenario-testing methodologies.

## Variants

### 1. test-kitchen.dot (dotpowers + test-kitchen)

Adds always-parallel exploration at two points:

**Omakase (Phase 0 — replaces brainstorm):**
- Fan-out to 3 agents (Opus, GPT-5.4, Gemini) each independently exploring the idea
- Each produces a full design brief
- Fan-in: Opus consolidates the 3 briefs, cherry-picking the best architectural ideas
- Human still approves the consolidated brief via y/n gate

**Cookoff (Phase 3 — replaces single-agent implement):**
- For each task from the plan, fan-out to 3 parallel agents
- Each reads the same task spec, writes their own test + implementation
- Fan-in: run tests on all 3, pick the one with best results
- If multiple pass, Opus picks based on code quality
- Winning implementation gets committed, losers discarded

### 2. scenario-testing.dot (dotpowers + scenario testing)

Adds a scenario validation phase between implementation and final review:

**Scenario Validation (Phase 4.5 — after all tasks complete):**
- Write end-to-end scenarios in `.scratch/` using real dependencies
- Zero mocks allowed — real APIs, real storage, real auth (sandbox/test mode OK)
- Run all scenarios, all must pass
- Extract robust patterns to `scenarios.jsonl`
- `.scratch/` stays in `.gitignore`
- Gate: scenarios must pass before proceeding to final review

### 3. kitchen-sink.dot (all of the above)

Combines omakase + cookoff + scenario testing.

## Pipeline Configuration

- **Models:** Same as dotpowers — Opus 4.6 (reasoning), GPT-5.4 (implementation), Gemini 3.5 Flash (third opinion)
- **Human Gates:** Same as dotpowers — y/n for design approval, batch checkpoints, finish gate
- **Retry Strategy:** Same as dotpowers — graduated retry with debug/replan/human escalation
- **Parallelism:**
  - Omakase: 3-way fan-out for design exploration (component → tripleoctagon)
  - Cookoff: 3-way fan-out per task for implementation (component → tripleoctagon)
  - Scenario testing: linear (scenarios run sequentially for deterministic output)
