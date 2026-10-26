# Conversations (fertigai_conversations_*)

## Overview
List or search conversations, fetch one in detail (transcript, outcome, extracted data), see the workflow route it took and the actions that ran after it, and flag it with a color. Conversations themselves are never created or edited here; agents produce them at runtime.

Shared conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_conversations_list` | `agent_id?`, `status?`, `date_from?`, `date_to?`, `search?`, `flag?`, `cursor?`, `page_size?` |
| `fertigai_conversations_get` | `id` |
| `fertigai_conversations_flag` | `id`, `color` |
| `fertigai_conversations_unflag` | `id` |
| `fertigai_conversations_workflow` | `id` |
| `fertigai_conversations_script_runs` | `id` |

## Filters
- `agent_id`: limit to one agent (its `agt_` id).
- `status`: one of `completed`, `transferred`, `dropped`.
- `date_from` / `date_to`: an RFC 3339 timestamp or a bare `YYYY-MM-DD` (read as midnight UTC). `date_from` is inclusive, `date_to` exclusive, so a full day is `date_from: "2026-08-11", date_to: "2026-08-12"`.
- `search`: matches the summary and the caller number.
- `flag`: comma-separated flag colors (for example `red,orange`), optionally including `none` for unflagged.
- Paginate with `cursor` and `page_size`. Results are newest-first.

## Flags
`fertigai_conversations_flag` sets a color: `red`, `orange`, `yellow`, `green`, `blue` or `purple` (any other value is a `422`). A conversation has at most one flag, so a new color replaces the old one. `fertigai_conversations_unflag` removes it. Find flagged conversations with the `flag` filter above.

## Workflow route
`fertigai_conversations_workflow` returns the workflow the conversation ran on and the route it took through it: one segment per branch, the conversation's own branch first, then one per agent handover in call order. It returns `404` when the conversation kept no workflow snapshot, for example older conversations and those with zero data retention. It also needs Agents-View.

## Action runs
`fertigai_conversations_script_runs` lists the action runs of one conversation, newest first and unpaginated, as `{ "runs": [...] }`: action, status and duration; `action_deleted` marks an action deleted since. Open one in full with `fertigai_action_executions_get` (actions.md).

## Example
```
fertigai_conversations_list { "agent_id": "agt_...", "status": "completed", "flag": "red", "page_size": 50 }
```
Then `fertigai_conversations_get { id }` for the full transcript and extracted variables, and `fertigai_conversations_unflag { id }` once it is handled.

## Common mistakes
- Passing an invalid `status` (must be completed, transferred, or dropped).
- Expecting a tool that creates or edits a conversation's content: only its flag can change.

Every tool requires the Conversations-View permission.
