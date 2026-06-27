---
name: workflow-doctor
description: Preflight/health check for the agentic Kanban workflow. Verifies git, the repos-root, the board, and pings each configured MCP (Jira, GitHub/GitLab) so you can see what's green before running the backlog manager loop. Use when setting up the workflow on a machine, or to diagnose why the loop won't run.
---

# Workflow Doctor

Run a preflight status check for the agentic Kanban workflow and report a clear
PASS / WARN / FAIL table. The Backlog Manager loop calls this at startup and
**refuses to run if any REQUIRED check FAILs** — so this is the single source of
truth for "is the workflow safe to run on this machine right now?".

This routine is read-only. It MUST NOT modify any repo, the board, or Jira.

## Steps

### 1. Load config + identify the machine
- Read `.claude/agentic-workflow/config.yaml` (relative to the vault root).
- Get the hostname: PowerShell `$env:COMPUTERNAME`, or bash `hostname`.
- Select `machines.<hostname>`. If there is no block for this hostname →
  **FAIL** with: "No config for `<hostname>` — add a machine block to config.yaml."
- Merge `defaults` with the machine block for the values below.

### 2. Checks (build a table as you go)

Mark each: ✅ PASS · ⚠️ WARN · ❌ FAIL. Note which are REQUIRED.

**git (REQUIRED)**
- `git --version` resolves.
- `git config user.name` and `git config user.email` are both set.

**repos-root (REQUIRED)**
- `reposRoot` exists and is a directory.
- Count immediate subfolders containing `.git`; report the count.
- If the vault root is inside `reposRoot` and `safety.excludeVaultAsRepo` is true,
  WARN: "vault sits under repos-root — it is excluded as a dev target."

**worktree-root (REQUIRED when `isolation: worktree`)**
- `worktreeRoot` is set for this machine and is NOT inside `reposRoot` (must be a sibling/elsewhere,
  so worktrees aren't mistaken for repos). Create it if missing; FAIL if its parent isn't writable.

**board (WARN if missing — setup may be incomplete)**
- `boardPath` exists.
- Its `## ` headings match `columns` exactly (order + names).
- The `kanban:settings` JSON `list-collapse` array length equals the number of
  columns. If not → WARN (lane state will be off; fix when creating/editing the board).

**folders (WARN if missing)**
- `taskNotesFolder` and `logsFolder` exist (create-on-first-use is fine; just report).

**git client (REQUIRED)**
- `git.client: cli` → covered by the git check above.
- `git.client: mcp` → confirm the configured git MCP tools are available
  (use ToolSearch); do one light read. FAIL if unavailable.

**review provider (severity depends on value)**
- `github-gh` (REQUIRED): `gh --version` resolves AND `gh auth status` shows logged in.
  - If `gh` missing → FAIL: "run /setup-github (install + `gh auth login`)."
  - If installed but not authed → FAIL: "run `gh auth login`."
- `github-mcp` / `gitlab-mcp` (REQUIRED): confirm the MCP's tools are available
  (ToolSearch) and do one light read. FAIL if unavailable.
- `none` → WARN: "PR creation + CI checks will be left as #needs-human."

**MCP — required (REQUIRED)**
- For each entry in `mcp.required`: load its `check` tool via ToolSearch and call it
  (e.g. Atlassian → `atlassianUserInfo`). PASS if it returns; FAIL otherwise with the
  hint that the MCP may need authentication/reconnection.

**MCP — optional (never fails)**
- For each entry in `mcp.optional`: note whether its tools are present. WARN if absent.

### 3. Report
- Print the table grouped as REQUIRED then OPTIONAL, each row: `status · check · detail`.
- End with one verdict line:
  - All REQUIRED green → `✅ READY — the backlog manager loop can run.`
  - Any REQUIRED red → `❌ NOT READY — fix the FAIL rows above before running /loop.`
- For every non-green row, give the one-line remediation.

Keep the output compact and scannable. Do not attempt fixes automatically — just diagnose.
