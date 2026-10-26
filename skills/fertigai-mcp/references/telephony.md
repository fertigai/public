# Telephony routing (fertigai_phone_numbers_*, fertigai_call_flows_*, fertigai_call_queues_*, fertigai_audios_*, fertigai_schedules_*)

## Overview
Telephony routing decides what happens when someone calls one of the workspace's phone numbers. A phone number answers in one of two ways, never both:
- **attached to an agent branch**: the call goes straight to that agent branch;
- **bound to a call flow**: the call runs through a graph of steps (play a prompt, check opening hours, offer a key menu, wait in a queue, hand over to an agent, ring an outside number, hang up).

The building blocks:
- **Phone numbers** (`phn_...`) and **phone number blocks** (`pnb_...`, ranges of consecutive numbers).
- **Call flows** (`cfl_...`): the routing graph.
- **Call queues** (`cq_...`): cap how many callers pass a point in a flow at once.
- **Audio prompts** (`aud_...`): WAV or MP3 files that flows and queues play.
- **Schedules** (`sch_...`): opening hours, date exceptions and statutory holidays.
- **SIP trunks** (`st_...`): the connection a number routes over. Creating, editing, testing and checking trunks is in `references/sip-trunks.md`.

Buying a phone number from the platform pool is not available over MCP: buy numbers in the portal. Over MCP you can register numbers you already own on one of your SIP trunks, and route any number of the workspace.

