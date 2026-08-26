---
name: securonix-measure-mitre-attack-coverage
description: Pull MITRE ATT&CK tactic, technique and sub-technique coverage for a Securonix tenant, in JSON or CSV, and join gaps back to the detection policies that would close them.
api: Securonix Policy Management API
spec: openapi/securonix-policy-management-api.json
operations:
  - getThreatCoverageMetrics
  - getTechniqueDetails
  - getPolicies
generated: '2026-08-26'
method: generated
source: openapi/securonix-policy-management-api.json
---

# Measure MITRE ATT&CK coverage

Securonix declares MITRE ATT&CK in the contract itself, not just in marketing copy — which means an
agent that already speaks ATT&CK technique IDs can read coverage with no bespoke mapping layer.

## Authenticate

Same two-step WS-token-then-JWT exchange as
[securonix-manage-detection-policies](securonix-manage-detection-policies.md). Send
`Authorization: Bearer <accessToken>`.

## 1. Pull the coverage matrix

`getThreatCoverageMetrics` — `GET /v1/policies/threat-coverage/metrics`.

- `subTenant` scopes the answer. Multi-tenant deployments must set it deliberately; the default is
  not the whole estate.
- Content negotiation is real here and worth using:
  - `Accept: application/json` (default) returns `ThreatCoverageMetricsResponse` — a
    `CoverageSummary`, `TacticCoverage[]` each holding `TechniqueCoverage[]` each holding
    `Subtechnique[]`, plus `PolicyDistributionByTacticSummary[]`.
  - `Accept: text/csv` returns a flat row per technique/sub-technique:
    `TacticID, TacticName, TechniqueID, TechniqueName, SubtechniqueID, SubtechniqueName, CoverageStatus`.

If you are producing a report or diffing coverage over time, ask for CSV — the flat shape is
already the join key, and you skip walking three levels of nesting.

## 2. Drill into an uncovered technique

`getTechniqueDetails` — `GET /v1/policies/threat-coverage/technique-details`.

Returns `TechniqueDetailsResponse` with `techniqueId`, `parentTechniqueId` (so sub-techniques point
at their parent), `AssociatedTactic[]`, `SubTechniqueDetails[]` and — the useful part —
`MappedPolicy[]` carrying `policyId` and `signatureId`.

## 3. Close the loop

`MappedPolicy.policyId` joins straight back to `getPolicies` (`GET /v1/policies/all`). So the whole
gap analysis is three calls:

1. coverage matrix → find `CoverageStatus` gaps by `TechniqueID`
2. technique details → find which policies (if any) map to that technique
3. policy list → read those policies, or author new ones with `createPolicies`

## Cautions

- Coverage status is computed from the policies deployed in that tenant. It is a statement about
  configuration, not about whether detections are firing correctly.
- Check `getLimit` before authoring a batch of new policies to close gaps.
- Nothing here writes, so nothing here needs reversing — this is a read-only flow.
