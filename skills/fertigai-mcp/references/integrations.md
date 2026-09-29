# Integrations (fertigai_integrations_*, fertigai_integration_actions_*)

## Overview
An **integration** is a ready-made function or action from the catalogue (built-in entries plus any integration source installed in the workspace). You do not create or edit it; you attach it to an agent branch by its **key** and pick which workspace connection fills each **connection role** it declares. Integrations are not in `fertigai_functions_list` or `fertigai_actions_list`; those list your own custom functions and actions. Shared MCP conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args | Returns |
|---|---|---|
| `fertigai_integrations_list` | `search?`, `letter?`, `cursor?`, `page_size?` | function catalogue entries |
| `fertigai_integrations_get` | `key` | one function entry with `parameter_schema` |
| `fertigai_integration_actions_list` | `search?`, `letter?`, `cursor?`, `page_size?` | action catalogue entries |
| `fertigai_integration_actions_get` | `key` | one action entry with `parameter_schema` |

`letter` is one of `A`..`Z` or `#` (names that start with anything else). The list omits `parameter_schema`; call the `get` tool for the entry you are about to attach.

## A catalogue entry
```
{ key: "zendesk_create_ticket", name: "Create Zendesk ticket", description: "...",
  category: "ticketing", icon: "...", source: "zendesk" | null,
  connections: [ { role: "default", slug: "zendesk", required: true,
                   connected: [ { public_id: "con_...", name: "Support desk" } ] } ],
  attachable: true,
  parameter_schema: { ... }   // get only
}
```
- `connections`: the roles the script reads through `ctx.connections[role]` (`ctx.connection` is the `default` role). `slug` is the connection type; `connected` lists the workspace connections of that type you can bind. An entry with an empty `connections` needs no connection.
- `attachable`: every required role has at least one connected candidate and the entry can run. You can attach an entry whose required role has no candidate; it stays attached and is skipped at runtime until you bind one.

## Attaching
- **Functions**: through `fertigai_agent_branch_configure` (attachments.md). An entry in `functions.branch` or `functions.nodes[...]` carries `integration_key`, optional `label`, `parameter_values` and `connection_bindings`; exactly one of `function_id`, `system_tool_type`, `integration_key`.
- **Actions**: `fertigai_agent_actions_attach { agent_id, branch_id, integration_key, label?, parameter_values, connection_bindings }` (actions.md).

`connection_bindings` is `[{ "role": "<role>", "connection_public_id": "con_..." }]`, one entry per role you bind; every required role must be bound or the call fails with 422. `label` (max 80 characters) is shown instead of the catalogue name and lets you attach the same integration twice with different connections.

## Example
```
fertigai_integration_actions_get { "key": "zendesk_create_ticket" }
fertigai_agent_actions_attach {
  "agent_id": "agt_...", "branch_id": "abr_...",
  "integration_key": "zendesk_create_ticket", "label": "Support desk",
  "parameter_values": { "subject": { "source": { "Static": { "value": "Call follow-up" } } } },
  "connection_bindings": [ { "role": "default", "connection_public_id": "con_..." } ]
}
```

## Common mistakes
- Looking for integrations in `fertigai_functions_list` / `fertigai_actions_list`: they are not there; use the catalogue tools.
- Creating a copy of an integration with `fertigai_functions_create`: unnecessary; attach by key.
- Sending `connection_public_id` for an integration: use `connection_bindings` with the role name.
- Binding a connection of the wrong type: the `connection_public_id` must come from that role's `connected` list (same `slug`).

Reads and writes need Integrations-Manage.
