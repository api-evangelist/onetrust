---
name: onetrust-scan-and-publish-cookie-consent
description: >-
  Run a OneTrust Cookie Consent (CMP) cycle — scan a website or mobile app, review the discovered
  cookies and SDKs, categorise them, and publish the updated banner script to the domain or app.
api: onetrust:consent-and-preferences-cookie-consent
spec: openapi/onetrust-consent-preferences-cookie-consent-openapi.json
base_url: https://{hostname}/api/cmp/v1
scopes:
  - COOKIE
  - COOKIE_READ
operations:
  - scanApplication
  - getScans
  - getSingleScan
  - getScanSdks
  - getScanSdkDetails
  - getScanPermissions
  - getAppPrivacyManifest
  - getAppPrivacyManifestDatatype
  - getCategorizedCookies
  - getScriptForWebsite
  - getScriptDetails
  - downloadScriptFile
  - publishScriptToSite
  - publishAppScript
  - getAppScriptDetails
  - getBrandingAttributeList
  - updateBrandingAttributesForPublicApi
  - createDomainGroup
  - deleteDomain
  - bulkDeleteCookies
---

# Scan and publish a OneTrust cookie banner

## Before you start

Resolve `{hostname}`, get a bearer token, and confirm the credential carries `COOKIE` for writes or
`COOKIE_READ` for reads only. All paths below are under `https://{hostname}/api/cmp/v1`.

## 1. Scan

**Mobile / OTT app:** `PUT /application/{id}/scan` — `scanApplication`. Then poll
`GET /applications/{id}/scans` (`getScans`) and read one with
`GET /applications/{id}/scans/{scanId}` (`getSingleScan`).

Scan results expose more than cookies:

- `GET /applications/{id}/scans/{scanId}/sdks` — `getScanSdks`, and
  `.../sdks/{sdk}` — `getScanSdkDetails`. This is the third-party SDK inventory.
- `GET /applications/{id}/scans/{scanId}/permissions` — `getScanPermissions`.
- `GET /applications/{id}/scans/{scanId}/privacy-manifest` — `getAppPrivacyManifest`, and
  `.../privacy-manifest/{dataType}` — `getAppPrivacyManifestDatatype`. Apple privacy-manifest data,
  which is what an App Store submission is checked against.

## 2. Categorise

`POST /cookies/categorized` — `getCategorizedCookies`. Uncategorised cookies land in the banner
ungrouped, which is the usual cause of a banner that looks wrong after a scan.

## 3. Publish

Publishing is what actually changes what a visitor sees. Nothing before this step is live.

- Website: `PUT /domains/publish` — `publishScriptToSite`
- App: `PUT /applications/{id}/publish` — `publishAppScript`

Then confirm the script that is now serving:

- `GET /domains/scripts` — `getScriptForWebsite`
- `GET /domains/{domainId}/script-details` — `getScriptDetails`
- `GET /domains/{domainId}/scripts` — `downloadScriptFile`
- `GET /applications/{id}/script-details` — `getAppScriptDetails`

The published script is loaded in the customer's page by
`https://cdn.cookielaw.org/scripttemplates/otSDKStub.js`, which is **unpinned** — it floats to the
current CDN build. See `components/onetrust-components.yml`.

## 4. Branding and grouping

- `GET|PUT /domains/{id}/branding-attributes` — `getBrandingAttributeList`,
  `updateBrandingAttributesForPublicApi`
- `POST /domains/{domainId}/domaingroup` — `createDomainGroup`

## Reversibility — this section is the reason to read the skill

Publishing is **live and immediate**. A `publishScriptToSite` against a production domain changes the
consent banner every visitor sees, and OneTrust documents no rollback operation, no version pin and no
republish-previous. Download the current script with `downloadScriptFile` **before** publishing so
there is something to compare against, and treat publish as a human-approved action.

`deleteDomain` (`DELETE /domains/{domainId}`) and `bulkDeleteCookies`
(`DELETE /webscans/cookies`) have **no documented restore path and no retention window**. Deleting a
domain removes its configuration, its categorisations and its banner. Require explicit confirmation
naming the `domainId`.

## Errors

Standard platform semantics: 403 = missing `COOKIE` scope; 404 = the domain or application id is not
in this tenant; 429 = honour `Retry-After`, read `ot-period`. Scans are asynchronous — a scan that has
not finished returns an in-progress state rather than an error, so poll `getSingleScan` instead of
retrying `scanApplication`, which would start a second scan (there is no idempotency key).
