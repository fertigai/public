# Managing actions (fertigai_actions_*)

## Overview
An **action** is an automation that runs in response to a conversation (for example a post-call step). It has a name, a description, a `style`, a parameter schema, and either a `script_source` (code) or a `pipeline` (structured steps).

Code actions are an ES module with a `run(ctx)` export (TypeScript accepted). The full built-in primitive set (`mail`, `llm`, `http`, `time`, `objects`, `secrets`, `functions.invoke`, `tickets`, ...), the runtime, and the sandbox limits are documented in the scripting reference (scripting.md). Shared MCP conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_actions_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_actions_get` | `id` |
| `fertigai_actions_create` | `name`, `description`, `style`, `script_source?`, `pipeline?`, `parameter_schema?` |
| `fertigai_actions_update` | `id`, `name`, `description`, `style`, `script_source?`, `pipeline?`, `parameter_schema?` |
| `fertigai_actions_delete` | `id` |
| `fertigai_actions_test` | `script`, `parameter_schema?`, `parameter_values?`, `ctx_params?`, one of `conversation_public_id` or `script_ctx`, `action_id?`, `connection_public_id?` |

## Fields
- `style`: integer. `2` = Code (put the action logic in `script_source`), `1` = Visual (uses `pipeline`). For a code action use `style: 2` with a `script_source`.
- Do NOT send `preset_key` on create; it is server-managed and rejected.
- `style` is immutable across updates (a different value is rejected).

## The action `ctx`
An action's `ctx` is conversation-centric, built from the conversation that triggered it:
```
{ conversation: { id, duration_ms, started_at, ended_at, caller: { number, called_number }, agent: { id, name } },
  transcript: [ { role: "agent"|"user", content, time_in_call_secs, sort_order, tool_calls: [...] } ],
  summary: "...",
  classification: { "<category_key>": "<value>" },
  variables: { "<key>": "<value>" },
  workspace: { id, slug },
  params: { /* the action's resolved parameters */ },
  event: { type: "post-call-analysis" },
  connection?: { /* credentials, when connection-backed */ } }
```
Use `conversation.normalize(ctx)` (see scripting.md) to render the transcript into readable lines.

## Return value
Unlike a function, an action's return value is stored verbatim as the execution result; there is no required `{ status, data }` shape. Returning something small and descriptive (for example `{ ok: true, ticketId }`) is good practice.

## Test before saving
`fertigai_actions_test` dry-runs a draft. It needs a context source: pass either a real `conversation_public_id` to run against a past conversation, or a `script_ctx` object with mock context. Provide exactly one of them, not both.

## Common mistakes
- Sending `preset_key` on create.
- Omitting the context source on `test`, or providing both (`conversation_public_id` XOR `script_ctx`).
- Forgetting to `await` an outbound primitive (mail, http, ...): the run fails with a "missing await" error.
- `test` returning a service-unavailable error: the action runtime must be available for test to run.

Writes need Integrations-Manage.
