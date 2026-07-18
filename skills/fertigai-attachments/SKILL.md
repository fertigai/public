---
name: fertigai-attachments
description: Use when attaching functions, built-in system tools, or knowledge-base items to a fertig.ai agent branch (or to a specific workflow node) through the workspace MCP, including wiring a function's parameter sources and its response-to-variable assignments.
---

# Managing branch attachments (fertigai_agent_attachments_get, fertigai_agent_branch_configure)

## Overview
An agent branch calls tools and reads knowledge during a conversation through **attachments**: a function (user-defined or a built-in system tool) or a knowledge-base item, bound either to the whole branch or to one workflow node. Attachments are read and written declaratively, alongside the branch's config, through `fertigai_agent_branch_configure`. Shared MCP conventions (ids, pagination, errors, permissions) are in the fertigai-mcp skill; branch config itself is in the fertigai-agents skill.

## Tools
| Tool | Args | Notes |
|---|---|---|
| `fertigai_agent_attachments_get` | `agent_id`, `branch_id` | current attachment state for the branch |
| `fertigai_agent_branch_configure` | `agent_id`, `branch_id`, `config?`, `attachments?` | writes config and/or attachments in one call; `config` is documented in fertigai-agents |
| `fertigai_system_tools_list` | none | the built-in tools available as attachments |
| `fertigai_knowledge_base_list` | `parent_id?`, `search?`, `cursor?`, `page_size?` | browse knowledge-base items to find their `kbi_...` ids |

## The read-modify-write loop
The `attachments` object is declarative and replaces its whole domain: whatever you send for `functions` becomes the entire function-attachment state, and whatever you send for `knowledge_bases` becomes the entire knowledge-base state. There is no partial merge, so anything you leave out is removed.

To edit a branch, ALWAYS read both its config and its attachments first, then send them back together in one call:
1. `fertigai_agent_branch_get { agent_id, branch_id }` to read the current `config` (see fertigai-agents).
2. `fertigai_agent_attachments_get { agent_id, branch_id }` to read the current `attachments`.
3. Edit whichever you are changing (add, remove, or change entries).
4. Send both back in one call: `fertigai_agent_branch_configure { agent_id, branch_id, config, attachments }`.

Reading and re-sending both is the safe default. Because each section is a full replace, sending an `attachments` section you did not read first silently detaches everything you left out (and sending a `config` you did not read drops variables and workflow edits). The two sections are technically independent, so you may send just one when you are certain the other is untouched, but when in doubt read and send both.

## The attachments shape
```json
{
  "functions": {
    "branch": [ /* FunctionAttachmentInput, attached at branch level */ ],
    "nodes":  { "<workflowNodeId>": { "items": [ /* FunctionAttachmentInput for that node */ ] } }
  },
  "knowledge_bases": {
    "branch": [ "kbi_...", "kbi_..." ],
    "nodes":  { "<workflowNodeId>": { "items": [ "kbi_..." ] } }
  }
}
```
- `functions.branch` / `knowledge_bases.branch`: available across the whole agent (conventionally bound at the `start_agent` node).
- `functions.nodes["<id>"]` / `knowledge_bases.nodes["<id>"]`: scoped to one workflow node. `<id>` is the `id` of a `function` or `subagent` node in the branch's `config.workflow` (see fertigai-agents and its `workflow-reference.md`).

## FunctionAttachmentInput fields
| Field | Type | Notes |
|---|---|---|
| `function_id` | string | the function's public id (`fn_...`), for a user-defined function |
| `system_tool_type` | string | one of the built-in tool slugs from `fertigai_system_tools_list`: currently `end_call`, `language_detection`, `transfer_to_number`. Mutually exclusive with `function_id`; exactly one of the two must be set |
| `parameter_values` | object | keyed by the function's parameter key, see below. Empty for system tools other than `transfer_to_number` |
| `connection_public_id` | string | required only when the underlying function needs a connection |
| `assignments` | array | response-to-variable assignments, see below |
| `transfer_routes` | array | only for the `transfer_to_number` system tool, see below |

## parameter_values: the four sources
Each key in `parameter_values` maps to a tagged object with exactly one source:

