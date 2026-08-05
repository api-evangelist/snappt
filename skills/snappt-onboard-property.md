---
name: Onboard a property and configure Snappt verification features
description: Create a Snappt property, enable identity and income verification, customize the applicant flow, and issue applicant document-upload links.
api: openapi/snappt-enterprise-api-openapi-original.yml
generated: '2026-08-05'
method: generated
source: https://snappt-enterprise-api.readme.io/docs/using-our-api-with-the-document-upload-portal-links
operations:
  - GET /account
  - POST /properties
  - GET /properties
  - GET /properties/{propertyId}
  - PUT /properties/{propertyId}
  - POST /properties/{propertyId}/identity-verification
  - POST /properties/{propertyId}/income-verification
  - POST /properties/{propertyId}/features
  - POST /properties/{propertyId}/send-document-upload-email
  - POST /properties/{propertyId}/generate-applicant-link
---

# Onboard a property and configure Snappt verification features

Auth: `Authorization: Bearer <SNAPPT_PARTNER_API_KEY>`.

## Steps

1. **Confirm scope.** `GET /account` returns the API key record and the `company` object with `id`
   and `shortId`. Your key is scoped to specific companies and properties — a resource outside that
   scope returns `404`, not `403`.
2. **Create or find the property.** `POST /properties`, or `GET /properties` /
   `GET /properties/{propertyId}` if it exists. Store the property `id`. Every applicant session is
   associated with a company and a property.
3. **Enable the products you sell on this property.**
   - `POST /properties/{propertyId}/identity-verification` — enable/disable ID verification.
   - `POST /properties/{propertyId}/income-verification` — enable/disable income verification. When
     enabled, the income result is folded into the same `REPORT_READY` event as the fraud verdict.
   - `POST /properties/{propertyId}/features` — customize the applicant flow.
   - `PUT /properties/{propertyId}/identity-verification/settings` — attach the `selfie` or
     `document` image to the IDV PDF report.
4. **Issue applicant links.**
   - `POST /properties/{propertyId}/send-document-upload-email` — Snappt emails the applicant. Store
     the returned `applicantDetailId`.
   - `POST /properties/{propertyId}/generate-applicant-link` — you get the link and deliver it
     yourself. Set `externalId` and `externalMetadata` here so every downstream webhook carries your
     identifiers.
   - Sending a second email? Pass the stored `applicantIdentifier` (the first
     `applicantDetailId`) so the submissions are linked.
5. **Retire a property.** Properties **cannot be deleted**. `PUT /properties/{propertyId}` with
   `status: DISABLED`. Since v0.9.2, `status` must be exactly `ACTIVE` or `DISABLED` or the call
   returns `400`.

## Rules

- **Document upload links never expire**, and generating a new link does not invalidate the old one —
  both stay usable. What matters for linking is that the `applicantDetailId` stays the same.
- **Billing is per applicant**, not per submission or per upload, so repeat submissions by one
  applicant are not charged again.
- `applicantId` is one-to-one with a submission; `applicantDetailId` is one-to-many across
  submissions by the same person.
