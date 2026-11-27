# Analytics and dashboards (fertigai_analytics_get, fertigai_dashboards_get)

## Overview
Read-only reporting: conversation analytics over a date range, and the agents, tasks and telephony dashboards the portal shows. Nothing here changes the workspace. Both tools are in the `analytics` toolset (`?toolsets=analytics`).

Shared conventions (ids, errors, permissions) are in the skill index (SKILL.md). Both tools also take an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_analytics_get` | `metric`, `date_from?`, `date_to?`, `agent_id?`, `branch_id?`, `status?`, `classification?`, `category?` |
| `fertigai_dashboards_get` | `kind` |

## Analytics metrics
`fertigai_analytics_get` returns `{ "data": ... }` with one aggregate, picked by `metric`:

| `metric` | `data` |
|---|---|
| `summary` | one object: `total_conversations`, `completed`, `transferred`, `dropped`, `total_duration`, `avg_duration`, and the selection's call time: `total_seconds`, `ai_seconds`, `talk_seconds`, `avg_duration_seconds`, `median_duration_seconds` |
| `volume` | per day: `date`, `total`, `completed`, `transferred`, `dropped` |
| `status_distribution` | per outcome: `status` (`COMPLETED`, `TRANSFERRED`, `DROPPED`), `count`, `percentage` |
| `duration_trend` | per day: `date`, `avg_duration`, `min_duration`, `max_duration`, `count` |
| `agents` | per agent: `agent_id`, `agent_name`, `total_conversations`, `avg_duration`, `total_duration`, `success_rate` |
| `agent_success_rates` | per agent: `agent_id`, `agent_name`, `total`, `completed`, `transferred`, `dropped`, `completion_rate` |
| `peak_hours` | per hour of day: `hour` (0 to 23), `count` |
| `minutes_per_month` | per month: `month` (`YYYY-MM`), `total_minutes`, `conversation_count` |
| `classification_distribution` | per classification category and value: `category` (the category's display name), `value`, `count`, `chat_count`, `percentage`; with `category`, also the call time per value and one `unclassified` row (see below) |

- Voice conversations only (chats are left out): `duration_trend`, `agent_success_rates`, `minutes_per_month`, and the durations and `success_rate` in `summary` and `agents`. Every other count includes chats, so `agents.total_conversations` can be higher than `agent_success_rates.total`.
- Durations are seconds. `total_minutes` is the month's seconds divided by 60, rounded down.
- Success and completion rates are percentages (0 to 100), count `completed` plus `transferred` as successful, and are `0` when there is nothing to rate.
- Days, hours and months are UTC. `volume`, `duration_trend` and `peak_hours` list only days or hours that had conversations; a missing one means zero.
- Conversations of a deleted agent are grouped as `agent_id: "deleted"`.

### Call time per classification value
With `category`, each `classification_distribution` row also carries the call time of its voice calls, and the totals for the whole selection are on `summary`:

- `total_seconds`: from answer to hang-up. `ai_seconds`: time the AI agent handled the call. `talk_seconds`: time after forwarding to a person. All are seconds, unrounded (divide by 60 for minutes).
- `minutes_share`: the row's share of the `total_seconds` of all rows, the unclassified row included, as a percentage (0 to 100); the rows sum to 100.
- `avg_duration_seconds` and `median_duration_seconds`: per call, over answered calls longer than 0 s.
- One extra row, `unclassified: true` with an empty `value`, covers the selection's conversations without a value in that category. Old periods grow this row as retention deletes classifications.
- `percentage`: with `category`, the share of all the selection's conversations, the unclassified row included, so the rows sum to 100; without `category`, the share of that category's classified conversations.
- `chat_count`: how many of `count` are chats; the rest are calls.
- The call-time fields are absent without `category` and on rows with no voice calls (chats only); the averages are absent when no call qualifies. On `summary`, `total_seconds`, `ai_seconds` and `talk_seconds` are always present (0 without calls) and the two averages are absent when no call qualifies. They come from the call records (test calls excluded, dated by call start), so they need not match `total_duration`. A call with several AI sessions splits its time across their values.
- `summary` covers every voice call of the selection, the same figures the rows split, so call it with the same filters (without `category`) for the totals.

## Filters
- `date_from` / `date_to`: `YYYY-MM-DD` (read as midnight UTC) or an RFC 3339 timestamp. `date_from` is inclusive, `date_to` exclusive, so a full day is `date_from: "2026-08-11", date_to: "2026-08-12"`. Without them the range is the last 30 days through tomorrow.
- `agent_id` (`agt_...`) and `branch_id` (`br_...`): only that agent's or branch's conversations. An unknown id is a `400`.
- `status`: only conversations with that outcome, `completed`, `transferred` or `dropped` (lower case). Works with every metric.
- `classification`: only conversations classified with these values, a map from category `key` to value, e.g. `{ "intent": "BILLING" }`; with several keys all must match. Works with every metric. Keys and values must not be empty, so it cannot select the unclassified conversations. The keys are the same category keys as `category` takes.
- `category`: only with `classification_distribution`, one classification category `key`; with any other metric the call fails. The key is not the display name the result shows: read it from the agent branch config, `config.post_call_analysis.classification.categories[].key` (`fertigai_agent_branch_get`, see agents-workflow.md). An unknown key returns an empty list.

## Dashboards
`fertigai_dashboards_get` returns one dashboard by `kind`. KPIs reflect now; series and windowed figures (suffix `last30d`) cover the last 30 days.

| `kind` | Holds | Needs |
|---|---|---|
| `agents` | `kpis` (`total_agents`, `active_agents`, `conversations_today`, `success_rate_last30d`) and `fleet`, one row per agent: `agent_id`, `name`, `status`, `sync_status`, `conversations_last30d`, `success_rate_last30d`, `last_active_at` | Agents-View |
| `tasks` | `kpis` (`open_tasks`, `new_today`, `solved_last30d`, `avg_resolution_seconds_last30d`), `daily` (`date`, `created`, `solved`), `open_by_status` and `open_by_priority` (`key`, `count`) | the Tasks product; counts only tasks you can see |
| `telephony` | `kpis` (`calls_last30d`, `total_minutes_last30d`, `ai_minutes_last30d`, `talk_minutes_last30d`, `cost_eur_micros_last30d`, `phone_numbers`, `sip_trunks`), `daily` (`date`, `calls`, `total_minutes`, `ai_minutes`, `talk_minutes`), `top_numbers` (the five busiest: `number`, `calls_last30d`, `minutes_last30d`) | Telephony-View |

- `cost_eur_micros_last30d` is millionths of a euro (divide by 1,000,000).
- Telephony minutes come from the call records and are rounded up, so they will not match `minutes_per_month` exactly.
- `success_rate_last30d` counts voice conversations only and is `null` when there were none; `avg_resolution_seconds_last30d` is `null` when no task was solved.
- The `daily` series list every day of the window, zeros included.

## Example
```
fertigai_analytics_get { "metric": "summary", "date_from": "2026-08-01", "date_to": "2026-09-01" }
fertigai_analytics_get { "metric": "volume", "agent_id": "agt_...", "date_from": "2026-08-01", "date_to": "2026-09-01" }
fertigai_analytics_get { "metric": "classification_distribution", "category": "intent", "status": "completed" }
fertigai_analytics_get { "metric": "summary", "classification": { "intent": "BILLING" } }
fertigai_dashboards_get { "kind": "telephony" }
```
To read the conversations behind a number, use `fertigai_conversations_list` with the same date range, `agent_id` and `status` (conversations.md); it has no classification filter.

## Common mistakes
- Treating `date_to` as inclusive: the last day of a month is covered by `date_to` set to the first of the next month.
- Passing `category` with a metric other than `classification_distribution`.
- Reading `avg_duration` or `total_seconds` as minutes: they are seconds.
- Passing `category` to narrow the result: it picks the category to break down; narrow with `classification` instead.

`fertigai_analytics_get` requires the Analytics-View permission; each dashboard needs the permission in the table above.
