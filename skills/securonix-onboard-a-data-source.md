---
name: securonix-onboard-a-data-source
description: Onboard an API-based or Syslog-based data source into Securonix Unified Defense SIEM — discover tenants, RINs and parsers, validate syslog details, configure the source, then watch the import jobs.
api: Securonix Datasource Onboarding API
spec: openapi/securonix-datasource-onboarding-api.json
operations:
  - healthCheckUsingGET
  - getAllTenantsByUsernameUsingGET
  - getAllRINsByTenantNameUsingGET
  - getParsersFromCRPUsingGET
  - getCRPContentUsingGET
  - createSyslogSourceUsingPOST
  - createSyslogFilterUsingPOST
  - validateSyslogSourceAndFilterUsingPOST
  - configureSyslogDataSourceUsingPOST
  - configureDataSourceUsingPOST
  - getResourceGroupByNameUsingGET
  - getResourceGroupByIdUsingGET
  - getJobImportSummaryUsingGET
  - getAllActivityImportJobDetailsUsingGET
  - getRunningJobsForRgUsingGET
  - jobReschedulerUsingPOST
  - stopARunningJobUsingPUT
generated: '2026-08-26'
method: generated
source: openapi/securonix-datasource-onboarding-api.json
---

# Onboard a data source into Securonix

The Datasource Onboarding API (OpenAPI 3.0.3, 22 operations, `/ingestion/v1/*`) is how a data
source becomes a **resource group** — the ingestion unit everything else in the platform is scoped
by.

## Authenticate

This service takes the WS token directly, in a header named **`wstoken`** — not `token`, and not a
bearer JWT. Get it from `GET https://{BASE_URL}/ws/token/generate` with `username`, `password`,
`validity` headers.

Confirm the service is up first: `healthCheckUsingGET` — `GET /ingestion/v1/health`.

## 1. Discover the context you have to name

You cannot configure anything until you know the tenant, the ingester and the parser.

- `getAllTenantsByUsernameUsingGET` — `GET /ingestion/v1/fetchalltenants`. Returns the tenants the
  calling account can see. On an MSSP deployment this is the step that decides whose data you are
  about to touch.
- `getAllRINsByTenantNameUsingGET` — `GET /ingestion/v1/fetchrins`. Returns the Remote Ingestion
  Nodes for that tenant, keyed by `ingesterId`. A syslog source must be bound to one.
- `getParsersFromCRPUsingGET` — `GET /ingestion/v1/parsersdetails`. Find a parser by vendor name or
  resource type. Securonix ships several hundred connector parsers; look before you write one.
- `getCRPContentUsingGET` — `GET /ingestion/v1/readcrpcontent`. Read the Custom Resource Parser
  content by `crpId` so you can see what it will actually do to your events.

## 2a. Syslog path

1. `createSyslogSourceUsingPOST` — `POST /ingestion/v1/createsyslogsource`. Needs `ingesterId` and
   `tenantId`.
2. `createSyslogFilterUsingPOST` — `POST /ingestion/v1/createsyslogfilter`. Needs `ingesterId` and
   the `sourceId` from step 1.
3. `validateSyslogSourceAndFilterUsingPOST` — `POST /ingestion/v1/validatesyslogdatasource`.
   **Run this.** It is the only validate-before-commit operation Securonix publishes anywhere on
   the platform — there is no general dry-run mode — so it is the one chance to find out you have
   the wrong filter before events start landing in the wrong resource group.
4. `configureSyslogDataSourceUsingPOST` — `POST /ingestion/v1/configuresyslogdatasource`. Takes
   `crpId`, `tenantId` and a `SyslogDetailsPayload` referencing `sourcesId`, `filtersId`,
   `ingesterId`.

Inspect existing wiring with `getSyslogSourcesByIngesterIdUsingGET`
(`GET /ingestion/v1/getsyslogsourcebyingesterid`) and `getSyslogFiltersByIngesterIdUsingGET`
(`GET /ingestion/v1/getsyslogfiltersbyingesterid`).

## 2b. API-based path

`configureDataSourceUsingPOST` — `POST /ingestion/v1/configuredatasource`, with `crpId` and
`tenantId`. This is the out-of-the-box API connector path (Okta, Jira, Meraki, Umbrella, Dropbox,
and the rest of the documented connector catalog).

## 3. Confirm the resource group exists

- `getResourceGroupByNameUsingGET` — `GET /ingestion/v1/getresourcegroupbyname`
- `getResourceGroupByIdUsingGET` — `GET /ingestion/v1/getresourcegroupbyid`
- `getResourceGroupDetailsByTenantIdUsingGET` — `GET /ingestion/v1/listresourcegroupconfigbytenantid`

The `resourceGroupId` you get back is the key you will use in device monitoring, in policy
resource bindings and in Spotter queries (`resourcegroupname`).

## 4. Watch the import

- `getJobImportSummaryUsingGET` — `GET /ingestion/v1/getjobimportsummary`
- `getAllActivityImportJobDetailsUsingGET` — `GET /ingestion/v1/listactivityimportjobtriggerdetails`
- `getRunningJobsForRgUsingGET` — `GET /ingestion/v1/fetchrunningjobs`
- `jobReschedulerUsingPOST` — `POST /ingestion/v1/jobrescheduler` (needs `crpId`, `resourceGroupId`)
- `stopARunningJobUsingPUT` — `PUT /ingestion/v1/stoprunningjob`

`stopARunningJobUsingPUT` is the reversal for a scheduling mistake; there is no published undo for
`configuredatasource` itself, so validate first.

## Cautions

- **No idempotency.** Retrying a create can produce a second syslog source or filter. Read back
  with the `getsyslogsourcebyingesterid` / `getsyslogfiltersbyingesterid` operations rather than
  re-POSTing on an ambiguous timeout.
- Errors: 401 Unauthorized, 403 Forbidden, 404 Not Found — `application/json`, vendor shape, no
  RFC 9457 problem details.
