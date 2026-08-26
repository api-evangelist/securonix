---
name: securonix-manage-detection-policies
description: Create, fetch, page, enable/disable and bulk-delete Securonix Unified Defense SIEM detection policies through the Policy Management API, including the JWT exchange the API requires.
api: Securonix Policy Management API
spec: openapi/securonix-policy-management-api.json
operations:
  - getClientId
  - createPolicies
  - updatePolicy
  - enableDisablePolicies
  - bulkDeletePolicies
  - getPolicy
  - getPolicies
  - getLimit
generated: '2026-08-26'
method: generated
source: openapi/securonix-policy-management-api.json, https://documentation.securonix.com/r/content/authentication.htm
---

# Manage Securonix detection policies

Detection policies are the unit Securonix evaluates activity against. This flow covers the whole
lifecycle over the Policy Management API (OpenAPI 3.1.0, base
`https://policymanagement.api.securonix.net`).

## 1. Authenticate — two steps, not one

Policy Management does **not** accept the WS token directly. It requires a JWT minted from one.

1. `GET https://{BASE_URL}/ws/token/generate` with request headers `username`, `password`,
   `validity` (days). `{BASE_URL}` is `https://<hostname or IP address>/Snypr`. The body is a bare
   UUID — that is the WS token.
2. `POST https://{REGION_BASE_URL}/shared/snypr-service-gateway/api/v2/oauth/token` with header
   `wstoken: <the UUID>` (and optionally `x-transaction-id: <UUID>` for tracing). The response
   carries `accessToken`, `accessTokenExp`, `refreshToken`, `refreshTokenExp`.
3. Send `Authorization: Bearer <accessToken>` on every Policy Management call.

Ask Securonix Support for your regional base URL — it is not published.

The calling account needs one of `ROLE_ADMIN`, `ROLE_CONTENT_DEVELOPER`, `ROLE_READ_ONLY` for
reads. If calls start returning "Access Denied" right after a password change, sign in to the
Securonix UI once with the new password — that is a documented known issue, not a bad token.

## 2. Get the client id

`getClientId` — `GET /v1/policies/client-id`. Several operations take a client-id header; this
operation derives it from the JWT so you do not have to be told it out of band.

## 3. Read before you write

- `getPolicies` — `GET /v1/policies/all`. Paginated, sortable, supports column selection. Use
  `offset` and the page-size parameter; the response envelope reports the total.
- `getPolicy` — `GET /v1/policies/fetch`. Fetch one policy by name.
- `getLimit` — `GET /v1/policies/limit`. Check the tenant's policy limit **before** a bulk create,
  so you fail on a number rather than halfway through a batch.

## 4. Create and update

- `createPolicies` — `POST /v1/policies/create`.
- `updatePolicy` — `PUT /v1/policies/update`.

A `Policy` carries a `LogSource`, a `detection` block, an `AnalyticalType` (with threshold deltas —
`AmountBasedThreshold`, `FrequencyBasedThreshold`, `ParentChildRelation`), zero or more
`AdditionalEventAnalytics` blocks (active list, lookup, watchlist, TPI check, email-sent-to-self,
match-string config) and `ViolationAction` entries, one of which can be `AddToWatchlist` with a
`RemovalPeriod`.

**There is no idempotency mechanism.** No `Idempotency-Key` header exists on this API. A retried
create can produce a duplicate policy. Read back with `getPolicy` before retrying rather than
firing the same create twice.

## 5. Turn a policy off before deleting it

- `enableDisablePolicies` — `PATCH /v1/policies/status`. This is the reversible control: call it
  again with the opposite state to undo.
- `bulkDeletePolicies` — `DELETE /v1/policies/erase`. **Not reversible.** No restore, undelete or
  trash endpoint is published, and no retention window is stated. Disable first, confirm nothing
  broke, delete later.

The bulk delete returns a `PolicyDeletionResult` per item with a `deletionTrackerId` — read every
row, because a 200 on the batch does not mean every policy in it was deleted.

## 6. Errors you will actually see

| Status | Meaning | What to do |
|---|---|---|
| 400 | Invalid input data. Cannot process policy. | Fix the body against the schema. |
| 401 | Invalid JWT token. | Re-mint the JWT from a fresh WS token. |
| 403 | Valid credentials, insufficient permissions. | Grant the role above. |
| 404 | One or more policies not found — detail in the response. | Read the per-item results. |
| 422 | Parent id not found in Auth token. | Re-mint the JWT for the right tenant. |
| 500 | Error fetching/processing policies. | Retry with backoff. |

All error bodies are `application/json` against a vendor schema — this API does not use RFC 9457
`application/problem+json`. See `errors/securonix-problem-types.yml`.
