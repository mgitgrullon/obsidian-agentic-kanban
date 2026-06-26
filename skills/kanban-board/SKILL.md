---
name: kanban-board
description: Read and safely mutate obsidian-kanban plugin boards (move cards between lanes, add/update cards, set tags, mark done, archive). Use when an agent needs to manipulate a Kanban board markdown file created by the obsidian-community/obsidian-kanban plugin.
---

# Obsidian Kanban Board manipulation

Safely read and edit boards produced by the **obsidian-kanban** plugin
(`github.com/obsidian-community/obsidian-kanban`). These are plain markdown files;
the rules below preserve the plugin's structure so the board never corrupts.

## File anatomy

```
---

kanban-plugin: board

---

## Lane Name

- [ ] A card
- [ ] [[Linked Task Note]] ^blockid
	#some-tag
	indented sub-content belongs to the card above
- [x] A completed card @{2026-06-25}

## Another Lane



***

## Archive

- [ ] [[Old card]]

%% kanban:settings
` ` `
{"kanban-plugin":"board","list-collapse":[false,false],...}
` ` `
%%
```

(The settings fence uses real triple backticks; spaced here only to show it.)

### Syntax rules
- **Frontmatter** must contain `kanban-plugin: board` (keep the blank lines around it).
- **Lane** = a `## Heading` line. Lane order in the file = left-to-right on the board.
- **Card** = a list item starting at column 0: `- [ ] text` (open) or `- [x] text` (checked / "complete").
- **Card → note link**: `- [ ] [[Task Note Name]]` — the wikilink is the card title.
- **Block id**: a trailing `^id` on the card's first line — a stable anchor. Keep it with the card.
- **Tags**: `#tag` anywhere in the card (often on an indented line). Used for `#needs-human`.
- **Dates / times**: date trigger `@` → `@{YYYY-MM-DD}`; time trigger `@@` → `@@{HH:mm}`.
- **Card body**: any TAB-indented lines following the card line belong to that card.
- **Archive**: everything after the `***` horizontal rule, under `## Archive`. Cards here are hidden from the board.
- **Settings**: the `%% kanban:settings … %%` block at the very end. Never reorder or drop it.

### A "card block" (what you move/cut as a unit)
A card's first line PLUS every following line until the next of: a column-0 `- [` card,
a `## ` heading, the `***` rule, or end-of-file. Always move the whole block (title +
block id + indented body + tags) together.

## Operations

Always **Read the file first**, compute the edit, then apply it with the Edit tool using
exact string matches. Do one logical change per Edit.

1. **Parse** — list lanes (in order) and, per lane, the card blocks.
2. **Move a card** — cut the full card block from its source lane and append it under the
   target lane's `## Heading` (after the heading line / existing cards). Preserve the block verbatim.
3. **Add a card** — append `- [ ] <title or [[Note]]>` under the target lane.
4. **Set a tag** (e.g. add `#needs-human`) — add the tag on an indented line within the card
   block if not already present.
4b. **Add a metadata line** (e.g. a PR link `🔗 [PR](<url>)`) — add it as a TAB-indented line inside
   the card block. Don't duplicate if an equivalent line already exists (update it instead).
5. **Mark done** — change `- [ ] ` to `- [x] ` on the card's first line.
6. **Archive a card** — move its block below the `***` rule into `## Archive`. If `archive-with-date`
   is on (or for our workflow), prepend the card title with `@{YYYY-MM-DD}`. Create the `***` and
   `## Archive` if missing.

## Invariants (do not break these)
- **`list-collapse` length == number of visible lanes.** If you add or remove a lane, update the
  `list-collapse` array in the settings JSON by the same count (use `false` for new lanes).
  The Archive (after `***`) does NOT count as a lane.
- Keep the `%% kanban:settings … %%` block last and intact (valid JSON on the fenced line).
- Lane headings are exactly `## Name`. Card lines start at column 0 with `- [ ] ` / `- [x] `.
- Never delete a card's block id or wikilink during a move.

## Concurrency
The board file is a **single-writer** resource. Only one agent (the Backlog Manager) should
mutate it at a time. Subagents must NOT write to the board concurrently — they report status
and let the manager move cards. (Task notes, being one-file-per-ticket, are safe to write in parallel.)
