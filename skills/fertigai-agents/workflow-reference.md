# Agent workflow and setting-group reference

Companion to the `fertigai-agents` skill. This is the detailed shape of `config.workflow` (the conversation graph) and the nested setting groups. The whole `config` is validated as one object on the `config` section of `fertigai_agent_branch_configure`, so the reliable way to author these is to `fertigai_agents_get` an agent, edit the returned `config`, and send it back.

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

### The 8 node types

Every node's `data` includes `type` (equal to the node-level type) plus the node-specific fields below.

| type | purpose | key `data` fields | rules |
|---|---|---|---|
| `start_agent` | conversation root; its `config.system_prompt` is the base prompt for the whole workflow; branch-level functions and knowledge bases attach here | `conversationGoal` (the overall goal), `label?` | exactly one; cannot be deleted; a routing node, so its outgoing edges must be conditioned (no `unconditional`) |
| `first_message` | speaks the welcome message | `label?` | exactly one; cannot be deleted; has NO incoming edge; exactly one outgoing edge to `start_agent`, `unconditional` |
| `subagent` | a nested agent step with its own goal; inherits the base system prompt unless overridden | `conversationGoal` (this step's goal), `overridePrompt` (default `false`: replace the base system prompt only when `true`), `overrideFunctions?`, `overrideKnowledgeBases?`, `voiceId`, `model`, `eagerness`, `spellingPatience` | routing node; outgoing edges must be conditioned |
| `say` | speaks a line, fixed or LLM-generated | `mode` (`"literal"`\|`"prompt"`), `text` (for literal), `prompt` (for prompt), `voiceId?` | at most one `unconditional` outgoing edge |
| `function` | runs one attached function | `label?` (the function attachment is bound to the node, not in `data`) | requires exactly one attached function, supplied via the `attachments` section of the same `fertigai_agent_branch_configure` call (see fertigai-attachments); can branch on a `result` edge (success/failure) |
| `update_context` | sets dynamic variables | `updates: [{ variableName, value }]` | every `variableName` must be a declared dynamic variable |
| `end_call` | ends the call | `label?` | terminal: no outgoing edges |
| `transfer` | phone transfer | `routes: [{ number, condition, transferType, timeoutSecs }]` where `transferType` is `"COLD"` or `"ATTENDED"` | terminal when all routes are `COLD`; an `ATTENDED` route returns control, so outgoing edges are then allowed but must be `unconditional`; at least one route needs a non-empty `number` |

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
- `{ "kind": "result", "successful": true|false, "label?": "..." }` - only from a `function` node (or a COLD/mixed `transfer`); branches on tool success or failure.

### Validation rules (a save is rejected if any is broken)

- Exactly one `start_agent` and exactly one `first_message`.
- `first_message` has no incoming edge and exactly one `unconditional` outgoing edge to `start_agent`.
- No self-loops; no edges referencing a missing node; every node reachable.
- Terminal nodes (`end_call`; an all-`COLD` `transfer`) have no outgoing edges.
- A `function` node has exactly one attached function, sent in the `attachments` section of the same configure call (a `function` node with no matching attachment is rejected; see fertigai-attachments).
- `start_agent` and `subagent` outgoing edges are conditioned (no `unconditional`).
- Every variable referenced by an expression edge or an `update_context` node is declared in `config.dynamic_variables`.

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
{ "consent_required": false, "anonymize_data": false, "data_retention_days": 90, "recording_enabled": false }
```
