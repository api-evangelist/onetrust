---
name: onetrust-provision-users-scim
description: >-
  Provision, update and deprovision OneTrust users and groups over SCIM 2.0 — the standards-based path
  that needs no bespoke connector, plus the platform-native Access Management operations for
  organizations and audit history.
api: onetrust:platform-user-provisioning
spec: openapi/onetrust-platform-user-provisioning-openapi.json
base_url: https://{hostname}/api/scim
scopes:
  - SCIM
  - USER
  - ORGANIZATION
operations:
  - getSchemasUsingGET
  - getSchemasByNameUsingGET
  - getResourceTypesUsingGET
  - getResourceTypesByNameUsingGET
  - getAllUsersUsingGET
  - createUserUsingPOST
  - getUserUsingGET
  - updateUserUsingPUT
  - patchUserUsingPATCH
  - deleteUserUsingDELETE
  - listGroupsUsingGET
  - getGroupResourceUsingGET
  - updateGroupMembersUsingPUT
  - updateGroupMembersUsingPATCH
  - getGroups
  - createGroup
  - getGroupById
  - updateGroup
  - deleteGroup
  - modifyGroup
---

# Provision OneTrust users over SCIM 2.0

OneTrust implements SCIM 2.0 properly — not just a `/Users` endpoint borrowing the name. It serves
`ServiceProviderConfig`, `Schemas` and `ResourceTypes`, and declares the registered URNs
(`urn:ietf:params:scim:schemas:core:2.0:User`, `...:Group`,
`urn:ietf:params:scim:schemas:extension:enterprise:2.0:User`) plus one vendor extension,
`urn:ietf:params:scim:schemas:onetrust:Group`. An IdP that already speaks SCIM can provision into
OneTrust without a bespoke connector.

## Before you start

- Resolve `{hostname}` for the tenant.
- Get a bearer token from `POST /api/access/v1/oauth/token`. **SCIM endpoints require an OAuth 2.0
  access token in the `Authorization` header** — unlike the rest of the platform they do not accept
  the API-key-as-query-parameter form.
- The credential needs the `SCIM` scope, which grants the whole SCIM surface (Users, Groups,
  Resources, Schemas, Service Provider) as a single unit — there is no read-only SCIM scope.

## 1. Discover before you write

Always start here rather than assuming the shape:

- `GET /api/scim/v3/ServiceProviderConfig` — which SCIM features this tenant supports.
- `GET /api/scim/v3/Schemas` — `getSchemasUsingGET`; `GET /api/scim/v3/Schemas/{schemaName}` —
  `getSchemasByNameUsingGET`.
- `GET /api/scim/v3/ResourceTypes` — `getResourceTypesUsingGET`.

The vendor extension `urn:ietf:params:scim:schemas:onetrust:Group` is where OneTrust-specific group
attributes live; read it from `Schemas` rather than hard-coding fields.

## 2. Pick v2 or v3 — they are both live

| | v2 | v3 |
|---|---|---|
| Users | `/api/scim/v2/Users` | `/api/scim/v3/Users` |
| Groups | `/api/scim/v2/Groups` | `/api/scim/v3/Groups` |
| Create group | not available | `createGroup` |
| Delete group | not available | `deleteGroup` |
| Schemas / ResourceTypes / ServiceProviderConfig | not available | yes |

v2 can read and update groups but cannot create or delete them. **Use v3 for anything new**; v2 exists
for already-integrated IdPs.

## 3. Users

- List: `GET /api/scim/v2/Users` — `getAllUsersUsingGET`
- Create: `POST /api/scim/v2/Users` — `createUserUsingPOST`
- Read: `GET /api/scim/v2/Users/{id}` — `getUserUsingGET`
- Replace: `PUT /api/scim/v2/Users/{id}` — `updateUserUsingPUT`
- Patch: `PATCH /api/scim/v2/Users/{id}` — `patchUserUsingPATCH`
- Delete: `DELETE /api/scim/v2/Users/{id}` — `deleteUserUsingDELETE`

Prefer `PATCH` over `PUT`. A `PUT` replaces the resource, so a payload missing an attribute clears it —
that is the standard way an IdP sync accidentally wipes a user's organization assignment.

## 4. Groups

v3: `getGroups`, `createGroup`, `getGroupById`, `updateGroup`, `modifyGroup`, `deleteGroup`.
v2: `listGroupsUsingGET`, `getGroupResourceUsingGET`, `updateGroupMembersUsingPUT`,
`updateGroupMembersUsingPATCH`.

## 5. Organizations and audit (platform-native, not SCIM)

These live in `openapi/onetrust-platform-access-management-openapi.json` under
`https://{hostname}/api/access/v1` and need the `ORGANIZATION` or `USER` scope:

- `GET|POST /external/organizations`, `PUT|DELETE /external/organizations/{externalId}`
- `GET /login-history` — login audit records
- `GET /api/audit/v1/users/{userId}/activities` — per-user activity

`orgGroupId` is the most-referenced identifier in the whole platform (59 schemas). Getting a user's
organization wrong scopes them out of records they should see, silently.

## Reversibility

`deleteUserUsingDELETE` and `deleteGroup` have **no documented restore path and no published
retention window**. OneTrust does not publish an undelete for either. Deprovisioning is a routine
IdP-driven operation, so this matters: an over-broad IdP filter can delete users in bulk with nothing
to roll back to. Require confirmation on any delete that was not driven by an explicit IdP
deprovision event, and log the SCIM `id` before deleting.

## Errors

403 means the credential lacks `SCIM` — it cannot be widened at token time, the credential must be
re-created in Global Settings. 404 means the id does not exist **in this tenant**; SCIM ids are not
portable across OneTrust environments. See `errors/onetrust-problem-types.yml`.
