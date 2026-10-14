# Managing tasks (fertigai_task_*, fertigai_tasks_*)

## Overview
**Tasks** are the workspace's work items: support requests, call-backs, follow-ups. Each task belongs to a **task type**, which defines its workflow (statuses) and its form (fields picked from the workspace **field library**). Every workspace also has one built-in type, "No type" (key `TASK`, or `TASK2` when a type already used `TASK`): tasks created without a type land in it. It is always open, has no fields and cannot be archived; its name, description and statuses can be edited. **Labels** are shared across types, and **task triggers** automate changes when tasks are created, updated, or sit too long.

Shared conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Concepts
- **Task type** (`tty_...`): `key` (2 to 5 uppercase letters or digits, starting with a letter, immutable), `name`, `description`, `statuses`, `default_status_id`, the selected `fields`, and `private`. Tasks of a type are numbered per type and addressed as `KEY-n` (for example `SUP-12`).
- **Statuses**: `[{ id, label, category, color? }]`. `category` is one of `NEW`, `OPEN`, `PENDING`, `SOLVED`, `CLOSED` (responses return it uppercase). Omit `id` (or send `""`) on a new status and the server generates an `sts_...` id. A type needs at least one status; a status still used by tasks cannot be removed.
- **Private types**: an open type is visible to everyone with Tasks access. A private type, and its tasks, are visible only to its members and to Tasks-Admin. The creator of a private type becomes its first member; replace the member list with `fertigai_task_types_set_members`.
- **Field library** (`tfd_...`): workspace-wide fields that types select as `fields: [{ field_id, required }]` (list order is form order). Each field has a `key` and a `type` (both immutable), a `label`, and optional `help_text`, `placeholder`, `default`, and `config`. `type` is one of `SHORT_TEXT`, `LONG_TEXT`, `NUMBER`, `DATE`, `DATE_TIME`, `DROPDOWN`, `MULTI_SELECT`, `CHECKBOX`, `EMAIL`, `PHONE`, `USER` (value: a `usr_...` id), `CONTACT` (value: a `con_...` id). `DROPDOWN` and `MULTI_SELECT` need `config.options` (`[{ value, label }]`); an option that tasks still use cannot be removed.
- **Labels** (`tlb_...`): `name` (unique, case-insensitive) and an optional `color`. Labels are archived, never deleted.
- **Task** (`tsk_...`): `title`, markdown `description`, `status_id`, `priority` (`NONE`, `URGENT`, `HIGH`, `MEDIUM`, `LOW`), an assignee user and/or an assigned group, a requester (a workspace member or a contact, not both), an optional `parent` task, field values, labels, free-form `tags`, and a `source` (`EMAIL`, `SMS`, `WHATSAPP`, `WEB`, `PHONE`). Field values are written keyed by field **key**; responses return `field_values` keyed by field **id** (`tfd_...`).

## Tools
**Task types**
| Tool | Args |
|---|---|
| `fertigai_task_types_list` | `include_archived?`, `search?`, `cursor?`, `page_size?` |
| `fertigai_task_types_get` | `id` |
| `fertigai_task_types_create` | `body`: `{ key, name, description?, fields?: [{ field_id, required }], statuses: [{ id?, label, category, color? }], default_status_id?, private? }` |
| `fertigai_task_types_update` | `id`, `body`: as for create without `key` (full replace) |
| `fertigai_task_types_archive`, `fertigai_task_types_unarchive` | `id` |
| `fertigai_task_types_set_members` | `id`, `body`: `{ user_ids: ["usr_..."] }` (replaces the member list) |

**Field library**
| Tool | Args |
|---|---|
| `fertigai_task_fields_list` | `include_archived?`, `search?`, `cursor?`, `page_size?` |
| `fertigai_task_fields_get` | `id` |
| `fertigai_task_fields_create` | `body`: `{ key, label, type, help_text?, placeholder?, default?, config?: { options: [{ value, label }] } }` |
| `fertigai_task_fields_update` | `id`, `body`: `{ label, help_text?, placeholder?, default?, config? }` (full replace; `key` and `type` never change) |
| `fertigai_task_fields_archive`, `fertigai_task_fields_unarchive` | `id` (archived fields are hidden on forms; stored values are kept) |

