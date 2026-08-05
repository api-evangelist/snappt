---
name: Submit a rental application for Snappt fraud review
description: Create an applicant session, attach the applicant's PDF documents, submit for review, and retrieve the fraud verdict and PDF report.
api: openapi/snappt-enterprise-api-openapi-original.yml
generated: '2026-08-05'
method: generated
source: https://snappt-enterprise-api.readme.io/docs/getting-started-fraud-detection
operations:
  - POST /session
  - PUT /session/application
  - POST /session/documents
  - POST /session/submit
  - GET /applicants/{applicantId}
  - GET /applicants/{applicantId}/report
note: The published Snappt OpenAPI declares no operationId on any operation, so steps below reference HTTP method + path, which is what the contract actually exposes.
---

# Submit a rental application for Snappt fraud review

Base URL: `https://enterprise-api.snappt.com` (production) or `https://enterprise-api.demo.snappt.com` (demo).

## Auth

Send the partner API key as a bearer token on every partner-scoped call:

```
Authorization: Bearer <SNAPPT_PARTNER_API_KEY>
```

Session-scoped calls (`/session/*`) use the session token instead:

```
x-unauthenticated-session-token: <token from POST /session>
```

The session token expires **24 hours after creation or on submission, whichever comes first**. It is
scoped to one applicant session and is safe to hand to client-side JavaScript.

## Steps

1. **Create the session.** `POST /session` with your partner API key. The response carries `token`
   (the session token), `id` (session id), and `applicantDetailId`. Store `applicantDetailId` — it is
   the key that links repeat submissions by the same person.
2. **Attach applicant data.** `PUT /session/application` with the session token. Set `firstName`,
   `lastName`, `email`, `phone`, and `unit`.
3. **Upload each document.** `POST /session/documents` as `multipart/form-data` with the file in the
   `upload` field and the document `type` (`BANK_STATEMENT`, `PAYSTUB`, or `OTHER`). **PDFs only.**
   Add `?checks=true` to run the synchronous checks (document type detection, Print-to-PDF, scanned,
   password protected, PDF encryption, page limit) so you get an immediate accept/reject instead of
   waiting on the deprecated `ACCEPTED_DOCUMENT` webhook. Rejections come back as a 400 whose
   `failedChecks` array names the failing checks.
   **Limit: 15 documents per application** — a 16th returns
   `Only 15 documents may be attached to a single application.`
4. **Submit.** `POST /session/submit`. Do not call this while any document is still `PENDING` —
   since v0.9.1 the API rejects that, and before v0.9.1 it silently dropped documents. The response
   returns the `applicantId`. **The session token is invalid after this call.**
5. **Wait for the verdict.** Either:
   - poll `GET /applicants/{applicantId}` with the partner API key until `status` becomes `READY`, or
   - subscribe to the `REPORT_READY` webhook (preferred — see the webhook skill).
6. **Read the verdict.** `result` is `CLEAN`, `EDITED`, or `UNDETERMINED` (or `PASSED` / `FAILED` /
   `UNDETERMINED` if your company uses the pass-fail display setting). `note` carries a
   human-readable reason on some `EDITED` and `UNDETERMINED` verdicts.
7. **Download the report.** `GET /applicants/{applicantId}/report` with
   `?preset=summary`, `?preset=summary-and-documents`, or `?preset=fcra` (thumbnails replaced with
   generic thumbnails for FCRA-compliant reporting). Do **not** use the deprecated
   `includeDocuments` / `showThumbnails` parameters.

## Rules

- **No request idempotency.** Snappt exposes no `Idempotency-Key`. Re-POSTing `/session` creates a
  new session and a new applicant. Track `applicantDetailId` yourself to avoid duplicate submissions.
- **Errors.** `400` = invalid payload or query parameter (check `failedChecks`); `401` = bad or
  expired credential — note the edge returns 401 for unknown paths too; `404` = unknown id or an id
  outside your key's scope. There are no declared 5xx, 409, 422 or 429 responses.
- **Correlate with your own system** by passing `externalId` (max 255 chars) and `externalMetadata`
  (max 10 KB serialized) when you generate the applicant link; both are echoed back on
  `GET /applicants/{applicantId}` and on every applicant-scoped webhook.
- **FCRA / Fair Housing.** These reports are consumer-facing screening decisions. Do not let an agent
  auto-decline an applicant on an `EDITED` verdict — route it to a human. See
  `agentic-access/snappt-agentic-access.yml`.
