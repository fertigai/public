---
name: fertigai-mail-templates
description: Use when creating, editing, or deleting fertig.ai mail templates (reusable HTML email templates) through the workspace MCP.
---

# Managing mail templates (fertigai_mail_templates_*)

## Overview
A **mail template** is a reusable HTML email with named `variables` that get filled in when the mail is sent. It has a `name`, a `variables` list, and an `html` body.

Shared conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill.

## Tools
| Tool | Args |
|---|---|
| `fertigai_mail_templates_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_mail_templates_get` | `id` |
| `fertigai_mail_templates_create` | `name`, `variables`, `html` |
| `fertigai_mail_templates_update` | `id`, `name`, `variables`, `html` |
| `fertigai_mail_templates_delete` | `id` |

## Fields
- `variables`: an array of `{ name, description, required }` objects declaring the placeholders the `html` uses.
- `html`: the template body (up to a size limit). It is compiled and validated on save; a broken template returns a validation error.
- The URL slug is derived from `name` by the server, so you do not set or change a slug. The slug stays fixed once created even if you rename.

## Example
```
fertigai_mail_templates_create {
  "name": "Welcome",
  "variables": [ { "name": "first_name", "description": "recipient given name", "required": true } ],
  "html": "<p>Hello {{ first_name }}, welcome!</p>"
}
```

## Common mistakes
- Trying to set a `slug`: it is server-derived and any value you send is ignored.
- Sending `html` that fails to compile: fix the template syntax and retry.

Writes need Integrations-Manage.
