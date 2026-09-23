---
name: onetrust-fulfill-dsr-request
description: >-
  Create, triage and close a Data Subject Request (DSAR/DSR) in OneTrust Privacy Automation — search
  the queue, read a request, verify the requester, add subtasks, move it through stages and retrieve
  the deletion certificate.
api: onetrust:privacy-automation-data-subject-request-dsr-automation
spec: openapi/onetrust-privacy-automation-data-subject-request-dsr-automation-openapi.json
base_url: https://{hostname}/api/datasubject/v2
scopes:
  - DSAR_READ
  - DSAR_WRITE
operations:
  - createRequestQueueV2UsingPOST
  - getRequestCreationLogsUsingGET
  - searchForRequestUsingPOST
  - getAllRequestQueuesV2UsingGET
  - getRequestByIdUsingGET
  - getAllV2VerificationMethodsUsingGET
  - createV2VerificationMethodUsingPOST
  - updateV2VerificationMethodUsingPUT
  - getAllSubTaskByRefIdUsingGET
  - createSubTaskUsingPOST
  - createSubTaskFromTemplateUsingPOST
  - addCommentsUsingPUT
  - moveStatusByRequestRefIdUsingPUT
  - pauseOrResumeDeadlineUsingPUT
  - getRequestHistory
  - getDeletionCertificateUsingGET
  - bulkDeleteUsingPUT
---

# Fulfill a Data Subject Request in OneTrust

## Before you start

1. **Resolve the environment host.** Every path below is relative to `https://{hostname}`, where
   `{hostname}` is the tenant's OneTrust environment — `app.onetrust.com`, `app-eu.onetrust.com`,
   `trial.onetrust.com`, and so on. There is no global host. Ask, or read
   `conventions/onetrust-conventions.yml` → `base_url.shared_environments`.
2. **Get a token.** `POST /api/access/v1/oauth/token` (operationId `GetOAuthToken`) with the client
   credential, then send `Authorization: Bearer {token}`. The credential must already carry
   `DSAR_READ` and `DSAR_WRITE` — scopes are chosen when the credential is created and cannot be
   widened at token time. A 403 here means the wrong credential, not the wrong call.
3. **Know that there is no idempotency key.** OneTrust has none. If `createRequestQueueV2UsingPOST`
   times out, do NOT retry blindly — search first with `searchForRequestUsingPOST` and only create if
   nothing matches.

## 1. Create the request

`POST /api/datasubject/v2/requestqueues/{templateId}` — `createRequestQueueV2UsingPOST`.
The template decides the workflow, the deadline and the required fields.

The response gives you a `requestTraceId`. Creation is asynchronous: poll
`GET /api/datasubject/v2/requestqueues/status/{requestTraceId}` — `getRequestCreationLogsUsingGET`
— until it reports the request has been created, then take the `requestQueueRefId` from there.
That ref id, not the trace id, is what every later call uses.

> `createRequestQueueFromMessageUsingPOST` (`POST /api/datasubject/v2/requestqueues`) also creates a
> request but is marked `deprecated: true` in the contract. Use the templated form.

## 2. Find an existing request

- `POST /api/datasubject/v2/requestqueues/search/{language}` — `searchForRequestUsingPOST`, for
  criteria-based lookup. This is the one to use before creating anything.
- `GET /api/datasubject/v2/requestqueues/{language}` — `getAllRequestQueuesV2UsingGET`, paged listing.
- `GET /api/datasubject/v2/requestqueues/{requestQueueRefId}/language/{language}` —
  `getRequestByIdUsingGET`.

`{language}` is a required path segment on these, not a query flag.

Paging follows the platform convention: `page` (0-based), `size`, `sort=property,asc|desc`, with
`totalPages` / `totalElements` in the envelope. See
`conventions/onetrust-conventions.yml` → `pagination`.

## 3. Verify the requester

- `GET .../{requestQueueRefId}/verificationmethods` — `getAllV2VerificationMethodsUsingGET`
- `POST .../{requestQueueRefId}/verificationmethods` — `createV2VerificationMethodUsingPOST`
- `PUT .../{requestQueueRefId}/verificationmethods` — `updateV2VerificationMethodUsingPUT`

Do not advance a request past verification on an agent's own judgement. Verification status is a
compliance artifact.

## 4. Work the request

- Subtasks: `getAllSubTaskByRefIdUsingGET`, `createSubTaskUsingPOST`, or
  `createSubTaskFromTemplateUsingPOST` when the tenant has subtask templates.
- Comments: `PUT .../{requestQueueRefId}/comments` — `addCommentsUsingPUT`.
- Custom fields: `PUT .../{requestQueueRefId}/customfields/{language}` — `updateCustomFieldsUsingPUT`.
- Stage: `PUT .../{requestQueueRefId}/movestages/{language}` — `moveStatusByRequestRefIdUsingPUT`.
- Deadline: `PUT .../{requestQueueRefId}/pausedeadline` — `pauseOrResumeDeadlineUsingPUT`. Pausing a
  statutory deadline is a compliance decision; surface it to a human rather than deciding it.

## 5. Close out

- Audit trail: `GET .../{requestQueueRefId}/requesthistory` — `getRequestHistory`.
- Deletion certificate: `GET /api/datasubject/v2/requestqueues/delete-certificate/{requestRefId}` —
  `getDeletionCertificateUsingGET`. This is the evidence a deletion request was honoured; fetch and
  store it rather than asserting completion.

## Reversibility — read this before any delete

`bulkDeleteUsingPUT` (`PUT /api/datasubject/v2/requestqueues/bulkdelete/{language}`) deletes requests
in bulk. **OneTrust publishes no restore path and no retention window for it.** There is no undo
documented. Treat it as permanent and require explicit human confirmation naming the exact
`requestQueueRefId`s.

## Errors

| Status | What it usually means |
|---|---|
| 400 | Body failed schema validation, or a UUID path parameter is malformed. |
| 401 | Token missing/expired, or issued against a different environment host. |
| 403 | Credential lacks `DSAR_READ` / `DSAR_WRITE`. Re-create the credential; you cannot widen it. |
| 404 | The ref id does not exist **in this tenant**. Identifiers are not portable across environments. |
| 429 | Back off for `Retry-After` seconds. Read `ot-period` to see whether you hit the per-minute endpoint cap (1,000) or the per-hour account cap (200,000). |
| 500 | Retryable — but OneTrust returns 500 for an invalid `sort` criterion, so cap your retries. |

There is no RFC 9457 problem+json and no stable error code registry; the body may be a JSON string or
an untyped object. See `errors/onetrust-problem-types.yml`.