```json
{
  "name":  { "source": { "Llm":             { "description_override": "the customer's name" } } },
  "count": { "source": { "Static":          { "value": 5 } } },
  "key":   { "source": { "Secret":          { "secret_public_id": "sec_..." } } },
  "tier":  { "source": { "DynamicVariable":  { "name": "customer_tier" } } }
}
```
- `Llm`: the model decides the value at call time. `description_override` optionally replaces the parameter's schema description shown to the model.
- `Static`: a literal value sent on every call. `value` must match the parameter's declared type (string, number, boolean, or a nested object/array).
- `Secret`: resolved from a workspace secret (`sec_...`) at call time; the plaintext value never appears in the config.
- `DynamicVariable`: resolved from a branch dynamic variable's current value at call time. `name` must be a variable already declared in `config.dynamic_variables`.

## assignments: writing a result into a variable
```json
{ "dynamic_variable": "order_id", "value_path": "data.id", "sanitize": false, "preserve_native_type": false }
```
Each assignment extracts a value from the function's JSON result using a dot-notation `value_path` (for example `"data.0.id"`) and writes it into the named dynamic variable, so later nodes and prompts can reference it as `{{ order_id }}`. `sanitize: true` keeps the extracted value out of what the model and the caller see in the transcript while still assigning it; `preserve_native_type` keeps the original JSON type instead of turning it into a string.

`fertigai_agent_branch_configure` auto-creates the output dynamic variable for every `dynamic_variable` name referenced in an assignment. Do not also declare these yourself in `config.dynamic_variables`.

- Each `dynamic_variable` must be non-empty, unique within that attachment's assignments, and must not start with `system__`; `value_path` must be non-empty. Otherwise the call is rejected.
- `assignments` apply only to user-defined functions. A system tool (`system_tool_type`) ignores them, so do not put assignments on `end_call`, `language_detection`, or `transfer_to_number`.

## transfer_routes: transfer_to_number only
```json
{ "number": "+15551234567", "condition": "customer asks for a human", "transfer_type": "ATTENDED", "timeout_secs": 15 }
```
`transfer_type` is `"COLD"` (unattended, connects directly) or `"ATTENDED"` (rings the destination first before connecting); these are the only two accepted values. `timeout_secs` is the maximum ring time for an `ATTENDED` transfer before the call returns to the agent; it defaults to `15` seconds when omitted (`0` is a literal zero, it does not select a default, so use `15` for the standard behavior). This field only applies to a `transfer_to_number` system-tool entry; other attachments leave it empty.

## Function nodes need a matching attachment
A workflow `function` node runs exactly one attached function. If a branch's `config.workflow` has a `function` node, the same `fertigai_agent_branch_configure` call MUST include a matching entry under `attachments.functions.nodes["<that node's id>"]`, or the whole call is rejected. Set the workflow and its function-node attachments together in one call.

## Knowledge bases
`fertigai_knowledge_base_list` browses the workspace's knowledge-base tree (folders and files) to find item ids: pass a folder's id as `parent_id` to look inside it, or `search` to filter by name. Each item's `id` (`kbi_...`) is what goes in `knowledge_bases.branch` or `knowledge_bases.nodes["<id>"].items`.

## Example: attach a function to a workflow node
Read current attachments:
```
fertigai_agent_attachments_get { "agent_id": "agt_...", "branch_id": "br_..." }
```
Edit the returned object, add an entry under `functions.nodes["fn1"].items`, and send the whole thing back:
```
fertigai_agent_branch_configure {
  "agent_id": "agt_...", "branch_id": "br_...",
  "attachments": {
    "functions": {
      "branch": [],
      "nodes": { "fn1": { "items": [ {
        "function_id": "fn_...",
        "parameter_values": {},
        "assignments": [ { "dynamic_variable": "order_id", "value_path": "data.id", "sanitize": false, "preserve_native_type": false } ]
      } ] } }
    },
    "knowledge_bases": { "branch": [], "nodes": {} }
  }
}
```

## Common mistakes
- Sending a partial `attachments` object and expecting a merge: each section (`functions`, `knowledge_bases`) fully replaces its current state. Read first with `fertigai_agent_attachments_get`, edit, send everything back.
- Leaving a workflow `function` node without a matching node-scoped attachment in the same call: the call is rejected.
- Setting both `function_id` and `system_tool_type` on the same entry, or setting neither.
- Manually declaring an assignment's `dynamic_variable` in `config.dynamic_variables`: it is created automatically.
- Omitting `connection_public_id` for a function that requires a connection.

Writes need Integrations-Manage for the `attachments` section and Agents-Edit for the `config` section (both, when a call sends both).
