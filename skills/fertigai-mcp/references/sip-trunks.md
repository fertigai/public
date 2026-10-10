# Checking SIP trunk health (fertigai_sip_trunks_health)

## Overview
SIP trunk health is **read-only** here: you can check whether one trunk, or every trunk in the workspace, is registered and able to carry calls. You cannot create, edit, enable or disable trunks through this API; configure them in the portal.

Shared conventions (ids, errors, permissions) are in the skill index (SKILL.md). The tool also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_sip_trunks_health` | `id?` |

- With `id` (a SIP trunk's `st_` id), the tool returns that trunk's verdict object.
- Without `id`, it returns a JSON array of verdict objects, one per trunk in the workspace (the first 100 trunks). The tool takes no `cursor`: a workspace with more than 100 trunks reports only the first 100. A trunk deleted while the check runs is left out of the array.

Each verdict has these fields:

| Field | Meaning |
|---|---|
| `sip_trunk_id` | The trunk's `st_` id. |
| `status` | The overall verdict: `HEALTHY`, `UNHEALTHY`, `DISABLED` or `UNKNOWN`. |
| `registration_status` | The registration state the verdict is based on (see below). |
| `registration_status_at` | When the registration state was last reported, as an RFC 3339 timestamp. Empty when it was never reported. |

## Status vocabulary
`status` is derived from whether the trunk is enabled and from `registration_status`:

| `status` | When | What it means for the user |
|---|---|---|
| `DISABLED` | The trunk is switched off in the portal, whatever its registration state. | The trunk carries no calls until it is enabled again. |
| `HEALTHY` | `registration_status` is `REGISTERED`. | The registration is current; the trunk can carry calls. |
| `UNHEALTHY` | `registration_status` is `FAILED`, `STALE` or `UNREGISTERED`. | The trunk is not registered; calls over it are likely to fail. Check the trunk's registration settings in the portal and on the carrier or PBX side. |
| `UNKNOWN` | `registration_status` is `STATIC` or `UNKNOWN`. | Registration says nothing about this trunk's health (see below). |

`registration_status` values:

| Value | Meaning |
|---|---|
| `REGISTERED` | The registration succeeded and has not yet expired. |
| `STALE` | The last registration was successful but has expired without being renewed. |
| `FAILED` | The last registration attempt was refused or got no answer. |
| `UNREGISTERED` | The trunk is not registered, for example after the registration was removed. |
| `STATIC` | The trunk authenticates by IP address and never registers. |
| `UNKNOWN` | No registration state has been reported yet, for example for a trunk created moments ago. |

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
- Reading `UNKNOWN` for an IP-authenticated trunk (`registration_status: "STATIC"`) as an error. Such a trunk never registers, so registration cannot tell whether it is healthy; it is not a fault.
- Treating `STALE` as healthy because the trunk once registered. The registration has expired, so the verdict is `UNHEALTHY`.
- Expecting the tool to create or revoke a health URL for an uptime monitor. It only reads verdicts; create health URLs on the SIP trunk in the portal.
- Inventing a trunk id. Call the tool without `id` first and take an `st_` id from its result.

Requires the Telephony-View permission.