Shared conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`). List tools take `cursor?` and `page_size?` (1 to 100, default 20).

## Tools
**Phone numbers**
| Tool | Args |
|---|---|
| `fertigai_phone_numbers_list` | `search?` (number or display name), `block_id?`, `exclude_block_members?`, `sip_trunk_id?`, `cursor?`, `page_size?` |
| `fertigai_phone_numbers_create` | `number`, `sip_trunk_id`, `kind?`, `country_code?`, `display_name?`, `codec_override?` |
| `fertigai_phone_numbers_update` | `id`, then any of `display_name`, `call_flow_id`, `caller_id_pref`, `called_id_pref`, `codec_override`, and the routing fields `number`, `kind`, `country_code`, `sip_trunk_id` |
| `fertigai_phone_numbers_delete` | `id` |
| `fertigai_phone_numbers_countries` | none (only where number buying is enabled, see below) |
| `fertigai_phone_numbers_prefixes` | `country_code` (only where number buying is enabled) |

**Phone number blocks**
| Tool | Args |
|---|---|
| `fertigai_phone_number_blocks_list` | `cursor?`, `page_size?` |
| `fertigai_phone_number_blocks_create` | `sip_trunk_id`, `head_number`, `country_code`, `size`, `includes_head?`, `display_name?`, `validate_only?` |
| `fertigai_phone_number_blocks_update` | `id`, `display_name` |
| `fertigai_phone_number_blocks_delete` | `id` |

**Numbers on agents**
| Tool | Args |
|---|---|
| `fertigai_agent_phone_numbers_list` | `agent_id` (numbers on any branch of the agent) |
| `fertigai_agent_branch_phone_numbers_list` | `agent_id`, `branch_id` |
| `fertigai_agent_branch_phone_numbers_attach` | `agent_id`, `branch_id`, `phone_number_id` |
| `fertigai_agent_branch_phone_numbers_detach` | `agent_id`, `branch_id`, `phone_number_id` |

**Call flows**
| Tool | Args |
|---|---|
| `fertigai_call_flows_list` | `search?` (name), `cursor?`, `page_size?` |
| `fertigai_call_flows_get` | `id` |
| `fertigai_call_flows_create` | `name`, `enabled?`, `graph` |
| `fertigai_call_flows_update` | `id`, `name`, `enabled?`, `graph` (full replace) |
| `fertigai_call_flows_delete` | `id` |
| `fertigai_call_flows_phone_numbers` | `id` (the numbers bound to the flow) |
| `fertigai_agent_call_flows_list` | `agent_id` (the flows whose `voice_agent` nodes route to the agent) |

**Call queues**
| Tool | Args |
|---|---|
| `fertigai_call_queues_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_call_queues_get` | `id` |
| `fertigai_call_queues_create` | `name`, `capacity`, `hold_audio_id`, `max_wait_secs?`, `enabled?` |
| `fertigai_call_queues_update` | `id`, `name`, `capacity`, `hold_audio_id`, `max_wait_secs?`, `enabled?` (full replace) |
| `fertigai_call_queues_delete` | `id` |

**Audio prompts**
| Tool | Args |
|---|---|
| `fertigai_audios_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_audios_get` | `id` |
| `fertigai_audios_upload` | `name`, `content_base64`, `file_name?`, `content_type?` |
| `fertigai_audios_url` | `id` |
| `fertigai_audios_update` | `id`, `name` |
| `fertigai_audios_delete` | `id` |

**Schedules and holidays**
| Tool | Args |
|---|---|
| `fertigai_schedules_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_schedules_get` | `id` |
| `fertigai_schedules_create` | `name`, `timezone`, `rules` |
| `fertigai_schedules_update` | `id`, `name`, `timezone`, `rules` (full replace) |
| `fertigai_schedules_delete` | `id` |
| `fertigai_holidays_list` | `region`, `from`, `to` |

**Permissions**: phone numbers, blocks, branch numbers, call flows and call queues need Telephony-View to read and Telephony-Manage to change. Audio prompts, schedules and holidays need Integrations-View to read and Integrations-Manage to change.

## Phone numbers
Each number has: `id` (`phn_...`), `number`, `display_name`, `type` (`MANAGED` from the platform pool, `CUSTOM` registered by you), `status` (`ACTIVE`, `PENDING`, `INACTIVE`), `kind` (`DID` a public number, `EXTENSION` an internal number of its trunk), `country_code`, `monthly_cost`, `sip_trunk_id`, `call_flow_id` (empty when not bound to a flow), `routing_editable`, `scope`, `caller_id_pref`, `called_id_pref` and `codec_override`.

**Create (bring your own number)**:
- `number`: E.164 (`+4930123456`) for a `DID`, digits only for an `EXTENSION`. A number exists once per trunk (409 otherwise).
- `sip_trunk_id`: a trunk from `fertigai_sip_trunks_list`.
- `kind`: `DID` (default) or `EXTENSION`.
- `country_code`: ISO 3166-1 alpha-2 (`DE`). Required for a `DID`; must be empty for an `EXTENSION`.
- `codec_override`: `{ "codecs": [...] }`, an ordered list from `PCMA`, `PCMU`, `G.722`, `G.729`, `G.726`, `Opus`. Leave it out to use the trunk's codecs.
- A trunk set to a single device holds at most one `DID` and one `EXTENSION` (422).

A new number routes nowhere until you attach it to a branch or bind it to a call flow.

**Update** (only the fields you send change):
- `display_name`.
- `call_flow_id`: a `cfl_` id binds the number to that flow; `""` unbinds it. See "Routing a number" below.
- `caller_id_pref` (the caller ID sent on outbound and transfer calls) and `called_id_pref` (the P-Asserted-Identity header): `INHERIT`, `AUTO`, `DIALED_DID` or `ORIGINAL_CALLER`, in any case; another value is a 400.
- `codec_override`: leave it out to keep it, `{ "codecs": [] }` to inherit the trunk's list again, or a list to replace it.
- **Routing fields** `number`, `kind`, `country_code`, `sip_trunk_id`: change only where the number's `routing_editable` is true. They are locked (422) on `MANAGED` numbers, block members, numbers lent to the workspace by the organization, and numbers that route over a shared organization or global SIP trunk. `sip_trunk_id: ""` is a 422: a number always has a trunk.

**Delete**: a number you registered is deleted; a pool number goes back to the pool, where it may be given to someone else. Either way it stops routing and its branch attachments go. A block member cannot be deleted alone (409): delete its block.

**Countries and prefixes**: `fertigai_phone_numbers_countries` and `fertigai_phone_numbers_prefixes` show what the platform pool offers (prefix, `country_code`, `monthly_cost`, `type` such as `LOCAL` or `MOBILE`). They answer 403 unless number buying is enabled for the workspace. Buying itself is portal-only.

## Phone number blocks
A block registers a range of your own consecutive DIDs on one trunk: one phone number per suffix of `head_number`.
- `sip_trunk_id`: not a trunk set to a single device (422).
- `head_number`: the E.164 prefix every member shares, at least 5 digits.
- `country_code`: ISO 3166-1 alpha-2.
- `size`: `10`, `100`, `1000` or `10000` (the head followed by 1 to 4 digits).
- `includes_head`: also register the head itself (default false); the head must then be a valid E.164 number.
- `validate_only`: check the request and return the would-be block with an empty `id`, without creating anything.

Numbers of the workspace already in the range join the block (`adopted_count`); any other existing number in the range refuses the create with 409. Members are ordinary phone numbers, but their number, kind, country and trunk belong to the block. List them with `fertigai_phone_numbers_list` and `block_id`. A block shows `head_number`, `size`, `includes_head`, `number_count` and `routed_count`.

`fertigai_phone_number_blocks_update` only renames the block (members keep their own labels). `fertigai_phone_number_blocks_delete` deletes the block, all its member numbers and their branch attachments.

## Routing a number
A number answers **either** an agent branch **or** a call flow:
- **To an agent branch**: `fertigai_agent_branch_phone_numbers_attach` with `agent_id`, `branch_id` (from `fertigai_agents_get`) and `phone_number_id`. 409 while the number is bound to a call flow, or when it is already attached to that branch. `fertigai_agent_branch_phone_numbers_detach` removes the attachment; the number stays in the workspace, unrouted.
- **To a call flow**: `fertigai_phone_numbers_update` with `id` and `call_flow_id`. 409 while the number is attached to an agent branch: detach it first. The flow must belong to the same workspace (422 otherwise). `call_flow_id: ""` unbinds it.

To see what routes where: `call_flow_id` on each phone number, `phone_number_count` on each flow, `fertigai_call_flows_phone_numbers` for one flow, and `fertigai_agent_branch_phone_numbers_list` / `fertigai_agent_phone_numbers_list` for agents.

## Call flows
A flow has `id` (`cfl_...`), `name` (1 to 255 bytes), `enabled` (default true; a disabled flow leaves its numbers unrouted), `graph`, `version` (goes up on every save), `phone_number_count`, `created_at` and `updated_at`. Changes reach the bound numbers on the next call.

`fertigai_call_flows_update` replaces the whole flow: get it, change the graph, and send `name`, `enabled` and `graph` back. An omitted `enabled` means true.

`fertigai_call_flows_delete` answers 409 while numbers are bound to the flow: unbind them first.

### The graph
`graph` is the editor document `{ "nodes": [...], "edges": [...] }`, sent as a JSON object exactly as `fertigai_call_flows_get` returns it.
- **Node**: `{ "id", "type", "position": { "x", "y" }, "data": { ... } }`. `id` is any non-empty string, unique in the flow. `position` places the node in the portal editor. `data` keys are camelCase.
- **Edge**: `{ "id", "source", "sourceHandle", "target" }`. `source` and `target` are node ids; `sourceHandle` is the exit of the source node the edge leaves from.
- An exit without an edge hangs up for the main exits (`done`, `failed`, `closed`, ...) and does nothing for `caller_ended_call` and the `*.picked_up` exits.

### Node types
| `type` | `data` | Exits |
|---|---|---|
| `entry` | none | `start` |
| `voice_agent` | `agentId`, `branchId` (a branch of that agent) | `agent_ended_call`, `caller_ended_call`, `transferred.picked_up`, `transferred.ended`, `failed` |
| `redirect` | `number` (E.164), `ringTimeoutSecs?` (1 to 120, default 30) | `connected.picked_up`, `connected.ended`, `caller_ended_call`, `failed` |
| `audio` | `audioId`, `loop?` (default false) | `done`, `caller_ended_call` |
| `opening_hours` | `scheduleId` | `open`, `closed` |
| `menu` | `audioId`, `retryAudioId?`, `entries`, `timeoutSecs?`, `interdigitSecs?`, `retries?` | `key:<digits>` per entry, `no_input`, `invalid`, `caller_ended_call` |
| `balancer` | `outputs`, `stickyByCaller?` | one exit per output `id` |
| `queue_enter` | `queueId`, `entries?` | `admitted`, `timeout`, `key:<digits>` per entry, `caller_ended_call` |
| `queue_end` | `queueId` | `next` |
| `hang_up` | none | none |

- **`voice_agent`** hands the call to an agent branch. `transferred.picked_up` fires when a transfer target answers; `transferred.ended` when that call ends.
- **`redirect`** rings an outside number and connects the caller. `failed` covers busy, no answer within `ringTimeoutSecs`, and a rejected call.
- **`audio`** plays a prompt; with `loop: true` it repeats it.
- **`opening_hours`** takes `open` or `closed` from a schedule, at the moment the caller reaches the node.
- **`menu`** plays `audioId` and routes on the keys pressed:
  - `entries`: 1 to 64 items `{ "digits": "1", "label": "Sales" }`. `digits` is 1 to 16 of `0`-`9`, `*`, `#`, unique within the node; each entry adds the exit `key:<digits>`. `label` is for the editor only.
  - `retryAudioId`: played before the prompt repeats; `""` or absent for none.
  - `timeoutSecs`: wait for input after the prompt, 1 to 60 (default 10).
  - `interdigitSecs`: wait after a digit while a longer entry could still match, 1 to 10 (default 3).
  - `retries`: extra attempts after no input or an invalid key, 0 to 5 (default 2).
