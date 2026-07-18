---
name: fertigai-agents
description: Use when creating, configuring, renaming, or deleting fertig.ai agents through the workspace MCP, or when editing an agent's model, voice, language, prompt, or other behavior settings.
---

# Managing agents (fertigai_agents_*, fertigai_agent_branch_*)

## Overview
An **agent** is a container: it has a name, status, and display colors, but none of its behavior configuration. All behavior (model, voice, languages, system prompt, and more) lives on a **branch**. Every agent has one active branch (the first is named "Main"). So "configure an agent" always means "update a branch's config".

Shared conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill.

## Tools
| Tool | Args | Notes |
|---|---|---|
| `fertigai_agents_list` | `search?`, `status?`, `sort?`, `order?`, `cursor?`, `page_size?` | list agents |
| `fertigai_agents_get` | `id` | returns the agent plus its full `config`, `branches`, and `active_branch_id` |
| `fertigai_agents_create` | `name`, `config?` | creates the agent and a "Main" branch; omit `config` for a blank agent |
| `fertigai_agents_rename` | `id`, `name` | renames the container only |
| `fertigai_agents_delete` | `id` | deletes the agent |
| `fertigai_agent_branch_get` | `agent_id`, `branch_id` | one branch, including its config |
| `fertigai_agent_branch_create` | `agent_id`, `name`, `from_branch_id` | new branch copied from an existing one |
| `fertigai_agent_branch_update_config` | `agent_id`, `branch_id`, `config` | the main "configure the agent" tool |
| `fertigai_agent_set_active_branch` | `agent_id`, `branch_id` | make a branch the live one |

There is no "list branches" tool; branches come embedded in `fertigai_agents_get`. There is no `fertigai_agents_update`; config changes only through `fertigai_agent_branch_update_config`.

## The config object
`config` (an AgentConfig) top-level fields: `name`, `model`, `voice_id`, `languages` (array, e.g. `["de"]`), `system_prompt`, `welcome_message`, `eagerness`, `spelling_patience`, `dynamic_variables` (array), plus optional nested groups `workflow`, `speech`, `call`, `post_call_analysis`, `security`, `guardrails`, `gdpr`.

Here `config.name` is the BRANCH name (the first branch is `"Main"`), not the agent's name. On `fertigai_agents_create`, `config` is optional: pass a full AgentConfig to seed the Main branch, or omit it to get a blank agent with backend defaults. If you only need a few settings, the simplest reliable path is to create with just `name`, then configure with the get + update pattern below. `fertigai_agents_create` returns the agent, but always `fertigai_agents_get` it afterward to read the `active_branch_id` you will configure.

## Editing an agent's behavior (the reliable pattern)
1. `fertigai_agents_get { id }` and take the returned `config` and `active_branch_id`.
2. Modify the fields you want in that config object (for example change `system_prompt` or `model`).
3. `fertigai_agent_branch_update_config { agent_id, branch_id: active_branch_id, config }` with the whole edited object.

Start from what `get` returns so required nested groups are present; do not hand-build a config from scratch.

## Example: create and configure an agent
```
fertigai_agents_create { "name": "Support Bot" }
```
Read the new agent's active branch and starting config:
```
fertigai_agents_get { "id": "agt_..." }
```
Edit the returned `config` (set `model`, `voice_id`, `languages`, `system_prompt`, and so on) and send it back:
```
fertigai_agent_branch_update_config { "agent_id": "agt_...", "branch_id": "<active_branch_id>", "config": <edited config> }
```

## Common mistakes
- Trying to change config via `fertigai_agents_rename` or a nonexistent `fertigai_agents_update`: config only changes through `fertigai_agent_branch_update_config`.
- Sending a partial `config`: the update validates the whole object. Start from `get`.
- Confusing `agent_id` and `branch_id`: both are separate arguments on the branch tools.
- A `403` even though the key has Agents-Edit: the workspace also needs the agents product enabled for these tools.

Writes need Agents-Edit and the agents product.
