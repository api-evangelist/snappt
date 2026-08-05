---
name: Run a Snappt identity verification session
description: Enable ID verification on a property, generate a single-use identity upload link, and retrieve the PASS/FAIL verdict and PDF report.
api: openapi/snappt-enterprise-api-openapi-original.yml
generated: '2026-08-05'
method: generated
source: https://snappt-enterprise-api.readme.io/docs/getting-started-identity-verification
operations:
  - POST /properties/{propertyId}/identity-verification
  - POST /id-verification/generate-identity-upload-link
  - POST /id-verification/send-identity-upload-email
  - GET /id-verification/applicants/{idVerificationId}
  - GET /id-verification/applicants/{idVerificationId}/report
  - PUT /properties/{propertyId}/identity-verification/settings
---

# Run a Snappt identity verification session

Base URL: `https://enterprise-api.snappt.com`. Auth: `Authorization: Bearer <SNAPPT_PARTNER_API_KEY>`.

## Steps

1. **Enable the feature on the property.**
   `POST /properties/{propertyId}/identity-verification`. Identity verification is off by default.
2. **Create the session.** `POST /id-verification/generate-identity-upload-link`. The response
   carries an `identityUploadLink` and an `idVerificationId`. **Store `idVerificationId`** — it is
   the only handle you get on this session.
   - Pass a `metadata` object to carry your own identifiers through; it is echoed back on the
     `IDV_REPORT_READY` payload.
   - Pass `redirectUrl` to send the applicant somewhere when the session completes. Without it the
     applicant stays on the Snappt completion page.
   - **The link is single-use and expires 72 hours after creation.**
   - Alternatives: `POST /id-verification/send-identity-upload-email` to have Snappt email it, or the
     static per-property `GET /id-verification/{propertyShortId}/link` and
     `GET /id-verification/{propertyShortId}/qr-code`.
3. **Wait for the verdict.** Either poll
   `GET /id-verification/applicants/{idVerificationId}` or subscribe to the `IDV_REPORT_READY`
   webhook. `status` is `PASS` or `FAIL`.
4. **Download the report.** `GET /id-verification/applicants/{idVerificationId}/report`.
   Configure what the PDF contains per property with
   `PUT /properties/{propertyId}/identity-verification/settings` — you can attach the `selfie` or the
   `document` image from the session.
5. **List sessions** with `GET /id-verification/applicants`.

## Rules

- **The IDV graph is disjoint from the fraud-detection graph.** Snappt states explicitly that
  identity-verification records and fraud-detection applicants **do not share ids and are not
  linked**. `IDV_REPORT_READY` webhooks carry `applicantId: null`, `applicantDetailId: null`,
  `externalId: null` and `externalMetadata: null`. If you need the two joined, join them yourself
  using the `metadata` you passed on the upload link.
- **`IDV_REPORT_READY` is not retried** — a single delivery attempt. If your receiver is down you
  must fall back to polling `GET /id-verification/applicants/{idVerificationId}`.
- **Biometric data.** This flow collects a government ID and a selfie. Treat the report and the
  attached images as regulated biometric/PII; do not cache them in agent context.
