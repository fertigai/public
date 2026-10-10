# Listing SIP trunks and checking their health (fertigai_sip_trunks_list, fertigai_sip_trunks_health)

## Overview
SIP trunks are **read-only** here: you can list the workspace's trunks and check whether one trunk is registered (or, for trunks that authenticate by IP address, reachable) and able to carry calls. You cannot create, edit, enable or disable trunks through this API; configure them in the portal.

Shared conventions (ids, errors, permissions) are in the skill index (SKILL.md). The tool also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

## Tools
| Tool | Args |
|---|---|
| `fertigai_sip_trunks_list` | `cursor?`, `page_size?` |
| `fertigai_sip_trunks_health` | `id` |

- `fertigai_sip_trunks_list` returns the workspace's trunks with their settings and last reported registration state (paginated, see SKILL.md). Use it to find a trunk's `st_` id. It never contains passwords.
- `fertigai_sip_trunks_health` checks exactly one trunk: pass its `st_` id and it returns that trunk's verdict object. To check several trunks, call it once per trunk.
- Checking a trunk that authenticates by IP address sends it a live reachability check, so the call can take a few seconds. A result is reused for about 30 seconds.

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

## Status vocabulary
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

On `UNHEALTHY`, calls over the trunk are likely to fail: check its registration or address settings in the portal and on the carrier or PBX side.

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
- Expecting the tool to create a health URL: it only reads health; the credential-free health URL for uptime monitors is created on the trunk in the portal.
- Inventing a trunk id. Call `fertigai_sip_trunks_list` first and take an `st_` id from its result.
- Calling the health tool without `id`: it checks one trunk per call and refuses a call without one.

Both tools require the Telephony-View permission.
