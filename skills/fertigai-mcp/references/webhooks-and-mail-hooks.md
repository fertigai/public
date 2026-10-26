# Webhooks and inbound mail hooks (fertigai_webhooks_*, fertigai_webhook_*, fertigai_mail_hooks_*, fertigai_mail_hook_*)

## Overview
Two ways to connect the workspace to other systems by HTTP and email:
- **Webhooks** (`wh_...`): outbound. The workspace POSTs a signed JSON event to your `https` URL each time a subscribed event happens. Every send is recorded as a **delivery** (`whd_...`) with its status and retries.
- **Inbound mail hooks** (`mhk_...`): inbound. Each hook gives the workspace an email address; every email sent to it runs one of the workspace's functions. Each received email is recorded as a mail hook delivery (`mhdl_...`).

The **signing secret** of a webhook is never returned over MCP, also not by `fertigai_webhooks_create`. Read and rotate it in the portal's webhook settings. There are no MCP tools for it.

Shared conventions (ids, pagination, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`). List tools take `cursor?` and `page_size?` (1 to 100, default 20), except `fertigai_mail_hook_deliveries_list`, which is not paginated.

## Tools
**Webhooks**
| Tool | Args |
|---|---|
| `fertigai_webhook_event_types_list` | none |
| `fertigai_webhooks_list` | `cursor?`, `page_size?` |
| `fertigai_webhooks_get` | `id` |
| `fertigai_webhooks_create` | `name`, `url`, `events` |
| `fertigai_webhooks_update` | `id`, then any of `name`, `url`, `events`, `enabled` (absent fields stay) |
| `fertigai_webhooks_delete` | `id` |
| `fertigai_webhooks_test` | `id` |
| `fertigai_webhook_deliveries_list` | `id` (the webhook), `status?`, `cursor?`, `page_size?` |
| `fertigai_webhook_deliveries_get` | `id` (the webhook), `delivery_id` |

**Inbound mail hooks**
| Tool | Args |
|---|---|
| `fertigai_mail_hooks_list` | `search?` (name or local part), `cursor?`, `page_size?` |
| `fertigai_mail_hooks_get` | `id` |
| `fertigai_mail_hooks_create` | `function_public_id`, `match_type`, `local_part?`, `name?`, `enabled?` |
| `fertigai_mail_hooks_update` | `id`, `function_public_id`, `match_type`, `local_part?`, `name?`, `enabled?` (full replace) |
| `fertigai_mail_hooks_delete` | `id` |
| `fertigai_mail_hook_deliveries_list` | `id` (the mail hook) |

## Webhooks

### Fields
- `name`: 1 to 100 characters, trimmed.
- `url`: where deliveries are POSTed. See URL rules below.
- `events`: the event types the webhook subscribes to, at least one. Each must be a type from `fertigai_webhook_event_types_list`; duplicates are dropped and the order is kept.
- `enabled`: a new webhook is enabled. `false` pauses deliveries.
- `last_delivery_status` (read only, in list results): the status of the most recent delivery.

A workspace has at most 20 webhooks; creating one more returns `409`. If webhooks are turned off for the workspace, the webhook tools return `404`.

### Event types
Call `fertigai_webhook_event_types_list` for the current catalogue: each entry has a `type` and a `description`. Subscribe only to types it returns; an unknown type is a validation error. Today the catalogue holds `webhook.test`, the event that `fertigai_webhooks_test` sends. More event types are added to the catalogue over time.

### URL rules
The `url` must be:
- an absolute URL of at most 2048 characters;
- `https` (plain `http` is rejected);
- without a user name or password in it;
- with a host, and not a private, loopback or link-local IP address.

The URL is checked on create and update, and again before every delivery attempt, including the addresses its host name resolves to. An attempt to a blocked address fails with `last_error` starting `blocked: `.

### Updating
`fertigai_webhooks_update` changes only the fields you send. `events`, when sent and not empty, replaces the whole subscription list; an absent or empty `events` keeps the current one. Send `"enabled": false` to pause a webhook and `"enabled": true` to resume it.

### What a delivery looks like
Each delivery is an HTTP `POST` with `Content-Type: application/json` and this body:
```json
{
  "type": "webhook.test",
  "timestamp": "2026-10-26T09:00:00Z",
  "workspace_id": "<workspace-id>",
  "data": {}
}
```
`data` depends on the event type. The receiver has 10 seconds to answer. Any `2xx` answer counts as success; anything else, a timeout or a connection error counts as a failed attempt.

### Verifying signatures
Deliveries follow the [Standard Webhooks](https://www.standardwebhooks.com) specification, so its open-source libraries verify them. Each request carries three headers:
- `webhook-id`: the delivery id (`whd_...`). It stays the same across retries of one delivery, so use it to ignore duplicates.
- `webhook-timestamp`: the attempt time in Unix seconds.
- `webhook-signature`: one or more space-separated signatures, each `v1,<base64>`.

Each signature is an HMAC-SHA256 of the string `<webhook-id>.<webhook-timestamp>.<raw body>`. The key is the signing secret (`whsec_...`) with the `whsec_` prefix removed and the rest base64-decoded. To verify:
1. Read the raw request body as received. Do not parse and re-serialize the JSON first.
2. Compute the HMAC as above and base64-encode it.
3. Accept the request if any `v1,` signature in the header matches. Compare in constant time.
4. Reject timestamps far from the current time (for example more than 5 minutes) to stop replays.

After you rotate the secret in the portal, the old secret keeps signing for 24 hours next to the new one, so both signatures are in the header. Deploy the new secret to the receiver within that window.

Example in Node.js:
```javascript
import crypto from "node:crypto";

function verifyWebhook(secret, headers, rawBody) {
  const key = Buffer.from(secret.slice("whsec_".length), "base64");
  const signed = `${headers["webhook-id"]}.${headers["webhook-timestamp"]}.${rawBody}`;
  const expected = Buffer.from(
    crypto.createHmac("sha256", key).update(signed).digest("base64"),
  );
  const age = Math.abs(Date.now() / 1000 - Number(headers["webhook-timestamp"]));
  if (!(age <= 300)) return false;
  return headers["webhook-signature"].split(" ").some((entry) => {
    const [version, value] = entry.split(",");
    const given = Buffer.from(value ?? "");
    return version === "v1" && given.length === expected.length && crypto.timingSafeEqual(given, expected);
  });
}
```

The signing secret is shown and rotated only in the portal. Never put it in a function, an action or a chat with the assistant; store it on the receiving side.

### Test send
`fertigai_webhooks_test` sends one `webhook.test` event to that webhook now, also when the webhook is disabled or not subscribed to `webhook.test`. No other webhook receives it. The call waits for the first attempt and returns the recorded delivery with `status`, `last_status_code`, `last_error`, `payload` and `last_response_body`. A failed test delivery is then retried on the normal schedule. Test sends share a per-user rate limit with function and action tests; over the limit the call returns `429`.

### Delivery statuses and retries
| Status | Meaning |
|---|---|
| `PENDING` | Waiting for its next attempt (`next_attempt_at` is set). |
| `SUCCEEDED` | The receiver answered with a `2xx`. |
| `FAILED` | All 8 attempts failed. Final. |

A delivery is attempted at most 8 times. After a failed attempt the next one follows after 5 seconds, 5 minutes, 30 minutes, 2 hours, 5 hours, 10 hours and 10 hours. The 8th failure is final.

### Delivery log
- `fertigai_webhook_deliveries_list`: a webhook's deliveries, newest first. Each has `id`, `event_type`, `status`, `attempts`, `next_attempt_at`, `last_attempt_at`, `last_status_code`, `last_duration_ms`, `last_error`, `created_at` and `completed_at`. Filter with `status`: `PENDING`, `SUCCEEDED` or `FAILED` (any other value is a `400`).
- `fertigai_webhook_deliveries_get`: one delivery, plus the `payload` that was sent and `last_response_body`, the first 1 KiB of the receiver's last answer.

To debug a receiver: run `fertigai_webhooks_test`, then read `last_status_code`, `last_error` and `last_response_body`.

### Example
```
fertigai_webhooks_create {
  "name": "CRM sync",
  "url": "https://hooks.example.com/fertigai",
  "events": ["webhook.test"]
}
```
The result has the new webhook's `id`. Then run `fertigai_webhooks_test` with that `id`, and tell the user to copy the signing secret from the portal's webhook settings into the receiver.

## Inbound mail hooks

### Fields
- `function_public_id`: the function (`fn_...`) each matching email runs. It must be a function of the same workspace; otherwise the call returns `404`.
- `match_type` (case-insensitive):
  - `exact`: the address's local part (before `@`) equals `local_part`;
  - `prefix`: the local part starts with `local_part`;
  - `catchall`: every address of the workspace.
- `local_part`: trimmed and stored lowercased. Required for `exact` and `prefix`; cleared for `catchall`.
- `name`: a display name, trimmed.
- `enabled`: `false` stops matching emails from running the function. Defaults to `true`.

Two live hooks cannot have the same `local_part` and `match_type`; the second create returns `409`.

### The address
Each hook's result has a `receiving_address`: `<local_part>@<workspace handle>.<mail domain>`, or `*@<workspace handle>.<mail domain>` for a catchall. Read the address from `receiving_address` rather than building it yourself. List results also carry `function_name`.

### Updating is a full replace
`fertigai_mail_hooks_update` overwrites every field. A field you leave out is not kept: an omitted `name` or `local_part` becomes empty, and an omitted `enabled` becomes `true` (so a disabled hook is re-enabled). Always:
1. `fertigai_mail_hooks_get` the hook;
2. change the fields you need;
3. send all of `function_public_id`, `match_type`, `local_part`, `name` and `enabled` back.

```
fertigai_mail_hooks_update {
  "id": "mhk_...",
  "function_public_id": "fn_...",
  "match_type": "exact",
  "local_part": "invoices",
  "name": "Invoice intake",
  "enabled": false
}
```

### Delivery log
`fertigai_mail_hook_deliveries_list` returns the hook's 50 most recent deliveries, newest first, not paginated. Each has `id`, `status` (`delivered` or `failed`), `recipient`, `detail` (the error, on failure) and `created_at`.

### Example
```
fertigai_mail_hooks_create {
  "function_public_id": "fn_...",
  "match_type": "exact",
  "local_part": "invoices",
  "name": "Invoice intake"
}
```

## Permissions
- Webhooks: reads (list, get, event types, deliveries) need Integrations-View; create, update, delete and test need Integrations-Manage.
- Mail hooks: any workspace member can read them; create, update and delete need Integrations-Manage.

## Deleting
Deletes are permanent; there is no restore.
- `fertigai_webhooks_delete` removes the webhook and its whole delivery log. Pending retries stop.
- `fertigai_mail_hooks_delete` removes the hook; its address stops running the function, and later `get` calls return `404`.

## Common mistakes
- Asking for the signing secret over MCP: no tool returns it. Send the user to the portal's webhook settings.
- Using an `http`, private or loopback `url`: rejected. Use a public `https` endpoint.
- Verifying the signature over re-serialized JSON: sign-check the raw body bytes.
- Updating a mail hook with only the changed field: the other fields are cleared and `enabled` turns `true`. Get, change, send everything back.
- Subscribing to an event type not in `fertigai_webhook_event_types_list`.
