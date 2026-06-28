# Agentic Kanban Workflow — setup & run

A self-running dev pipeline driven by an Obsidian-Kanban board. The Backlog Manager scans the board
on an interval, spawns Developer + Code Reviewer subagents per ticket, and keeps everything in task
notes. Runs **interactively via `/loop`** (not headless — see "Why not headless").

## Pieces
**Provided by the plugin** (installed globally): skills `backlog-manager` / `kanban-board` /
`workflow-doctor`, agents `developer` / `code-reviewer`, command `workflow-init`.

**Scaffolded into your vault** by `/workflow-init`:
- `.claude/agentic-workflow/config.yaml` — hostname-keyed config (edit per machine)
- `.claude/agentic-workflow/templates/task-note.md` — per-ticket durable record
- `Kanban/Agentic Workflow Board.md` — the board
- `Tasks/Agentic/tickets/` — per-ticket notes (old Done tickets move to `tickets/archive/`)
- `Tasks/Agentic/logs/manager-log.md` — board-wide operational log (created on first use)

## Install on any machine — step by step
1. **Prerequisites:**
   - Obsidian + the **obsidian-kanban** plugin installed and enabled.
   - **git** installed; `git config --global user.name` and `user.email` set.
   - Your repos cloned under one **repos-root** folder.
   - If `review.provider: github-gh`: **gh** installed and `gh auth login` completed.
   - Jira/Atlassian MCP connected in this Claude Code instance (only if you use Jira as the source).
2. **Install the plugin** (run Claude Code from your vault):
   ```
   /plugin marketplace add mgitgrullon/obsidian-agentic-kanban
   /plugin install obsidian-agentic-kanban@obsidian-agentic-kanban
   ```
3. **Scaffold the vault:** `/workflow-init` — creates `config.yaml` (with a block for this host),
   the board, the task-notes folder, and copies the template. Answer the prompts (reposRoot,
   worktreeRoot, provider).
4. **Preflight:** `/workflow-doctor` — fix every ❌ until it prints **READY**.
5. **Create tickets:** add cards to the board's `Backlog` column. Each card must include a **git repo
   link** (resolved to a local repo) and, if applicable, a **Jira link** (read-only). No repo link →
   bounced with `#needs-human`.
6. **Queue work:** move cards `Backlog → Ready`.
7. **Start the loop:** `/loop 15m /backlog-manager`
   (10m is also fine — idle ticks are cheap. Lanes: Backlog → Ready → Blocked → In Progress →
   In Review → Done.)
8. **Stop:** interrupt the loop in the session.

## Day-to-day
- The manager moves cards and writes task notes; you review/merge PRs (full-auto opens them
  ready-for-review). Anything it can't handle lands in `Blocked` with `#needs-human`.
- Watch `Tasks/Agentic/logs/manager-log.md` for a per-tick summary.

### Two kinds of ticket
- **Code ticket (default):** branch in a worktree → review → push → PR. Card flows to `In Review` → `Done`.
- **Artifact ticket:** add the `#artifact` tag to the Backlog card. The agent writes a NEW untracked
  file into the project's checkout (default `docs/`, or a path you name in the ticket) — **no branch,
  commit, or PR** — then moves the card straight to `Done` with a `📄 <path>` marker. This is the only
  case the agent writes into your real checkout, and only ever as a new untracked file. To iterate,
  add a note in `## Human response` and drag the card back to `Ready`; it regenerates.

### Priority & dependencies
- **Priority = `Ready` lane order.** The manager works `Ready` top-down, so drag the most important
  card to the top. (Jira priority, if imported, is saved in the note as a hint, but lane order wins.)
- **Dependencies.** Set `dependsOn: [KEY, …]` in a ticket's note (or let `/jira-to-backlog` import Jira
  "is blocked by" links). The manager won't start a ticket until every dependency reaches `Done`; while
  it waits, the card shows `⏳ waiting on <dep>` and the manager moves on to the next unblocked card.

### Ticket type tags (optional, for code tickets)
Tag a card `#feat`, `#bug`, `#chore`, `#docs`, or `#refactor` to set the conventional-commit prefix
and branch type (`#bug`→`fix:`, `#feat`→`feat:`, …). **You don't have to tag** — with `autoClassify`
on, the manager reads the ticket's intent and adds the right tag (artifact vs code, and which type)
before handing it to the developer, logging `auto-classified as <tag> (retag to override)` in the note.
If it can't tell artifact-vs-code, it asks via `#needs-human` instead of guessing.

### Answering a blocked ticket (human-in-the-loop)
When a card is in `Blocked` it shows `#needs-human` and a `❓ <short question>` on the card:
1. Open the card's task note → read the full question under `## Needs human — agent's question`.
2. Write your answer under `## Human response`.
3. Drag the card `Blocked → Ready`.
On the next tick the developer reads your answer (treating it as a requirement), the manager clears
the `❓`/`#needs-human` from the card, and work resumes — no need to repeat anything in chat.

### Requesting changes on a PR (pull it back from In Review)
Want more work on a PR that's already in `In Review`?
1. Write what you want changed under `## Human response` in the task note.
2. Tag the card `#rework` (leave it in `In Review`).
On the next tick, Loop 2 revises the **existing branch** and pushes to the **same PR** (briefly showing
the card in `In Progress`), then clears `#rework` and returns it to `In Review`.

> ⚠️ Do **not** move a reviewed card to `Ready` — that's treated as a brand-new ticket and would cut a
> fresh branch and ignore the open PR. Use the `#rework` tag instead.

## Permissions (for unattended runs)
A `/loop` running unattended **stalls on permission prompts**, so allow-list what it needs in your
vault's `.claude/settings.local.json` (per-user, gitignored — **not** shipped in the plugin). A
hands-off setup allows `Bash`, `WebFetch`, `WebSearch`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, plus
the **read-only** Atlassian/Jira MCP tools (`getJiraIssue`, `fetch`, `search`, …). Keep Jira **writes**
off the list so a stray write stalls instead of running. The workflow's own rails (worktree isolation,
never-push-to-main, artifact = new files only) apply regardless of permissions.
`/workflow-init` offers to create this file for you — it asks which scope (hands-off / scoped / skip).

## Why not headless (for now)
Decided to run interactively because, under `claude -p` on a scheduler:
- the **Jira/Atlassian MCP may be unavailable** (interactive auth),
- **overlapping scheduled runs would break the single-writer board guarantee** (must enforce no-overlap),
- **non-interactive can't prompt for permissions** (everything must be pre-allowlisted).
Graduate to headless later only after verifying Jira works headless, enforcing no-overlap, and
allowlisting the needed tools.

> Tip: no `gh` yet? Set `review.provider: none` to dry-run — the agent pushes the branch and stops at
> `#needs-human` for the PR step until a provider is configured.
