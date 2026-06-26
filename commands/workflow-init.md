---
description: Scaffold the agentic Kanban workflow into the current Obsidian vault — creates config.yaml (with a machine block for this host), the board, the task-notes folder, copies templates, then runs the health check. Safe to re-run (never overwrites existing files).
allowed-tools: Bash Read Write Edit Skill AskUserQuestion
---

# Initialize the agentic Kanban workflow in this vault

Scaffold everything the workflow needs into the **current vault** (the directory Claude Code was
launched in — treat its root as `<vault>`). The reusable logic (skills/agents) comes from the
installed plugin; this command creates only the per-vault / per-machine files. **Idempotent:** never
overwrite an existing file — create only what's missing, and report what you did.

The plugin's bundled data lives at `${CLAUDE_PLUGIN_ROOT}` — copy from there.

## Steps

1. **Detect the machine.** Hostname (`$env:COMPUTERNAME` on Windows / `hostname` elsewhere) and OS.

2. **Create folders** (if missing): `<vault>/.claude/agentic-workflow/templates`,
   `<vault>/Tasks/Agentic`, `<vault>/Kanban`.

3. **Templates + docs** (copy if missing):
   - `${CLAUDE_PLUGIN_ROOT}/agentic-workflow/templates/task-note.md` → `<vault>/.claude/agentic-workflow/templates/task-note.md`
   - `${CLAUDE_PLUGIN_ROOT}/agentic-workflow/SETUP.md` → `<vault>/.claude/agentic-workflow/SETUP.md`

4. **Board** (copy if missing):
   - `${CLAUDE_PLUGIN_ROOT}/agentic-workflow/board-template.md` → `<vault>/Kanban/Agentic Workflow Board.md`

5. **Config.**
   - If `<vault>/.claude/agentic-workflow/config.yaml` does **not** exist, copy
     `${CLAUDE_PLUGIN_ROOT}/agentic-workflow/config.example.yaml` there.
   - Then ensure the `machines:` map has a block for **this hostname**. If absent, ask the user
     (AskUserQuestion or prompt) for:
     - `reposRoot` — absolute path to the folder containing their git repos
       (suggest `~/Documents/GitHub` on Windows, `~/GitHub` on macOS).
     - `worktreeRoot` — absolute path **outside** reposRoot (suggest a sibling like `~/agent-worktrees`).
     - `review.provider` — `github-gh` | `gitlab-mcp` | `none` (default `github-gh`).
     Insert a block under `machines:` keyed by the hostname with `os`, `reposRoot`, `worktreeRoot`,
     `git: { client: cli }`, and `review: { provider: <chosen> }`. Use forward slashes in paths.

6. **Health check.** Invoke the `workflow-doctor` skill and show its report.

7. **Summary.** Print what was created/skipped and the next steps:
   ```
   /workflow-doctor      # re-run until READY
   /loop 15m /backlog-manager
   ```
   Remind them: install `gh` + `gh auth login` if `review.provider: github-gh`, and connect the Jira
   MCP if they use Jira as the source.

Keep output concise. Do not start the loop yourself.
