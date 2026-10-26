# Managing functions (fertigai_functions_*)

## Overview
A **function** is a reusable custom tool (JavaScript) that an agent can call during a conversation. It has a name, a description (what the model sees), a parameter schema, and a `script` (or a `visual_graph` or an `http_request`). Functions can optionally declare connection roles for external credentials.

Functions generated from a workspace's external MCP servers are listed with `source: "mcp"` (see MCP server functions below); ready-made integrations are a separate catalogue (integrations.md).

The script is an ES module with a `run(ctx)` export (TypeScript accepted), and the full built-in primitive set (`mail`, `llm`, `http`, `time`, `objects`, `secrets`, `functions.invoke`, `tasks`, ...), the runtime, and the sandbox limits are documented in the scripting reference (scripting.md). Shared MCP conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_functions_list` | `search?`, `source?` (`custom` default, `mcp`, `all`), `cursor?`, `page_size?` |
| `fertigai_functions_get` | `id` |
| `fertigai_functions_create` | `name`, `description`, `style`, `parameter_schema`, `script`, `connections?`, `visual_graph?`, `http_request?` |
| `fertigai_functions_update` | `id`, `name`, `description`, `parameter_schema`, `script`, `connections?`, `visual_graph?`, `http_request?`, `usage_fixes?`, `dry_run?` |
| `fertigai_functions_delete` | `id` |
| `fertigai_functions_test` | `script?`, `parameter_schema`, `parameter_values`, `ctx_params?`, `function_id?`, `connection_bindings?`, `visual_graph?`, `http_request?` |
| `fertigai_function_usages` | `id` |
| `fertigai_function_keys_list` | `function_id`, `attached_function_id?` |
| `fertigai_function_keys_delete` | `function_id`, `key_id` |
| `fertigai_function_executions_list` | `function_id`, `cursor?`, `page_size?` |
| `fertigai_function_executions_get` | `function_id`, `execution_id` |
| `fertigai_agent_function_executions_list` | `agent_id`, `attached_function_id?`, `cursor?`, `page_size?` |
| `fertigai_scheduled_functions_list` | `status?`, `cursor?`, `page_size?` |
| `fertigai_scheduled_functions_get` | `id` |
| `fertigai_scheduled_functions_create` | `function_id`, `run_at`, `params?` |
| `fertigai_scheduled_functions_delete` | `id` |

## Fields
- `style`: integer. `1` = Script (a JavaScript function, the common case), `2` = Visual (the drag-and-drop builder, which uses `visual_graph` instead of `script`), `3` = HTTP (one HTTP request; `http_request` instead of `script`, see HTTP functions below).
- `parameter_schema`: the arguments the agent must supply, `{ "parameters": [{ "key", "description", "type", "required" }] }`. `key` is lowercase letters, digits and underscores, starting with a letter, max 32 characters. `type` is an integer: `1` string, `2` boolean, `3` integer, `4` number, `5` object (nested `properties`, a list of parameters), `6` array (`item_schema`, one parameter). `enum_values` restricts a string or integer. Send `{ "parameters": [] }` when the function takes no arguments.
- `script`: the JavaScript module (see the return contract below).
- `connections`: the connection roles the script reads, `[{ "role", "slug", "required" }]`, at most 8. `role` is lowercase letters, digits and underscores, max 32 characters; `slug` is a connection type from the workspace's connection catalogue; a non-empty list must contain a required `default` role. The script reads `ctx.connections[role]`, and `ctx.connection` is the `default` role. Omit it or send `[]` when the function needs no connection. Responses return `connections` in the catalogue entry shape, with the `connected` candidates for each role (integrations.md).
- `style` is immutable, so `update` does not take it.

## The function `ctx`
A function's `ctx` is param-centric:
```
{ params: { /* the typed arguments, matching parameter_schema */ },
  event: { type: "agent-invocation" | "mailhook" | "task-trigger" | "manual-test" | "function-call", ... },
  connection?: { /* the default role's credentials */ }, connections?: { [role]: { /* credentials */ } },
  conversation_id?: "conv_...",   // present when an agent calls it mid-call
  agent_id?: "agt_..." }
