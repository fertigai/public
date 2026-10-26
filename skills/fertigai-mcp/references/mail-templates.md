# Managing mail templates (fertigai_mail_templates_*)

## Overview
A **mail template** is a reusable HTML email with named `variables` that get filled in when the mail is sent. It has a `name`, a `variables` list, and an `html` body.

Shared conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_mail_templates_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_mail_templates_get` | `id` |
| `fertigai_mail_templates_create` | `name`, `variables`, `html` |
| `fertigai_mail_templates_update` | `id`, `name`, `variables`, `html` |
| `fertigai_mail_templates_delete` | `id` |
| `fertigai_mail_templates_preview` | `html`, `variables?` |

## Fields
- `variables`: an array of `{ name, description, required }` objects declaring the placeholders the `html` uses. Variables marked `required` are enforced at send time.
- `html`: the template body, Jinja-style syntax (`{{ name }}` placeholders), max 256 KiB. It is compiled and validated on save; a broken template returns a validation error. Interpolated values are HTML-escaped automatically at render time.
- The URL slug is derived from `name` by the server, so you do not set or change a slug. The slug stays fixed once created even if you rename.

## Preview before saving
`fertigai_mail_templates_preview` renders `html` with sample `variables` (an object keyed by variable name) and returns the resulting html. Nothing is stored, so it works on a draft that has no template yet.
- Values are HTML-escaped; a template opts one value out with `| safe` (`{{ signature | safe }}`).
- A variable you do not send renders empty.
- `html` is at most 256 KB; larger html, or malformed template syntax, is a `422`.
```
fertigai_mail_templates_preview {
  "html": "<p>Hello {{ first_name }}, welcome!</p>",
  "variables": { "first_name": "Ada" }
}
```

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
- Sending `html` that fails to compile: fix the template syntax and retry. Check a draft with `fertigai_mail_templates_preview` first.

Writes need Integrations-Manage; preview needs only workspace access.
