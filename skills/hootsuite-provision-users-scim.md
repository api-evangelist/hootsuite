---
name: hootsuite-provision-users-scim
description: >-
  Provision and deprovision Hootsuite members and teams over SCIM 2.0, or over the native
  organization endpoints, and read back the permissions that actually govern what those users can do.
api: Hootsuite REST API
base_url: https://platform.hootsuite.com
spec: openapi/hootsuite-rest-api-openapi.yml
generated: '2026-08-13'
method: generated
source: >-
  openapi/hootsuite-rest-api-openapi.yml,
  https://developer.hootsuite.com/docs/api-permissions-matrix
operations:
  - getScimResourceTypes
  - createScimUser
  - getScimUsers
  - getScimUser
  - replaceScimUser
  - modifyScimUser
  - createScimGroup
  - getScimGroups
  - getScimGroup
  - replaceScimGroup
  - modifyScimGroup
  - createMember
  - retrieveMember
  - retrieveMemberOrganizationsById
  - retrieveOrganizationMembers
  - removeMemberFromOrganization
  - createTeam
  - addMemberToTeam
  - getTeamMembers
  - getTeam
  - retrieveOrganizationMemberOrganizationPermissions
  - retrieveOrganizationMembersTeamPermissions
scopes:
  - offline
permissions:
  - Organization permission "Admin or above"
  - Custom permission "Manage Members" (users) / "Manage Teams" (teams)
---

# Provision Hootsuite users and teams

Two surfaces do overlapping work. Pick one deliberately.

| | SCIM 2.0 (`/scim/v2/*`) | Native (`/v1/members`, `/v1/organizations/*`) |
|---|---|---|
| Shape | RFC 7643 / RFC 7644 | Hootsuite `data`/`errors` envelope |
| Errors | RFC 7644 §3.12 (`scimType`) | Numeric Hootsuite codes |
| Partial update | `PATCH` with SCIM `PatchOp` | `POST` with a field subset |
| Use when | Wiring an IdP (Okta, Entra, OneLogin) | Bespoke integration, or you need permissions/teams reads |

Both act on the **same** members and teams. A Hootsuite **Team** is a SCIM **Group**.

## 1. Authenticate and confirm authority

OAuth 2.0 as everywhere else — client credentials in an **HTTP Basic** header on
`POST /oauth2/token`, never in the body. There is no admin scope; `offline` is all you need at the
scope layer.

The real gate is the dashboard role. Every operation here requires **Admin or above** on the
organization, plus the custom permission **Manage Members** or **Manage Teams**. Verify with
`retrieveOrganizationMemberOrganizationPermissions` —
`GET /v1/organizations/{organizationId}/members/{memberId}/permissions` — before a bulk run, or you
will discover the gap partway through as codes **4002**, **4009** or **1201**.

## 2. Discover the SCIM surface

`getScimResourceTypes` — `GET /scim/v2/ResourceTypes` — returns the configuration for every
supported resource type. Call it first; it tells you what this tenant actually accepts.

## 3. Users

- `createScimUser` — `POST /scim/v2/Users`
- `getScimUsers` — `GET /scim/v2/Users` — supports **equals filtering on `username` only**. No
  `co`, `sw` or complex filters — plan reconciliation around an exact-match lookup.
- `getScimUser` — `GET /scim/v2/Users/{memberId}`
- `replaceScimUser` — `PUT /scim/v2/Users/{memberId}` — full replace. Call `/Schemas` to see the
  attributes the PUT body requires.
- `modifyScimUser` — `PATCH /scim/v2/Users/{memberId}` — SCIM `PatchOp` body, one or more attributes.

`ScimUser` carries `userName`, `name`, `emails[]`, `displayName`, `title`, `timezone`,
`preferredLanguage`, `groups[]` and `active`. `timezone` defaults if you omit it.

Native equivalents: `createMember` — `POST /v1/members` (requires `email` and `fullName`),
`retrieveMember` — `GET /v1/members/{memberId}`.

**Deprovisioning:** SCIM deactivation is `active: false` via `modifyScimUser`. Removing someone from
an organization outright is native only — `removeMemberFromOrganization` —
`DELETE /v1/organizations/{organizationId}/members/{memberId}`. A member holding the payment role
cannot be removed (code **3301**).

## 4. Teams / Groups

- `createScimGroup` / `getScimGroups` / `getScimGroup` / `replaceScimGroup` / `modifyScimGroup`
  under `/scim/v2/Groups`. `getScimGroups` supports equals filtering on `displayName` only.
- Native: `createTeam` — `POST /v1/organizations/{organizationId}/teams`; `addMemberToTeam` —
  `POST /v1/organizations/{organizationId}/teams/{teamId}/members/{memberId}`; `getTeamMembers`;
  `getTeam` — `GET /v1/teams/{teamId}`.

Team-name rules surface as errors, not schema: **3002** (name must be 2–200 characters), **3006**
(name must be unique in the organization), **3008** (member is not seated in the organization).

## 5. Read back the permissions that matter

Creating a user grants nothing by itself. What they can do is the intersection of organization, team
and social-network permissions:

- `retrieveOrganizationMemberOrganizationPermissions` — organization-level.
- `retrieveOrganizationMembersTeamPermissions` —
  `GET /v1/organizations/{orgId}/teams/{teamId}/members/{memberId}/permissions`.
- `retrieveOrganizationSocialProfilePermissions` —
  `GET /v1/organizations/{orgId}/members/{memId}/socialProfiles/{profileId}/permissions`.

Publishing needs "Limited or above" plus **Publish Message**; deleting a scheduled message needs
"Editor or above". Provision the profile permission or the user's API calls will 403 with no scope
error to explain it.

## 6. Two error shapes on one host

SCIM paths return
`{"schemas":["urn:ietf:params:scim:api:messages:2.0:Error"],"status":"400","detail":"...","scimType":"invalidValue"}`
with `scimType` in `invalidSyntax | mutability | invalidValue | uniqueness`.

Native paths return `{"errors":[{"code":n,"message":"...","id":"..."}]}`. Relevant codes: 2000–2008
(member field validation), 2303–2306 (invalid/missing ids), 3002–3010 (team and email), 3301
(payment member), 4002–4010 (insufficient permissions), 4303 (member not in organization).

**There is no idempotency key.** A retried `createScimUser` or `createMember` can create a duplicate.
Reconcile with `getScimUsers?filter=userName eq "..."` before every create, and again after any
ambiguous failure.

Rate limits apply to bulk provisioning like anything else: 20 req/s and 100,000 calls/day. A large
directory sync is a quota problem, not a latency problem.

## References

- `scopes/hootsuite-scopes.yml`
- `conformance/hootsuite-conformance.yml`
- `errors/hootsuite-problem-types.yml`
- `data-model/hootsuite-data-model.yml`
