# Managing branch attachments (fertigai_agent_attachments_get, fertigai_agent_branch_configure)

## Overview
An agent branch calls tools and reads knowledge during a conversation through **attachments**: a function (user-defined or a built-in system tool) or a knowledge-base item, bound either to the whole branch or to one workflow node. Attachments are read and written declaratively, alongside the branch's config, through `fertigai_agent_branch_configure`. Shared MCP conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md); branch config itself is in agents.md. Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args | Notes |
|---|---|---|
| `fertigai_agent_attachments_get` | `agent_id`, `branch_id` | current attachment state for the branch |
| `fertigai_agent_branch_configure` | `agent_id`, `branch_id`, `config?`, `attachments?` | writes config and/or attachments in one call; `config` is documented in agents.md |
| `fertigai_system_tools_list` | none | the built-in tools available as attachments |
| `fertigai_knowledge_base_list` | `parent_id?`, `search?`, `cursor?`, `page_size?` | browse knowledge-base items to find their `kbi_...` ids |

## The read-modify-write loop
The `attachments` object is declarative and replaces its whole domain: whatever you send for `functions` becomes the entire function-attachment state, and whatever you send for `knowledge_bases` becomes the entire knowledge-base state. There is no partial merge, so anything you leave out is removed - and that includes a whole missing sub-section: an `attachments` object that carries only `knowledge_bases` detaches EVERY function (and vice versa). Always send both sub-sections.

To edit a branch, ALWAYS read both its config and its attachments first, then send them back together in one call:
1. `fertigai_agent_branch_get { agent_id, branch_id }` to read the current `config` (see agents.md).
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
- `functions.nodes["<id>"]` / `knowledge_bases.nodes["<id>"]`: scoped to one workflow node. `<id>` is the `id` of a `function`, `subagent`, `directory_choose` or `directory_transfer` node in the branch's `config.workflow` (see agents.md and its `agents-workflow.md`).

## FunctionAttachmentInput fields
| Field | Type | Notes |
|---|---|---|
| `function_id` | string | the function's public id (`fn_...`), for a user-defined function |
| `system_tool_type` | string | one of the built-in tool slugs from `fertigai_system_tools_list`: currently `end_call`, `language_detection`, `transfer_to_number`, `transfer_to_agent`. Exactly one of `function_id`, `system_tool_type`, `integration_key` must be set |
| `integration_key` | string | the catalogue key of an integration function (integrations.md) |
| `label` | string | integration attachments only: shown instead of the catalogue name, max 80 chars |
| `connection_bindings` | array | `[{ role, connection_public_id }]`, one per connection role the function or integration declares; required roles must be bound |
| `parameter_values` | object | keyed by the function's parameter key, see below. Empty for system tools other than `transfer_to_number` |
| `assignments` | array | response-to-variable assignments, see below |
| `transfer_type`, `number_source`, `transfer_dynamic_variable`, `transfer_prompt`, `transfer_timeout_secs`, `transfer_routes` | | only for the `transfer_to_number` system tool, see below |
| `agent_transfer_targets` | array | only for the `transfer_to_agent` system tool, see below |
| `tool_call_settings` | object | how the call behaves while the function runs (sound, timeout, interruptions, speech before the call), see below. Ignored for system tools |

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
- `Llm`: the model decides the value at call time. `description_override` is REQUIRED (non-empty): it is the description the model reads when choosing the value.
- `Static`: a literal value sent on every call. `value` must match the parameter's declared type (string, number, boolean, or a nested object/array); a required string parameter needs a non-empty value.
- `Secret`: resolved from a workspace secret (`sec_...`) at call time; the plaintext value never appears in the config. Required parameters need a non-empty `secret_public_id`.
- `DynamicVariable`: resolved from a branch dynamic variable's current value at call time. If `name` is not declared in `config.dynamic_variables` yet, it is created on the branch when the attachment is saved, with no default value. A variable without a default value is fine: it resolves empty until the conversation sets it. Names the platform supplies itself (`caller_id`, `called_id`, and names starting with `system__` or `fertig__`) are never created. A name that differs only by case from a declared variable (`City` beside `city`), or that is not a valid variable name (it must match `^[a-zA-Z_][a-zA-Z0-9_]*$` and have at most 64 characters), is never created. `fertigai_agent_branch_configure` and the branch attachments sync still save: they store the attachment as stale until its source uses the declared spelling or a valid name. Writing a single attachment's values, and a `usage_fixes` entry on a function update, answer `422` instead.

## assignments: writing a result into a variable
```json
{ "dynamic_variable": "order_id", "value_path": "data.id", "sanitize": false, "preserve_native_type": false }
```
Each assignment extracts a value from the function's JSON result using a dot-notation `value_path` (for example `"data.0.id"`) and writes it into the named dynamic variable, so later nodes and prompts can reference it as `{{ order_id }}`. `sanitize: true` keeps the extracted value out of what the model and the caller see in the transcript while still assigning it; `preserve_native_type` keeps the original JSON type instead of turning it into a string.

