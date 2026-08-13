# Managing ticket templates (fertigai_ticket_templates_*)

## Overview
A **ticket template** defines the shape of a support ticket: its custom `fields` and its `statuses` (the workflow states). Templates are archived rather than hard-deleted.

Shared conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_ticket_templates_list` | `include_archived?`, `cursor?`, `page_size?` |
| `fertigai_ticket_templates_get` | `id` |
| `fertigai_ticket_templates_create` | `key`, `name`, `description?`, `fields`, `statuses`, `default_status_id?` |
| `fertigai_ticket_templates_update` | `id`, `name`, `description?`, `fields`, `statuses`, `default_status_id?` |
| `fertigai_ticket_templates_archive` | `id` |
| `fertigai_ticket_templates_unarchive` | `id` |

## Fields
- `key`: a stable identifier for the template, set on create and not changeable on update.
- `fields`: an array of `{ id?, key, label, type, required?, help_text?, placeholder?, default?, config? }`. `type` is one of `SHORT_TEXT`, `LONG_TEXT`, `NUMBER`, `DATE`, `DATE_TIME`, `DROPDOWN`, `MULTI_SELECT`, `CHECKBOX`, `EMAIL`, `PHONE` (case-insensitive). `DROPDOWN` and `MULTI_SELECT` require at least one option in `config.options` (`[{ value, label }]`). Keys and ids must be unique within the template.
- `statuses`: an array of `{ id?, label, category, color? }`. `category` is one of `NEW`, `OPEN`, `PENDING`, `SOLVED`, `CLOSED` (case-insensitive; responses return it uppercase).
- **Ids**: omit `id` (or send `""`) on a new field or status and the server generates one; a non-empty `id` you send is kept as-is. On `update`, always reuse the ids returned by `get` so existing tickets keep pointing at the same fields and statuses.
- `default_status_id` names the status new tickets start in and must match an `id` in `statuses`. To set it on create, give that status your own `id` and reference it; otherwise omit it, `get` the generated ids, and set it with `update`.
- There is no delete: use `fertigai_ticket_templates_archive` (and `unarchive` to restore). `list` hides archived templates unless you pass `include_archived: true`.

## Editing pattern
Get the template, edit the returned `fields` and `statuses` arrays, and send them back via `update`. Building these arrays from scratch risks a validation error; start from `get`.

## Common mistakes
- Expecting a delete tool: archive instead.
- A `DROPDOWN` or `MULTI_SELECT` field without `config.options`: rejected.
- `default_status_id` not matching any status `id` in the same payload: rejected.
- Sending fresh ids on `update` instead of the ones `get` returned: existing tickets lose their link to those fields/statuses.
- Forgetting `include_archived: true` when looking for an archived template.
- Missing the ticketing product: ticket templates need Tickets-Manage AND the ticketing product; without the product every call returns a not-licensed error.

Requires Tickets-Manage and the ticketing product.
