# Reading workspace admin (members, groups, roles, API keys, licenses, billing)

## Overview
These tools let you read how a workspace is set up: the workspace itself and your role in it, its members and pending invites, team groups, roles and the permission catalogue, API key metadata, products and licenses, the chat models a chat agent can use, and the billing plan and usage.

**Everything here is read-only.** There are no tools to invite, remove or change members, groups, roles, workspace settings or licenses, and none to create, rotate or revoke API keys. Those stay in the portal: tell the user to make the change there, and do not invent a tool for it.

Shared conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args | Notes |
|---|---|---|
| `fertigai_workspace_get` | none | the workspace's id and slug, your user id and your role. Any workspace member. |
| `fertigai_members_list` | `search?`, `role?`, `status?`, `group?` | members and pending invites, in one page. Team-View. |
| `fertigai_members_get` | `member_id` | one member (`usr_...`) or pending invite (`wsi_...`). Team-View. |
| `fertigai_groups_list` | none | team groups, each with its members. Groups-View. |
| `fertigai_roles_list` | none | roles and their permission keys. Roles-View. |
| `fertigai_permissions_list` | none | the permission catalogue roles are made of. Any workspace member. |
| `fertigai_api_keys_list` | `cursor?`, `page_size?` | API key metadata, never the key. API-Keys-Manage, signed-in user only (see below). |
| `fertigai_api_keys_get` | `id` | one API key's metadata (`wak_...`). API-Keys-Manage, signed-in user only. |
| `fertigai_products_list` | none | the products a member can hold a license for. Licenses-View. |
| `fertigai_licenses_list` | none | the workspace's product licenses and their seat counts. Licenses-View. |
| `fertigai_chat_models_list` | `cursor?`, `page_size?` | enabled chat models. Needs the Chat product. |
| `fertigai_billing_plan_get` | none | the billing plan. Billing-View. |
| `fertigai_billing_usage_get` | `period_start?`, `period_end?` | usage counters for a period. Billing-View. |

## Workspace
`fertigai_workspace_get` returns who you are in the targeted workspace:
```json
{
  "workspace_public_id": "wsp_...",
  "workspace_slug": "support",
  "user_public_id": "usr_...",
  "role": "ADMIN"
}
```
- `role` is the role key in upper case: `OWNER`, `ADMIN`, `EDITOR`, `VIEWER`, `MANAGER` for a Workspace Manager, or a custom role's key.
- It returns no workspace name or settings. For the workspace names you can reach, use `fertigai_whoami`.

## Members and pending invites
`fertigai_members_list` returns joined members and open invites together. Each entry has:
- `id`: `usr_...` for a member, `wsi_...` for a pending invite;
- `email`, `first_name`, `last_name`, `color`;
- `role_public_id` (`role_...`) and `role_name`;
- `product_licenses`: the product keys this person holds a seat for (the owner holds every product);
- `status`: `ACTIVE` (joined) or `PENDING` (invited, not joined yet);
- `joined_at` (RFC 3339).

**How invites appear**: an invite has `status: "PENDING"`, a `wsi_` id, an empty `joined_at`, and empty `first_name` and `last_name`. To list only invites, pass `status: "PENDING"`.

Filters, all optional:
- `search`: case-insensitive substring of a member's name or email, or an invite's email;
- `role`: a role id (`role_...`) from `fertigai_roles_list`;
- `status`: `ACTIVE` or `PENDING` (any other value is a `400`);
- `group`: a group id (`grp_...`) from `fertigai_groups_list`; it never matches an invite.

The list always comes back as one page: `has_more` is `false` and `cursor` is ignored. `fertigai_members_get` takes either id kind and returns the same fields for one entry.

## Groups
`fertigai_groups_list` returns every team group, unpaginated, in `data`, with its members inside, so there is no separate group members tool:
```json
{
  "data": [
    {
      "id": "grp_...",
      "name": "Support",
      "description": "First-line support",
      "color": "#2563eb",
      "icon": "headset",
      "member_count": 1,
      "members": [
        { "id": "usr_...", "first_name": "Ana", "last_name": "Lopez", "email": "ana@example.com", "color": "#16a34a" }
      ],
      "created_at": "2026-01-15T09:00:00Z",
      "updated_at": "2026-01-15T09:00:00Z"
    }
  ]
}
```
`color` (`#rrggbb`) and `icon` (an icon name) can be empty. Groups contain members only, never pending invites.

