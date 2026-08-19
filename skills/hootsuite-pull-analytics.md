---
name: hootsuite-pull-analytics
description: >-
  Pull organic post and profile metrics, and paid campaign/ad-set/ad metrics, out of the Hootsuite
  Analytics API into an ETL pipeline — including the app setup that has to happen first, the
  batching limits, and the cursor loop.
api: Hootsuite Analytics REST API
base_url: https://platform.hootsuite.com
spec: openapi/hootsuite-analytics-api-openapi.yml
generated: '2026-08-13'
method: generated
source: >-
  openapi/hootsuite-analytics-api-openapi.yml,
  https://developer.hootsuite.com/docs/using-the-api,
  https://developer.hootsuite.com/docs/networks-reference
operations:
  - retrieveMeOrganizations
  - getSocialProfiles
  - retrieveOrganizationMembersSocialProfiles
  - getMyAdAccounts
  - listPosts
  - listProfilesMetrics
  - listPaid
  - listPaidMetrics
scopes:
  - offline
  - analytics:read
---

# Pull Hootsuite analytics into a pipeline

## 0. One-time setup that cannot be skipped

The `analytics:read` scope must be **enabled on the app itself** before a token can ever request it:
App directory → Developer apps → *[your app]* → **Security** → Rest API tab → tick `analytics:read`
→ Save. Then install the app from the App Directory.

Hootsuite recommends creating a dedicated **service-account member** (not a real person) that has
access to every organization you need to query, and building the app under that account. An app
developer can install and use their own app without review.

Copy the **Client ID**, **Client secret** and your **Member ID** (Developer Profile page).

## 1. Authenticate

For unattended ETL use the `member_app` grant — no browser round trip:

```
curl -X POST https://platform.hootsuite.com/oauth2/token \
  -H 'Authorization: Basic <base64 client_id:client_secret>' \
  -F 'grant_type=member_app' \
  -F 'member_id=<MEMBER_ID>' \
  -F 'scope=analytics:read'
```

The token expires in **1 hour**. Refresh on a timer; do not let a long extraction run past it.

## 2. Resolve the profile universe

- `retrieveMeOrganizations` — `GET /v1/me/organizations` — the organizations in reach.
- `getSocialProfiles` — `GET /v1/socialProfiles` — every profile the token can see.
- `retrieveOrganizationMembersSocialProfiles` —
  `GET /v1/organizations/{organizationId}/members/{memberId}/socialProfiles` — use this instead when
  you need profiles correlated to a specific organization.

Bucket profile ids by network `type`. **Not every profile type is supported for analytics** — check
<https://developer.hootsuite.com/docs/networks-reference> and drop the rest, otherwise the request
fails rather than partially succeeding.

## 3. Organic metrics

- `listPosts` — `POST /v1/analytics/posts` — published posts plus lifetime metrics per post.
- `listProfilesMetrics` — `POST /v1/analytics/profiles` — daily profile metrics.

`profileId` is a required filter on both. Hard limits, all published:

- **Maximum 10 social profile IDs per call.** Batch in tens.
- **Maximum 100 results per call** via `limit`; a cursor comes back when more exist.
- `reportingPeriod` may start at most **2 years** in the past. When filtering by `reportingPeriod`
  the post's **creation date** is used.
- `lastModified` may start at most **30 days** in the past.

## 4. Paid metrics

Paid analytics are scoped to **ad accounts**, not organic profiles.

1. `getMyAdAccounts` — `GET /v1/me/adAccounts` — get `ad_account_id` values. Note the AdAccount
   entity is the one place in the API using `snake_case`. Check `can_access_api` and
   `social_profile_token_status` before relying on an account.
2. `listPaid` — `POST /v1/analytics/paid/{adEntityCollection}` — entity metadata.
3. `listPaidMetrics` — `POST /v1/analytics/paid/{adEntityCollection}/metrics` — daily metrics.

`{adEntityCollection}` is `campaigns`, `adsets` or `ads`. Query: `adaccountType` (`FACEBOOK` or
`TWITTER`), `datatype=PAID`, optional `limit` (default 10, max 100), optional `cursor`.
Body: `filters.adAccounts` — **at most 10 entries**, each `{adAccountId, organizationId}`. Group by
`organizationId` when batching.

Unified vocabulary: Campaign → Campaign / Campaign; Ad Set → Ad Set / **Ad Group** on Twitter/X;
Ad → Ad. The API path segment is always `adsets`.

## 5. The cursor loop

Every list call is cursor-paginated. Read `metadata` for the cursor, pass it back **verbatim** on the
next call, stop when none is returned. There is no offset or page number. A malformed cursor returns
error code **3020**.

## 6. Known data gaps

A set of Facebook profile and post metrics was **deprecated on 2026-06-03** following upstream
Facebook API changes. Historical data is still returned for dates before that; do not build new
extractions on them. See
<https://developer.hootsuite.com/changelog/changes-to-facebook-metrics> and
`lifecycle/hootsuite-lifecycle.yml`.

## 7. Errors and throughput

Same envelope as the rest of the platform:
`{"errors":[{"code":n,"message":"...","id":"..."}]}`. A partial failure returns `data` **and**
`errors` in one `200`.

20 requests/second, 100,000 calls/day per account. Because every analytics call is capped at 10
profiles and 100 rows, a wide backfill hits the **daily quota** long before the per-second limit —
budget the extraction against 100,000 calls, not against wall-clock time. Exhaustion is a **429**
with code 1003/1004/1043 and no `Retry-After`.

## References

- `rate-limits/hootsuite-rate-limits.yml`
- `errors/hootsuite-problem-types.yml`
- `scopes/hootsuite-scopes.yml`
- `data-model/hootsuite-data-model.yml`
