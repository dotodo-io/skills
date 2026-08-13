---
name: dotodo
description: >-
  Manage issues, notes, lists and the Next queue through the dotodo.io MCP server.
  Use MCP tools for issues, Next (not a tag), notes, lists, and search.
  Prefer tools over file edits. Tool schemas own argument lists — this file is behavior only.
---

# DOTODO

Use MCP tools over HTTP `/mcp`. Parse JSON from `content[0].text`. Reply in a short informal summary — **never raw JSON or tables**.

`search` first for “find / do I have…”. Args, filters, and pagination live on the tool schemas.

For read-only context, discover MCP Resources with `resources/list` instead of hardcoding their URIs.

## Time and dates

Chat time is stale. **Before any `dueDate` / `reminderAt`**, call `get_realtime` and use `currentLocal` + `timezone` as “now”.

- `dueDate` = calendar day: `YYYY-MM-DD` or `YYYY-MM-DDT12:00:00.000Z`. Never year-only (`"2026"`).
- `reminderAt` = full ISO with clock time, converted from the user’s zone to UTC.
- “Monday” / “by Friday” → that day’s `dueDate` (noon UTC).
- “remind me tomorrow” / “call at 2pm” → `reminderAt` from `currentLocal`.
- Recurring needs `dueDate`. Prefer `recurrenceAnchor: "due"`. Weekly `BYDAY` accepts one or more comma-separated days (for example `MO,WE,FR`) and is independent of the due weekday (API snaps due forward). Complete or delete-one advances (new `#id`); `delete_issue series: true` stops this + future only.

## Completing issues

**Do not call `complete_issue` unless the user explicitly asks** (“close #12”, “mark these done”). Finishing implementation is not enough.

1. You may suggest: “Close #12?”
2. Call only after a clear yes.
3. Never batch-complete for housekeeping.
4. Recurring complete may return `nextIssue` — mention the new id/due. Completing also drops the issue from Next.

## Next queue (not a tag)

Next is a pinned focus queue (`onNext` on the issue), not a tag and not a list. `#next` / `tags: ["next"]` does **nothing** for the queue.

| Intent            | Do                                       | Don’t                     |
| ----------------- | ---------------------------------------- | ------------------------- |
| Add / pin to Next | `add_to_next` (after `add_issue` if new) | `tags: ["next"]`          |
| What’s next       | `list_next`                              | search/list by tag `next` |
| Unpin             | `remove_from_next`                       | clear a `next` tag        |

Use the literal tag `next` only if the user asked for that label **and** still call `add_to_next` if they wanted the queue.

## Lists and new issues

Lists = projects. **Before `add_issue` for code / a repo / a project bug:**

1. `list_lists` (active lists only — archive is web-only).
2. Map folder/repo to a list (`dotodo.io` → `dotodo`).
3. Match → pass `listId`. No match → inbox, or ask.
4. Personal reminders → omit `listId`. Unclear → ask.

Do **not** create lists, invent `listId`s, or assign personal stuff to a project. Archived names stay reserved.

Archived lists and their issues/notes are hidden from `list_*`, search, Next, and MCP Resources; archive/unarchive is web-only.

**`delete_list`:** MCP has no confirm UI. `list_issues` + `list_notes` first. If anything is there, ask:

- **Detach** (default): delete the list; issues → inbox, notes unassigned.
- **Move** to another list, then delete.
- **Delete contents**, then the list.

## Notes and related

- Note `#5` and issue `#5` are different sequences.
- `save_note`: infer `tags` (project, topic); set `listId` when it clearly belongs to a list.
- Related is M2M (`relate_issue_note`, `relatedNoteIds` / `relatedIssueIds`). Edit/update **replaces** the set when those fields are sent.
- Related ≠ `note` (hint string on an issue) ≠ note `context` (body). Link when creating issues from a note or attaching a note as a source.

## How to write issues

- Appointments: short phrases (“Doctor Friday”, “Meeting 3pm”).
- Tasks: imperative when it fits (“Implement validation”).
- Fix typos; don’t over-formalize.
- Priority: urgent/important → `high`; backlog → `low`; else `medium` or omit.
- `#tag` in text → `tags` array (except “Next” — that is the queue above).

## How to answer

Casual and short. Examples: “Added #172: …”, “Marked as done: #172”, “Added #172 to Next”. Lists as checkboxes with due/repeat when present. Search: group by type if mixed. Mention `hasMore*` when pagination continues.

## When tools fail

Check URL (path `/mcp`), that the server is up, then `initialize` / `tools/list`. Auth is the client `Authorization` header, not a tool arg. Ask before restarting a local process.