```
It has NO transcript or conversation history (that is the action `ctx`; see actions.md). Mail-hook runs additionally receive `ctx.mail` (the inbound message) and task-trigger runs receive `ctx.task`: `{ id, key, number, type_id, title, status_id, status_category, priority, source, tags, label_ids }`. They also receive `ctx.ticket`, a legacy view of the same task for scripts written for ticket triggers (`priority` reads `"normal"` for `NONE` and `MEDIUM`, and the source is `channel`); new scripts read `ctx.task`. `ctx.event` carries the trigger detail (see scripting.md).

## The return contract (strict)
A function MUST return `{ status: <integer 100-599>, data: <any JSON> }`. `status` becomes the HTTP status the caller or agent sees; `data` becomes the raw JSON response body (no wrapping envelope). Returning any other shape fails as `502 function-invalid-return-shape`. A thrown error surfaces as `500`, a timeout as `504`.

## HTTP functions
A style `3` function is one HTTP request, described by `http_request` (send it on `create`, `update` and `test`, and send no `script`). Enums are integers:
```
{ "method": 2,                                   // 1 GET, 2 POST, 3 PUT, 4 PATCH, 5 DELETE
  "url": "https://{{connection.subdomain}}.zendesk.com/api/v2/tickets",
  "query":   [ { "key": "async", "value": "{{params.async}}" } ],
  "headers": [ { "key": "accept", "value": "application/json" } ],
  "auth": { "kind": 3, "username": "{{connection.email}}/token", "password": "{{connection.api_token}}" },
  "body": { "kind": 2, "content": "{ \"ticket\": { \"subject\": \"{{params.subject}}\", \"priority\": \"{{params.priority}}\" } }" },
  "version": 1 }
```
- `auth.kind`: `1` None, `2` Bearer (`token`), `3` Basic (`username`, `password`), `4` API key (`api_key_name`, `api_key_value`, `api_key_placement`: `1` header, `2` query).
- `body.kind`: `1` None, `2` JSON, `3` Text. GET and DELETE take no body (kind `1`). JSON and Text set `content-type` unless a header row already does.
- Query and header keys are literal and unique (headers case-insensitive). `method`, `auth.kind` and `body.kind` are required, `version` is `1`, `url` is non-empty without `#`, header names are HTTP tokens, the fields of the chosen auth kind are non-empty, and a key auth already sets (`Authorization`, the API-key name) may not appear as a row.

Every `url`, query and header value, auth value and body `content` is a template: literal text with `{{ path }}` placeholders.
| Placeholder | Resolves to |
|---|---|
| `{{params.<key>}}` | a parameter value (`ctx.params`) |
| `{{connection.<field>}}` | a field of the `default` connection role |
| `{{connections.<role>.<field>}}` | a field of a named connection role |
| `{{secrets.<NAME>}}` | a workspace secret |

Spaces inside the braces are allowed (`{{ params.x }}`); segments are `[A-Za-z_][A-Za-z0-9_]*`, `params.*` may nest (`params.a.b`), `connection.*` and `secrets.*` take exactly one field, `connections.*` takes role and field, `summary` and `transcript` take none. An unknown root, a missing field or an unclosed `{{` fails with the field path.

Encoding: placeholders in the URL and in query values are URL-encoded, except a `connection.*`, `connections.*` or `secrets.*` placeholder that starts the URL (a base URL such as `{{connection.base_url}}/tickets`), which is inserted as-is (any other placeholder there is rejected at save with an error on `url`); header, auth and text-body values are inserted as text (objects as JSON). A query row whose value resolves to the empty string is left out, so optional parameters stay optional. In a JSON body every placeholder must sit inside a JSON string; a string that is exactly one placeholder passes the value through with its type (numbers, booleans, objects, and an unset value drops the key), while mixed text is stringified. Placeholders are not allowed in JSON keys.

Result: `{ status, data }` with the upstream status and the response body, parsed as JSON when it is JSON, else the raw text. A network failure returns `{ status: 502, data: { error } }`. An invalid definition fails `create`/`update` with 422 `validation-error` and `test` with 422 `http-request-invalid`, detail `http_request invalid: <field>: <message>` (fields like `url`, `headers[0].key`, `body.content`). A missing `http_request` on a style-3 function fails as `invalid-function-body`. Responses for HTTP rows return an empty `script`; read `http_request` instead.

