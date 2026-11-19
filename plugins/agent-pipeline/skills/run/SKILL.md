---
name: run
description: Run a multi-agent pipeline defined in a Markdown workflow file — sequential stages, file handoffs, stop gates, verdicts and bounded fix loops. Use when the user types /agent-pipeline:run, wants to run, list or create a pipeline workflow.
argument-hint: <workflow> <request> | list | new <workflow>
---

# Pipeline engine

Arguments: $ARGUMENTS

You are the **conductor**. You do NOT do any stage's work yourself (no writing code, scripts,
drafts or documents). You only: read the workflow definition → assign each stage to a subagent →
check its handoff file → decide whether to continue, loop back, or stop.

## 1. Find the workflow

A workflow is a file `<name>.md`. Look in this order and use the first match:

1. `.claude/workflows/<name>.md` in the current project — project-specific workflows
2. `~/.claude/workflows/<name>.md` — the user's personal workflows
3. `${CLAUDE_PLUGIN_ROOT}/skills/run/workflows/<name>.md` — workflows bundled with this plugin

Handle the arguments:
- Empty or `list`: list every workflow in all three places (name + its `description`, where it
  lives, and which ones shadow others). Stop.
- `new <name>`: follow section 6. Stop.
- Otherwise: the first word is the workflow name, the rest is the **request**. If the workflow
  is not found, STOP and list the available ones.

Read the whole definition file. Its syntax is described in section 5.

## 2. Stage 0 — preflight

Let `D` = the `handoff-dir` from the frontmatter (default `.pipeline`).

Run each check in `preflight`:
- `git-repo`: the current directory must be a git repository with at least one commit.
  If not → STOP and suggest `git init` plus an initial commit.
- `feature-branch`: `git branch --show-current` must not be `main` or `master`.
  If it is → STOP and suggest `git checkout -b <branch-name>`.
- `clean`: `git status --porcelain` must be empty. If not → STOP and list the files.
- Anything else is a plain-language condition → check it as written; if it fails, STOP.

Then:
- If this is a git repository and `.gitignore` has no `D/` line, add it.
- Create `D/` if missing. If `D/` contains handoff files from a previous run (`.md` files directly
  inside `D/`), MOVE them to `D/history/<YYYYMMDD-HHMM>-<workflow>/`. Never delete them.
  Keep at most the 10 most recent history folders; delete older ones.
- Write `D/request.md`: workflow name, date and time, and the user's request verbatim.

## 3. Run the stages

Run stages in file order. Never skip, never run in parallel unless a stage says
`parallel-with: <stage>`.

For each stage:

1. **Assign** it with the Agent tool:
   - With `agent: <name>` → `subagent_type` = `<name>`. Agents bundled with this plugin need the
     plugin prefix, e.g. `agent-pipeline:planner`. If the name is not found, retry with the
     `agent-pipeline:` prefix added; if still not found, STOP and tell the user.
   - Without `agent` but with `role` → `subagent_type` = `general-purpose`, `model` = the stage's
     `model` (if given), and put the `role` text at the top of the assignment.
   - The assignment contains: the user's request (point to `D/request.md`), the PATHS of the files
     to read (`reads`), the PATH of the file to write (`writes`), and the stage's `notes` if any.
     Pass paths only — never paste file contents into the assignment.
   - On a fix round (section 4), add: "This is fix round N. Read <reason file> and fix only the
     points it lists."
2. **Check the handoff**: the `writes` file must exist and be non-empty. If missing, re-assign the
   stage once, saying which file is missing; if still missing → STOP.
3. **Evaluate gates** (`stop-if`, `verdict`) — see section 4.

If a stage has `confirm: yes`, summarise what it is about to do in 1–2 lines and ask the user
before assigning it.

## 4. Gates, verdicts and fix loops

- `stop-if: <condition>`: read the stage's handoff file; if the condition holds → STOP and show
  the user exactly the relevant part (e.g. the open questions, the failing tests). Do not resolve
  it yourself.
- `verdict`: the first line of the handoff file reads `VERDICT: <VALUE>`. Each value maps to an
  action:
  - `continue` → go to the next stage (or finish if this is the last one).
  - `back-to <stage>` → fix loop: re-run from `<stage>` through the end, with the fix-round
    instruction from 3.1. The reason file is the handoff file that holds the verdict.
  - `stop` → STOP and show the user the reasons.
  If the verdict cannot be read, treat it as `stop`.
- `max-fix-rounds` (frontmatter, default 0) caps the total number of `back-to` loops in one run.
  When it is used up without reaching `continue` → STOP and report the last verdict and reasons.
- Before each loop, move the handoff files of the stages about to re-run into
  `D/history/<this run's folder>/round-<N>/` so the trail is kept.

## 5. Workflow file syntax

```markdown
---
name: <name>                    # same as the file name
description: <one line>
handoff-dir: .pipeline          # relative to the current directory
preflight: [git-repo, feature-branch]   # may be []
max-fix-rounds: 2               # default 0
---

## Stage <stage-name>
- agent: agent-pipeline:planner # OR use role + model
- model: opus                   # only without agent: opus | sonnet | haiku
- role: |                       # only without agent
    You are ... You do NOT ...
- reads: [request.md, CLAUDE.md]   # bare names live in the handoff dir; paths with / are project paths
- writes: plan.md
- stop-if: the file has a non-empty "OPEN QUESTIONS" section
- verdict: {APPROVE: continue, REVISE: back-to coder, REJECT: stop}
- confirm: yes
- notes: <extra instructions for this stage>

## Finish
<what to do after the last stage: what to summarise, what is forbidden>
```

Choosing `agent` vs `role`: a stage that needs **enforced permissions** (read-only, test files
only, no shell) must use an `agent` whose file restricts `tools:`. A `role` constrains only by
instruction, which suits low-risk stages (ideation, drafting).

## 6. Create a workflow (`new <name>`)

1. Ask the user (one round, at most 4 questions): the goal and final deliverable; the steps they
   do by hand today, in order; which steps need human approval or tend to go wrong; whether it is
   for this project only or for all their projects.
2. Design the stages by these principles:
   - One role per stage, one output file per stage. A stage knows only what earlier stages wrote.
   - Deciding/judging stages use a strong model (opus); executing stages use sonnet;
     checking/counting stages use haiku.
   - End with an independent checking stage that has a `verdict`, and put at least one
     `stop-if` where ambiguity is likely.
   - Reuse bundled agents (`agent-pipeline:planner`, `agent-pipeline:reviewer`, …) when they fit.
3. Write the file: this project → `.claude/workflows/<name>.md`; all projects →
   `~/.claude/workflows/<name>.md`.
4. Show the user a stage table (stage | agent/model | reads | writes | gate) and the command to
   try it.

## 7. Finish

Follow the workflow's `## Finish` section. Always, unless the workflow explicitly says otherwise:
- Summarise in 3–5 lines: what was produced, the outcome of each gate, the final verdict, and the
  number of fix rounds used.
- Do NOT commit, merge, push, open pull requests, or publish anything externally. A human
  approves the result.
