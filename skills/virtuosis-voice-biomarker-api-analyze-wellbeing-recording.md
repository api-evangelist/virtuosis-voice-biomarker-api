---
generated: '2026-08-18'
method: generated
name: Analyze a voice recording for wellbeing insights
description: >-
  Create an end-user account, upload a Base64-encoded speech recording for wellbeing analysis, and
  poll until stress, anxiety and depression insights are returned.
api: openapi/virtuosis-voice-biomarker-api-openapi.yml
operations: [createAccount, uploadRecording, getRecordingAnalysis]
source: >-
  operationIds verified verbatim in openapi/virtuosis-voice-biomarker-api-openapi.yml; runtime rules
  quoted from https://docs.virtuosis.ai/api-reference and
  https://docs.virtuosis.ai/api-reference/recordings/upload-recording.
---

# Analyze a voice recording for wellbeing insights

Base URL `https://api.virtuosis.ai/v1.3`. Three calls: create the account once per user, upload the
recording, then poll.

## Auth
- `Authorization: Bearer <API_KEY>` on every request. Organisation-scoped; see
  `authentication/virtuosis-voice-biomarker-api-authentication.yml`.
- Requests with a JSON body must also send `Content-Type: application/json`.

## Before you start
- `wellbeing` and `communication_coach` are self-service. `parkinsons` and `alzheimers` are released
  only after manual validation by Virtuosis and will return 403 until enabled.
- Uploads consume credits. Check `getRecordingsUsage` first if the balance may be low.

## Audio requirements (enforced server-side)
- Format WAV, MP3, MP4 or OGG, mono, ≥ 8,000 Hz, ≥ 32,000 bps.
- At least 30 seconds of free speech. If the clip is not part of a conversation, prompt the speaker
  with an open question such as "How's your day going?" or "Can you introduce yourself?".
- Maximum 50 MB. Over that, `uploadRecording` returns 413.
- The audio goes in the JSON body as a Base64 string. **Multipart upload is not accepted.**

## Steps
1. **Create the account** — `createAccount` (`POST /accounts`). Returns `account_id` (UUID) and
   `organisation_role`, which is always `speaker`. Create one account per end user and store the
   `account_id`; there is no list-accounts operation to recover it later.
2. **Upload the recording** — `uploadRecording` (`POST /recordings`) with `account_id`, `recorded_at`
   (ISO 8601, when the recording began), `analysis: ["wellbeing"]`, and `audio` (Base64). Optionally
   set `isolate_oldest_speaker: true` for multi-speaker clips. Returns `recording_id` plus the
   detected audio properties. **`analysis` is required and must list at least one type.**
3. **Poll for results** — `getRecordingAnalysis`
   (`GET /recordings/{recording_id}/analysis`). Poll every 15–30 seconds. Never poll more often than
   every 5 seconds. Give up after 5 minutes. Optionally narrow the response with the `analysis` query
   parameter (comma-separated).
4. **Read the status before the values** — each family object carries a status of `completed`,
   `processing`, `error` or `not_requested`. Only treat a family as final at `completed`. A
   `not_requested` family is not a failure; it means you did not ask for it.
5. **Extract the insights** — for wellbeing, `WellbeingInsights` carries stress (`low`/`moderate`/
   `high`), anxiety (`minimal`/`mild`/`moderate`/`severe`) and depression (`minimal`/`moderate`/
   `severe`) ratings.

## Errors
- Envelope is `{"error": {"type": "...", "message": "..."}}` — not RFC 9457. See
  `errors/virtuosis-voice-biomarker-api-problem-types.yml`.
- `403` splits three ways and they need different handling: `UploadLimitExceeded` (out of credits),
  `Suspended` (account action), `ApiNotProvisioned` (billing never set up). Do not retry any of them.
- `429` means slow down. There is **no `Retry-After` and no rate-limit header** — back off
  exponentially from the 15–30s baseline.
- `SpeakerSpeechTooShort` and `RecordingTooLongForIsolation` only occur with
  `isolate_oldest_speaker: true`.

## Retry safety — read this
There is **no idempotency key** on `POST /recordings`. Processing takes up to five minutes, so client
timeouts are normal, and a blind retry will bill a second credit and create a second `recording_id`.
Persist the `recording_id` the moment the upload returns, and on an ambiguous failure resume by
polling rather than re-uploading.

## Mandatory display obligation
Any UI that shows Virtuosis outputs must carry this disclaimer:

> Not intended to provide a medical diagnosis or to replace clinical judgment. Outputs are for
> clinical decision support and must be interpreted by a qualified healthcare professional.

See <https://docs.virtuosis.ai/guidelines> for the legal manufacturer information requirement.
