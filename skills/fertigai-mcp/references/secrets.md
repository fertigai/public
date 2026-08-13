# Managing secrets (fertigai_secrets_*)

## Overview
A **secret** is an encrypted key/value used by functions, actions, and integrations. Secrets are **write-only over this API**: you can create, update (rotate), and delete them, and list their names, but you can never read a secret's value back.

Shared conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args | Notes |
|---|---|---|
| `fertigai_secrets_list` | `search?`, `cursor?`, `page_size?` | returns metadata only: `id`, `key`, `created_at`. No values. |
| `fertigai_secrets_create` | `key`, `value` | store a new secret |
| `fertigai_secrets_update` | `id`, `value` | rotate the value |
| `fertigai_secrets_delete` | `id` | remove it |

## Rules
- **No read of values**: there is no get-value or reveal tool, by design. `list` shows only names and ids.
- **Reserved prefixes**: keys starting with `conn:` or `mcp:` are managed by the platform and cannot be created, updated, or deleted here (you get a 400 or 409 error).
- Deleting a secret that a connection still references is refused with a conflict error; remove the reference first.

## Common mistakes
- Expecting to read a value back: it is impossible; store it elsewhere if you need it.
- Using a `conn:` or `mcp:` prefix: reserved.

Writes need Integrations-Manage.