## Roles and the permission catalogue
- `fertigai_roles_list` returns every role, unpaginated: `id` (`role_...`), `key` (the system roles are `owner`, `admin`, `editor`, `viewer`), `name`, `description`, `is_system`, `permission_keys` (for example `billing.manage`), `members_count`, `created_at`, `updated_at`.
- `fertigai_permissions_list` returns the catalogue that `permission_keys` come from, grouped:
```json
{
  "data": {
    "groups": [
      {
        "key": "billing",
        "label": "Billing",
        "permissions": [
          { "key": "billing.view", "label": "View billing", "description": "..." }
        ]
      }
    ]
  }
}
```
A group's `key` is the prefix of its permission keys. Use the catalogue to explain in plain words what a role can do: match each of the role's `permission_keys` to its `label`.

## API keys (metadata only)
`fertigai_api_keys_list` (newest first, every status, paginated) and `fertigai_api_keys_get` return metadata only:
- `id` (`wak_...`), `name`, `role_key`;
- `last_four`: the last four characters of the key, for recognising it. It is not the key;
- `status`: `ACTIVE`, `EXPIRED` or `REVOKED`;
- `expires_at` (empty = never expires), `last_used_at` (empty = never used), `revoked_at`, `created_at`;
- `created_by` (`usr_...`, or empty).

No tool ever returns the key itself. Creating, rotating and revoking API keys stay in the portal.

**These two tools work only when the MCP client is connected as a signed-in user** (an OAuth connection, per-workspace or org-wide) that has API-Keys-Manage. On a connection authenticated with a workspace API key (`wsk_...`), both tools always fail with `403`, whatever that key's role: an API key cannot read API keys. `fertigai_whoami` tells you which you have: `principal.kind` is `"user"` or `"api_key"`. On an API key connection, do not retry; tell the user to look in the portal or to connect as a user.

## Products and licenses
- `fertigai_products_list` returns `data.products[]`: `key` (for example `agents`, `chat`, `tasks`, `telephony`), `label`, `description`.
- `fertigai_licenses_list` returns the workspace's product licenses: `id` (`wpl_...`), `product_key`, `assigned_count` (seats held by active members, which is the billable count), `created_at`, `updated_at`.
- Who holds a seat is on each member: the `product_licenses` field in `fertigai_members_list`.

Assigning or removing a license stays in the portal.

## Chat models
`fertigai_chat_models_list` (paginated) returns the enabled models a chat agent can use:
- `id` (`mdl_...`), `display_name`, `model_name`, `provider`, `description`;
- `region`: `EU`, `US` or `GLOBAL`;
- `input_cost_per_million`, `output_cost_per_million`: USD per million tokens, as decimal strings;
- `capabilities`: any of `VISION`, `TOOLS`, `WEB_SEARCH`, `REASONING`;
- `max_context_length`, `has_image`, `is_new`, `is_enabled`, `sort_order`.

It needs the Chat product.

## Billing plan and usage
- `fertigai_billing_plan_get`: `plan_name`, `price` (decimal string), `currency`, `included_minutes`, `included_agents` (`-1` = unlimited), `billing_cycle` (`MONTHLY` or `YEARLY`), `next_billing_date`. `data` is `null` when the workspace has no plan.
- `fertigai_billing_usage_get`: `conversations`, `agent_seconds`, `agent_chats`, `forwarded_minutes`, `emails_sent`, `sms_sent`, `actions_executed`, plus the `period_start` and `period_end` used.
  - Pass `period_start` and `period_end` as `YYYY-MM-DD`. They default to the first of this month and today. A malformed date is a `400`.

Changing the plan stays in the portal.

## Permissions
- Workspace and permission catalogue: any workspace member.
- Members and invites: Team-View. Groups: Groups-View. Roles: Roles-View.
- API keys: API-Keys-Manage, on a signed-in user connection only.
- Products and licenses: Licenses-View. Billing plan and usage: Billing-View.
- Chat models: the Chat product.

A call without the permission returns `isError: true` with `403: forbidden`.

## Common mistakes
- Asking for a write: inviting, removing or re-roling a member, editing groups or roles, assigning licenses, or creating, rotating or revoking an API key. There is no tool; send the user to the portal.
- Calling the API key tools on a `wsk_` connection: always `403`. Check `fertigai_whoami` first.
- Treating `last_four` as the key: it only identifies a key; the key itself is never returned.
- Paging `fertigai_members_list`: it is always one page.
- Looking for a group members tool: members are inside `fertigai_groups_list`.
