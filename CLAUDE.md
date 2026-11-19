# agent-pipeline

Public Claude Code plugin + marketplace (both named `agent-pipeline`). All content in English.

## Layout
- `.claude-plugin/marketplace.json` — marketplace, lists the plugin at `./plugins/agent-pipeline`
- `plugins/agent-pipeline/.claude-plugin/plugin.json` — manifest; bump `version` on every release
  (users only receive updates when it changes)
- `plugins/agent-pipeline/skills/run/SKILL.md` — the engine; `workflows/` holds bundled workflows
- `plugins/agent-pipeline/skills/ship/SKILL.md` — shortcut for `run code`
- `plugins/agent-pipeline/agents/` — planner, coder, tester, reviewer

## Conventions
- Nothing personal or machine-specific in the plugin (no absolute paths, no personal projects).
- Inside the plugin, reference bundled agents with the `agent-pipeline:` prefix and bundled files
  with `${CLAUDE_PLUGIN_ROOT}/...`.
- Before committing: `claude plugin validate .` and `claude plugin validate plugins/agent-pipeline`.
- Try local changes: `claude --plugin-dir plugins/agent-pipeline`.