- **`balancer`** splits calls by percentage:
  - `outputs`: at least 2 items `{ "id": "a", "percent": 50 }`. Percents are whole numbers from 1 to 100 and add up to 100. Each `id` is unique in the node and becomes an exit; it must not be an exit name used elsewhere (`start`, `agent_ended_call`, `caller_ended_call`, `transferred.picked_up`, `transferred.ended`, `connected.picked_up`, `connected.ended`, `failed`, `done`, `open`, `closed`, `admitted`, `timeout`, `next`, `no_input`, `invalid`). Every output needs an edge.
  - `stickyByCaller: true` sends the same caller number to the same output each time.
- **`queue_enter`** holds the caller in a call queue until a slot is free (`admitted`). `timeout` fires after the queue's `max_wait_secs`; a `timeout` edge needs a queue with a non-zero `max_wait_secs`. `entries` (same shape as `menu`) are keys that let the caller leave the queue while waiting.
- **`queue_end`** frees the caller's queue slot, then takes `next`. Place it where the queued part of the call is over, for example on the agent's exits.
- **`hang_up`** ends the call.

### Validation rules
A create or update that breaks a rule answers 422 and names the failing path, for example `graph.nodes[2].data.audioId` or `graph.edges[4]`.
- Exactly one `entry` node, and it has a `start` edge. Nothing points to the entry.
- Node ids are unique and non-empty; every edge's `source` and `target` are nodes of the flow.
- `sourceHandle` must be an exit of the source node, and each exit has at most one edge.
- Every node is reachable from the entry.
- The branch after `transferred.picked_up`, `connected.picked_up` or `caller_ended_call` may hold only `opening_hours` and `queue_end` nodes: the caller is already talking to someone or gone, so nothing may play. Do not put `hang_up` there.
- No loop made only of instant nodes (`balancer`, `opening_hours`, `queue_end`, and `queue_enter` on a disabled queue); a loop must pass a node that plays or waits.
- Every agent branch, audio, schedule and queue the graph names exists in this workspace.

