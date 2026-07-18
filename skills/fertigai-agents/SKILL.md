---
name: fertigai-agents
description: Use when creating, configuring, renaming, or deleting fertig.ai agents through the workspace MCP, or when editing an agent's model, voice, prompt, conversation workflow, guardrails, or other branch settings.
---

# Managing agents (fertigai_agents_*, fertigai_agent_branch_*)

## Overview
An **agent** is a container: a name, status, and display colors, with none of its behavior. All behavior lives on a **branch**: the model, voice, languages, system prompt, welcome message, the conversation **workflow** (a node/edge graph), and setting groups (speech, call, post-call analysis, security, guardrails, GDPR). Every agent has exactly one active branch (the first is named "Main"); extra branches let you version or A/B a configuration. So "configure an agent" always means "update a branch's config".

Shared conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill. The full workflow node and edge model, and the exhaustive setting-group fields, are in `workflow-reference.md` next to this skill.

## Tools
| Tool | Args | Notes |
|---|---|---|
| `fertigai_agents_list` | `search?`, `status?`, `sort?`, `order?`, `cursor?`, `page_size?` | list agents |
| `fertigai_agents_get` | `id` | returns the agent plus its full `config`, `branches`, and `active_branch_id` |
| `fertigai_agents_create` | `name`, `config?` | creates the agent and a "Main" branch; omit `config` for a blank agent |
| `fertigai_agents_rename` | `id`, `name` | renames the container only |
| `fertigai_agents_delete` | `id` | deletes the agent |
| `fertigai_agent_branch_get` | `agent_id`, `branch_id` | one branch, including its config |
| `fertigai_agent_branch_create` | `agent_id`, `name`, `from_branch_id` | fork a branch from an existing one |
| `fertigai_agent_branch_update_config` | `agent_id`, `branch_id`, `config` | the main "configure the agent" tool |
| `fertigai_agent_set_active_branch` | `agent_id`, `branch_id` | make a branch the live one |

There is no "list branches" tool (branches are embedded in `fertigai_agents_get`) and no `fertigai_agents_update` (config only changes through `fertigai_agent_branch_update_config`).

## The config object (AgentConfig)
The `config` passed to `fertigai_agent_branch_update_config` (and optionally to `fertigai_agents_create`) is an **AgentConfig**. `config.name` is the BRANCH name (use `"Main"` for the first branch), not the agent's name.

| Field | Type | Notes |
|---|---|---|
| `model` | string | prefixed LLM id, e.g. `"mistral/mistral-large-latest"` |
| `voice_id` | string | voice id for this branch |
| `languages` | string[] | BCP-47 codes, e.g. `["en","de"]` |
| `system_prompt` | string | system instruction; supports `{{ variable }}` tokens |
| `welcome_message` | string | first spoken line |
| `eagerness` | string | `eager` \| `normal` \| `patient` |
| `spelling_patience` | string | `auto` \| `off` |
| `dynamic_variables` | array | declared variables (see below) |
| `workflow` | object | the conversation node/edge graph (see `workflow-reference.md`) |
| `speech` | object | voice tuning: `speed`, `stability`, `similarity`, `tts_model`, `ambient_sound` |
| `call` | object | `max_duration` (sec), `silence_timeout` (sec) |
| `post_call_analysis` | object | classification categories, extracted variables, summary, `actions_enabled` |
| `security` | object | `rate_limit_per_minute` |
| `guardrails` | object | `focus`, `manipulation`, `content` (+ 7 content categories), `custom` rules |
| `gdpr` | object | `consent_required`, `anonymize_data`, `data_retention_days`, `recording_enabled` |

Nested groups are optional; omit one to keep backend defaults. The exact sub-fields, defaults, and enums for every group are in `workflow-reference.md`.

## Dynamic variables
`dynamic_variables` is an array of `{ name, default_value?, source? }`. `name` must match `^[a-zA-Z_][a-zA-Z0-9_]*$`. Reference a variable anywhere in config strings, prompts, and welcome messages with `{{ name }}`. Workflow expression edges and Update Context nodes may only reference a **declared** variable, or the save is rejected. Function outputs can also be written back into a variable.

## The reliable edit pattern (important)
`fertigai_agent_branch_update_config` REPLACES the whole config and runs strict validation, especially on the workflow graph (see `workflow-reference.md` for all rules: exactly one Start Agent and one First Message node, the fixed First Message to Start Agent edge, no dangling or self edges, all nodes reachable, terminal nodes with no outgoing edges, function nodes with exactly one attached function, and every referenced variable declared). Hand-building a config or workflow from scratch will almost certainly fail validation. Always:
1. `fertigai_agents_get { id }` (or `fertigai_agent_branch_get`) to read the current `config` and `active_branch_id`.
2. Modify only the fields you need in that object.
3. Send the whole object back via `fertigai_agent_branch_update_config`.

## Branches
- Agent create seeds one active branch named "Main".
- `fertigai_agent_branch_create { agent_id, name, from_branch_id }` forks a branch: it copies the entire config (model, prompt, the whole workflow graph, variables, and every setting group) plus classification categories, extracted variables, and attachments (post-call actions, knowledge bases, function attachments). The new branch is created inactive.
- `fertigai_agent_set_active_branch` makes a branch live. You cannot delete the active branch.
- Editing config never changes the agent's name (use `fertigai_agents_rename`).

## Example: create then configure
```
fertigai_agents_create { "name": "Support Bot" }
```
Read the new agent's active branch and starting config:
```
fertigai_agents_get { "id": "agt_..." }
```
Edit the returned `config` (set `model`, `voice_id`, `languages`, `system_prompt`, tweak `workflow` nodes, and so on) and send it back:
```
fertigai_agent_branch_update_config { "agent_id": "agt_...", "branch_id": "<active_branch_id>", "config": <edited config> }
```

## Common mistakes
- Trying to change config via `fertigai_agents_rename` or a nonexistent `fertigai_agents_update`: config only changes through `fertigai_agent_branch_update_config`.
- Building a `config` or `workflow` from scratch and hitting a `422` validation error: start from `get` and edit; the workflow has many structural rules (see `workflow-reference.md`).
- Referencing an undeclared variable in an expression edge or Update Context node: declare it in `dynamic_variables` first.
- Confusing `agent_id` and `branch_id`: both are separate arguments on the branch tools.
- A `403` even though the key has Agents-Edit: the workspace also needs the agents product enabled for these tools.

Writes need Agents-Edit and the agents product.
