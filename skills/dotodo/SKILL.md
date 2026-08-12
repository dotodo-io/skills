---
name: dotodo
description: >-
  Manage issues, notes, lists and the Next queue through the dotodo.io MCP server.
  Use MCP tools for issues (list/add/edit/complete/delete), Next queue pins (list_next/add_to_next/remove_from_next — Next is NOT a tag),
  notes (list/get/save/update/delete), lists, and search. Server also supports MCP Resources (dotodo://...).
  Avoid direct file edits; troubleshoot MCP connectivity or server health when needed.
---

# DOTODO

## Workflow

1. Confirm the MCP server is configured as a URL endpoint (not stdio) in the active MCP config.
2. Confirm the MCP path is `/mcp`.
3. If tool calls fail, check server reachability first (`/health` when available), then retry MCP calls.
4. Use MCP tools for all todo operations.
5. Return concise operation results – informal summaries, never raw JSON or tables

## Response format

Tools return **pure JSON** in `content[0].text` (parse with `JSON.parse`). Examples: `{ issue }`, `{ issues }`, `{ notes }`, `{ lists }`, `{ deleted: true, id }`. Fields like `dueDate` and `reminderAt` are in that JSON when set — read them from the payload. `search` returns `{ issues, notes, lists, hasMoreIssues, hasMoreNotes, hasMoreLists }` (also mirrored in `structuredContent`).

## Presenting results to the user

**Never show raw JSON or tables.** Always summarize in a short, natural, informal way:

- **add_issue** → "Added #172: Conditionally run migrations..."
- **edit_issue** → "Updated #172: ..."
- **complete_issue** → "Marked as done: #172 ..." (mention next occurrence id/due when `nextIssue` is present)
- **list_next** → short numbered list of pinned active tasks (Next queue)
- **add_to_next** → "Added #172 to Next"
- **remove_from_next** → "Removed #172 from Next"
- **delete_issue** → "Deleted #172" (if `series`, note that this and future were stopped)
- **list_issues** → short list with checkboxes; include due when present in JSON, e.g. "- [ ] #172 Task (due: 16 Jul)"; mention repeat when `recurrenceRule` / `inRecurrenceSeries`
- **search** → group by type when more than one section has hits, e.g. "Found 2 issues and 1 note about zebra: …"; each item like in list_issues/list_notes
- **get_issue** → include status, list, and dueDate from JSON when present
- **save_note** → "Saved note: Title"
- **list_notes** / **list_lists** → simple list, not a table
- **delete_note** / **delete_list** → "Deleted note #1" / "Deleted list"

Tone: casual, concise, conversational. Avoid formal headers like "Operation result" or "Status".

## Tool Usage

### Search (cross-resource)

- `search`: **First-choice tool for "find…" / "do I have anything about…" requests.** Searches issues, notes and lists by phrase in one call. Args: `q` (required, min 2 chars; whitespace-separated terms are AND-ed, each term may match any text field — issue text/note/context/tag, note title/context/tag, list name; **tags match exact, not substring**), `types` (optional subset: `issues`/`notes`/`lists`), `listIds`, `inboxOnly`, `tags`, `done`, `priority`, `createdAfter/Before`, `completedAfter/Before`, `dueAfter/Before`, `hasDueDate`, `hasReminder`, `orderBy` (`updatedAt` default / `createdAt` / `dueDate` / `completedAt`), `skip`/`take` (per section, default 20, max 50). Non-applicable filters are ignored per type. Returns `{ issues, notes, lists, hasMore* }` — when `hasMoreX` is true, mention there are more results (paginate with `skip`).

### Issues

- `list_issues`: List issues. Args: `listId` (optional) – omit for ALL issues, pass list id to filter, or `inboxOnly: true` for inbox only. Optional filters: `done`, `tags`, `priority`, `completedAfter`, `completedBefore`, `dueBefore`, `dueAfter` (ISO), `hasDueDate` (boolean), `orderBy` ("id" | "dueDate"). Pagination: `skip` (default 0), `take` (default 50, max 50). **For "tasks for today" or "scheduled tasks"**: use `hasDueDate: true`, `orderBy: "dueDate"` or read resource `dotodo://issues/today`.
- `add_issue`: Add a new issue. Args: `text` (required), `listId` (optional), `tags` (optional), `priority` (low/medium/high), `dueDate` (ISO), `reminderAt` (ISO), `context` (optional – chat context), `note` (optional – hints, details), `relatedNoteIds` (optional – public note ids to link), `recurrenceRule` (optional RRULE subset; **requires** `dueDate`), `recurrenceAnchor` (optional; UI/clients use `"due"`). See "add_issue quality", "Related issues & notes", "Recurring issues", and "Projects" below.
- `edit_issue`: Edit issue. Args: `id` (required), `text`, `listId`, `tags`, `priority`, `dueDate`, `reminderAt`, `context`, `note`, `relatedNoteIds` (optional; replaces related set when provided), `recurrenceRule` / `recurrenceAnchor` (optional; `null` clears rule).
- `get_issue`: Get a single issue by `id` (full JSON including dueDate, reminderAt, relatedNoteIds, recurrenceRule, inRecurrenceSeries).
- `get_realtime`: Get time context (timezone, currentUtc, currentLocal). **Call before add_issue/edit_issue when user mentions dates** – use as reference for parsing "tomorrow 3pm" etc.
- `complete_issue`: Mark issue done by numeric `id`. **Never call without explicit user approval** — ask first (see "Completing issues"). Recurring → may return `nextIssue` (new open occurrence). Also removes the issue from Next.
- `list_next` / `add_to_next` / `remove_from_next`: **Next queue** — see "Next queue (not a tag)" below. Do **not** simulate Next with `tags`.
- `delete_issue`: Delete issue by numeric `id`. Recurring without `series`: like complete (creates next, soft-deletes this). `series: true`: stop from here — delete this + future descendants only (past history kept, no new occurrence).
- `relate_issue_note`: Link note to issue (M2M). Args: `issueId`, `noteId` (public display numbers).
- `unrelate_issue_note`: Remove that link. Args: `issueId`, `noteId`.

## Next queue (not a tag)

**Next is a pinned queue, not a tag and not a list.** It is a cross-list “focus” view of active issues the user pinned (ordered by position). Membership is stored separately (`onNext` on issue JSON). Tagging an issue with `"next"` does **nothing** for this queue.

| User intent                      | Correct action                                      | Wrong action                           |
| -------------------------------- | --------------------------------------------------- | -------------------------------------- |
| “Add to Next” / “pin to Next”    | `add_to_next` with issue `id`                       | `tags: ["next"]` or `#next`            |
| “Show Next” / “what’s next”      | `list_next`                                         | `list_issues` / `search` with tag next |
| “Remove from Next” / “unpin”     | `remove_from_next`                                  | clearing a `next` tag                  |
| New issue **and** put it on Next | `add_issue` → then `add_to_next` with returned `id` | only `add_issue` with `tags: ["next"]` |

- `list_next`: List active issues in the Next queue, ordered by position. No args.
- `add_to_next`: Append issue to Next by numeric `id`. Idempotent if already pinned.
- `remove_from_next`: Remove from Next by `id` (does not delete the issue). Completing an issue also removes it from Next.
- Truth check: `get_issue` / issue payloads include `onNext` (`true` = in the queue). A `next` tag is unrelated.

**Never** invent a `next` tag to mean the queue. Only use `tags: ["next"]` if the user explicitly asked for that literal tag as a label (rare) — and still call `add_to_next` if they also wanted the queue.

### Notes

- Note `id` is a **display number** per account (`#1…`), same style as issues (separate sequence).
- `list_notes`: List notes. Args: `listId` (optional – filter by list/project), `tags` (optional – filter by tags). Pagination: `skip` (default 0), `take` (default 50, max 50). Use for queries like "show notes related to project X".
- `get_note`: Get a single note by display number `id` (includes `relatedIssueIds`).
- `save_note`: Create a note. Args: `title` (required), `context` (optional), `listId` (optional), `tags` (optional), `relatedIssueIds` (optional). **Add tags based on note content** (project name, topic) so user can query e.g. "show notes related to project xxx" – e.g. `tags: ['project-x', 'design']`.
- `update_note`: Update note. Args: `id` (required), `title` (optional), `context` (optional), `listId` (optional), `tags` (optional), `relatedIssueIds` (optional; replaces set when provided).
- `delete_note`: Delete note by display number `id`.

## Related issues & notes

Issues and notes can be linked **many-to-many** as related sources (not the same as `Issue.note` text field).

- **Note ids** use the same public numbering style as issues: display number `#1…` per account (separate sequence from issues — note `#5` and issue `#5` can both exist).
- `relate_issue_note` / `unrelate_issue_note`: link or unlink one pair (`issueId`, `noteId` = public display numbers).
- `add_issue` / `edit_issue`: optional `relatedNoteIds` (on edit, **replaces** the full set when provided).
- `save_note` / `update_note`: optional `relatedIssueIds` (same replace semantics on update).
- `get_issue` / `get_note` return `relatedNoteIds` / `relatedIssueIds`.

**When to link**

1. User asks to create issues **from a note** → `add_issue` with `relatedNoteIds: [noteId]` (and/or `relate_issue_note` after).
2. User asks to attach a note as an **information source** for an issue → `relate_issue_note`.
3. Do **not** confuse with `note` (string hints on an issue) or the Note entity body (`context`).

## Notes – tags and lists

When saving a note, infer and add relevant `tags` from the content (project name, topic, context). This enables queries like "show notes related to project xxx" via `list_notes` with `tags: ['xxx']` or `listId` when the note belongs to a list. Assign `listId` when the note clearly relates to an existing list/project.

### Lists

- `list_lists`: List **active** (non-archived) lists only. Pagination: `skip` (default 0), `take` (default 50, max 50).
- `add_list`: Create a list. Args: `name` (required). Name must be unique among active **and archived** lists for the user (archived names stay reserved until the project is deleted).
- `update_list`: Rename list. Args: `id`, `name` (required).
- `delete_list`: Delete list. Args: `id`. **Default API behavior:** issues move to inbox (`listId=null`), notes become unassigned (`listId=null`) — contents are **not** deleted. See "Deleting lists" below.

**Archived projects:** Users can archive a project in the web app (reversible hide). Archived lists and their issues/notes do **not** appear in `list_lists`, `list_issues`, `list_notes`, `search`, Next, or MCP resources. There is no MCP archive/unarchive tool — archive is web-only. Do not invent `listId` values for missing projects; call `list_lists` first.

## Recurring issues

- Repeat requires `dueDate`. Example rule: `FREQ=DAILY;INTERVAL=1`, `FREQ=WEEKLY;INTERVAL=1;BYDAY=MO`, `FREQ=WEEKLY;INTERVAL=1;BYDAY=MO,WE,FR` (multi-day), `FREQ=WEEKLY;INTERVAL=1;BYDAY=MO,TU,WE,TH,FR` (weekdays), `FREQ=MONTHLY;INTERVAL=1`.
- Weekly `BYDAY` is independent of the due weekday: API snaps `dueDate` forward to the first selected day on/after the given date (e.g. due Saturday + `BYDAY=SU` → due Sunday).
- Prefer `recurrenceAnchor: "due"` (advance from due date).
- Complete or delete-one advances the series (new `#id`); past occurrences stay. `delete_issue` with `series: true` stops this and future only.

## Completing issues

**Do not call `complete_issue` unless the user explicitly asks to close/mark done.** Finishing implementation is not enough.

1. After work that relates to an issue, you may _suggest_ closing it (e.g. "Close #215?").
2. Call `complete_issue` only after a clear yes (or an explicit instruction like "mark these done" / "close #215").
3. Never batch-complete issues "for housekeeping" without asking.

## Deleting lists

MCP has no confirm UI. Before `delete_list`:

1. Call `list_issues` with that `listId` and `list_notes` with that `listId`.
2. If either returns items, **ask the user** what to do with the project contents, e.g.:
   - **Detach / keep** (default): delete the list only; issues stay in inbox, notes stay without a list.
   - **Move** to another list: `edit_issue` / `update_note` with the new `listId`, then `delete_list`.
   - **Delete contents**: `delete_issue` / `delete_note` for each item the user wants removed, then `delete_list`.
3. Only call `delete_list` after the user chooses (or after they explicitly said to delete the empty/unwanted list).
4. Summarize the outcome informally (e.g. "Deleted list X; 3 tasks moved to inbox, 2 notes kept unassigned").

## add_issue quality

When adding issues, keep text clear and natural:

- **Reminders / appointments**: short phrases (e.g. "Doctor appointment on Friday", "Meeting at 3pm").
- **Actionable tasks**: imperative when it fits (e.g. "Implement validation", "Check API").
- **Grammar**: fix obvious spelling/grammar errors; don't over-formalize simple notes.

## Projects – match list before add_issue (required for code/dev)

Lists = projects. **Before `add_issue` when the task is about code / a repo / a bug in a project:**

1. Call `list_lists`.
2. Map folder/repo/project name to a list (e.g. `kupbilecik-app` → list `kupbilecik`, `dotodo.io` → `dotodo`).
3. If matched → `add_issue` with that `listId`.
4. If no match → inbox (omit `listId`), or ask "Add to list X?" when ambiguous.

| Issue type               | Action                                           |
| ------------------------ | ------------------------------------------------ |
| Bug / dev task in a repo | `list_lists` → match → `add_issue` with `listId` |
| Personal reminder        | `add_issue` without `listId`                     |
| Unclear                  | Ask the user                                     |

**Do NOT**

- Create new lists automatically.
- Assign personal reminders to a project unless the user asks.
- Guess — when unclear, ask.

## Date parsing (AI → ISO)

**Dates:** `dueDate` is a calendar day — prefer `YYYY-MM-DDT12:00:00.000Z` (or `YYYY-MM-DD`). `reminderAt` needs a real time in full ISO 8601. **Never pass year only** like `"2026"`.

**Before parsing dates**, call `get_realtime` to get user timezone and current time. The chat may not have real time – use `currentUtc` and `currentLocal` as reference for "now". Interpret times like "tomorrow 3pm" in the user's timezone and convert to UTC (ISO 8601) before sending to the API.

Parse natural language to ISO dates for `dueDate` and `reminderAt`:

- "on Monday", "by Friday" → `dueDate: "2025-02-17T12:00:00.000Z"` (calendar day, UTC noon)
- "remind me tomorrow" → `reminderAt: "2025-02-14T09:00:00Z"` (use `currentLocal` to determine "tomorrow")
- "call at 2pm", "remind me to call at 14:00" → `reminderAt: "2025-02-13T14:00:00Z"` (today + time in user timezone)

## Priorities

Map phrases to `priority` (low/medium/high):

- "important", "urgent", "must do tomorrow" → `high`
- "add to backlog", "not urgent" → `low`
- Default: `medium` or omit

## Tags

Extract `#tag` from text and pass as `tags` array (e.g. `#bug` → `tags: ["bug"]`). Keep or remove hashtags from text per user preference.

**Exception — “Next”:** phrases like “add to next”, “put on Next”, “pin next”, or a bare “next” meaning the focus queue are **not** tags. Use `add_to_next` / `list_next` / `remove_from_next` (see “Next queue (not a tag)”). Do not pass `tags: ["next"]` for that intent.

## Auth note

MCP auth is **Bearer token in the HTTP `Authorization` header** (per request), not a tool argument. Clients configure the token in MCP server settings; do not pass tokens as tool parameters.

## Execution Rules

- Validate required inputs before calling a tool (`text` for add, positive integer `id` for complete/delete/edit, `title` for save_note, `name` for add_list/update_list). Dates (`dueDate`, `reminderAt`) – always ISO 8601 format.
- Prefer `list_issues` or `list_notes` or `list_lists` before destructive actions when state may be stale.
- Surface backend errors directly when useful (for example: issue not found, server unavailable).
- Keep responses short and action-oriented. **Never present raw JSON or tables** – always use informal summaries (see "Presenting results to the user").

## Resources

The server exposes MCP Resources for read-only context (picker, auto-context). URIs: `dotodo://issues/inbox`, `dotodo://issues/today` (scheduled tasks: overdue, today, upcoming), `dotodo://notes`, `dotodo://lists`, `dotodo://issues/list/{listId}`, `dotodo://issue/{id}`, `dotodo://note/{id}`. Clients may use `resources/list` and `resources/read`; agents use tools for actions.

## Troubleshooting

When the MCP server appears unavailable:

1. Verify the configured URL is correct (scheme, host, port, path `/mcp`).
2. Check whether the Node server process is running.
3. Confirm `initialize` and `tools/list` succeed before retrying business calls.
4. If needed, ask for permission to restart the local server process.