## MCP server functions
MCP servers are connected in the portal. Each tool a connected server offers appears as a function: `fertigai_functions_list` returns them with `source: "mcp"` (or `"all"`), and they read back with `is_preset_link: true`. They mirror the server's tools and stay in sync with it: a changed tool updates its function, and a tool the server no longer offers removes its function.

- They are read-only. `fertigai_functions_update` is rejected with `422` (`preset_link_locked: function is managed by its MCP server and cannot be edited`), and `fertigai_functions_delete` with `409` (`mcp_generated_locked: MCP tool functions cannot be deleted`).
- Attach one like any function, by its `fn_...` id (attachments.md). Its `parameter_values` are set for you: every parameter is filled by the model, described by the tool's own parameter description, and values you send are replaced.

## Optional parameters
When a function is called, an optional parameter (`required: false`) may be omitted or sent as `null`; both count as not supplied, and `null` reaches the script unchanged. A required parameter sent as `null` is rejected as missing.

## Test before you save
`fertigai_functions_test` runs a draft (without saving) and returns the outcome and logs. Always test a new or changed `script` (or `visual_graph`, `http_request`) before `create` or `update`. `script` is optional when `http_request` is set.

## Stale usages
An update may not break the places that use the function: agent and workflow attachments, task triggers that run it, and pending scheduled runs. `fertigai_function_usages { id }` lists every usage before you change anything: agent attachments, task trigger actions, scheduled runs and mail hooks, stale ones first, each with `is_stale`, `stale_reason` (set only when stale) and its `current_values`. It needs Integrations-Manage. If the new parameters or connection roles would leave any of them stale, nothing is saved and the tool returns a `409` error with one line per usage: why it no longer fits, and its `current_values`.

- A usage whose only problem is a dynamic variable that its agent branch no longer declares does not block the update; declare the variable on the branch again or pick another source.
- Mail hooks have no parameter values, so they are never stale.
- Send the same update again with `usage_fixes`, one entry per listed usage. The update and every fix are saved together, or nothing is.
- `id` is the usage's id. `parameter_values` are its full new values, in the shape of its `current_values` (omit to keep them). `connection_bindings` replaces its role bindings, `[{ "role", "connection_public_id" }]` (omit to keep them).
- An agent or workflow attachment (`afn_...`) takes `parameter_values` in the four-source shape (attachments.md).
- A task trigger (`trg_...`) also needs its `action_index`, and its values are the trigger's inputs, which may use task placeholders such as `{{title}}` or `{{custom.<field>}}` (tasks.md). Without access to the Tasks product, the trigger is listed with only its id, index and reason. Fixing it also needs Tasks-Manage: with the product but without Tasks-Manage you see its name and values, but its fix fails with `403`.
- A scheduled run (`sfn_...`) can only be cancelled: `{ "id": "sfn_...", "cancel": true }`.
- A fix for an id that is not a usage of this function fails with `422`.
- `dry_run: true` runs the same checks and fixes without saving anything: the answer is what a real update would answer.

```
fertigai_functions_update {
  "id": "fn_...", "name": "Customer Lookup", "description": "Find a customer by id.",
  "parameter_schema": { "parameters": [{ "key": "customer_id", "description": "The customer's id", "type": 1, "required": true }] },
  "script": "export async function run(ctx) {\n  return { status: 200, data: { id: ctx.params.customer_id } }\n}",
  "usage_fixes": [
    { "id": "afn_...", "parameter_values": { "customer_id": { "source": { "Llm": { "description_override": "The customer's id" } } } } },
    { "id": "trg_...", "action_index": 0, "parameter_values": { "customer_id": "{{custom.customer_id}}" } },
    { "id": "sfn_...", "cancel": true }
  ]
}
```

