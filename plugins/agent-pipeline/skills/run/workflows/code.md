---
name: code
description: Ship a code change — planner → coder → tester → reviewer, with up to 2 automatic fix rounds when the reviewer asks for revisions
handoff-dir: .pipeline
preflight: [git-repo, feature-branch]
max-fix-rounds: 2
---

## Stage planner
- agent: agent-pipeline:planner
- reads: [request.md, CLAUDE.md, README.md]
- writes: plan.md
- stop-if: the file has a non-empty "OPEN QUESTIONS" section

## Stage coder
- agent: agent-pipeline:coder
- reads: [plan.md]
- writes: changes.md

## Stage tester
- agent: agent-pipeline:tester
- reads: [plan.md, changes.md]
- writes: test-results.md
- stop-if: any test failed, or the file says the test command is missing

## Stage reviewer
- agent: agent-pipeline:reviewer
- reads: [plan.md, changes.md, test-results.md]
- writes: review.md
- verdict: {APPROVE: continue, REVISE: back-to coder, REJECT: stop}

## Finish
Show the user `review.md`. Summarise in 3–5 lines: what was built, number of tests, verdict,
fix rounds used. Do not commit, merge, push or open a pull request.
