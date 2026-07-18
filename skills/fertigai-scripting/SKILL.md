---
name: fertigai-scripting
description: Use when writing the JavaScript inside a fertig.ai function or action - the run(ctx) entry point, the sandbox and limits, and the built-in primitives like mail.send, llm.send, http, secrets, functions.invoke, and tickets.create.
---

# fertig.ai script environment (functions and actions)

## Overview
Functions and actions run the SAME sandboxed JavaScript: an ES module with a single `run(ctx)` export, executed in a sandboxed JavaScript runtime targeting **ECMAScript 2020**. There is no DOM, no Node.js, no `require`, and no npm imports; the only capabilities are the built-in primitives below. What differs between a function and an action is the `ctx` payload and how the return value is handled (see fertigai-functions and fertigai-actions).

## Entry point
```js
export async function run(ctx) {
  // ...your logic...
  return { status: 200, data: { ok: true } }
}
```
The export MUST be an async function named `run`. **`await` every asynchronous primitive**: if `run` returns while an outbound call (mail, http, tickets, ...) is still pending, the run fails with a "missing await" error.

## Built-in primitives
Global; the same set is available to functions and actions.

**HTTP**
- `fetch(url, init?)` -> `{ status, headers, body /* string */, error? }`. Low-level; `init` = `{ method?, headers?, body? }`. On transport error returns `status: 0` with `error` set (does not throw).
- `http.get/post/put/patch/delete(url, body?, init?)` -> `{ status, data }`. Higher-level JSON client (serializes and parses JSON). `init` = `{ headers?, timeoutMs? }`.

**Secrets**
- `secrets.get(name)` -> `Promise<string>`. The decrypted workspace secret, or `""` if missing.

**Logging**
- `log.info(msg, fields?)`, `log.warn(...)`, `log.error(...)` - structured log entries.
- `console.log/warn/error` - aliases of `log.info/warn/error`.

**Mail**
- `mail.send({ to, cc?, bcc?, replyTo?, subject, body?, html?, template?, variables?, attachments? })` -> `{ status, mailId?, error? }`. Managed relay (the From address is platform-fixed; use `replyTo`). `template` (a stored mail-template slug) is mutually exclusive with `html`; `variables` fills template placeholders; `attachments` is `[{ filename, contentType, contentBase64 }]`. Caps: 50 recipients, 10 attachments, 10 MiB total. A delivery failure returns a 4xx/5xx `status` (does not throw).
- `mail.smtp({ to, subject, body })` -> `{ status, data: { messageId, error } }`. Sends over configured SMTP.

**LLM**
- `llm.send(prompt, opts?)` -> `Promise<string>`. `opts` = `{ model?, temperature?, maxTokens? }`.
- `llm.sendStructured(prompt, schema, opts?)` -> `Promise<object>`. `schema` is a field map (value = a description string, or `{ type, description }`; `type` is `string`|`number`|`integer`|`boolean`, with a `[]` suffix for arrays). Both throw on failure.

**Other functions**
- `functions.invoke(key, params?)` -> `{ status, data }`. Calls another workspace function by its `key`. Nesting is capped at 3 levels.

**Tickets**
- `tickets.create({ template, title, fields?, requester?, priority?, statusId? })` -> `{ id, created }`. `template` is a ticket-template key or `ttp_` id; `priority` is `LOW`|`NORMAL`|`HIGH`|`URGENT` (default `NORMAL`). Idempotent per action execution.

**Transcript helper**
- `conversation.normalize(ctx)` -> `string` (synchronous, no `await`). Formats `ctx.transcript` into `Agent:` / `User:` lines.

**Encoding**
- `btoa(s)` / `atob(s)` / `base64Encode(s)` / `base64Decode(s)` - UTF-8-aware base64.

**Workspace integrations (dynamic, per workspace)**
- `integrations.<provider>.<fn>(connectionPublicId, params)` -> `Promise`. Integration presets your workspace has configured; the editor autocomplete lists them. Connectionless presets are `integrations.standalone.<fn>(params)`.
- `mcp.<server>.<tool>(params)` -> `Promise`. Calls a tool on an MCP server registered in the workspace.

**Not implemented (throw if called):** `sms.send`, `urlshort.shorten`.

## Sandbox limits
- Wall-clock timeout: 30 s (overridable per function invoke).
- Memory 64 MiB; stack 1 MiB.
- Outbound calls per run: fetch + http 50, integrations 50, mcp 50, `functions.invoke` 10, mail 10, `tickets.create` 5, llm 5.
- HTTP response body cap 10 MiB.
- Private-network addresses are blocked.
- No DOM, no Node APIs, no `require`, no npm.

## ctx and return shape differ by script type
- **Function**: `ctx` is param-centric (`ctx.params`, plus optional `connection`, `conversation_id`, `agent_id`, `mail`). It MUST return `{ status: <int 100-599>, data: <any> }`. See fertigai-functions.
- **Action**: `ctx` is conversation-centric (`conversation`, `transcript`, `summary`, `classification`, `variables`, `workspace`, `params`). Its return value is stored verbatim, with no required shape. See fertigai-actions.
