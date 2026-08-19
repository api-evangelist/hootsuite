---
name: hootsuite-schedule-social-post
description: >-
  Schedule a post to one or more connected social profiles through the Hootsuite REST API,
  including optional media, and confirm it landed. Handles the fan-out response, the reauth trap
  and the fact that this write has no idempotency protection.
api: Hootsuite REST API
base_url: https://platform.hootsuite.com
spec: openapi/hootsuite-rest-api-openapi.yml
generated: '2026-08-13'
method: generated
source: >-
  openapi/hootsuite-rest-api-openapi.yml,
  https://developer.hootsuite.com/docs/message-scheduling,
  https://developer.hootsuite.com/docs/uploading-media
operations:
  - retrieveMe
  - getMySocialProfiles
  - createMedia
  - getMedia
  - scheduleMessage
  - retrieveMessage
  - retrieveMessages
  - deleteMessage
scopes:
  - offline
permissions:
  - Social network permission "Limited or above"
  - Custom permission "Publish Message" OR "Publish Message with Approval"
---

# Schedule a social post with Hootsuite

Publishes or schedules a message to one or more connected social profiles.

**This writes to live social networks. It is not idempotent.** Read step 6 before you retry anything.

## 1. Authenticate

Get an access token from `POST /oauth2/token`. Client credentials go in an **HTTP Basic header** —
Hootsuite does not accept them in the request body.

- Interactive integration: `grant_type=authorization_code` (the code is single-use and expires in
  10 minutes; reusing it revokes every token issued from it).
- Unattended single-customer integration: `grant_type=member_app` with `member_id`.

Request the `offline` scope so you get a refresh token. Refresh tokens never expire but are
**single-use** — persist the new one returned on every refresh or you are locked out.
Access tokens last 3599 seconds.

## 2. Resolve the caller and the target profiles

- `retrieveMe` — `GET /v1/me` — confirms which member the token acts as.
- `getMySocialProfiles` — `GET /v1/me/socialProfiles` — returns the profiles this token can use.

**Check `isReauthRequired` on every profile you intend to post to.** A value of `1` means the
upstream network token has lapsed; the publish will fail and only a human reconnecting the account
in the dashboard fixes it. Drop those profile ids and report them rather than scheduling into a
guaranteed failure.

Note the `type` field. Hootsuite's publishing API does **not** support TikTok, YouTube or Bluesky,
and cannot post to Instagram Business profiles (Instagram Personal only). Pinterest can be
scheduled, but **only on its own** — it cannot be bundled with any other profile in one call.

## 3. Attach media (optional)

Media is a two-step upload — `createMedia` does not accept bytes.

1. `createMedia` — `POST /v1/media` — declare the MIME type and size. The response carries a
   `mediaId` and a presigned Amazon S3 `uploadUrl`.
2. `PUT` the bytes to that `uploadUrl` with `Content-Type` and `Content-Length` **exactly matching**
   what you declared. Only the first valid upload to a URL is kept; later uploads are accepted by S3
   and immediately discarded by Hootsuite.
3. `getMedia` — `GET /v1/media/{mediaId}` — poll until the upload has finished processing. Attaching
   media that is not ready returns error code **40021** (`Uploaded media is not yet ready to be
   used`).

Hootsuite deletes uploaded media 90 days after it is used in a message.

## 4. Schedule the message

`scheduleMessage` — `POST /v1/messages` — with
`Content-Type: application/json;charset=utf-8` (Hootsuite requires the charset).

Body fields that matter: `text`, `socialProfileIds` (array), `scheduledSendTime` (UTC ISO-8601),
`media`, `tags`, `targeting`, `privacy`, `location`, `webhookUrls`, `emailNotification`.

**The response fans out.** One request against N profiles returns an **array of N messages**, each
with its own `id`. Persist every id — they are the only handles you have for the rest of the
lifecycle. Do not assume one request means one message.

Watch for: **40023** (Twitter allows only 1 network per message), **40024** (Facebook Groups no
longer supported), **40025** (scheduled message limit reached), **40001** (social profile is not
owned by the organization — required when posting with tags).

## 5. Confirm

- `retrieveMessage` — `GET /v1/messages/{messageId}` — per returned id.
- `retrieveMessages` — `GET /v1/messages` — filter by `socialProfileIds` and `state`; percent-encode
  every query value, and repeat the key for arrays
  (`?socialProfileIds=1234&socialProfileIds=5678`). More than 50 results returns a cursor in
  `metadata` — echo it back verbatim, opaque. A malformed cursor is error **3020**.

States: `SCHEDULED`, `PENDING_APPROVAL`, `APPROVED`, `SUBMITTED` (video still uploading), `SENT`,
`SEND_FAILED_PERMANENTLY`, `DELETED`, `REJECTED`.

If you registered `webhookUrls`, you will also receive `com.hootsuite.messages.event.v1` callbacks
on state change. They carry `seq_no`, `state`, `message.id`, `organization.id` and `timestamp` —
**ids only**, so call back into the API for anything else. Return `200 OK` or Hootsuite disables the
callback URL. These callbacks are **unsigned**.

## 6. Retrying safely

There is no `Idempotency-Key` and no replay-safe retry contract anywhere in the Hootsuite platform.
A retried `POST /v1/messages` after a timeout **can publish twice**.

On a timeout or ambiguous failure:

1. Do **not** blind-retry.
2. Call `retrieveMessages` filtered by the target `socialProfileIds` and a tight `startTime`/
   `endTime` window around your attempt.
3. If the message is there, treat the original call as succeeded.
4. If it is there twice, `deleteMessage` — `DELETE /v1/messages/{messageId}` — the duplicate while
   it is still `SCHEDULED`. Deleting requires "Editor or above". Error **40003** means the message
   can no longer be deleted in its current state.

## 7. Errors and rate limits

Errors come back as `{"errors":[{"code":n,"message":"...","id":"...","resource":{...}}]}` —
**not** RFC 9457 problem+json. A partially-failed request returns **both** `data` and `errors` in a
`200`; never treat that as clean. Quote the error `id` to `dev.support@hootsuite.com`.

Limits are 20 requests/second and 100,000 calls/day per account. Watch `X-Account-Quota-Used` and
`X-Account-Rate-Limit-Requests-Remaining` (reported per cluster node — a guide, not a budget).
Exhaustion returns **429** with code 1003, 1004 or 1043 and **no `Retry-After`** — back off to the
next one-second window yourself.

## References

- `conventions/hootsuite-conventions.yml`
- `errors/hootsuite-problem-types.yml`
- `rate-limits/hootsuite-rate-limits.yml`
- `asyncapi/hootsuite-webhooks.yml`
