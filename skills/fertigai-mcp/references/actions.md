# Managing actions (fertigai_actions_*)

## Overview
An **action** is an automation that runs in response to a conversation (for example a post-call step). It has a name, a description, a `style`, a parameter schema, and one body: a `script_source` (code), a `pipeline` (structured steps), or an `http_request` (one HTTP request).

Code actions are an ES module with a `run(ctx)` export (TypeScript accepted). The full built-in primitive set (`mail`, `llm`, `http`, `time`, `objects`, `secrets`, `functions.invoke`, `tickets`, ...), the runtime, and the sandbox limits are documented in the scripting reference (scripting.md). Shared MCP conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_actions_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_actions_get` | `id` |
| `fertigai_actions_create` | `name`, `description`, `style`, `script_source?`, `pipeline?`, `http_request?`, `parameter_schema?`, `connections?` |
| `fertigai_actions_update` | `id`, `name`, `description`, `style`, `script_source?`, `pipeline?`, `http_request?`, `parameter_schema?`, `connections?` |
| `fertigai_actions_delete` | `id` |
| `fertigai_actions_test` | `script?`, `parameter_schema?`, `parameter_values?`, `ctx_params?`, one of `conversation_public_id` or `script_ctx`, `action_id?`, `connection_bindings?`, `http_request?` |

## Attaching actions to a branch
Actions run after a conversation ends, once per attachment on the branch.
| Tool | Args |
|---|---|
| `fertigai_agent_actions_list` | `agent_id`, `branch_id` |
| `fertigai_agent_actions_attach` | `agent_id`, `branch_id`, one of `action_id` / `integration_key`, `parameter_values`, `label?`, `connection_bindings?` |
| `fertigai_agent_actions_update` | `agent_id`, `branch_id`, `attached_action_id`, `parameter_values`, `label?`, `connection_bindings?` |
| `fertigai_agent_actions_detach` | `agent_id`, `branch_id`, `attached_action_id` |

`attached_action_id` is the `aba_...` id from the list or attach response. `update` is a full replacement: send every value you want kept. `parameter_values` uses the same four sources as function attachments (attachments.md). `connection_bindings` is `[{ "role", "connection_public_id" }]`, one entry per role the action (custom or integration) declares; every required role must be bound. Integration actions come from `fertigai_integration_actions_list` (integrations.md).

## Fields
- `style`: integer. `2` = Code (put the action logic in `script_source`), `1` = Visual (uses `pipeline`), `3` = HTTP (uses `http_request`, no `script_source`). For a code action use `style: 2` with a `script_source`.
- Do NOT send `preset_key` on create; it is server-managed and rejected.
- `style` is immutable across updates (a different value is rejected).
- `connections`: the connection roles the script reads, `[{ "role", "slug", "required" }]`, at most 8. `role` is lowercase letters, digits and underscores, max 32 characters; `slug` is a connection type from the workspace's connection catalogue; a non-empty list must contain a required `default` role. The script reads `ctx.connections[role]`, and `ctx.connection` is the `default` role. Omit it or send `[]` when the action needs no connection. Responses return `connections` in the catalogue entry shape, with the `connected` candidates for each role (integrations.md).

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
  connection?: { /* the default role's credentials */ }, connections?: { [role]: { /* credentials */ } } }
```
Use `conversation.normalize(ctx)` (see scripting.md) to render the transcript into readable lines.

## HTTP actions
`http_request` has the same shape, enums, placeholder syntax, encoding and `{ status, data }` result as for functions (functions.md, HTTP functions); the result is stored as the execution result. On `test`, `script` is optional when `http_request` is set. Responses for HTTP rows return an empty `script_source`; read `http_request` instead. Actions fail `create`/`update` with 422 `invalid-http-request` (same detail format, also for a missing `http_request`) and `test` with `http-request-invalid`. Actions accept these placeholders on top of the function ones (`params`, `connection`, `connections`, `secrets`):
| Placeholder | Resolves to |
|---|---|
| `{{conversation.<field>}}` | a conversation field, e.g. `conversation.caller.number` |
| `{{summary}}` | the call summary |
| `{{transcript}}` | the transcript as readable lines |
| `{{variables.<key>}}` | a conversation variable |
| `{{classification.<key>}}` | a classification value |
| `{{workspace.<field>}}` | `workspace.id` or `workspace.slug` |

## Return value
Unlike a function, an action's return value is stored verbatim as the execution result; there is no required `{ status, data }` shape. Returning something small and descriptive (for example `{ ok: true, ticketId }`) is good practice.

## Test before saving
`fertigai_actions_test` dry-runs a draft. It needs a context source: pass either a real `conversation_public_id` to run against a past conversation, or a `script_ctx` object with mock context. Provide exactly one of them, not both.

## Common mistakes
- Sending `preset_key` on create.
- Sending `script_source` or `pipeline` with an HTTP action: rejected; send only `http_request`.
- Declaring `connections` without a required `default` role: rejected.
- Omitting the context source on `test`, or providing both (`conversation_public_id` XOR `script_ctx`).
- Forgetting to `await` an outbound primitive (mail, http, ...): the run fails with a "missing await" error.
- `test` returning a service-unavailable error: the action runtime must be available for test to run.

Writes need Integrations-Manage.
