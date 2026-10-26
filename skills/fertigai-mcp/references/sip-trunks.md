# SIP trunks: setup, test calls and health (fertigai_sip_trunks_*)

## Overview
A SIP trunk (`st_...`) connects the workspace to a carrier or a PBX; phone numbers route over it (phone numbers, call flows and the rest of telephony routing are in `references/telephony.md`). Over MCP you can list, create, replace and delete trunks, start from a provider template, place a test call over a trunk, and check whether a trunk is registered (or, for trunks that authenticate by IP address, reachable) and able to carry calls.

The trunk password is **write-only**: you send it on create or update, and no tool ever returns it. The credential-free health URL for uptime monitors is created on the trunk in the portal; there is no tool for it.

Shared conventions (ids, errors, permissions) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_sip_trunks_list` | `cursor?`, `page_size?` |
| `fertigai_sip_trunks_create` | the trunk fields (below) |
| `fertigai_sip_trunks_update` | `id`, then every trunk field (full replace) |
| `fertigai_sip_trunks_delete` | `id` |
| `fertigai_sip_trunks_predefined` | none |
| `fertigai_sip_trunks_config` | none |
| `fertigai_sip_trunks_test_call_start` | `id`, `to_number`, `agent_id`, `caller_id?` |
| `fertigai_sip_trunks_test_call_get` | `id`, `call_id` |
| `fertigai_sip_trunks_test_call_end` | `id`, `call_id` |
| `fertigai_sip_trunks_health` | `id` |

- `fertigai_sip_trunks_list` returns the workspace's trunks with every setting and the last reported registration state (paginated, see SKILL.md). It is also the only way to read one trunk: there is no `get` tool. It never contains passwords.
- `fertigai_sip_trunks_delete` is permanent.

## Trunk fields
Required: `name`, `host` (the carrier's SIP host name or IP address), `port` (usually 5060 for UDP and TCP, 5061 for TLS) and `transport` (`UDP`, `TCP` or `TLS`).

Optional (an omitted field takes the default shown):

| Field | Meaning |
|---|---|
| `auth_mode` | `REGISTER` (default: the platform registers at the carrier with `username` and `password`), `IP` (the carrier is allowed by source address, no credentials) or `REGISTRAR` (your PBX registers to the platform with `username` and `password`). |
| `username` | SIP username. Required for `REGISTRAR`. |
| `password` | SIP password, write-only. Required for `REGISTRAR`. |
| `auth_username` | Digest username when the carrier authenticates under another name than `username`; empty = `username`. |
| `contact_user` | Contact user part to register with; empty = `username`. |
| `allowed_ips` | Source IP addresses or CIDR ranges allowed to send calls. At least one is required for `IP`; for `REGISTER` it is an optional restriction. |
| `codecs` | Ordered list of `PCMA`, `PCMU`, `G.722`, `G.729`, `G.726`, `Opus`; `G.711` expands to `PCMA`, `PCMU`. The portal's default is `["G.722", "PCMA", "PCMU"]`. |
| `registration_expiry` | REGISTER expiry in seconds (default 300). |
| `outbound_proxy` | `host[:port]`; empty = none. |
| `custom_headers` | Extra SIP headers as a JSON object encoded in a string, for example `"{\"X-Account\":\"42\"}"`; empty = none. Returned as written, so never put a secret in a header. |
| `media_encryption` | `no` (default), `optional` or `force`. |
| `caller_id_pref` / `called_id_pref` | The default caller ID and called ID for the trunk's numbers: `AUTO` (empty), `DIALED_DID` or `ORIGINAL_CALLER`. |
| `identity_header` | The header that presents the caller, for example `P_ASSERTED_IDENTITY`; empty = default. |
| `number_format` | The digit shape in that header, for example `E164`; empty = default. |
| `country_calling_code` | The carrier's country calling code, digits only (for example `49`); empty turns off national-number conversion. |
| `national_prefix` | The carrier's trunk prefix (for example `0`). |
| `caller_source_signal` | How the carrier states where a call came from, for example `ALERT_INFO_URN`; empty = read the caller's number shape. |
| `rewrite_contact` | Send in-dialog requests to the source address (carrier behind NAT). |
| `signal_to_call_source` | Send the legs of an inbound call to the address it came from (PBX behind NAT). UDP only. |
| `tls_verify_chain_only` | Verify the carrier's TLS certificate chain without requiring the host name in the certificate. TLS only. |
| `contact_by_address` | Register with the bound address instead of the public host in Contact. |
| `contact_without_line` | Register without the `;line=` routing token in Contact. |
| `single_destination` | The trunk serves exactly one inbound number: unmatched inbound calls go to its single attached number. |
| `enabled` | In service. Absent: enabled on create, unchanged on update. |
| `transfer_method` | `DIAL` (default), `REFER`, `DISABLED`, `DIAL_RELAY` or `REFER_ATTENDED`. Absent: `DIAL` on create, unchanged on update. |

## Creating a trunk
Start from a provider template or a manual config.

- **Template**: `fertigai_sip_trunks_predefined` lists the provider templates. Each has an `id` (for example `easybell`), `provider`, `description`, `defaults` (host, port, transport, codecs, registration_expiry, media_encryption, username, password, outbound_proxy) and the `fields` the user fills in (`key`, `label`, `placeholder`, `type`, `required`). Strings in `defaults` may hold `{key}` placeholders such as `{sip_user_id}`: ask the user for each field, substitute the values yourself, add a `name`, and pass the result to `fertigai_sip_trunks_create`. A placeholder for an optional field that the user leaves empty becomes an empty string.
- **IP trunk** (`auth_mode: "IP"`): the carrier must send calls to the platform and allow its source ranges. `fertigai_sip_trunks_config` returns `sip_ingress_host` (where the carrier sends calls) and `sip_egress_cidrs` (the ranges to allowlist at the carrier). Both are empty when none are configured.

```
fertigai_sip_trunks_create {
  "name": "Carrier main",
  "host": "sip.example-carrier.com", "port": 5061, "transport": "TLS",
  "auth_mode": "REGISTER", "username": "1234567", "password": "<sip-password>",
  "codecs": ["G.722", "PCMA", "PCMU"], "media_encryption": "optional"
}
```
The response is the new trunk without its password.

## Updating a trunk
`fertigai_sip_trunks_update` is a **full replace**. Every field you omit resets to its default (empty, `false`, or 300 for `registration_expiry`), with three exceptions:
- an empty `password` keeps the stored password, except when the update switches the trunk into `REGISTRAR`, which needs a new one;
- an absent `enabled` keeps the trunk's state;
- an absent `transfer_method` keeps the trunk's transfer method.

The safe pattern:
1. Find the trunk in `fertigai_sip_trunks_list`.
2. Copy all of its settings, change only what you need, and leave `password` empty unless you are changing it.
3. Send every field back with `id`.

To take a trunk out of service, send it back unchanged with `"enabled": false`.

## Test calls
A test call is a **real call**: it rings a real phone, it is billed, and the agent's active branch answers it. Place one only when the user asks for it, and confirm the number first.

1. `fertigai_sip_trunks_test_call_start { "id": "st_...", "to_number": "+4930...", "agent_id": "agt_..." }`
   - `to_number`: an E.164 number or an internal extension of 1 to 6 digits.
   - `agent_id`: the agent whose active branch answers the call.
   - `caller_id` (optional): the identity to present as the calling party, at most 64 letters, digits or `+-._~`; empty presents none.
   - Returns `call_public_id` (`call_...`).
2. Poll `fertigai_sip_trunks_test_call_get { "id": "st_...", "call_id": "call_..." }` every few seconds. `status` is `DIALLING` until the call record exists, then `INITIATED`, `RINGING`, `IN_PROGRESS`, `TRANSFERRING`, `COMPLETED`, `FAILED`, `NO_ANSWER` or `BUSY`, with the call record in `call`.
3. Hang up with `fertigai_sip_trunks_test_call_end` (same args) if the call is still running. It is best effort and succeeds for a call that has already ended.

Rules:
- A start is **not idempotent**: every call places another phone call. Never retry a start after a timeout or a connection error. Check first whether the call was placed: look for it in `fertigai_conversations_list` (`agent_id`, newest first), and ask the user before you start another.
- At most 6 test calls per workspace per minute; more return `429`.
- `409` when the trunk is disabled or the agent has no active branch; `404` when the trunk or agent is not in the workspace; `400` for an unusable `to_number` or `caller_id`; `503` when telephony is temporarily unavailable.

## Health
`fertigai_sip_trunks_health` checks exactly one trunk: pass its `st_` id and it returns that trunk's verdict object. To check several trunks, call it once per trunk. Checking a trunk that authenticates by IP address sends it a live reachability check, so the call can take a few seconds. A result is reused for about 30 seconds.

Each verdict has these fields:

| Field | Meaning |
|---|---|
| `sip_trunk_id` | The trunk's `st_` id. |
| `status` | The overall verdict: `HEALTHY`, `UNHEALTHY`, `DISABLED` or `UNKNOWN`. |
| `registration_status` | The registration state the verdict is based on (see below). |
| `registration_status_at` | When the registration state was last reported, as an RFC 3339 timestamp. Empty when it was never reported. |
| `reachable` | IP-address trunks only: whether the carrier answered the reachability check. Absent when the check could not run. |
| `probe_sip_status` | IP-address trunks only: the SIP status code the carrier answered with (for example `200` or `403`). Absent when it did not answer. |
| `probe_rtt_ms` | IP-address trunks only: the round-trip time of the check in milliseconds. Absent when the carrier did not answer. |
| `probed_at` | IP-address trunks only: when the check ran, as an RFC 3339 timestamp. |

### Status vocabulary
A disabled trunk reports `DISABLED` whatever its registration state and carries no calls until it is enabled again. For an enabled trunk, `status` follows `registration_status`:

| `registration_status` | `status` | Meaning |
|---|---|---|
| `REGISTERED` | `HEALTHY` | The registration succeeded and has not expired; the trunk can carry calls. |
| `STALE` | `UNHEALTHY` | The last successful registration expired without being renewed. |
| `FAILED` | `UNHEALTHY` | The last registration attempt failed (this also covers a PBX that registers to the workspace). |
| `UNREGISTERED` | `UNHEALTHY` | No live registration is reported. |
| `STATIC` | from the reachability check | The trunk authenticates by IP address and never registers; its verdict comes from the reachability check (see below). |
| `UNKNOWN` | `UNKNOWN` | No registration state has been reported yet. |

A trunk that authenticates by IP address (`STATIC`) is checked by sending its carrier a SIP OPTIONS request:

| Check result | `status` |
|---|---|
| The carrier answered with any status except `401`, `403` or `407` | `HEALTHY` |
| The carrier answered `401`, `403` or `407` (it does not accept the workspace's traffic) | `UNHEALTHY` |
| The carrier did not answer | `UNHEALTHY` |
| The check could not run | `UNKNOWN` (the check fields are absent) |

On `UNHEALTHY`, calls over the trunk are likely to fail: check the trunk's registration or address settings and the carrier or PBX side.

### Example
One trunk:
```
fertigai_sip_trunks_health { "id": "st_..." }
```
```json
{
  "sip_trunk_id": "st_...",
  "status": "UNHEALTHY",
  "registration_status": "FAILED",
  "registration_status_at": "2026-10-10T08:15:02+00:00"
}
```

An IP-address trunk:
```
fertigai_sip_trunks_health { "id": "st_..." }
```
```json
{
  "sip_trunk_id": "st_...",
  "status": "HEALTHY",
  "registration_status": "STATIC",
  "registration_status_at": "",
  "reachable": true,
  "probe_sip_status": 200,
  "probe_rtt_ms": 42,
  "probed_at": "2026-10-10T08:14:52+00:00"
}
```

## Common mistakes
- Sending only the changed fields to `fertigai_sip_trunks_update`. It replaces the whole trunk: every omitted field resets. Copy the trunk from `fertigai_sip_trunks_list` and send it all back.
- Sending a placeholder (`{sip_password}`) or the wrong password on update when you meant to keep it. Leave `password` empty to keep the stored one.
- Putting a token or password in `custom_headers`. Headers are returned as written in every list.
- Retrying a test call start after a timeout. It may already be ringing; check `fertigai_conversations_list` first.
- Placing test calls in a loop. Each one is billed, and the workspace limit is 6 per minute.
- Expecting a tool that creates a health URL. Health URLs for uptime monitors are created in the portal.
- Inventing a trunk id. Call `fertigai_sip_trunks_list` first and take an `st_` id from its result.
- Calling the health tool without `id`: it checks one trunk per call and refuses a call without one.

Listing, templates, config and health need the Telephony-View permission. Create, update, delete and test calls need Telephony-Manage.
