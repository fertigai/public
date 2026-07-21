---
name: fertigai-functions
description: Use when creating, editing, testing, or deleting fertig.ai workspace functions (custom JavaScript tools agents can call) through the workspace MCP.
---

# Managing functions (fertigai_functions_*)

## Overview
A **function** is a reusable custom tool (JavaScript) that an agent can call during a conversation. It has a name, a description (what the model sees), a parameter JSON Schema, and a `script`. Functions can optionally require a connection for external credentials.

The script is an ES module with a `run(ctx)` export, and the full built-in primitive set (`mail`, `llm`, `http`, `secrets`, `functions.invoke`, `tickets`, ...), the runtime (sandboxed JavaScript, ECMAScript 2020), and the sandbox limits are documented in the **fertigai-scripting** skill. Shared MCP conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill. Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see fertigai-mcp / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_functions_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_functions_get` | `id` |
| `fertigai_functions_create` | `name`, `description`, `style`, `parameter_schema`, `script`, `preset_key?`, `requires_connection_slug?`, `visual_graph?` |
| `fertigai_functions_update` | `id`, `name`, `description`, `parameter_schema`, `script`, `requires_connection_slug?`, `visual_graph?` |
| `fertigai_functions_delete` | `id` |
| `fertigai_functions_test` | `script`, `parameter_schema`, `parameter_values`, `ctx_params?`, `function_id?`, `connection_public_id?`, `visual_graph?` |

## Fields
- `style`: integer. `1` = Script (a JavaScript function, the common case), `2` = Visual (the drag-and-drop builder, which uses `visual_graph` instead of `script`).
- `parameter_schema`: a JSON Schema object describing the arguments the agent must supply. Send at least `{ "type": "object", "properties": {} }`.
- `script`: the JavaScript module (see the return contract below).
- `requires_connection_slug`: set only if the function needs an external connection's credentials.
- `style` and `preset_key` are immutable, so `update` does not take them.

## The function `ctx`
A function's `ctx` is param-centric:
```
{ params: { /* the typed arguments, matching parameter_schema */ },
  connection?: { /* credentials, when requires_connection_slug is set */ },
  conversation_id?: "con_...",   // present when an agent calls it mid-call
  agent_id?: "agt_..." }
```
It has NO transcript or conversation history (that is the action `ctx`; see fertigai-actions). Inbound-mail-hook functions additionally receive `ctx.mail`.

## The return contract (strict)
A function MUST return `{ status: <integer 100-599>, data: <any JSON> }`. `status` becomes the HTTP status the caller or agent sees; `data` becomes the raw JSON response body (no wrapping envelope). Returning any other shape fails as `502 function-invalid-return-shape`. A thrown error surfaces as `500`, a timeout as `504`.

## Test before you save
`fertigai_functions_test` runs a draft (without saving) and returns the outcome and logs. Always test a new or changed `script` before `create` or `update`.

## Example
```
fertigai_functions_create {
  "name": "Coin Flip", "description": "Flip a coin and return heads or tails.",
  "style": 1, "parameter_schema": { "type": "object", "properties": {} },
  "script": "export async function run(ctx) {\n  const result = Math.random() < 0.5 ? 'heads' : 'tails'\n  return { status: 200, data: { result } }\n}"
}
```

## Common mistakes
- Returning a bare value instead of `{ status, data }`: fails as `502`. Always return the two-field object.
- Forgetting to `await` an outbound primitive (mail, http, ...): the run fails with a "missing await" error.
- Passing `style` or `preset_key` to `update`: they are immutable; omit them.
- Expecting a "run this function now" tool over MCP: there is none. Functions are invoked by agents at runtime; use `fertigai_functions_test` for dry runs.

Writes need Integrations-Manage.
