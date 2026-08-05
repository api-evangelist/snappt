---
name: Receive and verify Snappt webhooks
description: Register a Snappt webhook, verify the Snappt-Signature-v2 HMAC, and build a receiver that survives retries and out-of-order delivery.
api: openapi/snappt-enterprise-api-openapi-original.yml
generated: '2026-08-05'
method: generated
source:
  - https://snappt-enterprise-api.readme.io/docs/webhook-snappt-signature-v2
  - https://snappt-enterprise-api.readme.io/docs/webhook-delivery-retry-behavior
  - https://snappt-enterprise-api.readme.io/docs/webhook-event-types
operations:
  - POST /webhooks
  - GET /webhooks
  - GET /webhooks/{id}
  - PUT /webhooks/{id}
  - DELETE /webhooks/{id}
  - GET /webhooks/{id}/signing-secret
---

# Receive and verify Snappt webhooks

## 1. Register

`POST /webhooks` with your partner API key. Supply your endpoint URL, an `events` array (a webhook
only receives event types it lists), and any custom headers you want echoed on delivery. The response
returns the **signing secret**, prefixed `whsec_`. If you lose it, call
`GET /webhooks/{id}/signing-secret`. Set `isActive: false` via `PUT /webhooks/{id}` to stop all
deliveries and redeliveries.

## 2. Verify the signature — before parsing the body

Snappt sends two headers. **Use `Snappt-Signature-v2`.** `Snappt-Signature` (v1) is keyed on the API
Key ID, is deprecated, and will be removed.

```
Snappt-Signature-v2: t=<unix-ms>,v2=<signature>
```

Verification:

1. Capture the **raw request body** before any JSON middleware parses it. Snappt signs compact JSON
   with no whitespace; a parse-and-re-serialize changes the bytes and the signature will not match.
2. Split the header on `,` and read `t` and `v2`.
3. Reject if `Date.now() - t` exceeds 5 minutes.
4. Compute `HMAC-SHA256(key = whsec_..., message = "{t}.{rawBody}")` and encode as **base64url**
   (RFC 4648 §5: `+`→`-`, `/`→`_`, no `=` padding) — not standard base64.
5. Compare in constant time (`crypto.timingSafeEqual`, `CryptographicOperations.FixedTimeEquals`).

## 3. Respond fast

- Only **HTTP 200** counts as success. `3xx` (redirects are not followed), `4xx`, `5xx`, connection
  errors, and TLS failures are all recorded as failures.
- **30-second timeout.** Return 200 as soon as you have matched the applicant, then do database
  writes, report downloads, and downstream calls afterward.

## 4. Handle retries and reordering

- `REPORT_READY` retries every **~61–70 minutes for 24 hours** (~20–24 attempts) and stops on a 200.
- `APPLICATION_SUBMITTED`, `REPORT_UPDATED` and `IDV_REPORT_READY` are **single attempt, never
  retried** — you need a polling fallback for those.
- **Order is not guaranteed**, even for the same applicant. A retry of an older event can land after
  a newer one. Compare against your stored state before overwriting a verdict.
- **Deduplicate.** Use `data.id` + `eventType` as the idempotency key; treat duplicates as no-ops and
  still return 200.

## 5. Correlate

`externalId` and `externalMetadata` are echoed on `APPLICATION_SUBMITTED`, `REPORT_READY` and
`REPORT_UPDATED`. They are **always null on `IDV_REPORT_READY`**, which is not tied to an applicant
record. Set them on `POST /properties/{propertyId}/generate-applicant-link` or via the Embedded SDK.

## 6. Always re-fetch

Snappt's own recommendation: verify the signature, then make a follow-up API call to retrieve the
full application data rather than trusting the webhook body as the record of truth.

## Event reference

| Event | Fires | Retried |
|---|---|---|
| `APPLICATION_SUBMITTED` | documents submitted, pre-review (`result`/`status` = `PENDING`) | No |
| `REPORT_READY` | verdict rendered (`CLEAN` / `EDITED` / `UNDETERMINED`, `status` `READY`) | Yes, 24h |
| `REPORT_UPDATED` | a delivered report changed (income dispute, override, fraud re-review) — carries an extra `update` object | No |
| `IDV_REPORT_READY` | ID verification terminal (`PASS` / `FAIL`) | No |
| `ACCEPTED_DOCUMENT` | **deprecated** — use `POST /session/documents?checks=true` instead | — |
