---
name: onetrust-record-consent
description: >-
  Record, read, update and withdraw a data subject's consent in OneTrust Universal Consent &
  Preference Management — collection points, purposes, preference centres, receipts and withdrawal.
api: onetrust:consent-and-preferences-universal-consent-and-preference-management-oas
spec: openapi/onetrust-consent-preferences-universal-consent-preference-management-oas-openapi.json
base_url: https://{hostname}/api/consentmanager
scopes:
  - CONSENT
  - CONSENT_READ
operations:
  - getCollectionPointsUsingGET
  - createCollectionPointUsingPOST
  - getTokenUsingGET
  - getPurposesUsingGET
  - createPurposeUsingPOST
  - publishPurposeUsingPUT
  - createNewPurposeVersionUsingPOST
  - setRetirementUsingPUT
  - getDataSubjectProfileUsingGET
  - createOrUpdateDataSubjectUsingPOST
  - getDataSubjectPurposesByIdentifierUsingGET_1
  - updatePreferencesForDataSubjectApiUsingPUT
  - withdrawPreferencesApiUsingDELETE
  - getReceiptListDetailsUsingGET
  - findReceiptUsingGET
  - mergeDataSubjectsUsingPOST
---

# Record and manage consent in OneTrust

## Before you start

Resolve `{hostname}` (the tenant's environment), get a bearer token from
`POST /api/access/v1/oauth/token`, and confirm the credential carries `CONSENT` (read/write/delete) or
`CONSENT_READ` (read only). See `authentication/onetrust-authentication.yml`.

**Version sprawl is the trap here.** v1, v2, v3 and v4 of the data-subject surface are all live at
once and they are not interchangeable. OneTrust publishes a
[V1 to V4 migration guide](https://developer.onetrust.com/onetrust/reference/v1-to-v4-migration-guide).
For new work prefer the newest version the tenant has enabled; the v1 operations named below as
deprecated are flagged `deprecated: true` in the contract.

## 1. Understand the model

- A **Collection Point** is where consent is captured (a form, an app screen, an API caller).
- A **Purpose** is what consent is for. Purposes are versioned and must be **published** before use.
- A **Preference Center** groups purposes for a data subject to manage.
- A **Receipt** is the immutable record that consent was given or withdrawn.

`data-model/onetrust-data-model.yml` has the full relationship graph.

## 2. Set up purposes and collection points

- `GET /v1/purposes` — `getPurposesUsingGET` *(deprecated in spec; prefer the v2 purposes surface)*
- `POST /v1/purposes` — `createPurposeUsingPOST`
- `POST /v1/purposes/{purposeGuid}` — `createNewPurposeVersionUsingPOST`
- `PUT /v1/purposes/{purposeId}/publish` — `publishPurposeUsingPUT`
- `PUT /v1/purposes/{purposeId}/retire` — `setRetirementUsingPUT`
- `GET /v1/collectionpoints` — `getCollectionPointsUsingGET`
- `POST /v1/collectionpoints` — `createCollectionPointUsingPOST`
- `GET /v1/collectionpoints/{collectionpointGuid}/token` — `getTokenUsingGET`

An unpublished purpose will not accept consent. Publish before you capture.

## 3. Capture consent

Consent is written by posting a **receipt**. The high-volume ingestion path is
`POST /request/v1/consentreceipts` (single) and `POST /v1/consentreceipts/bulk` (bulk), both
rate-provisioned separately — 2,000/min and 3,000/min respectively. See
`rate-limits/onetrust-rate-limits.yml`.

**There is no idempotency key.** A retried receipt post creates a second receipt. Before retrying a
timed-out write, read back with `getReceiptListDetailsUsingGET` and match on the data subject
identifier and purpose.

## 4. Read consent state

- `GET /v1/datasubjects/profiles` — `getDataSubjectProfileUsingGET` (paged: `page`, `size`, `sort`;
  large tenants should use the `requestContinuation` keyset token from `pageable`)
- `GET /v1/preferencecenters/{prefcenterId}/datasubjects/preferences` —
  `getDataSubjectPurposesByIdentifierUsingGET_1`
- `GET /v1/receipts` — `getReceiptListDetailsUsingGET`
- `GET /v1/receipts/{id}` — `findReceiptUsingGET`

## 5. Update and withdraw

- `PUT /v1/preferencecenters/{prefcenterId}/datasubjects/preferences` —
  `updatePreferencesForDataSubjectApiUsingPUT`
- `POST /v1/datasubjects/dataelements` — `createOrUpdateDataSubjectUsingPOST`
- `DELETE /v1/preferencecenters/{prefcenterId}/datasubjects/preferences` —
  `withdrawPreferencesApiUsingDELETE` — withdraws consent for **all** purposes in that preference
  centre.

## Reversibility

Withdrawal is a **forward-only consent event**, not an undo: the original receipt is retained and a
withdrawal receipt is written. Re-consenting is a new capture, not a restore.

Deletion is different and is genuinely destructive:
`deleteDataSubjectProfilesUsingDELETE` (`DELETE /api/consentmanager/v2/datasubjects/profiles`) and
`deleteDataSubjectUsingTTL` (`DELETE /rest/api/consent/v4/data-subjects`) have **no documented
restore path and no published retention window** — the `UsingTTL` name implies a delay, but OneTrust
does not state one, so do not assume it. Require explicit human confirmation.

`mergeDataSubjectsUsingPOST` (`PUT /v1/datasubjects/merge`) deduplicates profiles. Merges are also
not documented as reversible; export first with
`GET /v1/export-duplicate-datasubject/{mergeRequestId}` (`exportduplicatedatasubject`).

## Errors and limits

429 carries `Retry-After` plus `ot-period`, `ot-requests-allowed`, `ot-request-made` and
`ot-ratelimit-event-id`. There is no proactive budget header on successful calls, so an agent running
a bulk import cannot see how close it is until it is throttled. Honour `Retry-After` and back off
exponentially. Full catalogue in `errors/onetrust-problem-types.yml`.