**Labels**
| Tool | Args |
|---|---|
| `fertigai_task_labels_list` | `include_archived?`, `search?`, `cursor?`, `page_size?` |
| `fertigai_task_labels_get` | `id` |
| `fertigai_task_labels_create` | `body`: `{ name, color? }` |
| `fertigai_task_labels_update` | `id`, `body`: `{ name, color? }` (full replace) |
| `fertigai_task_labels_archive`, `fertigai_task_labels_unarchive` | `id` |

**Triggers**
| Tool | Args |
|---|---|
| `fertigai_task_triggers_list` | `search?`, `cursor?`, `page_size?` (evaluation order; triggers have no archive) |
| `fertigai_task_triggers_get` | `id` |
| `fertigai_task_triggers_create` | `body`: `{ name, position?, active?, run_on, conditions: { all: [...], any: [...] }, actions: [...] }` (see Triggers) |
| `fertigai_task_triggers_update` | `id`, `body`: as for create (full replace) |
| `fertigai_task_triggers_delete` | `id` (permanent) |

**Tasks**
| Tool | Args |
|---|---|
| `fertigai_tasks_list` | `type?`, `status_category?`, `assignee?`, `priority?`, `label?`, `tag?`, `q?`, `cursor?`, `page_size?` |
| `fertigai_tasks_get` | `ref` |
| `fertigai_tasks_create` | `body` (see below) |
| `fertigai_tasks_update` | `ref`, `body` (see below) |
| `fertigai_tasks_comment` | `ref`, `body`: the comment text (markdown) |

- `ref` is a task key (`SUP-12`), a task id (`tsk_...`), or, for a migrated ticket, its `tkt_...` id or old ticket number (for example `42`). After a task moves to another type, its old key keeps resolving.
- `fertigai_tasks_list` filters are comma-separated lists, newest activity first: `type` (type ids or keys), `status_category` (`NEW`, `OPEN`, `PENDING`, `SOLVED`, `CLOSED`), `assignee` (`usr_...`, `grp_...`, `ME`, or `NONE`), `priority`, `label` (`tlb_...`), `tag`, and `q` (free text in titles). **`ME` needs a user session (OAuth); with an API key it matches nothing.** The list shows only tasks the caller can see.
- In a comment or description, `[@Name](mention:usr_...)` mentions a member and subscribes them to the task.

`fertigai_tasks_create` body:
```json
{
  "type_id": "SUP",
  "title": "Call back about invoice 1042",
  "description": "Customer asked for a copy of the invoice.",
  "status_id": "sts_...",
  "priority": "HIGH",
  "assignee_user_id": "usr_...",
  "assigned_group_id": "grp_...",
  "requester_contact_id": "con_...",
  "parent": "SUP-7",
  "fields": { "invoice_number": "1042" },
  "label_ids": ["tlb_..."],
  "tags": ["billing"],
  "source": "PHONE"
}
```
Only `title` is required. Without `type_id` (a `tty_...` id or a type key) the task lands in the built-in type. `status_id` defaults to the type's default status and `priority` to `NONE`. Use `requester_user_id` instead of `requester_contact_id` for a workspace member as requester; send at most one of the two. `parent` takes a task id or key.

`fertigai_tasks_update` body (a partial update):
```json
{
  "status_id": "sts_...",
  "priority": "LOW",
  "assignee_user_id": "",
  "fields": { "invoice_number": "1043", "notes": null },
  "labels_add": ["tlb_..."],
  "labels_remove": ["tlb_..."],
  "tags_add": ["vip"],
  "tags_remove": ["billing"],
  "conversations_link": ["conv_..."],
  "conversations_unlink": ["conv_..."]
}
```
- Absent keys stay unchanged; `""` clears a clearable property (here: unassigns the user).
- Also accepted: `title`, `description`, `assigned_group_id`, `requester_user_id`, `requester_contact_id` (sending either requester key replaces the requester), `parent` (`""` makes the task top level), and `type_id` (moves the task to another type: it gets a new key, and its status is mapped by category).
- `fields` merges by field key; `null` clears a value.
- Labels, tags and linked conversations change only through the `_add`/`_remove` and `_link`/`_unlink` lists; there is no `label_ids` or `tags` on update.

