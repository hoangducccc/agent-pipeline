---
name: ship
description: Run the full code pipeline (planner → coder → tester → reviewer) for a feature request. Shortcut for /agent-pipeline:run code.
argument-hint: <feature request>
---

This is a shortcut for the `code` workflow. Read `${CLAUDE_PLUGIN_ROOT}/skills/run/SKILL.md` and
follow it exactly with the arguments:

code $ARGUMENTS

(If the project has `.claude/workflows/code.md`, that file takes precedence, per the lookup order
in section 1.)