Smallest valid flow (answers and hangs up):
```json
{
  "nodes": [
    { "id": "entry", "type": "entry", "position": { "x": 0, "y": 0 }, "data": {} },
    { "id": "end", "type": "hang_up", "position": { "x": 0, "y": 150 }, "data": {} }
  ],
  "edges": [
    { "id": "e1", "source": "entry", "sourceHandle": "start", "target": "end" }
  ]
}
```

## Call queues
A queue caps how many callers pass a flow's `queue_enter` node at once; the rest wait and hear the hold audio. The cap is shared by every flow that uses the queue.
- `name`: 1 to 255 bytes.
- `capacity`: callers admitted at once, 1 to 1000.
- `max_wait_secs`: the longest wait before the `timeout` exit, 0 to 7200 (default 0 = no limit).
- `hold_audio_id`: an audio prompt of the workspace (upload one first).
- `enabled`: default true. A disabled queue admits every caller at once, so you can switch a queue off without editing the flows.

Responses add the live `waiting` and `active` counts. `fertigai_call_queues_update` replaces every field: an omitted `max_wait_secs` becomes 0 and an omitted `enabled` becomes true. Delete answers 409 naming the flows that still use the queue.

## Audio prompts
An audio has `id` (`aud_...`), `name`, `file_name`, `format` (`wav` or `mp3`), `size_bytes` and `duration_ms`. `audio` and `menu` nodes and queue hold music play them.