## Editing pattern
- **Types, fields, labels, triggers** are replaced in full by their `update` tools. Get the entry, change the returned object, and send all of it back. On a type, reuse the status ids that `get` returned, or tasks lose their status.
- **Tasks** are patched (`fertigai_tasks_update`): send only what changes.
- There is no delete for types, fields, or labels: archive them. `list` hides archived entries unless you pass `include_archived: true`.

## Triggers
A trigger runs its actions on a task when its conditions match. Triggers are evaluated in `position` order.

**`run_on`**
- `CREATED`, `UPDATED`, or `BOTH` (the default): on task writes. A comment is an `UPDATED` event without changes.
- `TIME`: checked every 5 minutes against a clock condition (see below).

**Conditions**: `{ "all": [...], "any": [...] }` matches when every `all` condition holds and at least one `any` condition holds; an empty list adds no constraint. Each condition is `{ field, field_key, operator, value }`.
- `field`: `STATUS` (a status id), `STATUS_CATEGORY`, `PRIORITY`, `ASSIGNEE` (a `usr_...` id), `REQUESTER_EMAIL`, `CREATED_VIA` (`MANUAL`, `ACTION`, `API`, `FUNCTION`, `TRIGGER`), `TITLE`, `FIELD` (with `field_key` = a field key; `field_key` is `""` for every other field), `TAGS`, `TYPE` (a `tty_...` id), `LABEL` (`tlb_...` ids), and the time fields `TIME_IN_STATUS`, `TIME_SINCE_ACTIVITY`, `TIME_SINCE_CREATED`.
- `operator`: `IS`, `IS_NOT`, `IS_EMPTY`, `CONTAINS`, `GREATER`, `LESS`, or the transitions `CHANGED`, `CHANGED_TO`, `CHANGED_FROM`.
- `TAGS` and `LABEL` take `CONTAINS`, `IS_NOT`, `IS_EMPTY`, or a transition, with a list value. `TYPE` takes `IS`, `IS_NOT`, or a transition.
- **Time conditions** are allowed only in `TIME` triggers and only in `all`. They take `GREATER` or `LESS` with whole minutes from 1 to 525600 (one year). A `TIME` trigger needs exactly one `GREATER` time condition in `all` (its clock) and cannot use transition operators. It fires at most once per task while the clock keeps running: a status change restarts `TIME_IN_STATUS`, user or script activity restarts `TIME_SINCE_ACTIVITY`, and trigger writes restart nothing.

```json
{
  "name": "Escalate pending support tasks after 2 days",
  "run_on": "TIME",
  "conditions": {
    "all": [
      { "field": "TIME_IN_STATUS", "field_key": "", "operator": "GREATER", "value": 2880 },
      { "field": "STATUS_CATEGORY", "field_key": "", "operator": "IS", "value": "PENDING" },
      { "field": "TYPE", "field_key": "", "operator": "IS", "value": "tty_..." }
    ],
    "any": []
  },
  "actions": [
    { "action": { "SetPriority": { "priority": "URGENT" } } },
    { "action": { "AddComment": { "body": "Pending for 2 days, escalated." } } }
  ]
}
```

