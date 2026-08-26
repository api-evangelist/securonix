---
name: securonix-run-a-spotter-search
description: Run an asynchronous Spotter (or SQL) search over the Securonix data lake — trigger, poll status, page results — including the two-layer escaping the API requires.
api: Securonix Spotter API
spec: null
operations:
  - 'POST /shared/snypr-service-gateway/spotter-api/spotter/api/v1/search/queries'
  - 'GET /shared/snypr-service-gateway/spotter-api/spotter/api/v1/search/queries/{queryId}/status'
  - 'GET /shared/snypr-service-gateway/spotter-api/spotter/api/v1/search/queries/{queryId}/results'
generated: '2026-08-26'
method: generated
source: https://documentation.securonix.com/r/content/spotter-api.htm
---

# Run a Spotter search

> Securonix publishes no OpenAPI for the Spotter API. The operations below are documented HTTP
> paths taken verbatim from the Spotter API reference page, not `operationId`s — there are none to
> cite. Everything here is grounded in that page.

## Authenticate and authorize

Bearer JWT (see the two-step exchange in
[securonix-manage-detection-policies](securonix-manage-detection-policies.md)). Send
`Authorization: Bearer <accessToken>`.

The account needs one of `ROLE_ADMIN`, `ROLE_CASE_ANALYST`, `ROLE_HUNTERS`, `ROLE_SECURITY_ANALYST`,
`ROLE_CASE_ADMIN`. `ROLE_CASE_ADMIN` is the least-privilege option of the five, but it still carries
UI privileges beyond the API. For genuine least privilege, build a role with only the four
SpotterServices privileges (execute asynchronously, check status, retrieve results, cancel
execution) plus the Administration privilege that allows generating a web-services token.

## 1. Trigger

`POST .../spotter-api/spotter/api/v1/search/queries`

```json
{
  "query": "index = violation",
  "fromTime": 1739332315000,
  "toTime": 1749028747000,
  "timeout": 3600,
  "limit": 1000,
  "sortBy": "policyname",
  "sortOrder": "desc"
}
```

Required: `query`, `fromTime`, `toTime` (epoch ms), `columns` (`["*"]` for all), `sortBy`,
`sortOrder`. Optional: `timeout` (default 3600s), `limit` (default 1000, **max 10,000**),
`countOnly` (default false), `dataLabels` (default false).

Set `"queryLanguageType": "SQL"` to send SQL instead — same endpoint, same workflow.

The response is `{"queryId": "..."}`. That is all you get; results are not inline.

## 2. Get the escaping right — this is where calls actually fail

Two layers apply, in this order:

1. **Spotter level.** Field values go in double quotes. Escape `"` and `\` inside them. `*` matches
   zero or more characters, `?` matches exactly one; escape them as `\*` and `\?` to use them
   literally.
2. **JSON level.** The query is then a JSON string, so backslashes multiply.

Published mapping: `\` → `\\`, `"` → `\\\"`, `?` → `\\?`, `*` → `\\*`.

Supported operators and commands: `=`, `!=`, `contains` / `not contains`, `starts with` /
`not starts with`, `ends with` / `not ends with`, `in` / `not in`, `null` / `not null`,
`between` / `not between`, `before`, `after`, `where`, `stats`, `table`, `top`, `rare`.

`STATS`, `TABLE` and `TOP` only work when `countOnly` is false, and must be used without `LIMIT`,
`MIN`, `MAX`, `EVAL` or `ORDERBY` modifiers.

## 3. Poll

`GET .../search/queries/{queryId}/status` → `{"queryId": ..., "status": ..., "message": null}`.

Documented statuses: `PARTIALLY_COMPLETED` (some results ready, more pending) and `COMPLETED`
(all results available). `message` is an optional note from the executor.

## 4. Page the results

`GET .../search/queries/{queryId}/results?offset={offset}&limit={limit}`

The envelope carries `total`, `offset`, `count`, `timezone` and `records[]`. Page with `offset`
until `offset + count >= total`. Calling this before the query finishes returns "still in
progress" rather than blocking.

## 5. Cancel

A cancel-execution endpoint exists — it is one of the four SpotterServices privileges — but its
path and method are **not published** on the reference page. Do not guess a URL. If you need to
cancel programmatically, ask Securonix Support for the endpoint.

## Indexes you can query

`activity`, `violation`, `riskscorehistory`, `asset`, `geolocation`, `lookup`, `tpi`, `users`,
`watchlist`, `whitelist` — one documented REST API category each.

## Cautions

- 10,000 records is a hard ceiling per query. Narrow the time window rather than expecting to page
  past it.
- No rate limit is published for this API and no `Retry-After` or `RateLimit-*` header is returned,
  so an agent cannot compute backoff from a throttled response. Pace yourself conservatively.
- This flow is read-only: nothing here needs reversing.
