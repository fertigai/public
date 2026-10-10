# Checking SIP trunk health (fertigai_sip_trunks_health)

## Overview
SIP trunk health is **read-only** here: you can check whether one trunk, or every trunk in the workspace, is registered and able to carry calls. You cannot create, edit, enable or disable trunks through this API; configure them in the portal.

Shared conventions (ids, errors, permissions) are in the skill index (SKILL.md). The tool also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_sip_trunks_health` | `id?` |

- With `id` (a SIP trunk's `st_` id), the tool returns that trunk's verdict object.
- Without `id`, it returns an array of verdicts for the workspace's trunks (the first 100; there is no `cursor`).

Each verdict has these fields:

| Field | Meaning |
|---|---|
| `sip_trunk_id` | The trunk's `st_` id. |
| `status` | The overall verdict: `HEALTHY`, `UNHEALTHY`, `DISABLED` or `UNKNOWN`. |
| `registration_status` | The registration state the verdict is based on (see below). |
| `registration_status_at` | When the registration state was last reported, as an RFC 3339 timestamp. Empty when it was never reported. |

## Status vocabulary
A disabled trunk reports `DISABLED` whatever its registration state and carries no calls until it is enabled again. For an enabled trunk, `status` follows `registration_status`:

| `registration_status` | `status` | Meaning |
|---|---|---|
| `REGISTERED` | `HEALTHY` | The registration succeeded and has not expired; the trunk can carry calls. |
| `STALE` | `UNHEALTHY` | The last successful registration expired without being renewed. |
| `FAILED` | `UNHEALTHY` | The last registration attempt failed (this also covers a PBX that registers to the workspace). |
| `UNREGISTERED` | `UNHEALTHY` | No live registration is reported. |
| `STATIC` | `UNKNOWN` | The trunk authenticates by IP address and never registers, so registration says nothing about its health. This is not a fault. |
| `UNKNOWN` | `UNKNOWN` | No registration state has been reported yet. |

On `UNHEALTHY`, calls over the trunk are likely to fail: check its registration settings in the portal and on the carrier or PBX side.

## Example
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

Every trunk in the workspace:
```
fertigai_sip_trunks_health {}
```
```json
[
  { "sip_trunk_id": "st_...", "status": "HEALTHY", "registration_status": "REGISTERED", "registration_status_at": "2026-10-10T08:14:40+00:00" },
  { "sip_trunk_id": "st_...", "status": "UNKNOWN", "registration_status": "STATIC", "registration_status_at": "" }
]
```

## Common mistakes
- Expecting the tool to create a health URL: it only reads health; the credential-free health URL for uptime monitors is created on the trunk in the portal.
- Inventing a trunk id. Call the tool without `id` first and take an `st_` id from its result.

Requires the Telephony-View permission.
