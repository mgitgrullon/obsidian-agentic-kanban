---
ticket: "{{TITLE}}"
kind: "code"             # code (branch → PR)  |  artifact (untracked file in the checkout, no PR)
jira: "{{JIRA_URL}}"
repo: "{{REPO_LINK}}"
repoPath: ""
artifacts: []            # paths of delivered artifact files (kind: artifact)
worktreePath: ""         # dedicated git worktree for this ticket (isolation: worktree)
branch: ""
base: ""                 # default branch the feature branch was cut from
pushed: false            # has the branch been pushed to origin yet?
pr: ""
status: "backlog"        # backlog | ready | blocked | in-progress | in-review | done
needsHuman: false
created: "{{DATE}}"
updated: "{{DATE}}"
---

# {{TITLE}}

> Durable record for this ticket. Survives across loop ticks and machine restarts —
> this note, not any session id, is the source of truth for continuity.

## Jira snapshot
<!-- Filled once on first pickup (read-only fetch). Do not write back to Jira. -->

## Requirements / acceptance

## Needs human — agent's question
<!-- The agent writes its blocking question/concern HERE (dated, with phase) when it can't proceed.
     Empty = nothing outstanding. -->

## Human response
<!-- YOU answer here. When done, drag the card  Blocked → Ready  to resume.
     The agent reads this first on the next pickup and treats it as a requirement. -->

## Decisions & blockers
<!-- Running log of decisions the agent made and why. -->

## History
- {{DATE}} — created.
