# Agent Pipeline

A Claude Code plugin that turns one request into a finished, reviewed result by passing it
through a chain of specialised subagents.

```
request ──► planner ──► coder ──► tester ──► reviewer ──► you approve
             (opus)    (sonnet)   (sonnet)    (opus)
                          ▲                      │
                          └──── REVISE (max 2) ──┘
```

Each stage is a focused subagent with only the tools its role needs. Stages never talk to each
other directly: each one reads the files earlier stages wrote and writes exactly one file of its
own. The pipeline stops at **gates** (open questions, failing tests) instead of guessing, and an
independent reviewer can send work back for a bounded number of fix rounds.

The code pipeline is just one workflow. The engine runs **any** workflow you describe in a short
Markdown file: writing, research, content production, release checklists.

- [Install](#install)
- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [How it keeps runs efficient](#how-it-keeps-runs-efficient)
- [How it protects output quality](#how-it-protects-output-quality)
- [Write your own workflow](#write-your-own-workflow)
- [Reference](#reference)

---

## Install

In Claude Code:

```
/plugin marketplace add hoangducccc/agent-pipeline
/plugin install agent-pipeline@agent-pipeline
```

Or from a shell: `claude plugin marketplace add hoangducccc/agent-pipeline`, then
`claude plugin install agent-pipeline@agent-pipeline`.

Update later with `/plugin marketplace update agent-pipeline`.

## Quick start

```bash
cd your-project
git checkout -b add-vat        # the code workflow refuses to run on main/master
claude
```

```
/agent-pipeline:ship add a calculate_vat(price) function that adds 10% VAT, rounded to 2 decimals; negative prices raise ValueError
```

When it finishes you get a 3–5 line summary and these files:

```
.pipeline/
├── request.md        # your request, verbatim
├── plan.md           # planner: files, signatures, edge cases, test command
├── changes.md        # coder: what changed and where to look
├── test-results.md   # tester: command run, pass/fail
└── review.md         # reviewer: VERDICT + reasons
```

Nothing is committed, merged or pushed. You read `review.md`, look at the diff, and decide.

Tip: put your test command in the project's `CLAUDE.md` or `README.md`
(e.g. `Test command: npm test`) so the planner and tester never have to guess.

---

## How it works

### The three parts

| Part | What it is | What it does |
|---|---|---|
| **Conductor** | Your main Claude Code session, following the `run` skill | Reads the workflow, assigns each stage, checks handoff files, applies gates and verdicts. Never does a stage's work itself |
| **Stages** | Subagents, each started fresh | Do one job, read the files they are pointed to, write exactly one file |
| **Workflow file** | A Markdown file | Declares the stages in order: who runs each one, what it reads and writes, when to stop |

### One run, step by step

**Stage 0: preflight (conductor)**
1. Finds the workflow file: project `.claude/workflows/`, then `~/.claude/workflows/`, then the
   ones bundled with the plugin.
2. Runs the workflow's checks. For `code`: inside a git repo with at least one commit, and not on
   `main`/`master`. A failed check stops the run with the command that fixes it.
3. Moves the previous run's files to `.pipeline/history/<timestamp>-<workflow>/` (the last 10
   runs are kept), adds `.pipeline/` to `.gitignore`, and writes your request to
   `.pipeline/request.md`.

**Each stage (conductor → subagent → conductor)**
1. **Assign.** The conductor starts the stage's subagent with a short assignment: where the
   request is, the **paths** of the files to read, and the **path** of the one file to write.
2. **Work.** The subagent opens those files itself, does its job with only the tools its role
   allows, and writes its output file.
3. **Check the handoff.** The conductor confirms the output file exists and is not empty. If it
   is missing, the stage gets one retry, told exactly which file is missing; then the run stops.
4. **Apply gates.** The conductor reads the output and checks the stage's `stop-if` condition and
   `verdict` line (below). Then it moves on, loops back, or stops.

**Finish (conductor)**
A 3–5 line summary covering what was produced, how each gate went, the final verdict and the
fix rounds used. No commits, pushes, pull requests or publishing.

### How stages know which files to use

Stages never message each other. Everything passes through files in the handoff directory
(`.pipeline/` by default):

```
request.md ──► planner ──► plan.md ──► coder ──► changes.md ──► tester ──► test-results.md
                                                                                  │
         review.md ◄── reviewer ◄── (plan.md + changes.md + test-results.md + git diff)
```

Two layers make this reliable:

1. **The assignment names the paths.** The workflow declares `reads:` and `writes:` for every
   stage, and the conductor puts those exact paths into each assignment.
2. **The bundled agents have defaults.** Each agent file says "the file named in your
   assignment (default `.pipeline/plan.md`)". It follows the assignment when the assignment names
   a file, and the defaults when it does not.

### Gates, verdicts and fix rounds

| Mechanism | Where | Effect |
|---|---|---|
| `stop-if` | After planner | Plan has `## OPEN QUESTIONS` → stop and show you the questions |
| `stop-if` | After tester | Any test fails, or no test command found → stop and show the failures |
| `verdict` | After reviewer | First line `VERDICT: APPROVE` → done |
| | | `VERDICT: REVISE` → back to the coder with `review.md` as the to-do list, then tester and reviewer again |
| | | `VERDICT: REJECT` → stop; the approach is wrong and needs a human |
| `max-fix-rounds` | Workflow frontmatter | Caps REVISE loops (2 for `code`). When it runs out, the run stops and reports |

On a fix round the coder is told it is fix round N and fixes **only** the points in
`review.md`. It then adds a `## Fix round N` section to `changes.md`. The files from the previous
round go to `.pipeline/history/.../round-N/`, so the full trail is kept.

---

## How it keeps runs efficient

The pipeline spends tokens and model time where they buy the most, and avoids paying twice for
the same context.

**Each model does the work it is suited to**

| Work | Model | Why |
|---|---|---|
| Planning, reviewing | `opus` | Short outputs whose quality decides everything downstream. A good plan prevents rework, and a sharp review catches what tests miss |
| Implementing, testing | `sonnet` | Larger volume of well-specified work. Following a tight plan does not need the strongest model |
| Counting and cross-checking (your workflows) | `haiku` | Mechanical comparison and listing |

The model is pinned in each agent file, so it stays the same however your main session is
configured. Because the conductor only coordinates, you can also run your main session on a
cheaper model (`/model sonnet`) without changing what each stage uses.

**Each stage starts with a clean context**
- Every stage is a fresh subagent. It does not carry the conversation history or the other
  stages' reasoning, only the files it is pointed to.
- The conductor passes **paths, never file contents**. Nothing is pasted twice, and the main
  session stays small however long the run is.
- The detail lives in the handoff files. The conductor reads only what it needs to apply a gate
  (the open-questions section, the test outcome, the verdict line).

**Stop early instead of redoing work**
- Ambiguity is caught at the cheapest point: the planner stops on open questions **before** any
  code is written.
- A failing test stops the run **before** a reviewer spends time on broken code.
- Fix rounds are targeted: the coder fixes only what `review.md` lists instead of redoing the
  whole task, and the number of rounds is capped so a run cannot loop indefinitely.

**Scope is held tight**
- The planner is told not to add requirements nobody asked for.
- The coder is told not to refactor or tidy unrelated code.
- Smaller diffs are faster to write, test and review.

---

## How it protects output quality

Several independent checks have to agree before a result reaches you.

**1. Separation of roles, enforced by tool permissions**

Roles are limited by the `tools:` line in each agent file. This is a permission boundary, not a
polite request:

| Agent | Cannot | So it cannot |
|---|---|---|
| planner | edit files or run commands | "just start coding" and skip the thinking |
| tester | change product code (by instruction) | edit the code until the tests pass |
| reviewer | edit anything except its verdict file; Bash for read-only git commands only | fix what it reviews and then approve its own fix |

The person who builds is never the person who checks.

**2. A plan the coder must follow**

The plan lists exact files, function signatures, edge cases and the conventions to copy (naming
the file to copy each one from). The coder reads only the plan, so gaps are caught early and the
result matches the codebase's existing style.

**3. Tests designed to catch real bugs**

The tester must cover three groups: the happy path, every edge case the plan names, and **at
least one case that must fail**. It tests behaviour, not internals, and it runs the project's own
test command. A failing test is a hard stop, never something to work around.

**4. An independent reviewer that reads the actual diff**

The reviewer does not trust the summaries. It reads `git status`, `git diff` and, in full, every
new untracked file (which `git diff` does not show). Then it answers three questions:
- Does the code match the plan?
- Do the tests have real value, or were they written for show?
- Are there security, performance or correctness problems?

Green tests with wrong code still get `REVISE` or `REJECT`.

**5. No guessing**

Every stage is told to stop and surface questions rather than assume. The planner turns ambiguity
into `## OPEN QUESTIONS`, the coder refuses to start while questions remain, and an unreadable
verdict counts as `stop`.

**6. An audit trail**

Every run and every fix round leaves its files in `.pipeline/history/`. You can see what was
planned, what changed, what was tested, why it was approved, and how earlier rounds differed.

**7. A human signs off**

The pipeline produces a verdict, not a merge. Committing, pushing and publishing stay with you.

---

## Write your own workflow

A workflow is a Markdown file. The engine looks for it in this order (first match wins):

1. `.claude/workflows/<name>.md`: in your project (commit it with the project)
2. `~/.claude/workflows/<name>.md`: your personal workflows, available in every project
3. Bundled with the plugin: `code`, `writing`

The fastest way to start is `/agent-pipeline:run new <name>`. Claude asks a few questions about
your process and writes the file.

Example: a research brief.

```markdown
---
name: research
description: Question → scope → brief → fact-check
handoff-dir: .pipeline
preflight: []
max-fix-rounds: 1
---

## Stage scope
- model: opus
- role: |
    You scope research questions. List the sub-questions to answer and what a good source
    looks like. Put anything the user must decide under "## OPEN QUESTIONS".
- reads: [request.md]
- writes: scope.md
- stop-if: the file has a non-empty "OPEN QUESTIONS" section

## Stage brief
- model: sonnet
- role: |
    You write a research brief that answers every sub-question in the scope, citing sources.
- reads: [scope.md]
- writes: brief.md

## Stage fact-check
- model: opus
- role: |
    You are an independent fact-checker. Do not rewrite the brief. First line must be
    VERDICT: APPROVE, VERDICT: REVISE or VERDICT: REJECT, followed by specific issues.
- reads: [scope.md, brief.md]
- writes: fact-check.md
- verdict: {APPROVE: continue, REVISE: back-to brief, REJECT: stop}

## Finish
Give the user the path to brief.md and the fact-checker's notes.
```

Run it with `/agent-pipeline:run research <your question>`.

### Designing a good workflow

- **One role, one output file per stage.** If a stage needs two outputs, it is probably two stages.
- **Put a gate where ambiguity is likely**, usually right after the first stage.
- **End with an independent check** that has a `verdict` and cannot edit what it checks.
- **Strong model to decide or judge, faster model to execute.**
- **Use `agent` when a limit must be enforced.** A `role` constrains a subagent only through its
  instructions. When a stage must be unable to do something, such as editing code or running
  commands, give it its own agent file in `.claude/agents/` with a restricted `tools:` line and
  reference it with `agent:`.

---

## Reference

### Commands

| Command | What it does |
|---|---|
| `/agent-pipeline:ship <request>` | Run the code workflow (shortcut for `run code`) |
| `/agent-pipeline:run <workflow> <request>` | Run any workflow |
| `/agent-pipeline:run list` | List available workflows and where they live |
| `/agent-pipeline:run new <name>` | Interview you and write a new workflow file |

### Bundled agents

| Agent | Model | Tools | Role |
|---|---|---|---|
| `agent-pipeline:planner` | opus | Read, Grep, Glob, Write | Plans; cannot edit or run code |
| `agent-pipeline:coder` | sonnet | Read, Write, Edit, Grep, Glob, Bash | Implements the plan, nothing more |
| `agent-pipeline:tester` | sonnet | Read, Write, Edit, Grep, Glob, Bash | Writes and runs tests; may only touch test files |
| `agent-pipeline:reviewer` | opus | Read, Grep, Glob, Bash, Write | Read-only review; writes only its verdict |

### Bundled workflows

| Workflow | Stages | Fix rounds |
|---|---|---|
| `code` | planner → coder → tester → reviewer | 2 |
| `writing` | outline (opus) → draft (sonnet) → edit (opus) | 1 |

### Workflow frontmatter

| Field | Default | Meaning |
|---|---|---|
| `name` | required | Same as the file name |
| `description` | required | One line, shown by `run list` |
| `handoff-dir` | `.pipeline` | Where handoff files live, relative to the current directory |
| `preflight` | `[]` | `git-repo`, `feature-branch`, `clean`, or any plain-language check |
| `max-fix-rounds` | `0` | Total `back-to` loops allowed per run |

### Stage fields

| Field | Meaning |
|---|---|
| `agent` | Subagent to use. Bundled ones need the `agent-pipeline:` prefix; your own agents in `.claude/agents/` work by plain name |
| `role` + `model` | Instead of `agent`: run a `general-purpose` subagent with this role text and model |
| `reads` | Files the stage reads. Bare names live in the handoff dir; names with `/` are project paths |
| `writes` | The single file the stage must produce, in the handoff dir |
| `stop-if` | Plain-language condition on the stage's output that stops the run for a human |
| `verdict` | Map from `VERDICT: <VALUE>` (first line of the output) to `continue`, `back-to <stage>` or `stop` |
| `confirm` | `yes` → ask the user before running this stage |
| `parallel-with` | Run alongside the named stage instead of after it |
| `notes` | Extra instructions passed to the stage |

Full engine specification:
[`plugins/agent-pipeline/skills/run/SKILL.md`](plugins/agent-pipeline/skills/run/SKILL.md).

### Troubleshooting

| Symptom | Fix |
|---|---|
| Commands don't appear | `claude plugin list` should show `agent-pipeline@agent-pipeline` enabled; then `/reload-plugins` |
| "refuses to run on main" | `git checkout -b <branch>` and run again |
| Tester reports a missing test command | Add `Test command: <cmd>` to the project's `CLAUDE.md` or `README.md` |
| Run stops with open questions | Answer them by re-running with a more detailed request |
| Want different agents for one project | Add `.claude/workflows/code.md` to the project, pointing at your own agents |

## License

MIT