## Function keys
A function key authenticates calls to a function. Every attachment gets its own key automatically; standalone keys are created and rotated in the portal. Over MCP you can only list and revoke keys, and no tool ever returns a key's value.
- `fertigai_function_keys_list` returns every key of the function, revoked ones included: `id`, `label`, `last_four`, `attached_function_id` (empty for a standalone key), `connection_bindings`, `created_at`, `last_used_at` and `revoked_at` (empty while the key works). Pass `attached_function_id` (`afn_...`) to see only the key of one attachment.
- `fertigai_function_keys_delete` revokes a standalone key at once; it stays in the list with `revoked_at` set. A key that belongs to an attachment cannot be revoked (`409`): detach the function instead. Revoking a key twice gives `404`. Needs Integrations-Manage.

## Run history
- `fertigai_function_executions_list` lists a function's runs, newest first: status, duration, what triggered the run, and the attachment and key used.
- `fertigai_agent_function_executions_list` lists the function runs of one agent, optionally only those of one attachment (`attached_function_id`).
- `fertigai_function_executions_get` returns one run in full: the parameters it got, the return value (or a note that it was too large to keep), logs, the error and the triggering event. It needs Integrations-Manage; the two list tools need only workspace access.

## Scheduled runs
A scheduled run calls a function once, at a set time, with fixed parameters.
```
fertigai_scheduled_functions_create {
  "function_id": "fn_...",
  "run_at": "2026-11-02T09:00:00Z",
  "params": { "customer_id": "C-1042" }
}
```
- `run_at` is an RFC 3339 timestamp and must be in the future; `params` is keyed by parameter key and must fit the function's `parameter_schema` (omit it, or send `{}`, when the function takes none). Either failing is a `422`; a function that is not in the workspace is a `404`.
- `fertigai_scheduled_functions_list` returns runs with the latest `run_at` first: function, `params`, `run_at`, `status` and error. Filter with `status`: `pending`, `running`, `succeeded` or `failed` (any other value is a `400`).
- There is no update. To change a run, delete it and create it again.
- `fertigai_scheduled_functions_delete` cancels a pending run or removes a finished one for good. A run that is running right now gives `409`. Finished runs are also cleaned up automatically after a while.
- Creating and deleting need Integrations-Manage.

## Example
```
fertigai_functions_create {
  "name": "Coin Flip", "description": "Flip a coin and return heads or tails.",
  "style": 1, "parameter_schema": { "parameters": [] },
  "script": "export async function run(ctx) {\n  const result = Math.random() < 0.5 ? 'heads' : 'tails'\n  return { status: 200, data: { result } }\n}"
}
fertigai_functions_create {
  "name": "Order Status", "description": "Look up an order's status by its number.",
  "style": 3,
  "parameter_schema": { "parameters": [{ "key": "order_number", "description": "The order number", "type": 1, "required": true }] },
  "http_request": { "method": 1, "url": "https://shop.example.com/api/orders/{{params.order_number}}",
    "query": [], "headers": [], "auth": { "kind": 2, "token": "{{secrets.SHOP_API_TOKEN}}" },
    "body": { "kind": 1, "content": "" }, "version": 1 }
}
```

## Common mistakes
- Returning a bare value instead of `{ status, data }`: fails as `502`. Always return the two-field object.
- Forgetting to `await` an outbound primitive (mail, http, ...): the run fails with a "missing await" error.
- Declaring `connections` without a required `default` role: rejected.
- Passing `style` to `update`: it is immutable; omit it.
- An unquoted placeholder in a JSON body (`"n": {{params.n}}`): rejected with `422`. Quote it (`"n": "{{params.n}}"`); a whole-string placeholder keeps the value's type.
- Sending `script` together with `http_request`: ignored on functions, rejected on actions; send only `http_request`.
- Re-sending an update after a `409` without `usage_fixes`: it is refused again. Add one entry per listed usage, or use `dry_run` to try first.
- Expecting a "run this function now" tool over MCP: there is none. Functions are invoked by agents at runtime; use `fertigai_functions_test` for dry runs, or `fertigai_scheduled_functions_create` for one real run at a future time.
- Expecting a tool that creates or rotates a function key, or shows a key's value: keys are managed in the portal.

Writes need Integrations-Manage.
