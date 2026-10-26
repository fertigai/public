# Integrations, connections, MCP servers and sources (fertigai_integrations_*, fertigai_connections_*, fertigai_mcp_servers_*, fertigai_integration_sources_*)

## Overview
- An **integration** is a ready-made function or action from the catalogue (built-in entries plus any integration source installed in the workspace). You do not create or edit it; you attach it to an agent branch by its **key** and pick which workspace connection fills each **connection role** it declares. Integrations are not in `fertigai_functions_list` or `fertigai_actions_list`; those list your own custom functions and actions.
- A **connection** (`conn_...`) holds the credentials for one external system, for example a helpdesk account. Integrations, functions and actions read it through a connection role.
- A **remote MCP server** (`mcp_...`) is an external MCP endpoint whose tools become workspace functions.
- An **integration source** (`isrc_...`) is a published package of connection types, actions and function presets that a workspace installs.
- A **function preset** is a catalogue function that scripts call directly.

Credentials are **write-only**: connection fields and MCP bearer tokens go in on create, reconnect or update, and no tool ever returns them.

Shared MCP conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
**Catalogue**
| Tool | Args | Returns |
|---|---|---|
| `fertigai_integrations_list` | `search?`, `letter?`, `cursor?`, `page_size?` | function catalogue entries |
| `fertigai_integrations_get` | `key` | one function entry with `parameter_schema` |
| `fertigai_integration_actions_list` | `search?`, `letter?`, `cursor?`, `page_size?` | action catalogue entries |
| `fertigai_integration_actions_get` | `key` | one action entry with `parameter_schema` |
| `fertigai_integration_presets_list` | `locale?` | the function presets scripts can call |

**Connections**
| Tool | Args |
|---|---|
| `fertigai_connections_catalogue` | `locale?` |
| `fertigai_connections_list` | none |
| `fertigai_connections_get` | `id` |
| `fertigai_connections_create` | `slug`, `name`, `fields` |
| `fertigai_connections_disconnect` | `id` |
| `fertigai_connections_reconnect` | `id`, `fields` |
| `fertigai_connections_delete` | `id` |

**Remote MCP servers**
| Tool | Args |
|---|---|
| `fertigai_mcp_servers_list` | `include_tools?` |
| `fertigai_mcp_servers_get` | `id` |
| `fertigai_mcp_servers_create` | `name`, `slug`, `mcp_url`, `transport`, `auth_kind`, `bearer_token?`, `auto_sync?` |
| `fertigai_mcp_servers_update` | `id`, `name`, `slug`, `mcp_url`, `transport`, `auth_kind`, `bearer_token?`, `auto_sync?` (full replace) |
| `fertigai_mcp_servers_delete` | `id` |
| `fertigai_mcp_servers_sync` | `id` |
| `fertigai_mcp_servers_functions` | `id` |

**Integration sources**
| Tool | Args |
|---|---|
| `fertigai_integration_sources_catalogue` | `cursor?`, `page_size?` |
| `fertigai_integration_sources_list` | `cursor?`, `page_size?` |
| `fertigai_integration_sources_install` | `source_id` |
| `fertigai_integration_sources_uninstall` | `source_id` |
| `fertigai_integration_sources_set_version` | `source_id`, `version_id` |

`letter` is one of `A`..`Z` or `#` (names that start with anything else). The list omits `parameter_schema`; call the `get` tool for the entry you are about to attach.

## A catalogue entry
```
{ key: "zendesk_create_ticket", name: "Create Zendesk ticket", description: "...",
  category: "ticketing", icon: "...", source: "zendesk" | null,
  connections: [ { role: "default", slug: "zendesk", required: true,
                   connected: [ { public_id: "conn_...", name: "Support desk" } ] } ],
  attachable: true,
  parameter_schema: { ... }   // get only
}
```
- `connections`: the roles the script reads through `ctx.connections[role]` (`ctx.connection` is the `default` role). `slug` is the connection type; `connected` lists the workspace connections of that type you can bind. An entry with an empty `connections` needs no connection. Custom functions and actions declare their roles the same way (functions.md, actions.md) and are bound through the same `connection_bindings`.
- `attachable`: every required role has at least one connected candidate and the entry can run. You can attach an entry whose required role has no candidate; it stays attached and is skipped at runtime until you bind one.

## Attaching
- **Functions**: through `fertigai_agent_branch_configure` (attachments.md). An entry in `functions.branch` or `functions.nodes[...]` carries `integration_key`, optional `label`, `parameter_values` and `connection_bindings`; exactly one of `function_id`, `system_tool_type`, `integration_key`.
- **Actions**: `fertigai_agent_actions_attach { agent_id, branch_id, integration_key, label?, parameter_values, connection_bindings }` (actions.md).

