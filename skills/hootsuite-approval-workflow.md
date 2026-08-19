---
name: hootsuite-approval-workflow
description: >-
  Drive Hootsuite's prescreen approval workflow — approve or reject messages and comments pending
  review, read the review history, and react to approval state changes over webhooks.
api: Hootsuite REST API
base_url: https://platform.hootsuite.com
spec: openapi/hootsuite-rest-api-openapi.yml
generated: '2026-08-13'
method: generated
source: >-
  openapi/hootsuite-rest-api-openapi.yml,
  https://developer.hootsuite.com/docs/webhooks
operations:
  - retrieveMessages
  - retrieveMessage
  - approveMessage
  - rejectMessage
  - getMessageHistory
  - retrieveComment
  - approveComment
  - rejectComment
  - getSocialProfiles
scopes:
  - offline
permissions:
  - Social network permission "Limited or above" with "Basic Usage" to read
  - Approval rights on the social profile to act
---

# Work the Hootsuite approval queue

Hootsuite's approval (prescreen) workflow is the part of this API that most social platforms do not
have: content can be held in `PENDING_APPROVAL` until a reviewer acts. Automating it means acting on
someone else's behalf — treat every approve as a consequential write.

## 1. Prerequisites

The prescreen component is an **organization app**, and Hootsuite states that organizations using
prescreen components must be on the **Enterprise plan**, with the organization-app installation
configured by Hootsuite. Authenticate with `grant_type=organization_app` and `organization_id`,
client credentials in an HTTP Basic header.

## 2. Find what is waiting

`retrieveMessages` — `GET /v1/messages` — filter with `state=PENDING_APPROVAL`, optionally scoped by
`socialProfileIds`. Percent-encode every query value; repeat the key for arrays
(`?socialProfileIds=1234&socialProfileIds=5678`). More than 50 results returns an opaque cursor in
`metadata` — echo it back verbatim.

Hootsuite notes that messages pending approval, including those created by *and/or actionable by*
the given user, are returned here.

Read one with `retrieveMessage` — `GET /v1/messages/{messageId}`. A message is always bound to a
**single** social profile, so a post that fanned out across five profiles is five independent
approval decisions.

## 3. Read the review history before acting

`getMessageHistory` — `GET /v1/messages/{messageId}/history` — returns the prescreening review
history as `ReviewAction` entries: `actorType`, `actorId`, `actionType`, `timestamp` (UTC).

Check it first. It tells you whether a human already acted, which is the only defence you have
against a double-approve — see step 6.

## 4. Act on messages

- `approveMessage` — `POST /v1/messages/{messageId}/approve`
- `rejectMessage` — `POST /v1/messages/{messageId}/reject`

Both take `Content-Type: application/json;charset=utf-8`. `sequenceNumber` identifies which step in
a multi-step approval chain you are acting on — send the one the message is currently at, not a
guess.

Approving moves the message to `APPROVED` and then on to `SCHEDULED`/`SENT`. Rejecting moves it to
`REJECTED` and it will **not** be sent. Neither is reversible through the API.

## 5. Act on comments

Comments run the same workflow with their own endpoints:

- `retrieveComment` — `GET /v1/comments/{commentId}` — retrieves a comment *if it has been through
  the approvals workflow*.
- `approveComment` — `POST /v1/comments/{commentId}/approve`
- `rejectComment` — `POST /v1/comments/{commentId}/reject`

`SocialProfileComment` carries `reviewState`, `sequenceNumber`, `creatorId`, `creatorName`,
`socialProfileId`, and `parentId`/`parentType` — where `parentId` is the **social network's native**
post id, not a Hootsuite id.

Comment states: `PENDING_APPROVAL`, `EXPIRED_APPROVAL`, `COMPLETED`, `FAILED`, `DELETED`,
`REJECTED`. `EXPIRED_APPROVAL` means it sat in review too long and lapsed on its own — an approval
bot that stalls silently produces expired content, not queued content.

## 6. Retry discipline

There is no idempotency key. A retried approve is a second approve.

Before any retry:

1. `getMessageHistory` (or `retrieveComment` for comments) and look for a `ReviewAction` matching
   your actor and timestamp window.
2. If it is already there, the original call landed — stop.
3. Only if it is absent should you re-issue.

Never blind-retry an approve on a timeout. Approving publishes to a live audience.

## 7. React instead of polling

Register a callback URL on your app (App directory → Developer apps → *[app]* → Security) and you
receive:

- `com.hootsuite.messages.event.v1` — `state` transitions including `PENDING_APPROVAL`, `APPROVED`,
  `REJECTED`, `SENT`, `SEND_FAILED_PERMANENTLY`.
- `com.hootsuite.comments.event.v1` — comment states, added 2019-03-20, triggered for any social
  profile configured with the prescreen app.

Each event carries `seq_no` (monotonic per callback URL), the state, `organization.id`, the object
id and an ISO-8601 `timestamp` — **ids only**, so call back for detail. Return `200 OK` or Hootsuite
disables the URL after consistent failures; a `com.hootsuite.ping.event.v1` answered `200` re-enables
it. **These callbacks are unsigned** — there is no signature header on the REST platform webhooks, so
verify by fetching the object rather than trusting the payload.

## 8. Errors

Standard envelope. Watch **4009** (insufficient organization permissions), **1037** (required scope
not granted), **1201** (not authorized to make changes to organization), **40003** (message cannot
be deleted in its current state). Errors carry a trace `id` — quote it to
`dev.support@hootsuite.com`.

## References

- `asyncapi/hootsuite-webhooks.yml`
- `errors/hootsuite-problem-types.yml`
- `data-model/hootsuite-data-model.yml`
- `authentication/hootsuite-authentication.yml`
