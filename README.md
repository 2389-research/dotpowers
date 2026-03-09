# dotpowers

A single DOT file that turns a one-line project idea into working, tested, reviewed software — autonomously.

dotpowers encodes the [superpowers](https://github.com/obra/superpowers) development methodology as a pipeline graph. It brainstorms a design with you, writes a TDD implementation plan, builds the project task-by-task, reviews the result through multi-model consensus, and presents shipping options. The pipeline runs on [tracker](https://github.com/2389-research/tracker), a DAG runner that executes LLM nodes, tool nodes, and human gates defined in Graphviz DOT format.

## What it does

Given a working directory containing an `idea.md` (even one sentence), dotpowers will:

1. **Brainstorm** the design with you — explores context, asks clarifying questions one at a time, proposes approaches, writes a design brief, and gets your approval before proceeding
2. **Plan** the implementation — GPT-5.2 drafts a strict TDD plan, a format validator catches vague instructions, Opus audits every requirement against the design brief, and GPT-5.2 patches any gaps (up to 3 iterations)
3. **Set up** the project — initializes git, installs dependencies, verifies a clean baseline
4. **Implement** each task with TDD — writes a failing test, verifies the failure, writes minimal code to pass, verifies green, commits. Every task gets a spec compliance review (Opus) and a code quality review (GPT-5.4) before it can be marked complete
5. **Review** the complete project — three models review independently, then cross-critique each other's reviews, then Opus synthesizes a consensus verdict
6. **Ship** — presents four options: merge locally, push and create a PR, keep the branch, or discard

## What it does not do

- **It is not a general-purpose agent.** It follows a fixed 6-phase pipeline. You cannot ask it to "fix this bug" or "add a feature to an existing project" — it builds new projects from scratch.
- **It does not run without human interaction.** The brainstorm phase requires your input to approve the design. The finish phase requires you to choose how to ship. If the pipeline gets stuck, it escalates to you.
- **It does not guarantee working software.** LLMs make mistakes. The multi-model review and TDD discipline catch most issues, but the pipeline can still produce bugs, especially in complex projects.
- **It does not manage infrastructure.** No deployment, no CI/CD, no cloud resources. It produces code in a git repository.
- **It does not work with existing codebases.** It expects a greenfield directory with an idea doc. It will not refactor, extend, or debug existing projects.
- **It is not cheap to run.** A single pipeline run uses Opus 4.6, GPT-5.4, GPT-5.2, and Gemini 3.5 Flash across dozens of nodes. Expect significant API costs, especially if loops iterate.

## Requirements

- [tracker](https://github.com/2389-research/tracker) — the pipeline runner
- API keys for OpenAI, Anthropic, and Google (set in `.env` or environment)
- `git` installed and available on PATH
- Language toolchains as needed (Rust, Python/uv, Node, Go — the pipeline detects project type automatically)

## Usage

```bash
# Create a project directory with an idea
mkdir my-project && cd my-project
echo "a terminal dashboard that shows system metrics" > idea.md
git init

# Run the pipeline
tracker /path/to/dotpowers.dot --tui
```

The `--tui` flag gives you an interactive terminal UI showing pipeline progress. When the pipeline reaches a human gate (brainstorm questions, design review, finish options), it will prompt you directly.

### Resuming a run

If a run is interrupted, tracker can resume from its last checkpoint:

```bash
tracker /path/to/dotpowers.dot --tui --resume
```

## Pipeline architecture

```
Brainstorm ──► Plan ──► Setup ──► Implement ──► Final Review ──► Finish
    │            │                    │              │
    ▼            ▼                    ▼              ▼
  Human      Draft/Audit/         TDD loop:       3 models ×
  approval   Patch loop        implement →       3 reviews +
             (max 5 iter)     spec review →     6 cross-critiques →
                             quality review →      consensus
                             mark complete
```

### Models

| Role | Model | Why |
|------|-------|-----|
| Reasoning, spec audit, consensus | Claude Opus 4.6 | Strongest at line-by-line verification |
| Implementation (TDD cycles) | GPT-5.4 | Fast code generation with tool use |
| Plan drafting, plan patching | GPT-5.2 | Structured output, cost-effective |
| Third opinion (reviews, critiques) | Gemini 3.5 Flash | Independent perspective, low cost |

### Escalation chain

When something fails, the pipeline escalates through four levels before asking for help:

1. **Node retry** (max 2) — retry the same node with error context
2. **Debug investigation** (max 2) — Opus does root cause analysis
3. **Replan task** (max 1) — rethink the approach for this task
4. **Human help** — escalate to you with full context

### Loop budgets

Every cycle in the pipeline has a hard termination limit to prevent runaway token burn:

| Loop | Budget | Escalation |
|------|--------|------------|
| Plan validation (DraftPlan ↔ ValidatePlanFormat) | 5 iterations | → Human help |
| Implement retry (review fail → ImplementTask) | 5 per task | → Human help |
| Final review rework (ReviewConsensus → ImplementTask) | 2 rework cycles | → Failure summary |

## Files produced

The pipeline writes all artifacts into `docs/plans/` in the working directory:

| File | Contents |
|------|----------|
| `docs/plans/brainstorm.md` | Brainstorm notes and decisions |
| `docs/plans/design-brief.md` | Approved design specification |
| `docs/plans/plan.md` | TDD implementation plan with task checkboxes |
| `docs/plans/plan-audit.txt` | Opus audit results (APPROVED or gap list) |
| `docs/plans/current_task_id.txt` | Currently executing task ID |

Pipeline state lives in `.tracker/` (run checkpoints, loop counters). This directory is gitignored.

## Known limitations

- **No parallel task execution.** Tasks run sequentially even when independent. The plan labels tasks as sequential but the pipeline does not currently fan out implementation.
- **Language detection is heuristic.** The pipeline checks for `pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml` in that order. Projects using other build systems need manual setup.
- **No incremental runs.** You cannot add tasks to a completed plan and resume. Each run is a full pipeline execution from brainstorm to finish.
- **Human gates block the pipeline.** When waiting for your input, no other work progresses. Keep the TUI open or the pipeline will sit idle.

## Development

The pipeline definition lives in a single file:

```
dotpowers.dot    — the complete pipeline (~1150 lines of Graphviz DOT)
docs/plans/      — design documents and iteration history
```

To validate the DOT file syntax:

```bash
dot -Tsvg dotpowers.dot > /dev/null  # graphviz syntax check
```

To generate a visual graph:

```bash
dot -Tpng -Gdpi=300 dotpowers.dot -o dotpowers.png
```

## License

[TBD]