`connection_bindings` is `[{ "role": "<role>", "connection_public_id": "conn_..." }]`, one entry per role you bind; every required role must be bound or the call fails with 422. `label` (max 80 characters) is shown instead of the catalogue name and lets you attach the same integration twice with different connections.

## Example
```
fertigai_integration_actions_get { "key": "zendesk_create_ticket" }
fertigai_agent_actions_attach {
  "agent_id": "agt_...", "branch_id": "abr_...",
  "integration_key": "zendesk_create_ticket", "label": "Support desk",
  "parameter_values": { "subject": { "source": { "Static": { "value": "Call follow-up" } } } },
  "connection_bindings": [ { "role": "default", "connection_public_id": "conn_..." } ]
}
```

## Connections
**Types.** `fertigai_connections_catalogue` lists the connection types the workspace can create (built-in types plus those of installed integration sources): `data.presets` and `data.categories`. Each type has `slug`, `name`, `description`, `category`, `enabled`, `kind` (`1` = credentials, `2` = OAuth) and the credential `fields` (`key`, `label`, `kind`: `1` text, `2` password, `3` email, `4` URL, `5` subdomain; `required`, `placeholder`, `helper_text`, `kind_data`). Pass `locale` (for example `de`) for translated names and descriptions; the default is English.

**Only credential types can be created over MCP.** A type with `kind: 2` (OAuth) needs a sign-in in the browser: tell the user to connect it in the portal.

**Create.** `fertigai_connections_create { slug, name, fields }`: `slug` is the type's slug, `name` a display name, and `fields` an object keyed by the type's `fields[].key` with string values. The credentials are stored encrypted and never returned. A workspace can have several connections of one type (for example a production and a sandbox account). `404` for a type the workspace's catalogue lacks; `422` for a disabled type or a missing or invalid field.

**Read.** `fertigai_connections_list` returns every connection in `data.data` (not paginated); `fertigai_connections_get` returns one. Each has `id`, `slug`, `name`, `status` (`1` connected, `2` disconnected), `secret_public_id` (the `sec_` id of the stored credentials, never their value) and timestamps.

**Use.** Bind a connection to a role through `connection_bindings` when you attach an integration, a function or an action (see Attaching above, functions.md, actions.md). Scripts read it as `ctx.connections[role]` (scripting.md).

**Rotate or pause.**
- `fertigai_connections_disconnect { id }` deletes the stored credentials and sets `status` to disconnected. The connection and its bindings stay, but integrations that use it cannot reach the external system until it is reconnected.
- `fertigai_connections_reconnect { id, fields }` stores a full new credential set and sets `status` to connected. Use it to rotate the credentials of a connected connection too. `422` when its type is no longer available to the workspace.

**Delete.** `fertigai_connections_delete { id }` removes the connection and its credentials permanently. `409` while an attachment role or a function key still uses it: rebind or remove those first.

```
fertigai_connections_catalogue {}
fertigai_connections_create {
  "slug": "zendesk", "name": "Zendesk production",
  "fields": { "subdomain": "acme", "email": "ops@example.com", "api_key": "<api-key>" }
}
```
Take the field keys from the type's `fields` in the catalogue; the keys above are only an example.

## Remote MCP servers
A remote MCP server's tools become workspace functions: one read-only function per tool, listed by `fertigai_functions_list` with `source: "mcp"` (functions.md). Attach them to a branch like any function. Scripts call a tool directly as `mcp["<slug>"]["<tool>"](params)` (or `mcp.<slug>.<tool>(params)` when the slug has no `-`).

Fields:
- `name`: display name.
- `slug`: the name scripts call the server by. Lowercase letters, digits, `-` and `_`, starting and ending with a letter or digit, unique in the workspace (`409` on a duplicate).
- `mcp_url`: the public `https` URL of the MCP endpoint. A private or loopback address is rejected with `422`.
- `transport`: `STREAMABLE_HTTP` or `SSE`.
- `auth_kind`: `NONE` or `STATIC_BEARER_TOKEN`. OAuth servers cannot be added over MCP; tell the user to add them in the portal.
- `bearer_token`: write-only, never returned. Required with `STATIC_BEARER_TOKEN` on create, empty with `NONE`.
- `auto_sync`: re-discover the tools in the background, about once an hour. Absent: on for create, unchanged on update.

**Create.** `fertigai_mcp_servers_create` connects to the server once to discover its tools and creates one function per tool. `424` when the server cannot be reached or offers more than 1000 tools.

**Read.** `fertigai_mcp_servers_list` returns `data.servers`; `fertigai_mcp_servers_get` returns one server. Each has `id`, `slug`, `name`, `mcp_url`, `transport` (`1` = `STREAMABLE_HTTP`, `2` = `SSE`), `auth_kind` (`0` = `NONE`, `1` = `STATIC_BEARER_TOKEN`), `auto_sync`, `tool_count`, `last_synced_at` and `last_sync_error`. Responses use the numbers; requests take the names. Pass `include_tools: true` to the list to embed each server's tools (name, description, `input_schema`). `fertigai_mcp_servers_functions { id }` lists the functions generated from the server.