**Upload** with `fertigai_audios_upload`:
- `name`: 1 to 255 bytes.
- `content_base64`: the file, base64-encoded (standard alphabet). WAV or MP3 only, at most 5 MB (5242880 bytes) after decoding. The format and duration are read from the file itself, so the file name and type do not matter.
- `file_name`: kept for display (default `audio.<format>`).
- `content_type`: informational only.

A file over the limit or invalid base64 is refused before anything is stored. A file that is not WAV or MP3, or has no playable length, answers 422.

`fertigai_audios_url` returns `{ "url": "..." }`, a temporary link to listen to or download the file, valid for 15 minutes. `fertigai_audios_update` renames the audio; the file itself never changes, so upload a new audio to replace it. Delete answers 409 naming the queues or flows that still play it.

## Schedules (opening hours)
A schedule has `id` (`sch_...`), `name` (1 to 255 bytes), `timezone` (an IANA name such as `Europe/Berlin`; every time is local to it) and `rules`:
- **`weekly`**: open ranges `[{ "weekday": "mon", "from": "08:00", "to": "18:00" }]`. `weekday` is `mon` to `sun`; times are `HH:MM` (24-hour), `to` may be `24:00`. `from` is before `to` (`to` is exclusive); at most 20 ranges per day; ranges of one day must not overlap (touching is fine).
- **`overrides`**: date exceptions that replace that day's weekly hours, at most 366, one per date. Either `{ "date": "2026-12-24", "closed": true }` or `{ "date": "2026-12-31", "open": [{ "from": "09:00", "to": "12:00" }] }`, never both.
- **`holidays`** (optional): `{ "region": "DE-BY", "skip": ["assumption_day"] }`. The schedule is closed on every statutory holiday of the region, except the keys listed in `skip`, which keep their normal hours. Skip keys must be holidays of the region and unique. An override wins over a holiday on its date.

Any time not covered by a range is closed; an empty `rules` object is always closed. `fertigai_schedules_update` replaces name, timezone and rules: get, change, send everything back. Delete answers 409 naming the flows that still use the schedule.

### Holidays
`fertigai_holidays_list` lists a region's statutory holidays between two dates, sorted by date, as `[{ "key": "good_friday", "date": "2026-04-03" }]`.
- `region`: `DE` (nationwide), a German state (`DE-BW`, `DE-BY`, `DE-BE`, `DE-BB`, `DE-HB`, `DE-HH`, `DE-HE`, `DE-MV`, `DE-NI`, `DE-NW`, `DE-RP`, `DE-SL`, `DE-SN`, `DE-ST`, `DE-SH`, `DE-TH`), `AT`, or a Swiss canton (`CH-AG`, `CH-AI`, `CH-AR`, `CH-BE`, `CH-BL`, `CH-BS`, `CH-FR`, `CH-GE`, `CH-GL`, `CH-GR`, `CH-JU`, `CH-LU`, `CH-NE`, `CH-NW`, `CH-OW`, `CH-SG`, `CH-SH`, `CH-SO`, `CH-SZ`, `CH-TG`, `CH-TI`, `CH-UR`, `CH-VD`, `CH-VS`, `CH-ZG`, `CH-ZH`). There is no Switzerland-wide region.
- `from`, `to`: `YYYY-MM-DD`; `to` is not before `from` and at most two years after it. 422 otherwise.

Use the returned keys in `rules.holidays.skip`. List a full year to see every key of a region: a few holidays fall only in some years.

## Example: opening hours in front of an agent
Goal: during office hours calls go to an agent; outside them, and on Bavarian holidays, callers hear a message and the call ends.

