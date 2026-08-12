---
name: fertigai-mcp
description: Use when managing a fertig.ai workspace through its MCP endpoint - creating or editing agents, branch attachments, functions, actions, ticket templates, mail templates, or secrets, or reading conversations - via the fertigai_* tools.
---

# fertig.ai Workspace MCP (fertigai_*)

## Overview
The workspace MCP exposes your fertig.ai workspace configuration as MCP tools, all named `fertigai_*`. Point an MCP client at your workspace endpoint, authenticate with a workspace API key, and the tools let you manage agents and their configuration and attachments, automation (functions and actions), ticket and mail templates, and secrets, and read conversation history. Every tool respects the API key's role permissions.

## Start with `fertigai_whoami`
Call `fertigai_whoami` FIRST, before any other tool, on every new connection. It takes no arguments and tells you how to call every other tool correctly:
```jsonc
{
  "principal": { "kind": "user", "name": "…", "email": "…" },   // or {"kind":"api_key"} for a wsk_ key
  "organization": { "slug": "acme", "name": "Acme" },
  "grant": "org_wide",                    // "org_wide" | "workspace"
  "scopes": ["mcp:read", "mcp:write"],
  "workspaces": [ { "slug": "support", "name": "Support", "role": "admin" }, … ]
}
```
- **`grant: "workspace"`**: this connection is already scoped to one workspace. Every other tool call works as before; you do not need to pass `workspace`.
- **`grant: "org_wide"`**: this connection can reach every workspace you're an active member of in the org. Every other `fertigai_*` workspace tool then REQUIRES a `workspace` argument: a slug taken from this response's `workspaces[]`. A call that omits it fails with an error telling you to pass one.
- **`workspaces`** lists only what this connection can reach: every workspace in the org for an org-wide grant, or the single bound workspace for a per-workspace grant (including a `wsk_` key).

Skipping `whoami` and guessing whether `workspace` is needed will fail on an org-wide connection and is never necessary: always call it first.

## Endpoint and auth
Two connection kinds, depending on whether you connect to one workspace or to every workspace you belong to in the org. Either way, transport is MCP Streamable HTTP.

- **Per-workspace** (the default):
  - URL: `https://<your-org-host>/api/v1/workspaces/<workspace-slug>/mcp`
  - Header: `Authorization: Bearer <wsk_...>` (a workspace API key; create one in workspace settings under API keys), or a per-workspace OAuth grant.
  - Every tool call is already scoped to `<workspace-slug>`; the optional `workspace` argument is ignored.
- **Org-wide** (opt-in):
  - URL: `https://<your-org-host>/api/v1/mcp` (no workspace slug in the path)
  - Header: an org-wide OAuth grant (connect once, reach every workspace you're an active member of in the org).
  - Every workspace tool call REQUIRES a `workspace` argument (a slug), since the connection has no single workspace to default to.

Call `fertigai_whoami` right after connecting to confirm which kind you have (its `grant` field) and, on an org-wide connection, which workspace slugs you can target.

## Tool catalogue by domain
| Domain | Tools (prefix `fertigai_`) | Skill |
|---|---|---|
| Identity (start here) | whoami | this skill |
| Agents + branches | agents_list/get/create/rename/delete, agent_branch_get/create/configure, agent_set_active_branch | fertigai-agents |
| Branch attachments | agent_attachments_get, agent_branch_configure (shared with config), system_tools_list, knowledge_base_list | fertigai-attachments |
| Functions | functions_list/get/create/update/delete/test | fertigai-functions |
| Actions | actions_list/get/create/update/delete/test | fertigai-actions |
| Ticket templates | ticket_templates_list/get/create/update/archive/unarchive | fertigai-ticket-templates |
| Mail templates | mail_templates_list/get/create/update/delete | fertigai-mail-templates |
| Secrets | secrets_list/create/update/delete | fertigai-secrets |
| Conversations (read only) | conversations_list/get | fertigai-conversations |

Load the per-domain skill for field details, workflows, and gotchas. Functions and actions also share a JavaScript scripting environment (the `run(ctx)` contract and built-in primitives like `mail.send` and `llm.send`); that is documented in the **fertigai-scripting** skill. `fertigai_whoami` is workspace-agnostic (no `workspace` argument, documented above); every other tool listed here is workspace-scoped (see Conventions below).

If a referenced skill is not installed locally, fetch its latest version from GitHub:
`https://raw.githubusercontent.com/fertigai/public/main/skills/<skill-name>/SKILL.md`
(for example `.../skills/fertigai-agents/SKILL.md`; the agent workflow reference is `.../skills/fertigai-agents/workflow-reference.md`). Every skill in the table lives there, always current, so installing only this skill is enough when your client can fetch URLs.

## Conventions shared by every tool
- **`workspace` targeting**: every tool except `fertigai_whoami` accepts an optional `workspace` argument (a workspace slug). On a per-workspace connection it's ignored, the connection is already scoped. On an org-wide connection it's REQUIRED: pass a slug from `fertigai_whoami`'s `workspaces[]`. Never guess a slug; use only a slug that `fertigai_whoami` (or a `list` tool) returned.
- **Public IDs** are prefixed and opaque: agents `agt_`, functions `fn_`, actions `act_`, secrets `sec_`, mail templates `mtpl_`, and similar. Always pass an id that a `list` or `get` returned; never invent one.
- **Pagination**: list tools accept `cursor` and `page_size` (1 to 100, default 20). Responses look like `{ "data": [...], "pagination": { "next": "<cursor|null>", "has_more": bool } }`. To page, pass the previous response's `next` as `cursor`.
- **Permissions and licenses**: writes need the matching role permission (agent config needs Agents-Edit; branch attachments, functions, actions, mail templates, and secrets need Integrations-Manage; ticket templates need Tickets-Manage plus the ticketing product). Reads need the view permission. A call without permission returns `isError: true` with text like `403: forbidden`.
- **Errors**: a failed call returns `isError: true` and a message like `404: agent not found` or `422: <validation detail>`. Read the status and detail, fix the arguments, and retry.
- **Nested JSON fields** (agent config, function/action parameter schema, ticket fields and statuses, mail variables) are passed as JSON exactly as the matching `get` returns them. The safe pattern for any edit is: get, modify, send back.

## Typical flow
1. `fertigai_whoami` once per connection, to learn your grant type and (if org-wide) which workspace slugs you can target.
2. `list` or `get` to discover what exists and copy an id or a config object.
3. `create` or `update` with the required fields (see the domain skill).
4. For functions and actions, `test` a draft before saving.