`fertigai_agent_branch_configure` auto-creates the output dynamic variable for every `dynamic_variable` name referenced in an assignment; you do not need to declare these in `config.dynamic_variables` (doing so anyway is harmless).

- Each `dynamic_variable` must match `^[a-zA-Z_][a-zA-Z0-9_]*$`, be unique within that attachment's assignments, and must not start with `system__` in any casing (`SYSTEM__x` and `System__x` are reserved too); `value_path` must be non-empty. Otherwise the call is rejected.
- A target becomes a declared variable, so no two targets anywhere in the call may differ only by case (the uniqueness check above is per attachment and exact, so such a pair passes it and fails on the merged list), and no target may differ only by case from a variable the branch already declares: an assignment targeting `City` beside `city` is a `422`.
- `assignments` apply only to user-defined functions. A system tool (`system_tool_type`) ignores them, so do not put assignments on `end_call`, `language_detection`, `transfer_to_number`, or `transfer_to_agent`; they are not applied at call time, but their targets still count for the naming rules above.

## Tool call settings
A function or integration entry can carry `tool_call_settings`, one object with five members. Every member is an integer, and `0` means the default:

| Member | Values |
|---|---|
| `sound` | sound played to the caller while the function runs: `0` none (default), `1` typing, `2` to `5` elevator music 1 to 4 |
| `sound_behavior` | when the sound plays: `0` or `1` auto, only after the agent spoke before calling the function (default); `2` always |
| `response_timeout_secs` | seconds the function may take before the agent stops waiting for it: `5` to `300`; `0` means `20` (default) |
| `interruption_mode` | `0` or `2` the caller cannot interrupt while the function runs (default); `1` the caller can interrupt; `3` the caller cannot interrupt while the function runs or during the agent's turn |
| `pre_tool_speech` | whether the agent speaks before calling the function: `0` or `1` auto, the model decides (default); `2` always; `3` never |

```json
{ "function_id": "fn_...",
  "parameter_values": {},
  "tool_call_settings": { "sound": 1, "sound_behavior": 2, "response_timeout_secs": 60, "interruption_mode": 1, "pre_tool_speech": 2 } }
```
- Send all five members when you send the object. Omit the whole object for the defaults.
- `fertigai_agent_attachments_get` returns the settings on every entry, so read-modify-write keeps them. An entry sent without `tool_call_settings` is saved with the defaults.
- An unknown value or a `response_timeout_secs` outside `5` to `300` is rejected with `400` naming the member, for example `tool_call_settings.response_timeout_secs: must be between 5 and 300 seconds`.

## Transfer settings: transfer_to_number only
A `transfer_to_number` entry carries its transfer settings at the attachment level:
```json
{ "system_tool_type": "transfer_to_number",
  "transfer_type": "COLD",
  "number_source": "LLM",
  "transfer_routes": [ { "number": "+15551234567", "condition": "customer asks for a human" } ] }
```
- `transfer_type`: exactly `"COLD"` (connects directly) or `"ATTENDED"` (rings the destination first; the call returns to the agent if unanswered), case-sensitive in the same way as `number_source`. Empty means `COLD`.
- `number_source`: where the number comes from. Empty means `LLM`. The three values are case-sensitive, so write them exactly as shown: `"llm_prompt"` is not `"LLM_PROMPT"`, and any other spelling or casing is rejected with `422`.
  - `"LLM"`: the model picks one of `transfer_routes`; at least one route needs a non-empty `number`.
  - `"DYNAMIC_VARIABLE"`: the number comes from a branch dynamic variable; `transfer_dynamic_variable` is then REQUIRED and no routes are needed. If the branch does not declare that variable yet, saving the attachment declares it with no default value, so give it a value: a `default_value` in `config.dynamic_variables`, or one set during the conversation. `caller_id`, `called_id`, and names starting with `system__` or `fertig__` are supplied by the platform and never declared. As for `DynamicVariable` parameter sources, a name that differs only by case from a declared variable, or that is not a valid variable name, is never declared for you: write the declared spelling exactly.
  - `"LLM_PROMPT"`: no routes are used; the model works the number out from `transfer_prompt`.

  In the two modes without routes, routes already on the entry are still stored and read back unchanged, so leaving the existing `transfer_routes` in place is safe and valid.
- `transfer_prompt`: the instruction the model reads to determine the number, 1 to 2000 characters, counted after surrounding whitespace is trimmed (so a whitespace-only prompt is rejected). REQUIRED for `"LLM_PROMPT"`; omit it or send `null` for the other two sources.
- `transfer_timeout_secs`: maximum `ATTENDED` ring time before the call returns to the agent; omitted reads as `30`.
- Per-route `transfer_type`/`timeout_secs` are deprecated; use the attachment-level fields.

Every write is checked: an unknown `number_source`, `"LLM"` without a route carrying a number, `"DYNAMIC_VARIABLE"` without a variable, and `"LLM_PROMPT"` without a prompt (or one longer than 2000 characters after trimming) are each rejected with `422`. The same rules apply to the `transfer` workflow node (see agents-workflow.md).