1. Find the holiday keys, then create the schedule (closed on all `DE-BY` holidays except Assumption Day, and on Christmas Eve):
```
fertigai_holidays_list { "region": "DE-BY", "from": "2026-01-01", "to": "2026-12-31" }
```
```json
{
  "name": "Office hours",
  "timezone": "Europe/Berlin",
  "rules": {
    "weekly": [
      { "weekday": "mon", "from": "08:00", "to": "18:00" },
      { "weekday": "tue", "from": "08:00", "to": "18:00" },
      { "weekday": "wed", "from": "08:00", "to": "18:00" },
      { "weekday": "thu", "from": "08:00", "to": "18:00" },
      { "weekday": "fri", "from": "08:00", "to": "16:00" }
    ],
    "overrides": [{ "date": "2026-12-24", "closed": true }],
    "holidays": { "region": "DE-BY", "skip": ["assumption_day"] }
  }
}
```
   Send this as `fertigai_schedules_create`; it returns the `sch_...` id.
2. Upload the closed message (`fertigai_audios_upload` with `name` and `content_base64`); it returns the `aud_...` id.
3. Take the agent's `agt_...` id and the branch id from `fertigai_agents_list` / `fertigai_agents_get`.
4. Create the flow with `fertigai_call_flows_create`:
```json
{
  "name": "Main line",
  "enabled": true,
  "graph": {
    "nodes": [
      { "id": "entry", "type": "entry", "position": { "x": 0, "y": 0 }, "data": {} },
      { "id": "hours", "type": "opening_hours", "position": { "x": 0, "y": 150 }, "data": { "scheduleId": "sch_..." } },
      { "id": "agent", "type": "voice_agent", "position": { "x": -200, "y": 300 }, "data": { "agentId": "agt_...", "branchId": "<branch-id>" } },
      { "id": "closed_msg", "type": "audio", "position": { "x": 200, "y": 300 }, "data": { "audioId": "aud_...", "loop": false } },
      { "id": "end", "type": "hang_up", "position": { "x": 200, "y": 450 }, "data": {} }
    ],
    "edges": [
      { "id": "e1", "source": "entry", "sourceHandle": "start", "target": "hours" },
      { "id": "e2", "source": "hours", "sourceHandle": "open", "target": "agent" },
      { "id": "e3", "source": "hours", "sourceHandle": "closed", "target": "closed_msg" },
      { "id": "e4", "source": "closed_msg", "sourceHandle": "done", "target": "end" }
    ]
  }
}
```
   The agent's exits have no edges, so the call ends when the agent or the caller hangs up.
5. Bind a number: find it with `fertigai_phone_numbers_list`. If it is attached to an agent branch, detach it first (`fertigai_agent_branch_phone_numbers_detach`). Then:
```
fertigai_phone_numbers_update { "id": "phn_...", "call_flow_id": "cfl_..." }
```
6. Check with `fertigai_call_flows_phone_numbers { "id": "cfl_..." }`.

## Common mistakes
- Trying to buy a number over MCP. There is no tool for it; buy numbers in the portal. `fertigai_phone_numbers_create` only registers a number you already own on your SIP trunk.
- Binding a number to a flow while it is attached to a branch, or attaching it while it is bound (409). Detach or unbind first.
- Looking for the binding on the flow. A number is bound through `fertigai_phone_numbers_update` (`call_flow_id`), not through the flow.
- Sending a partial flow, queue or schedule to an `update` tool. These replace the whole object: get, change, send everything back.
- Putting `hang_up` (or any node that plays) after `caller_ended_call`, `transferred.picked_up` or `connected.picked_up`. Leave those exits unconnected unless you need a `queue_end` there.
- A `timeout` edge from `queue_enter` on a queue whose `max_wait_secs` is 0.
- Balancer percents that do not add up to 100, or an output without an edge.
- snake_case keys in node `data` (`audio_id`). Node data is camelCase (`audioId`); tool arguments are snake_case.
- Thinking `holidays.skip` lists the closed days. The region's holidays are closed by default; `skip` lists the ones that keep normal hours.
- Uploading a file path or URL as `content_base64`. Send the file's bytes, base64-encoded, at most 5 MB decoded.
- Editing the routing fields of a number whose `routing_editable` is false (422). Only the label, caller-ID settings, `call_flow_id` and `codec_override` change there.
- Deleting a flow, queue, audio or schedule that is still in use (409 names the users). Unbind or edit those first.
