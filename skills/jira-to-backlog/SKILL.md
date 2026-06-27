---
name: jira-to-backlog
description: Import one or more Jira tickets (by link or KEY) into the local Kanban board's Backlog. Fetches title/description/type read-only, creates a task note with a Jira snapshot, sets the matching type tag, and adds the card. Use when the user wants to pull Jira issues into the agentic Kanban workflow.
argument-hint: "<jira-url-or-KEY> [more KEYs...] [repo:<git-url>]"
allowed-tools: Read Write Edit Bash Skill ToolSearch AskUserQuestion
---

# Jira → Backlog import

Turn Jira tickets into ready-to-work Backlog cards on the local board. **Read-only on Jira** — never
transition, edit, or comment.

## Input (from `$ARGUMENTS`)
- One or more Jira issue links or keys (e.g. `PROJ-123`, or `https://…/browse/PROJ-123`).
- Optional `repo:<git-url>` applied to all imported tickets this run.

## Steps
1. **Config.** Read `.claude/agentic-workflow/config.yaml` (merge `defaults` + this machine). Note
   `boardPath`, `taskNotesFolder`, the Backlog lane name (`columns[0]`), `typeTags`, and
   `projectRepoMap` (if present).
2. **Parse keys.** Extract each issue KEY from the args (from a browse URL, take the `PROJ-123` part).
3. **For each ticket:**
   a. **Fetch (read-only).** Use the Atlassian MCP — load tools with ToolSearch, then use
      `getJiraIssue` and/or `fetch` ONLY. Pull: summary (title), description, issue type, status,
      priority, **issue links (especially "is blocked by")**, key, and the canonical URL.
   b. **Resolve the repo link** (every Backlog card needs one):
      - use the `repo:` arg if given; else
      - `projectRepoMap[<project key>]` if mapped; else
      - ask the user for the repo for this project (AskUserQuestion / prompt), and offer to remember it
        in `projectRepoMap`.
   c. **Map the type tag** from the Jira issue type → a `typeTags` key:
      `Bug`/`Defect` → `#bug`; `Story`/`Feature`/`New Feature`/`Improvement`/`Epic` → `#feat`;
      `Task`/`Sub-task` → `#chore`; `Documentation` → `#docs`. Unknown type → leave **untagged**
      (the manager's triage auto-classifies it later).
   d. **Create the task note** in `<taskNotesFolder>/`, named `<KEY> <short-slug>.md`, from
      `.claude/agentic-workflow/templates/task-note.md`. Fill frontmatter: `ticket` (title),
      `kind: code`, `type` (mapped prefix or empty), `priority` (from Jira priority), `dependsOn`
      (KEYs from "is blocked by" links), `jira` (URL), `repo` (resolved), `status: backlog`,
      `created`/`updated` (today). Write the Jira summary + description into `## Jira snapshot`, seed
      `## Requirements / acceptance` from the description, and add a `## History` line
      `imported from Jira <KEY> (<today>)`.
   e. **Add the card** to the **Backlog** lane via the kanban-board skill:
      `- [ ] [[<KEY> <short-slug>]]`, with the type tag on an indented line (if one was mapped).
      Skip (and report) if a card already links that note.
4. **Report** a short table: each KEY → note path, repo, tag (or "untagged → will auto-classify"),
   and flag any that needed a repo decision or couldn't be fetched.

## Rules
- Jira stays read-only; the note's snapshot is the working copy from here on.
- One card per ticket; never duplicate an existing card.
- If a ticket can't be fetched (permissions / unknown key), report it and skip — don't create a card.
- Leave every new card in **Backlog**. Do NOT move a card to `Ready` yourself — promoting
  `Backlog → Ready` is the **human's** manual go-signal (their decision to start work), never this skill's job.
