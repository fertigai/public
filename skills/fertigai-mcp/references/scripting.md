# fertig.ai script environment (functions and actions)

## Overview
Functions and actions run the SAME sandboxed JavaScript: an ES module with a single `run(ctx)` export, executed in a sandboxed runtime targeting **ECMAScript 2020**. There is no DOM, no Node.js, no `require`, no npm imports, and no timers (`setTimeout`/`setInterval`); the only capabilities are the built-in primitives below. What differs between a function and an action is the `ctx` payload and how the return value is handled (see functions.md and actions.md).

**TypeScript is accepted**: scripts may use type annotations, which are stripped before execution (types are editor-only, never enforced at runtime). Only erasable syntax is allowed - `enum`, `namespace`, constructor parameter properties, `<T>expr` assertions, and decorators are rejected at save time. Plain JavaScript works unchanged.

## Entry point
```ts
export async function run(ctx) {
  // ...your logic...
  return { status: 200, data: { ok: true } }
}
```
The export MUST be an async function named `run`, and the return value must be JSON-serializable. **`await` every asynchronous primitive**: if `run` returns while an outbound call (mail, http, tasks, ...) is still pending, the run fails with a "missing `await`" error.

## ctx.event: what triggered this run
`ctx.event.type` identifies the trigger: `agent-invocation`, `mailhook` (with `hook_id`), `task-trigger` (with `trigger_id`; the task is `ctx.task`), `post-call-analysis` (actions), `manual-test` (editor/test runs, with `surface`), or `function-call` (invoked by another script, with `caller: { kind, id?, depth, params }` and a `parent` chain back to the original trigger). Extra fields sit directly on `ctx.event`.

## Built-in primitives
Global; the same set is available to functions and actions.

**HTTP**
- `fetch(url, init?)` -> `{ status, ok, statusText, headers, body /* string */, error? }`. Low-level; `init` = `{ method?, headers?, body? }`. Never throws on network failure: you get `status: 0` with `error` set. Fixed 15 s per-request timeout.
- `http.get/delete(url, init?)`, `http.post/put/patch(url, body?, init?)` -> `{ status, data }`. Higher-level JSON client (serializes the body, parses JSON responses). `init` = `{ headers?, timeoutMs? }` (default 15 s). Unlike `fetch`, THROWS on failure.

**Secrets**
- `secrets.get(name)` -> `Promise<string>`. The decrypted workspace secret, or `""` if missing (never throws).

**Logging**
- `log.info/warn/error(message, fields?)` - structured log entries; `console.log/info/warn/error` are equivalent. Not variadic; put extra data in the `fields` object.

**Date and time**
- `time.now(tz?)`, `time.from(input, tz?)` (epoch ms, ISO 8601 string, or `Date`), `time.parse(text, format, tz)` (strftime; `tz` is required) -> an immutable, timezone-aware time value: `{ iso, epochMs, timezone, year, month /* 1-based */, day, hour, minute, second, weekday /* 0=Sun */, weekdayName }`.
- Methods (each returns a new value): `.add(delta)` / `.subtract(delta)` with `{ weeks?, days?, hours?, minutes?, seconds? }` (unknown keys throw), `.startOfDay()`, `.tz(zone)`, `.format(fmt, locale?)` (default `"en"`). DST-aware: a nonexistent wall-clock time throws.

**Mail**
- `mail.send({ to, cc?, bcc?, replyTo?, subject, body?, html?, template?, variables? })` -> `{ status, mailId?, error? }`. Managed relay: the From address is platform-fixed, use `replyTo`. At least one of `body`/`html`/`template` (a stored mail-template slug; `variables` fills its placeholders); `template` and `html` are mutually exclusive. Attachments are NOT supported (a non-empty `attachments` throws). Caps: 50 recipients, 10 MiB body. A delivery failure returns a 4xx/5xx `status` (does not throw).
- `mail.smtp({ host, port /* 465 or 587 */, user, pass, from, subject, to, cc?, bcc?, text?, html?, template?, variables? })` -> `{ status, data: { messageId, error } }`. Sends through your own SMTP server; throws on bad arguments or connection failure.

**LLM**
- `llm.send(prompt, opts?)` -> `Promise<string>`. `opts` = `{ model?, temperature?, maxTokens? }`.
- `llm.sendStructured(prompt, schema, opts?)` -> `Promise<object>`. `schema` is a field map (value = a description string, or `{ type, description }`; `type` is `string`|`number`|`integer`|`boolean`, with a `[]` suffix for arrays; a nested object nests the map). Every schema field is required in the reply. Both throw on failure.

**Other functions**
- `functions.invoke(key, params?)` -> `{ status, data }`. Calls another workspace function by its `key` (the slug shown in `fertigai_functions_get`, not the `fn_` id). Throws on error. Nesting is capped at 3 levels.