**Actions**: each entry is `{ "action": { "<Variant>": { ... } } }` with exactly one variant. The changing actions (`SetStatus`, `SetPriority`, `SetAssignee`, `SetField`, tag and label adds and removes, `MoveToType`) fold into one task update, later ones winning; the others run after it, in order.
```json
[
  { "action": { "SetStatus": { "status_id": "sts_..." } } },
  { "action": { "SetPriority": { "priority": "HIGH" } } },
  { "action": { "SetAssignee": { "assignee": "usr_..." } } },
  { "action": { "SetField": { "key": "invoice_number", "value": "1042" } } },
  { "action": { "AddTags": { "tags": ["vip"] } } },
  { "action": { "RemoveTags": { "tags": ["new"] } } },
  { "action": { "AddLabels": { "label_ids": ["tlb_..."] } } },
  { "action": { "RemoveLabels": { "label_ids": ["tlb_..."] } } },
  { "action": { "MoveToType": { "type_id": "tty_..." } } },
  { "action": { "AddSubscriber": { "user_id": "usr_...", "group_id": "" } } },
  { "action": { "AddComment": { "body": "Escalated {{key}} from {{source}}." } } },
  { "action": { "SendEmail": { "body": "Hello, we received your request \"{{title}}\" and will get back to you." } } },
  { "action": { "CreateTask": { "task": { "type_id": "tty_...", "title": "Follow up on {{key}}", "parent": "{{id}}" } } } },
  { "action": { "RunFunction": { "function": "fn_...", "input": { "task_key": "{{key}}" }, "connection_bindings": [] } } }
]
```
- `SetAssignee`: a `usr_...` id, or `UNASSIGNED`.
- `AddSubscriber`: exactly one of `user_id` and `group_id` (a group subscribes its current members).
- `SendEmail`: a plain-text email to the task's requester (the contact's or member's email), subject `Re: <title> [<task id>]`. It is skipped, with an error on the task's timeline, when the requester has no email, the task's source is not `EMAIL` (a task without a source counts as email), or the body is empty.
- `CreateTask`: `task` takes the `fertigai_tasks_create` body. `"parent": "{{id}}"` makes the new task a sub-task of the triggering one. A trigger creates at most one task per triggering task.
- `RunFunction`: `function` is a `fn_...` id, `input` holds the function's parameters, and `connection_bindings` (`[{ "role", "connection_public_id" }]`) binds each connection role the function declares. The function runs with `ctx.event.type` `"task-trigger"` and receives the task as `ctx.task` (see functions.md).

**Placeholders**: `{{path}}` in `AddComment.body`, `SendEmail.body`, every string of `CreateTask.task`, and `RunFunction.input` resolves against the task after the trigger's own changes. Paths: `id`, `key`, `number`, `type_id`, `title`, `status_id`, `status_category`, `priority`, `assignee`, `requester_email`, `created_via`, `source`, `tags`, `labels`, and `custom.<field key>`. Use `{{source}}` for the task's source (`{{channel}}` no longer exists). An unknown path resolves to an empty string. A string that is exactly one placeholder keeps the value's type (numbers stay numbers, lists stay lists); inside longer text, lists join with `", "`. Values are inserted once and never re-scanned, so a title containing `{{...}}` stays literal. `SetField` values do not take placeholders.

## Permissions
- Every tool needs the **Tasks** product and the Tasks-View permission (`tasks.view`).
- Creating or changing tasks, types, fields, labels and triggers needs Tasks-Manage (`tasks.manage`). The trigger tools need it for reads too.
- Tasks-Admin (`tasks.admin`) also sees every private task type and its tasks. An API key needs it to create a private type.
- A private type, or a task of one, that the caller cannot see returns `404`.

## Migrated tickets
Workspaces that used tickets have them as tasks now:
- Each ticket template became a task type. Its key is derived from the template key and may be shortened (for example `support` became `SUPPO`). Template fields joined the field library; a field key that clashed with another template's field carries a suffix such as `_2`.
- A ticket's `tkt_...` id and its old ticket number still open its task in `ref`. The task's `legacy_ticket_id` shows the `tkt_...` id.
- Ticket triggers became task triggers on the migrated types, with `{{channel}}` rewritten to `{{source}}`.
- Migrated tasks keep their original timestamps, so a new `TIME` trigger also matches older tasks that meet its conditions.

## Common mistakes
- Sending only the changed keys to a full-replace tool (`fertigai_task_types_update`, `fertigai_task_fields_update`, `fertigai_task_labels_update`, `fertigai_task_triggers_update`): missing keys are cleared. Get, change, send it all.
- Writing field values keyed by field id: `fields` is keyed by field **key**.
- `label_ids` or `tags` on `fertigai_tasks_update`: use `labels_add`/`labels_remove` and `tags_add`/`tags_remove`.
- A `TYPE` condition with a type key: it takes a `tty_...` id.
- A time condition in an event trigger, in `any`, or a `TIME` trigger without exactly one `GREATER` clock: rejected with `422`.
- Filtering with `assignee: "ME"` on an API key: it matches nothing.
