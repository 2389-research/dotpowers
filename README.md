# dotpowers

You write `echo "a dvd bouncing tui" > idea.md`, run the pipeline, and come back to a tested, reviewed Rust project. One DOT file, ~1300 lines.

dotpowers is the [superpowers](https://github.com/obra/superpowers) dev methodology encoded as a pipeline graph. It talks through the design with you, writes a plan, builds the thing with TDD, has three models argue about whether it's any good, and then asks you how to ship it. Runs on any [attractor](https://github.com/strongdm/attractor)-compliant DOT runner ([tracker](https://github.com/2389-research/tracker), [mammoth](https://github.com/strongdm/mammoth), [smasher](https://github.com/strongdm/smasher), etc.) that executes LLM nodes, tool nodes, and human gates from Graphviz DOT files.

## What happens when you run it

1. **Brainstorm** -- reads your idea, asks you questions one at a time (YAGNI enforced), proposes 2-3 approaches with trade-offs, writes a design brief, waits for you to say "looks good"
2. **Plan** -- GPT-5.2 drafts a TDD plan, a shell script rejects vague hand-wavy steps, Opus audits every requirement against the brief, GPT-5.2 patches gaps. Up to 5 iterations before it gives up and asks you.
3. **Setup** -- create a `feature/<project-name-slug>` branch (never implements on main), install deps, run the test suite to confirm nothing is broken before we start
4. **Implement** -- for each task: write a failing test, watch it fail, write minimal code, watch it pass, commit. If the implementer has questions, it pauses at a human gate before proceeding. Opus checks spec compliance, GPT-5.4 checks code quality. Both have to pass before the task is marked done. Every 3rd completed task triggers a batch checkpoint where you review progress before the pipeline continues.
5. **Review** -- three models review the finished project independently. Then each model critiques the other two reviews. Opus reads everything and decides: ship, rework, or give up.
6. **Ship** -- you pick: merge locally, push a PR, keep the branch, or throw it away

## What it won't do

It builds new projects from scratch. That's it. You can't point it at an existing codebase and say "fix this bug" or "add a feature." It expects a directory with an `idea.md` and nothing else.

It does not deploy anything. No CI/CD, no cloud, no Docker. Output is code in a git repo.

It will not run unattended start to finish. You approve the design. You review batch checkpoints every 3 tasks during implementation. You choose how to ship. If the implementer has questions or gets stuck, it asks you.

It is not cheap. A full run hits Opus 4.6, GPT-5.4, GPT-5.2, and Gemini 3.5 Flash across dozens of nodes. If loops iterate (and they will), costs add up. We burned 3.5M OpenAI tokens on a single bad run before adding loop limits.

It does not guarantee the code works. Three models reviewing each other catches a lot, but LLMs still make mistakes.

## Requirements

- An [attractor](https://github.com/strongdm/attractor)-compliant DOT runner (tracker, mammoth, smasher, etc.)
- API keys for OpenAI, Anthropic, and Google (in `.env` or environment)
- git
- Whatever language toolchain your project needs (Rust, Python/uv, Node, Go -- the plan specifies the stack and the pipeline sets it up)

## Usage

```bash
mkdir my-project && cd my-project
echo "a terminal dashboard that shows system metrics" > idea.md
git init
# using tracker:
tracker /path/to/dotpowers.dot --tui

# or mammoth, smasher, etc. -- any attractor-compliant runner
mammoth /path/to/dotpowers.dot
```

## How it's wired

```
Brainstorm --> Plan --> Setup --> Implement --> Final Review --> Finish
    |            |                   |               |
    v            v                   v               v
  Human      Draft/Audit/       TDD loop:        3 reviews +
  approval   Patch loop      write test ->     6 cross-critiques ->
             (max 5)        spec review ->        consensus
                           quality review ->
                            mark complete
```

### Who does what

| Job | Model | Reasoning |
|-----|-------|-----------|
| Spec audits, consensus, debugging | Opus 4.6 | Best at careful line-by-line work |
| Writing code (TDD cycles) | GPT-5.4 | Fast, good with tools |
| Drafting and patching plans | GPT-5.2 | Structured output, cheaper |
| Third opinion on reviews | Gemini 3.5 Flash | Different perspective, cheap |

### When things go wrong

The pipeline tries four things before bothering you:

1. Retry the node (up to 2 times)
2. Debug investigation -- Opus does root cause analysis
3. Replan the task -- rethink the approach
4. Ask you

### Loop limits

Early runs burned millions of tokens in infinite loops. Every cycle now has a hard cap:

| Loop | Cap | Then what |
|------|-----|-----------|
| Plan validation | 5 iterations | Asks you for help |
| Review -> re-implement | 5 per task | Asks you for help |
| Final review rework | 2 full cycles | Writes a failure summary and stops |

Counters reset on each new run but persist across resume, so you won't hit a stale limit from a previous run.

## What it writes

All artifacts go in `docs/plans/` inside your project:

- `brainstorm.md` -- notes and decisions from the brainstorm
- `design-brief.md` -- the approved spec
- `plan.md` -- TDD plan with `- [ ] task-NNN:` checkboxes
- `plan-audit.txt` -- Opus audit (APPROVED or a list of gaps)
- `current_task_id.txt` -- which task is running right now

Pipeline state (checkpoints, loop counters) lives in `.tracker/`. Gitignored.

## The repo

```
dotpowers.dot    -- the pipeline (~1300 lines of DOT)
docs/plans/      -- design docs and iteration history
```

Validate syntax: `dot -Tsvg dotpowers.dot > /dev/null`

Generate a graph image: `dot -Tpng -Gdpi=300 dotpowers.dot -o dotpowers.png`

## Related

- [dot-files](https://github.com/2389-research/dot-files) — Standalone pipelines (speedrun, bug-hunter, refactor-express, doc-writer) and interactive games (20q, story-engine, model-debate) that don't use the superpowers framework
- [tracker](https://github.com/2389-research/tracker) — The DOT pipeline runner we use most
- [mammoth](https://github.com/2389-research/mammoth) — The DOT pipeline format and ecosystem
- [superpowers](https://github.com/obra/superpowers) — The dev methodology this encodes

## License

[TBD]