**Tasks**
- Bodies are the task REST bodies with snake_case keys (see tasks.md); an unknown key throws. `task` is a task id (`tsk_...`), a migrated ticket id (`tkt_...`), or a key such as `"SUP-12"`. Scripts act for the workspace, so private task types are included. Writes from a script that a task trigger started (directly or through scripts it called) do not fire task triggers again.
- `tasks.create({ type_id, title, description?, status_id?, priority?, assignee_user_id?, assigned_group_id?, requester_user_id?, requester_contact_id?, parent?, fields?, label_ids?, tags?, source? })` -> the task. `type_id` is a `tty_` id or a type key; `priority` is `NONE`|`URGENT`|`HIGH`|`MEDIUM`|`LOW` (default `NONE`); `fields` is keyed by field key; `source` defaults to `PHONE` inside a conversation, else `WEB`. With no requester on a call, the caller's contact becomes the requester, and the run's conversation is linked. Retries are deduplicated: one call creates at most one task.
- `tasks.update(task, { title?, description?, status_id?, priority?, assignee_user_id?, assigned_group_id?, requester_user_id?, requester_contact_id?, parent?, type_id?, fields?, labels_add?, labels_remove?, tags_add?, tags_remove?, conversations_link?, conversations_unlink? })` -> the task. Omitted keys stay unchanged; `""` clears; `fields` merges by key and `null` clears one. Sent once, never retried: after a timeout, check with `tasks.get` before retrying.
- `tasks.comment(task, body)` -> the comment event. Markdown; `[@Name](mention:usr_...)` mentions and subscribes a member. Sent once, never retried.
- `tasks.get(task)` -> the task, or `null` when it does not exist (does not throw).
- A task is returned as the REST API serves it: `{ id, key, number, type_id, title, description, status_id, status_category, priority, assignee, assigned_group_id, requester_user, requester_contact, parent_task_id, field_values /* keyed by field id */, label_ids, tags, source, created_via, legacy_ticket_id, conversation_ids, created_at, updated_at, ... }`.
- `tickets.create({ template, title, fields?, requester?, priority?, statusId? })` is a legacy alias: it creates a task in the task type migrated from the ticket template `template` and returns `{ id /* tsk_... */, created }`. New scripts use `tasks.create`.

**Custom object records**
- `objects.query(kind, { filter?, search?, externalId?, cursor?, limit? /* 1-100 */ })` -> `{ records, nextCursor, hasMore }`. `filter` is an equality map of scalar values.
- `objects.get(kind, id)` -> record or `null` (missing is not an error).
- `objects.create(kind, { name, externalId?, data? })` / `objects.update(kind, id, { name?, externalId?, data? })` -> the record. `update` merges `data` key by key; a `null` value clears that key.
- `objects.delete(kind, id)` -> `boolean` (`false` when already gone).
- A record is `{ id /* cor_... */, name, externalId, data, createdAt, updatedAt }`.

**Short links**
- `urlshort.shorten(url, { ttlSeconds?, oneTime? }?)` -> `Promise<string>`, the short URL. The link belongs to the running workspace (or organization) and is listed on its Short links page, where it can be revoked; `ttlSeconds` sets when it expires, `oneTime` stops it after the first visit. Links expire after 90 days unless `ttlSeconds` is given (at most 365 days). Throws on an invalid URL.

**Transcript helper**
- `conversation.normalize(ctx)` -> `string` (synchronous, no `await`). Formats `ctx.transcript` into `Agent:` / `User:` lines.

**Encoding**
- `btoa(s)` / `atob(s)` / `base64Encode(s)` / `base64Decode(s)` - UTF-8-aware base64.

**Workspace integrations (dynamic, per workspace)**
- `ctx.connection` / `ctx.connections[role]`: decrypted credentials of the bound connection(s). `ctx.connections` is keyed by the role the integration, function or action declares; `ctx.connection` is the `default` role. `integrations.<provider>.<fn>(connectionPublicId, params)` -> `Promise` calls an integration from a script; connectionless ones are `integrations.standalone.<fn>(params)`.
- `mcp.<server>.<tool>(params)` -> `Promise`. Calls a tool on an MCP server registered in the workspace.

**Not implemented (throw if called):** `sms.send`.

## Sandbox limits
- Wall-clock timeout: 30 s (a caller may request less, never more).
- Memory 64 MB; stack 1 MB; script source max 64 KiB.
- Outbound calls per run: fetch + http 50, integrations 50, mcp 50, objects 50, `functions.invoke` 10, mail 10, `tasks.*` 20 (`create`, `update`, `comment` and `get` together), `tickets.create` 5, llm 5, `urlshort.shorten` 10. Exceeding a quota throws a catchable error.
- HTTP response body cap 10 MiB.
- Private-network addresses are blocked for all outbound HTTP and SMTP.
- No DOM, no Node APIs, no timers, no `require`, no npm.

## ctx and return shape differ by script type
- **Function**: `ctx` is param-centric (`ctx.params`, plus `event` and, when applicable, `connection`, `conversation_id`, `agent_id`, `mail`, `task`). It MUST return `{ status: <int 100-599>, data: <any> }`. See functions.md.
- **Action**: `ctx` is conversation-centric (`conversation`, `transcript`, `summary`, `classification`, `variables`, `workspace`, `params`, `event`). Its return value is stored verbatim, with no required shape. See actions.md.
