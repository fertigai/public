---
name: fertigai-actions
description: Use when creating, editing, testing, or deleting fertig.ai actions (automations that run in response to a conversation, for example post-call steps) through the workspace MCP.
---

# Managing actions (fertigai_actions_*)

## Overview
An **action** is an automation that runs in response to a conversation (for example a post-call step). It has a name, a description, a `style`, a parameter schema, and either a `script_source` (code) or a `pipeline` (structured steps).

Shared conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill.

## Tools
| Tool | Args |
|---|---|
| `fertigai_actions_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_actions_get` | `id` |
| `fertigai_actions_create` | `name`, `description`, `style`, `script_source?`, `pipeline?`, `parameter_schema?` |
| `fertigai_actions_update` | `id`, `name`, `description`, `style`, `script_source?`, `pipeline?`, `parameter_schema?` |
| `fertigai_actions_delete` | `id` |
| `fertigai_actions_test` | `script`, `parameter_schema?`, `parameter_values?`, `ctx_params?`, one of `conversation_public_id` or `script_ctx`, `action_id?`, `connection_public_id?` |

## Fields
- `style`: integer. `2` = Code (provide the action logic in `script_source`), `1` = Visual (uses `pipeline`). For a code action use `style: 2` with a `script_source`.
- Do NOT send `preset_key` on create; it is server-managed and rejected.
- `style` is immutable across updates (a different value is rejected).

## Test before saving
`fertigai_actions_test` dry-runs a draft. It needs a context source: pass either a real `conversation_public_id` to run against a past conversation, or a `script_ctx` object with mock context. Provide exactly one of them, not both.

## Common mistakes
- Sending `preset_key` on create.
- Omitting the context source on `test`, or providing both (`conversation_public_id` XOR `script_ctx`).
- `test` returning a service-unavailable error: the action worker must be deployed for test to run.

Writes need Integrations-Manage.
