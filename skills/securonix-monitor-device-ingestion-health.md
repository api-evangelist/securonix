---
name: securonix-monitor-device-ingestion-health
description: List monitored devices and their ingestion health metadata from Securonix, with search, filtering, sorting and pagination — the check for "is this log source still sending?"
api: Securonix Device Monitoring API
spec: openapi/securonix-device-monitoring-api.json
operations:
  - listMonitoredDevices
generated: '2026-08-26'
method: generated
source: openapi/securonix-device-monitoring-api.json
---

# Monitor device ingestion health

One operation, and it answers the question that matters most in a SIEM: which log sources have gone
quiet.

## Call

`listMonitoredDevices` — `POST /devicealert/listdevices` on
`https://{tenantId}.securonix.net/Snypr/ws`.

Authentication is a WS token in a header named **`token`** (this API's scheme is `tokenAuth`, not
the `wstoken` the ingestion service uses and not a bearer JWT).

## Request

`DevicesRequest` fields, all optional:

| Field | Values | Purpose |
|---|---|---|
| `max` | integer | Page size |
| `offset` | integer | Pagination offset |
| `status` | `Trusted`, `Muted` | Which device state to list |
| `sort` | `device`, `rgname`, `functionality`, `device_state`, `tenantname` | Sort key |
| `order` | `asc`, `desc` | Sort direction |
| `searchUserText` | string | The search term |
| `searchAttrs` | `device`, `rgname`, `functionality`, `device_state`, `tenantname` | Which attribute to search |
| `operator` | `equals`, `contains` | Match mode |

The spec ships seven named request examples — pagination, exact device match, contains match,
by datasource, by state, by tenant (for MSSPs), by functionality. Read them before authoring a
body; they are the fastest way to get the `searchAttrs` / `operator` pairing right.

## Response

`ListDevicesResponse` holding `Device[]`. Each `Device` carries `rgId` (the resource group /
datasource it belongs to) and `tenantId`, plus its ingestion health metadata.

`rgId` joins straight to the Datasource Onboarding API — `getResourceGroupByIdUsingGET`
(`GET /ingestion/v1/getresourcegroupbyid`) — so a silent device can be traced back to the
configuration and the import jobs that feed it. See
[securonix-onboard-a-data-source](securonix-onboard-a-data-source.md).

## Errors

400, 401, 403, **429**, 500 — all `application/json` against `ErrorResponse`.

The 429 is worth knowing about: it is the **only** rate-limit response declared across all 242
documented Securonix operations, and it comes with no `Retry-After` and no published limit. If you
are polling device health on a schedule, back off exponentially on a 429; there is nothing in the
response to compute a better delay from.

## Reversibility

Read-only. Nothing to reverse.
