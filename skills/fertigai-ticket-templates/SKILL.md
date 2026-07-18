---
name: fertigai-ticket-templates
description: Use when creating, editing, archiving, or listing fertig.ai ticket templates (the field and status schema for support tickets) through the workspace MCP.
---

# Managing ticket templates (fertigai_ticket_templates_*)

## Overview
A **ticket template** defines the shape of a support ticket: its custom `fields` and its `statuses` (the workflow states). Templates are archived rather than hard-deleted.

Shared conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill.

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
- `fields`: an array of `{ key, label, type, required }` objects (the server assigns each field's `id`). `type` is a field-type string such as `text`, `number`, or `select`; `get` an existing template to see the exact set your workspace uses.
- `statuses`: an array of `{ label, category, color }` objects (the server assigns each status's `id`). `default_status_id` names the starting status. Because status ids are server-assigned on create, omit `default_status_id` on the first create, then `get` the template to read the generated status ids and set it with `update`.
- There is no delete: use `fertigai_ticket_templates_archive` (and `unarchive` to restore). `list` hides archived templates unless you pass `include_archived: true`.

## Editing pattern
Get the template, edit the returned `fields` and `statuses` arrays, and send them back via `update`. Building these arrays from scratch risks a validation error; start from `get`.

## Common mistakes
- Expecting a delete tool: archive instead.
- Forgetting `include_archived: true` when looking for an archived template.
- Missing the ticketing product: ticket templates need Tickets-Manage AND the ticketing product; without the product every call returns a not-licensed error.

Requires Tickets-Manage and the ticketing product.
