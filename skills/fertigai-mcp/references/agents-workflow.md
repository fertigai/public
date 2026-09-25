# Agent workflow and setting-group reference

Companion to `agents.md`. This is the detailed shape of `config.workflow` (the conversation graph) and the nested setting groups. The whole `config` is validated as one object on the `config` section of `fertigai_agent_branch_configure`, so the reliable way to author these is to `fertigai_agents_get` an agent, edit the returned `config`, and send it back.

## workflow

`config.workflow` is a directed graph:

```
"workflow": { "enabled": true, "version": 3, "nodes": [ ... ], "edges": [ ... ] }
```

- `enabled`: when `false`, the agent runs a flat single-prompt conversation and the graph is ignored.
- `version`: currently `3`. Older graphs are migrated up automatically on read and save, so a config you read back is already at the current version.

### Node

```
{ "id": "<stable id>", "type": "<node type>", "position": { "x": 0, "y": 0 },
  "data": { "type": "<node type>", ...node-specific fields } }
```

The node's kind appears in TWO places: the node-level `type` AND `data.type`. The editor reads the kind from `data.type`. The backend automatically mirrors `data.type` from the node-level `type` on every read and save, so a node that omits or mismatches `data.type` is repaired rather than rejected, but always set `data.type` equal to the node-level `type` so the editor renders it correctly. The rest of `data` is node-specific (see the table); mirror the shape from an existing branch's config rather than inventing fields.

### Node types

Every node's `data` includes `type` (equal to the node-level type) plus the node-specific fields below.

