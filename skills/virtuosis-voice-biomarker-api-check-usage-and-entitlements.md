---
generated: '2026-08-18'
method: generated
name: Check credit balance and API health before uploading
description: >-
  Read the organisation's trial, included and purchased credit balances and confirm the API is
  reachable, so an upload is not attempted against an exhausted or unprovisioned account.
api: openapi/virtuosis-voice-biomarker-api-openapi.yml
operations: [getRecordingsUsage, ping]
source: >-
  operationIds verified verbatim in openapi/virtuosis-voice-biomarker-api-openapi.yml; field semantics
  from https://docs.virtuosis.ai/api-reference/usage/get-recordings-usage.
---

# Check credit balance and API health before uploading

Every `uploadRecording` call spends a credit, and there is no way to un-spend one. Run this before a
batch.

## Auth
- `Authorization: Bearer <API_KEY>`. `getRecordingsUsage` is organisation-scoped.
- `ping` is the one operation that needs no credential.

## Steps
1. **Health check** — `ping` (`GET /ping`). Returns `{"message": "pong"}`. Unauthenticated, so it
   distinguishes "the API is down" from "my key is bad". This is the only availability signal
   Virtuosis offers; there is no status page.
2. **Read usage** — `getRecordingsUsage` (`GET /usage/recordings`). Returns:
   - `trial_remaining` / `trial_granted` — one-time trial credits, **consumed first**.
   - `included_remaining` / `included_granted` — monthly allotment, resets each billing cycle.
   - `purchased_remaining` — credit packs, roll over until used or refunded.
   - `tier` — the organisation's subscription tier (nullable).
3. **Gate the batch** — spend order is trial → included → purchased. Sum all three remaining buckets
   and compare against the number of recordings queued. If the total is short, stop before uploading
   rather than discovering it mid-batch as a 403.
4. **Compute burn** — `granted - remaining` for the trial and included buckets gives consumption this
   cycle; there is no usage-history endpoint, so snapshot this yourself if you need a trend.

## Errors
- `401` — missing or invalid bearer token.
- `403` — access suspended, or API billing not provisioned for the organisation
  (`ApiNotProvisioned`). Neither is retryable; both need Virtuosis to act.
- `500` — may arrive without a message.

## Notes
- Credit exhaustion surfaces on upload as **403 `UploadLimitExceeded`**, not 429. Do not conflate the
  two: 429 is a rate limit (back off and retry), 403 is a wallet problem (stop and top up).
- `tier` is informational. The public site publishes no pricing page and no named tiers, so do not
  branch business logic on its value without confirming it with Virtuosis.
