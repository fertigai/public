# Managing functions (fertigai_functions_*)

## Overview
A **function** is a reusable custom tool (JavaScript) that an agent can call during a conversation. It has a name, a description (what the model sees), a parameter JSON Schema, and a `script` (or a `visual_graph` or an `http_request`). Functions can optionally declare connection roles for external credentials.

Functions generated from a workspace's external MCP servers are listed with `source: "mcp"`; ready-made integrations are a separate catalogue (integrations.md).

The script is an ES module with a `run(ctx)` export (TypeScript accepted), and the full built-in primitive set (`mail`, `llm`, `http`, `time`, `objects`, `secrets`, `functions.invoke`, `tickets`, ...), the runtime, and the sandbox limits are documented in the scripting reference (scripting.md). Shared MCP conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_functions_list` | `search?`, `source?` (`custom` default, `mcp`, `all`), `cursor?`, `page_size?` |
| `fertigai_functions_get` | `id` |
| `fertigai_functions_create` | `name`, `description`, `style`, `parameter_schema`, `script`, `connections?`, `visual_graph?`, `http_request?` |
| `fertigai_functions_update` | `id`, `name`, `description`, `parameter_schema`, `script`, `connections?`, `visual_graph?`, `http_request?` |
| `fertigai_functions_delete` | `id` |
| `fertigai_functions_test` | `script?`, `parameter_schema`, `parameter_values`, `ctx_params?`, `function_id?`, `connection_bindings?`, `visual_graph?`, `http_request?` |

## Fields
- `style`: integer. `1` = Script (a JavaScript function, the common case), `2` = Visual (the drag-and-drop builder, which uses `visual_graph` instead of `script`), `3` = HTTP (one HTTP request; `http_request` instead of `script`, see HTTP functions below).
- `parameter_schema`: a JSON Schema object describing the arguments the agent must supply. Send at least `{ "type": "object", "properties": {} }`.
- `script`: the JavaScript module (see the return contract below).
- `connections`: the connection roles the script reads, `[{ "role", "slug", "required" }]`, at most 8. `role` is lowercase letters, digits and underscores, max 32 characters; `slug` is a connection type from the workspace's connection catalogue; a non-empty list must contain a required `default` role. The script reads `ctx.connections[role]`, and `ctx.connection` is the `default` role. Omit it or send `[]` when the function needs no connection. Responses return `connections` in the catalogue entry shape, with the `connected` candidates for each role (integrations.md).
- `style` is immutable, so `update` does not take it.

## The function `ctx`
A function's `ctx` is param-centric:
```
{ params: { /* the typed arguments, matching parameter_schema */ },
  event: { type: "agent-invocation" | "mailhook" | "ticket-trigger" | "manual-test" | "function-call", ... },
  connection?: { /* the default role's credentials */ }, connections?: { [role]: { /* credentials */ } },
  conversation_id?: "cv_...",   // present when an agent calls it mid-call
  agent_id?: "agt_..." }
```
It has NO transcript or conversation history (that is the action `ctx`; see actions.md). Mail-hook runs additionally receive `ctx.mail` (the inbound message) and ticket-trigger runs receive `ctx.ticket`. `ctx.event` carries the trigger detail (see scripting.md).

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

Encoding: placeholders in the URL and in query values are URL-encoded; header, auth and text-body values are inserted as text (objects as JSON). A query row whose value resolves to the empty string is left out, so optional parameters stay optional. In a JSON body every placeholder must sit inside a JSON string; a string that is exactly one placeholder passes the value through with its type (numbers, booleans, objects, and an unset value drops the key), while mixed text is stringified. Placeholders are not allowed in JSON keys.

Result: `{ status, data }` with the upstream status and the response body, parsed as JSON when it is JSON, else the raw text. A network failure returns `{ status: 502, data: { error } }`. An invalid definition fails `create`/`update` with 422 `validation-error` and `test` with 422 `http-request-invalid`, detail `http_request invalid: <field>: <message>` (fields like `url`, `headers[0].key`, `body.content`). A missing `http_request` on a style-3 function fails as `invalid-function-body`. Responses for HTTP rows return an empty `script`; read `http_request` instead.

## Test before you save
`fertigai_functions_test` runs a draft (without saving) and returns the outcome and logs. Always test a new or changed `script` (or `visual_graph`, `http_request`) before `create` or `update`. `script` is optional when `http_request` is set.

## Example
```
fertigai_functions_create {
  "name": "Coin Flip", "description": "Flip a coin and return heads or tails.",
  "style": 1, "parameter_schema": { "type": "object", "properties": {} },
  "script": "export async function run(ctx) {\n  const result = Math.random() < 0.5 ? 'heads' : 'tails'\n  return { status: 200, data: { result } }\n}"
}
fertigai_functions_create {
  "name": "Order Status", "description": "Look up an order's status by its number.",
  "style": 3,
  "parameter_schema": { "type": "object", "properties": { "order_number": { "type": "string" } }, "required": ["order_number"] },
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
- Expecting a "run this function now" tool over MCP: there is none. Functions are invoked by agents at runtime; use `fertigai_functions_test` for dry runs.

Writes need Integrations-Manage.
