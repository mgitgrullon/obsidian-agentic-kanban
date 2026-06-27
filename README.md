# Obsidian Agentic Kanban

A self-running dev pipeline driven by an [obsidian-kanban](https://github.com/obsidian-community/obsidian-kanban)
board, packaged as a **Claude Code plugin**. A **Backlog Manager** scans the board on an interval and,
per ticket, spawns **Developer** and **Code Reviewer** subagents — with review-before-push, git
**worktree isolation**, durable **task notes**, and a **human-in-the-loop** channel.

```
/loop 15m /backlog-manager  ──►  Backlog → Ready → Blocked → In Progress → In Review → Done → Archive
```

## What you get
- **Skills:** `backlog-manager` (the orchestrator, one tick), `kanban-board` (safe board edits),
  `workflow-doctor` (preflight health check).
- **Agents:** `developer` (implements in phases, review-before-push), `code-reviewer` (gates the branch).
- **Command:** `workflow-init` (scaffolds the per-vault files so it's ready to use).

## Highlights
- **Review before push** — the internal Code Reviewer gates the branch, so the PR is clean before
  CI / cursorbot / humans see it.
- **Worktree isolation** — every ticket works in its own git worktree; same-repo tickets run in
  parallel and your own checkout is never touched.
- **Durable continuity** — all state lives in per-ticket task notes (survives restarts), not session ids.
- **Human-in-the-loop** — blocked tickets ask a question in the note; you answer and drag the card back.
- **Two ticket kinds** — code (→ PR) and `#artifact` (an untracked file delivered into the repo, no PR).
- **Cost-aware** — cheap idle ticks, cached health checks, pinned subagent models.

## Install

Run Claude Code from your Obsidian vault, then:

```
/plugin marketplace add mgitgrullon/obsidian-agentic-kanban
/plugin install obsidian-agentic-kanban@obsidian-agentic-kanban
/workflow-init        # scaffolds config.yaml, the board, Tasks/Agentic, templates (answer the prompts)
/workflow-doctor      # verify — fix until READY
/loop 15m /backlog-manager
```

> Installing the plugin provides the skills/agents/command globally. `/workflow-init` creates the
> per-vault, per-machine files (it never overwrites existing ones, so it's safe to re-run).

## Prerequisites
- Obsidian with the **obsidian-kanban** plugin.
- **git** configured (`user.name` / `user.email`); your repos cloned under one folder ("repos-root").
- **gh** installed + `gh auth login` if you want PRs opened automatically (else set `review.provider: none`).
- An MCP for Jira (Atlassian) only if you use Jira as the read-only ticket source.

## Configuration
`/workflow-init` creates `.claude/agentic-workflow/config.yaml` (hostname-keyed, so one file works
across machines). Key knobs: `reposRoot`, `worktreeRoot`, `review.provider`, `concurrency`,
`maxReviewRounds`, `isolation`, `models`, `artifactTag` / `reworkTag`. See
[`agentic-workflow/config.example.yaml`](agentic-workflow/config.example.yaml) and the full guide in
[`agentic-workflow/SETUP.md`](agentic-workflow/SETUP.md).

## Tags

Tags on the cards drive special handling. The names are configurable (`artifactTag`, `reworkTag`,
`typeTags` in `config.yaml`); defaults below.

| Tag | Who sets it | On which card | What it does |
|-----|-------------|---------------|--------------|
| `#feat` | you (or auto) | a `Backlog` card | Marks a **feature** — sets the commit/branch prefix to `feat:`. |
| `#bug` | you (or auto) | a `Backlog` card | Marks a **bug fix** — sets the commit/branch prefix to `fix:`. (`#chore`/`#docs`/`#refactor` also supported via `typeTags`.) |
| `#artifact` | you (or auto) | a `Backlog` card | Deliver an untracked file (default `docs/`, or a path named in the ticket) — **no branch/commit/PR**. Card goes straight to `Done` with a `📄 <path>` marker. |
| `#rework` | **you** | an `In Review` card | Put your feedback in the note's `## Human response`, then add this tag. Loop 2 revises the **existing branch** and pushes to the **same PR**, then clears the tag. (Don't move a reviewed card to `Ready` — that restarts it fresh.) |
| `#needs-human` | **the agent** | a `Blocked` card | The agent couldn't proceed; the card shows a `❓ <question>` and the full question is in the note's `## Needs human` section. Answer under `## Human response` and drag the card `Blocked → Ready`; the manager clears the tag on resume. |

**Forgot to tag?** With `autoClassify` on (default), the manager reads the ticket's intent and adds the
right tag — artifact vs code, and which type (`#feat`/`#bug`/…) — before handing it to the developer,
logging `auto-classified as <tag> (retag to override)`. If it can't tell artifact-vs-code, it asks via
`#needs-human` rather than guessing.

## Day-to-day
- Create Backlog cards (each with a repo link); move to `Ready` to queue.
- Review/merge the PRs it opens. Need changes? Tag the card `#rework` + note feedback.
- Stuck tickets land in `Blocked` with a `❓` question — answer in the note, drag back to `Ready`.

## License
MIT
