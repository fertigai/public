# Agent workflow and setting-group reference

Companion to the `fertigai-agents` skill. This is the detailed shape of `config.workflow` (the conversation graph) and the nested setting groups. The whole `config` is validated as one object on `fertigai_agent_branch_update_config`, so the reliable way to author these is to `fertigai_agents_get` an agent, edit the returned `config`, and send it back.

## workflow

`config.workflow` is a directed graph:

```
"workflow": { "enabled": true, "version": 2, "nodes": [ ... ], "edges": [ ... ] }
```

- `enabled`: when `false`, the agent runs a flat single-prompt conversation and the graph is ignored.
- `version`: currently `2`. Older graphs are migrated up automatically on save.

### Node

```
{ "id": "<stable id>", "type": "<node type>", "position": { "x": 0, "y": 0 },
  "data": { "type": "<node type>", ...node-specific fields } }
```

The node's kind appears in TWO places and they must match: the node-level `type` AND `data.type`. The editor reads the kind from `data.type`, so a node whose `data` omits `type` will not render, and the backend rejects it. Always set `data.type` equal to the node-level `type`. The rest of `data` is node-specific (see the table); mirror the shape from an existing branch's config rather than inventing fields.

### The 8 node types

Every node's `data` includes `type` (equal to the node-level type) plus the node-specific fields below.

| type | purpose | key `data` fields | rules |
|---|---|---|---|
| `start_agent` | conversation root and config; branch-level functions and knowledge bases attach here | `conversationGoal`, `label?` | exactly one; cannot be deleted; a routing node, so its outgoing edges must be conditioned (no `unconditional`) |
| `first_message` | speaks the welcome message | `label?` | exactly one; cannot be deleted; has NO incoming edge; exactly one outgoing edge to `start_agent`, `unconditional` |
| `subagent` | a nested agent step with its own goal and optional overrides | `conversationGoal`, `overridePrompt`, `overrideFunctions?`, `overrideKnowledgeBases?`, `voiceId`, `model`, `eagerness`, `spellingPatience` | routing node; outgoing edges must be conditioned |
| `say` | speaks a line, fixed or LLM-generated | `mode` (`"literal"`\|`"prompt"`), `text` (for literal), `prompt` (for prompt), `voiceId?` | at most one `unconditional` outgoing edge |
| `function` | runs one attached function | `label?` (the function attachment is bound to the node, not in `data`) | requires exactly one attached function; can branch on a `result` edge (success/failure) |
| `update_context` | sets dynamic variables | `updates: [{ variableName, value }]` | every `variableName` must be a declared dynamic variable |
| `end_call` | ends the call | `label?` | terminal: no outgoing edges |
| `transfer` | phone transfer | `routes: [{ number, condition, transferType, timeoutSecs }]` where `transferType` is `"COLD"` or `"ATTENDED"` | terminal when all routes are `COLD`; an `ATTENDED` route returns control, so outgoing edges are then allowed but must be `unconditional`; at least one route needs a non-empty `number` |

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
- A `function` node has exactly one attached function.
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
{ "classification": { "enabled": true, "categories": [ { "key", "name", "options": [], "description" } ] },
  "extracted_variables": { "enabled": true, "variables": [ { "key", "name", "type", "description" } ] },
  "summary": { "enabled": true, "prompt_enabled": false, "prompt": "" },
  "actions_enabled": true }
```
- `extracted_variables[].type`: `ANY` | `STRING` | `NUMBER` | `BOOLEAN` | `ENUM`.
- `actions_enabled` triggers configured post-call actions after the call.

### security
```
{ "rate_limit_per_minute": 60 }
```
Max inbound calls per minute before rejecting.

### guardrails
```
{ "focus": true, "manipulation": true,
  "content": true,
  "content_config": { "sexual": { "enabled": true, "threshold": "medium" }, "violence": {...}, "harassment": {...},
                      "self_harm": {...}, "profanity": {...}, "religion_or_politics": {...}, "medical_and_legal_information": {...} },
  "custom": false, "custom_rules": [ { "enabled": true, "name": "", "prompt": "" } ] }
```
- `focus` refuses off-purpose topics; `manipulation` resists social engineering.
- `content` is the master toggle; each of the 7 `content_config` categories is `{ enabled, threshold: "low"|"medium"|"high" }`.

### gdpr
```
{ "consent_required": false, "anonymize_data": false, "data_retention_days": 90, "recording_enabled": false }
```
