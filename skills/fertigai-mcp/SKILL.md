---
name: fertigai-mcp
description: Use when managing a fertig.ai workspace through its MCP endpoint - creating or editing agents, functions, actions, ticket templates, mail templates, or secrets, or reading conversations - via the fertigai_* tools.
---

# fertig.ai Workspace MCP (fertigai_*)

## Overview
The workspace MCP exposes your fertig.ai workspace configuration as MCP tools, all named `fertigai_*`. Point an MCP client at your workspace endpoint, authenticate with a workspace API key, and the tools let you manage agents and their configuration, automation (functions and actions), ticket and mail templates, and secrets, and read conversation history. Every tool respects the API key's role permissions.

## Endpoint and auth
- URL: `https://<your-org-host>/api/v1/workspaces/<workspace-slug>/mcp`
- Header: `Authorization: Bearer <wsk_...>` (a workspace API key; create one in workspace settings under API keys)
- Transport: MCP Streamable HTTP

## Tool catalogue by domain
| Domain | Tools (prefix `fertigai_`) | Skill |
|---|---|---|
| Agents + branches | agents_list/get/create/rename/delete, agent_branch_get/create/update_config, agent_set_active_branch | fertigai-agents |
| Functions | functions_list/get/create/update/delete/test | fertigai-functions |
| Actions | actions_list/get/create/update/delete/test | fertigai-actions |
| Ticket templates | ticket_templates_list/get/create/update/archive/unarchive | fertigai-ticket-templates |
| Mail templates | mail_templates_list/get/create/update/delete | fertigai-mail-templates |
| Secrets | secrets_list/create/update/delete | fertigai-secrets |
| Conversations (read only) | conversations_list/get | fertigai-conversations |

Load the per-domain skill for field details, workflows, and gotchas. Functions and actions also share a JavaScript scripting environment (the `run(ctx)` contract and built-in primitives like `mail.send` and `llm.send`); that is documented in the **fertigai-scripting** skill.

## Conventions shared by every tool
- **Public IDs** are prefixed and opaque: agents `agt_`, functions `fn_`, actions `act_`, secrets `sec_`, mail templates `mtpl_`, and similar. Always pass an id that a `list` or `get` returned; never invent one.
- **Pagination**: list tools accept `cursor` and `page_size` (1 to 100, default 20). Responses look like `{ "data": [...], "pagination": { "next": "<cursor|null>", "has_more": bool } }`. To page, pass the previous response's `next` as `cursor`.
- **Permissions and licenses**: writes need the matching role permission (agents need Agents-Edit; functions, actions, mail templates, and secrets need Integrations-Manage; ticket templates need Tickets-Manage plus the ticketing product). Reads need the view permission. A call without permission returns `isError: true` with text like `403: forbidden`.
- **Errors**: a failed call returns `isError: true` and a message like `404: agent not found` or `422: <validation detail>`. Read the status and detail, fix the arguments, and retry.
- **Nested JSON fields** (agent config, function/action parameter schema, ticket fields and statuses, mail variables) are passed as JSON exactly as the matching `get` returns them. The safe pattern for any edit is: get, modify, send back.

## Typical flow
1. `list` or `get` to discover what exists and copy an id or a config object.
2. `create` or `update` with the required fields (see the domain skill).
3. For functions and actions, `test` a draft before saving.