These fields only apply to a `transfer_to_number` system-tool entry; other attachments leave them empty.

### Prompt as the number source
```json
{ "system_tool_type": "transfer_to_number",
  "transfer_type": "ATTENDED",
  "number_source": "LLM_PROMPT",
  "transfer_prompt": "Dial the branch the caller names: Berlin 030 1234, Hamburg 040 5678. Nothing else." }
```
`transfer_prompt` decides only WHICH number is dialled. WHETHER to transfer is still governed by the agent's own prompt and the built-in transfer guidance, so do not put the trigger condition in the prompt.

Keep the prompt as narrow as you can. The agent can dial any number the prompt allows, and the call is carried on the workspace's own trunk.

### Transfer numbers
Whichever source supplies it, a number is cleaned before dialling: spaces (ordinary, tab, and the non-breaking and thin kinds), hyphens (ordinary and non-breaking), parentheses, dots and slashes are removed. What remains must be digits, `*`, `#` and an optional leading `+`, with at least one digit and at most 32 characters. Anything else and the transfer is refused. These checks run when the call is placed, not when the configuration is saved: a number that fails them saves without error, and the transfer fails during the call.

- National numbers (`030 1234`) and internal extensions (`23`) are fine. A number with a leading zero is dialled as written; a bare international number (7 to 15 digits, no leading zero) gets its `+`.
- The `(0)` notation is not understood, and it is not an error either: `+49 (0)30 1234` becomes `+490301234` and dials a number nobody answers. Write `+49 30 1234`.
- Letters are never allowed, and a percent-escaped character (`%23` for `#`) is refused rather than decoded.

## Transfer to agent: transfer_to_agent only
A `transfer_to_agent` entry hands the call to a branch of an agent in the same workspace when a condition applies. It works like the `agent_transfer` workflow node (see agents-workflow.md), for agents without a workflow:
```json
{ "system_tool_type": "transfer_to_agent",
  "agent_transfer_targets": [
    { "agent_id": "agt_...", "branch_id": "br_...",
      "condition": "The caller wants to book, move or cancel an appointment.",
      "transfer_message": "One moment, I am connecting you to our appointment desk.",
      "play_welcome_message": false, "delay_ms": 0 }
  ] }
```
- 1 to 20 targets. Each `branch_id` must be a branch of that `agent_id` in this workspace. Unlike the node, a target may also be the branch being configured itself.
- `condition` is required, must not be empty after trimming and is at most 1000 characters: the agent hands the call to the target whose condition applies.
- `transfer_message` is spoken before the handover (at most 500 characters after trimming, empty = none); `play_welcome_message` (default `false`) lets the target agent open with its own welcome message; `delay_ms` is a whole number from 0 to 10000 (default 0).
- A target that breaks one of these rules is rejected with `422` naming it by position (counted from 1), for example `transfer_to_agent target 2 field 'condition' must not be empty`. An empty list is rejected with `transfer_to_agent field 'agent_transfer_targets' needs at least one target`, and more than 20 with `transfer_to_agent accepts at most 20 targets`. Sending `agent_transfer_targets` on any other attachment is rejected with `422` too.
- Branch level only: put it in `functions.branch`. Under `functions.nodes` it is rejected with `422`; inside a workflow use the `agent_transfer` node instead.
- A target branch must have been saved (synced) at least once before this branch is saved, otherwise this branch's sync fails and names the target.
- Deleting a branch that a target points at, or that branch's agent, is refused with `409` (see agents.md).

These fields only apply to a `transfer_to_agent` entry; other attachments leave `agent_transfer_targets` empty.

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
- Sending a partial `attachments` object and expecting a merge: each section (`functions`, `knowledge_bases`) fully replaces its current state, and a missing sub-section counts as empty. Read first with `fertigai_agent_attachments_get`, edit, send everything back.
- An `Llm` parameter source without a `description_override`: rejected; the description is required.
- An assignment target that differs only by case from another target in the same call or from a variable the branch declares, or one prefixed `system__` in any casing: each is a `422`, so match the declared spelling exactly.
- Leaving a workflow `function` node without a matching node-scoped attachment in the same call: the call is rejected.
- Setting more or fewer than one of `function_id`, `system_tool_type`, `integration_key`.
- Leaving a required connection role unbound, or sending a top-level `connection_public_id`: bind every role through `connection_bindings`.
- A `number_source` the entry cannot satisfy: `"LLM"` needs a route with a number, `"DYNAMIC_VARIABLE"` a `transfer_dynamic_variable`, `"LLM_PROMPT"` a `transfer_prompt` of 1 to 2000 characters after trimming. That, and any value outside the three (including a lowercase `"llm_prompt"`, since the values are case-sensitive), is a `422`.
- Writing a transfer number as `+49 (0)30 1234`: this is not a validation error and saves without complaint, then dials `+490301234`, which nobody answers. Write `+49 30 1234`. Letters are also caught only when the call is placed, never on save.

Writes need Integrations-Manage for the `attachments` section and Agents-Edit for the `config` section (both, when a call sends both).