**Update.** `fertigai_mcp_servers_update` is a **full replace**: send `name`, `slug`, `mcp_url`, `transport` and `auth_kind` as they should be (read them with `get` first). An empty `bearer_token` keeps the stored token. An update does not re-discover the tools: call `fertigai_mcp_servers_sync` after you change the URL or the authentication. A new `slug` or `name` renames the generated functions, and agents that use them follow; update scripts that call the old slug.

**Sync.** `fertigai_mcp_servers_sync { id }` re-discovers the tools now: a new tool gets a function, and the function of a tool the server no longer offers is retired. It returns the server and its tools; `424` when the server cannot be reached.

**Delete.** `fertigai_mcp_servers_delete { id }` removes the server, its token and its generated functions permanently, and agents that used those functions lose them.

```
fertigai_mcp_servers_create {
  "name": "Order lookup", "slug": "orders",
  "mcp_url": "https://mcp.example.com/mcp",
  "transport": "STREAMABLE_HTTP", "auth_kind": "STATIC_BEARER_TOKEN",
  "bearer_token": "<token>"
}
```

## Integration sources
An integration source adds its connection types, actions and function presets to the workspace catalogue.

- `fertigai_integration_sources_catalogue` lists the sources the workspace can install (paginated): `id` (`isrc_...`), `slug`, `name`, `publisher`, `repository_url`, `latest_version_id` (`isrv_...`), `latest_version`, `installed` and `latest_version_is_newer` (an installed source has a newer approved version).
- `fertigai_integration_sources_list` lists the installed sources (paginated): `id`, `slug`, `name`, `publisher`, `installed_version_id`, `installed_version`, `suspended` and `installed_at`. The integrations of a suspended source stop working.
- `fertigai_integration_sources_install { source_id }` installs the newest approved version; you cannot pick another version at install. `409` when the source is already installed or suspended; `422` when it has no approved version.
- `fertigai_integration_sources_set_version { source_id, version_id }` moves an installed source to another approved version of the same source, to upgrade or roll back. Over MCP the only version ids you can see are `latest_version_id` from the catalogue and `installed_version_id` from the list. Functions and actions already created from the source do not change. `422` for a version that is not approved.
- `fertigai_integration_sources_uninstall { source_id }` removes its connection types, actions and presets from the catalogue; functions and actions already created from it keep working. `409` while a connection of one of its types exists: delete those connections first.

## Function presets
`fertigai_integration_presets_list` returns `presets[]` (not wrapped in `data`): every catalogue function a script can call, with `key`, `provider`, `local_name`, `name`, `description`, `parameter_schema` and `connections` (its roles). Pass `locale` (for example `de`) for translated names and descriptions. `provider` is the connection type's slug, or `standalone` for a preset that needs no connection.

A preset is not copied into a function. Use it in one of two ways:
- attach the catalogue entry to a branch by `integration_key` (see Attaching above);
- call it from your own function or action script (scripting.md):

```javascript
export async function run(ctx) {
  // A preset with a connection: pass the bound connection's public id first.
  const ticket = await integrations.zendesk.get_ticket(ctx.params.connection_id, { ticket_id: ctx.params.ticket_id });
  // A preset without a connection.
  const now = await integrations.standalone.now_iso({});
  return { status: 200, data: { ticket, now } };
}
```
The provider and preset names above are examples; take them from `provider` and `local_name` in the list.

## Common mistakes
- Looking for integrations in `fertigai_functions_list` / `fertigai_actions_list`: they are not there; use the catalogue tools.
- Creating a copy of an integration with `fertigai_functions_create`: unnecessary; attach by key.
- Sending a top-level `connection_public_id`: every attachment binds through `connection_bindings` with the role name.
- Binding a connection of the wrong type: the `connection_public_id` must come from that role's `connected` list (same `slug`).
- Creating a connection of an OAuth type (`kind: 2`) or an MCP server with OAuth. Neither works over MCP; tell the user to use the portal.
- Expecting a connection's credentials or an MCP server's token in a response. They are write-only; ask the user again if they are needed.
- Sending only the changed fields to `fertigai_mcp_servers_update`. It replaces the server's settings: send every field.
- Changing an MCP server's URL or token and not calling `fertigai_mcp_servers_sync`.
- Deleting a connection that is still bound (409), or uninstalling a source while connections of its types exist (409).
- Inventing a source version id. Only `latest_version_id` and `installed_version_id` are known.

Permissions: the catalogue, presets, connections and the integration source lists need Integrations-Manage. MCP server reads (list, get, functions) need Integrations-View; their changes and sync need Integrations-Manage. Installing, uninstalling and changing the version of a source need Integration-Sources-Manage.
