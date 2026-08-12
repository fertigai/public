---
name: fertigai-agents
description: Use when creating, configuring, renaming, or deleting fertig.ai agents through the workspace MCP, or when editing an agent's model, voice, prompt, conversation workflow, guardrails, or other branch settings.
---

# Managing agents (fertigai_agents_*, fertigai_agent_branch_*)

## Overview
An **agent** is a container: a name, status, and display colors, with none of its behavior. All behavior lives on a **branch**: the model, voice, languages, system prompt, welcome message, the conversation **workflow** (a node/edge graph), setting groups (speech, call, post-call analysis, security, guardrails, GDPR), and its function and knowledge-base attachments. Every agent has exactly one active branch (the first is named "Main"); extra branches let you version or A/B a configuration. So "configure an agent" always means "configure a branch".

Shared conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill. Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see fertigai-mcp / `fertigai_whoami`). The full workflow node and edge model, and the exhaustive setting-group fields, are in `workflow-reference.md` next to this skill (or fetch `https://raw.githubusercontent.com/fertigai/public/main/skills/fertigai-agents/workflow-reference.md`). Attaching functions, system tools, and knowledge-base items to a branch or a workflow node is in the **fertigai-attachments** skill.

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
| `fertigai_agent_branch_configure` | `agent_id`, `branch_id`, `config?`, `attachments?` | the main "configure the agent" tool: sets a branch's config and/or its function and knowledge-base attachments in one call. Each section is optional; send only what you're changing |
| `fertigai_agent_set_active_branch` | `agent_id`, `branch_id` | make a branch the live one |

There is no "list branches" tool (branches are embedded in `fertigai_agents_get`) and no `fertigai_agents_update` (config only changes through `fertigai_agent_branch_configure`). The `attachments` section of that same tool is documented in the fertigai-attachments skill.

## The config object (AgentConfig)
The `config` section of `fertigai_agent_branch_configure` (and optionally `fertigai_agents_create`) is an **AgentConfig**. A returned config carries `config.name`, which is the BRANCH name (not the agent's name); it is a read-only echo, ignored on write.

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
`dynamic_variables` is an array of `{ name, default_value?, source? }`. `name` must match `^[a-zA-Z_][a-zA-Z0-9_]*$`; names starting with `system__` are reserved. Reference a variable anywhere in config strings, prompts, and welcome messages with `{{ name }}`. Workflow expression edges and Update Context nodes may only reference a **declared** variable, or the save is rejected. Function outputs can also be written back into a variable (see fertigai-attachments).

At call time the platform always supplies `caller_id` and `called_id` (the caller's and the called number) as conversation variables, even when undeclared; to reference one in an expression edge or Update Context node, declare it like any other variable.

## The reliable edit pattern (important)
Both the `config` section and each `attachments` section of `fertigai_agent_branch_configure` REPLACE their whole domain and run strict validation (the config especially on the workflow graph: exactly one Start Agent and one First Message node, the fixed First Message to Start Agent edge, no dangling or self edges, an incoming edge on every node except First Message, terminal nodes with no outgoing edges, function nodes with exactly one attached function, and every referenced variable declared, see `workflow-reference.md`). Because each section is a full replace, a section you send without reading it first is overwritten with only what you put in it, so hand-building config or attachments from scratch will wipe existing state and almost certainly fail validation.

To edit a branch, ALWAYS read both its config and its attachments first, then send them back together:
1. `fertigai_agents_get { id }` (or `fertigai_agent_branch_get { agent_id, branch_id }`) to read the current `config` and `active_branch_id`.
2. `fertigai_agent_attachments_get { agent_id, branch_id }` to read the current `attachments` (functions and knowledge bases, see fertigai-attachments).
3. Modify only what you need in each object.
4. Send both back in one call: `fertigai_agent_branch_configure { agent_id, branch_id, config, attachments }`.

Reading and re-sending both is the safe default: it stops a config edit from silently dropping the branch's attachments (and an attachments edit from dropping config), and it is required whenever the workflow has a `function` node, since that node needs its attachment in the same call or the call is rejected. The two sections are technically independent, so you may send just one when you are certain the other is untouched, but when in doubt read and send both.

## Branches
- Agent create seeds one active branch named "Main".
- `fertigai_agent_branch_create { agent_id, name, from_branch_id }` forks a branch: it copies the entire config (model, prompt, the whole workflow graph, variables, and every setting group) plus classification categories, extracted variables, and attachments (post-call actions, knowledge bases, function attachments). The new branch is created inactive.
- `fertigai_agent_set_active_branch` makes a branch live. You cannot delete the active branch.
- Editing config never changes the agent's name (use `fertigai_agents_rename`).

## Example: create then configure
Create the agent (this seeds an active branch named "Main"):
```
fertigai_agents_create { "name": "Support Bot" }
```
Read the branch's current config (which carries `active_branch_id`) and its current attachments:
```
fertigai_agents_get { "id": "agt_..." }
fertigai_agent_attachments_get { "agent_id": "agt_...", "branch_id": "<active_branch_id>" }
```
Edit the returned `config` (set `model`, `voice_id`, `languages`, `system_prompt`, tweak `workflow` nodes, and so on) and the returned `attachments` (attach functions or knowledge bases, see fertigai-attachments), then send both back in one call:
```
fertigai_agent_branch_configure {
  "agent_id": "agt_...", "branch_id": "<active_branch_id>",
  "config": <edited config>, "attachments": <edited attachments>
}
```

## Common mistakes
- Trying to change config via `fertigai_agents_rename` or a nonexistent `fertigai_agents_update`: config only changes through `fertigai_agent_branch_configure`.
- Building a `config` or `workflow` from scratch and hitting a `422` validation error: start from `get` and edit; the workflow has many structural rules (see `workflow-reference.md`).
- Referencing an undeclared variable in an expression edge or Update Context node: declare it in `dynamic_variables` first.
- Adding a `function` node to the workflow without also sending its attachment in the same `fertigai_agent_branch_configure` call: the call is rejected (see fertigai-attachments).
- Confusing `agent_id` and `branch_id`: both are separate arguments on the branch tools.
- A `403` even though the key has Agents-Edit: the workspace also needs the agents product enabled for these tools.

Writes to `config` need Agents-Edit and the agents product; writes to `attachments` need Integrations-Manage (see fertigai-attachments).
