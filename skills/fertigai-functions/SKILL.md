---
name: fertigai-functions
description: Use when creating, editing, testing, or deleting fertig.ai workspace functions (custom JavaScript tools agents can call) through the workspace MCP.
---

# Managing functions (fertigai_functions_*)

## Overview
A **function** is a reusable custom tool (JavaScript) that agents can call during a conversation. It has a name, a description (what the model sees), a parameter JSON Schema, and a `script`. Functions can optionally require a connection (external credentials).

Shared conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill.

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
- `script`: the JavaScript. Export a `run(ctx)` that returns `{ status, data }`.
- `requires_connection_slug`: set only if the function needs an external connection's credentials.
- `style` and `preset_key` are immutable, so `update` does not take them.

## Workflow: test before you save
`fertigai_functions_test` runs a draft against the worker and returns the outcome and logs, without saving. Always test a new or changed `script` before `create` or `update`.

## Example
```
fertigai_functions_create {
  "name": "Now (ISO)", "description": "Return the current server time as ISO 8601.",
  "style": 1, "parameter_schema": { "type": "object", "properties": {} },
  "script": "export async function run(ctx){ return { status: 200, data: { iso: new Date().toISOString() } } }"
}
```

## Common mistakes
- Passing `style` or `preset_key` to `update`: they are immutable; omit them.
- Skipping `parameter_schema`: send at least an empty object schema.
- Expecting a "run this function now" tool: there is none here. Functions are invoked by agents at runtime; use `fertigai_functions_test` for dry runs.

Writes need Integrations-Manage.
