---
name: fertigai-conversations
description: Use when reading or searching a fertig.ai workspace's conversation history (past calls and chats, their transcripts and outcomes) through the workspace MCP.
---

# Reading conversations (fertigai_conversations_*)

## Overview
Conversations are **read-only** here: you can list or search them and fetch one in detail (transcript, outcome, extracted data). You cannot create or edit conversations through this API; agents produce them at runtime.

Shared conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill. Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see fertigai-mcp / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_conversations_list` | `agent_id?`, `status?`, `date_from?`, `date_to?`, `search?`, `flag?`, `cursor?`, `page_size?` |
| `fertigai_conversations_get` | `id` |

## Filters
- `agent_id`: limit to one agent (its `agt_` id).
- `status`: one of `completed`, `transferred`, `dropped`.
- `date_from` / `date_to`: an RFC 3339 timestamp or a bare `YYYY-MM-DD` (read as midnight UTC). `date_from` is inclusive, `date_to` exclusive, so a full day is `date_from: "2026-08-11", date_to: "2026-08-12"`.
- `search`: matches the summary and the caller number.
- `flag`: comma-separated flag colors (for example `red,orange`), optionally including `none` for unflagged.
- Paginate with `cursor` and `page_size`. Results are newest-first.

## Example
```
fertigai_conversations_list { "agent_id": "agt_...", "status": "completed", "flag": "red", "page_size": 50 }
```
Then `fertigai_conversations_get { id }` for the full transcript and extracted variables.

## Common mistakes
- Passing an invalid `status` (must be completed, transferred, or dropped).
- Expecting a create or update tool: conversations are read-only here.

Requires the Conversations-View permission.