| type | purpose | key `data` fields | rules |
|---|---|---|---|
| `start_agent` | conversation root; its `config.system_prompt` is the base prompt for the whole workflow; branch-level functions and knowledge bases attach here | `conversationGoal` (the overall goal), `label?` | exactly one; cannot be deleted; a routing node, so its outgoing edges must be conditioned (no `unconditional`) |
| `first_message` | speaks the welcome message | `label?` | exactly one; cannot be deleted; has NO incoming edge; exactly one outgoing edge to `start_agent`, `unconditional` |
| `subagent` | a nested agent step with its own goal; inherits the base system prompt unless overridden | `conversationGoal` (this step's goal), `overridePrompt` (default `false`: replace the base system prompt only when `true`), `overrideFunctions?`, `overrideKnowledgeBases?`, `voiceId`, `model`, `eagerness`, `spellingPatience` | routing node; outgoing edges must be conditioned |
| `say` | speaks a line, fixed or LLM-generated | `mode` (`"literal"`\|`"prompt"`), `text` (for literal), `prompt` (for prompt), `voiceId?`; the selected mode's text/prompt must be non-empty | at most one `unconditional` outgoing edge |
| `function` | runs one attached function | `label?` (the function attachment is bound to the node, not in `data`) | requires exactly one attached function, supplied via the `attachments` section of the same `fertigai_agent_branch_configure` call (see attachments.md); can branch on a `result` edge (success/failure) |
| `update_context` | sets dynamic variables | `updates: [{ variableName, value }]` (see below) | every `variableName` must be a declared dynamic variable |
| `end_call` | ends the call | `label?` | terminal: no outgoing edges |
| `transfer` | phone transfer | `transferType` (exactly `"COLD"`\|`"ATTENDED"`, empty = COLD), `numberSource` (`"LLM"`\|`"DYNAMIC_VARIABLE"`\|`"LLM_PROMPT"`, empty = LLM), `dynamicVariable`, `numberPrompt`, `timeoutSecs`, `routes: [{ number, condition }]` | terminal when `transferType` is `COLD`; an `ATTENDED` transfer returns control, so outgoing edges are then allowed but must be `unconditional`. With `numberSource: "LLM"` at least one route needs a non-empty `number`; with `"DYNAMIC_VARIABLE"` no routes are needed but `dynamicVariable` must name the variable holding the number; with `"LLM_PROMPT"` no routes are used either and `numberPrompt` (1 to 2000 characters, counted after surrounding whitespace is trimmed) is what the model reads to determine the number. In both of those modes existing `routes` are kept and read back unchanged, so re-sending them is safe. `timeoutSecs` is the `ATTENDED` max ring time (omitted = 30) |
| `agent_transfer` | hands the caller to a branch of another agent in the same workspace | `targetAgentId` (`agt_…`), `targetBranchId` (`br_…`, a branch of that agent), `transferMessage` (spoken before the handover, at most 500 characters after trimming, empty = none), `playWelcomeMessage` (boolean, default `false`: the target agent opens with its own welcome message), `delayMs` (whole number 0 to 10000, default 0) | terminal: no outgoing edges. The target must be a branch of that agent in this workspace and not the branch being saved, otherwise the save is rejected with `422` naming the node. The target branch must have been saved (synced) at least once before this branch is saved, otherwise this branch's sync fails and names the target. Two new branches that hand over to each other cannot both be saved first: save one of them without the node, save the other, then add the node. |

An `update_context` entry's `value` is a tagged object: `{ "kind": "literal", "value": <literal>, "valueType": "string"|"number"|"boolean" }`, `{ "kind": "var", "name": "<declared variable>" }`, or `{ "kind": "llm", "prompt": "<instruction>", "valueType": ... }`.

(Per-route `transferType`/`timeoutSecs` on transfer routes are deprecated; the node-level fields above replace them. Old graphs still carrying them read back with defaults `COLD`/`30`.)

`numberPrompt` decides only WHICH number is dialled, not whether to transfer: that stays with the agent's prompt and the built-in transfer guidance. Keep it as narrow as you can, because the agent can dial any number the prompt allows and the call is carried on the workspace's own trunk. Numbers from every source must pass the same dial check (see the transfer section of attachments.md).

### System prompt and goals

Keep these three separate, they are easy to confuse:

- **Base system prompt = `config.system_prompt`.** This is the Start Agent's system prompt, and it is the BASE system prompt for the ENTIRE workflow. Every node, including every `subagent`, inherits it. Put the agent's shared persona, rules, and context here once, not repeated per node.
- **Conversation goal = `conversationGoal`.** The Start Agent and each `subagent` each have a `conversationGoal`: the Start Agent's is the overall goal, and each subagent's is that step's goal. A goal steers the agent toward an outcome on top of the base system prompt. It is a goal, NOT a system prompt, so do not paste the whole persona into it.
- **Subagent prompt override = `overridePrompt` (default `false`).** A subagent uses the base system prompt by default. Set `overridePrompt: true` ONLY when that one subagent should replace the base system prompt with its own instead of inheriting it. Leave it `false` unless you specifically need to overwrite the base for that step.

In short: write shared instructions once in `config.system_prompt`, give the Start Agent and every subagent a `conversationGoal`, and turn on a subagent's `overridePrompt` only when that step genuinely needs a different base prompt.

### Edge

```
{ "id": "...", "source": "<node id>", "target": "<node id>", "condition": { ... } }
```

`condition` is a tagged union:

- `{ "kind": "unconditional" }` - at most one per source node; NOT allowed from `start_agent` or `subagent`.
- `{ "kind": "llm", "condition": "<natural language>", "label?": "..." }` - an LLM decides whether to follow this edge.
- `{ "kind": "expression", "expression": { "kind": "compare", "left": { "name": "<var>" }, "op": "eq"|"neq"|"gt"|"lt"|"gte"|"lte", "right": <literal> }, "label?": "..." }` - compares a declared variable; `left.name` must be declared.
- `{ "kind": "result", "successful": true|false, "label?": "..." }` - only from a `function` node; branches on tool success or failure.

### Validation rules (a save is rejected if any is broken)

- An enabled workflow has at least one node; exactly one `start_agent` and exactly one `first_message`.
- `first_message` has no incoming edge and exactly one `unconditional` outgoing edge to `start_agent`.
- No self-loops; no edges referencing a missing node; every node except `first_message` has at least one incoming edge.
- Terminal nodes (`end_call`; `agent_transfer`; a `COLD` `transfer`) have no outgoing edges; an `ATTENDED` transfer's outgoing edges are `unconditional` only.
- A `function` node has exactly one attached function, sent in the `attachments` section of the same configure call (a `function` node with no matching attachment is rejected; see attachments.md).
- `start_agent` and `subagent` outgoing edges are conditioned (no `unconditional`).
- Every variable referenced by an expression edge or an `update_context` node is declared in `config.dynamic_variables`.
- A transfer can find a number: `numberSource: "LLM"` has at least one route with a non-empty `number`; `"DYNAMIC_VARIABLE"` a non-empty `dynamicVariable`; `"LLM_PROMPT"` a `numberPrompt` of 1 to 2000 characters after trimming. `numberSource` is case-sensitive, so a value outside those three exact spellings, `"llm_prompt"` included, is rejected as well.
- An `agent_transfer` node names both `targetAgentId` and `targetBranchId`, and that branch belongs to that agent in this workspace and is not the branch being saved.

## Setting groups (nested in config)

### speech
```
{ "speed": 1.0, "stability": 0.5, "similarity": 0.75, "tts_model": "balanced",
  "ambient_sound": { "enabled": false, "source_id": "office2", "volume": 0.1 } }
```
- `speed` 0.7-1.2 (1.0 = normal); `stability` 0-1 (higher = steadier); `similarity` 0-1.
- `tts_model`: `"flash"` | `"balanced"` | `"expressive"`.
- `ambient_sound.source_id`: `office1` | `office2` | `restaurant` | `city` | `typing` | `elevator1`..`elevator4`; `volume` 0-1.

### call
```
{ "max_duration": 600, "silence_timeout": 10 }
```
Seconds. `max_duration` auto-hangs up; `silence_timeout` re-prompts after silence.

### post_call_analysis
```
{ "classification": { "enabled": true, "categories": [ { "public_id", "key", "name", "options": [], "description" } ] },
  "extracted_variables": { "enabled": true, "variables": [ { "public_id", "key", "name", "type", "description" } ] },
  "summary": { "enabled": true, "prompt_enabled": false, "prompt": "" },
  "actions_enabled": true }
```
- `extracted_variables[].type`: `ANY` | `STRING` | `NUMBER` | `BOOLEAN` | `ENUM`.
- `actions_enabled` triggers configured post-call actions after the call.
- Each category and variable carries a `public_id`. Echo it back UNCHANGED on edit: the save reconciles by `public_id`, so a category or variable whose `public_id` is missing from the payload is deleted (and its stored classification history is dropped). Leave `public_id` empty only for a genuinely new category or variable.

### security
```
{ "rate_limit_per_minute": 60 }
```
Max inbound calls per minute before rejecting.

### guardrails
```
{ "focus": true, "manipulation": true,
  "content": true,
  "content_config": { "sexual": { "enabled": true, "threshold": 2 }, "violence": {...}, "harassment": {...},
                      "self_harm": {...}, "profanity": {...}, "religion_or_politics": {...}, "medical_and_legal_information": {...} },
  "custom": false, "custom_rules": [ { "enabled": true, "name": "", "prompt": "" } ] }
```
- `focus` refuses off-purpose topics; `manipulation` resists social engineering.
- `content` is the master toggle; each of the 7 `content_config` categories is `{ enabled, threshold }` where `threshold` is an integer: `1` (low), `2` (medium), or `3` (high) (`0` means unspecified).

### gdpr
```
{ "consent_required": false, "data_retention_days": 90,
  "recording_retention_days": null, "recording_enabled": true,
  "anonymization": { "enabled": false, "caller_number": true, "called_number": false, "entities": [] } }
```
Values shown are the defaults (an omitted `gdpr` block keeps them; a present block replaces the whole group, with absent fields falling back to these defaults). `data_retention_days` is capped at 36500. `recording_retention_days` controls how long call audio is kept: `null`/omitted follows `data_retention_days`, `0` keeps no audio, and any value is capped at `data_retention_days`.

#### anonymization

**An `anonymization` object you send is taken literally.** Sending `{ "enabled": true }` with no `entities` means NO entity is anonymized, only the number options apply. You have to list the entities you want anonymized. To change one option, read the config, edit the object, and send it back whole.

- `enabled` is the master switch. While it is `false` the other fields are kept but have no effect.
- `caller_number` (default `true`) removes the caller's number from the stored conversation; `called_number` (default `false`) removes the number that was called. These two govern the stored number fields only: a phone number spoken in the conversation or passed as a tool argument is covered by the `contact_number` entity.
- `entities` is an allow list of the entity paths anonymized in the transcript, tool-call arguments and results, the summary, and dynamic variables. An entity that is not listed is NOT anonymized. List only the entities you really need: anonymization accuracy decreases as more entities are selected. Unknown or duplicate paths are rejected with `422`.
- Anonymization runs after the call ends, and post-call analysis (summary, classification, extracted variables, actions) runs on the anonymized conversation. A conversation's `anonymized` flag means the branch's anonymization options were applied, so with every entity off, the transcript is stored as spoken.
- `anonymize_data` is deprecated. It is still accepted on write and still returned on read, mirroring `anonymization.enabled`, and `anonymization.enabled` wins when both are sent. It will be removed, so use `anonymization.enabled`.

The 46 valid entity paths, grouped by prefix:

| Prefix | Paths |
|---|---|
| (none) | `email_address`, `contact_number`, `dob`, `age`, `religious_belief`, `political_opinion`, `sexual_orientation`, `ethnicity_race`, `marital_status`, `occupation`, `physical_attribute`, `language`, `username`, `url`, `organization`, `date`, `date_interval` |
| `name.` | `name.name_given`, `name.name_family`, `name.name_other` |
| `financial_id.` | `financial_id.payment_card.payment_card_number`, `financial_id.payment_card.payment_card_expiration_date`, `financial_id.payment_card.payment_card_cvv`, `financial_id.bank_account.bank_account_number`, `financial_id.bank_account.bank_routing_number`, `financial_id.bank_account.swift_bic_code`, `financial_id.financial_id_other` |
| `location.` | `location.location_address`, `location.location_city`, `location.location_postal_code`, `location.location_coordinate`, `location.location_state`, `location.location_country`, `location.location_other` |
| `unique_id.` | `unique_id.government_issued_id`, `unique_id.account_number`, `unique_id.vehicle_id`, `unique_id.healthcare_number.medical_record_number`, `unique_id.healthcare_number.health_plan_beneficiary_number`, `unique_id.device_id`, `unique_id.unique_id_other` |
| `medical.` | `medical.medical_condition`, `medical.medication`, `medical.medical_procedure`, `medical.medical_measurement`, `medical.medical_other` |
