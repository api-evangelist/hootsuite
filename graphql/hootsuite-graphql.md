# Hootsuite GraphQL Schema

## Overview

This conceptual GraphQL schema represents the Hootsuite social media management platform API surface. Hootsuite provides a REST API at `https://platform.hootsuite.com/v1` with OAuth 2.0 authentication. This schema models the core domain objects and operations available through the Hootsuite platform, covering social profile management, message scheduling and publishing, analytics, team collaboration, inbox management, content libraries, and approval workflows.

## Schema Design

The schema is organized around the primary Hootsuite platform capabilities:

- **Social Profiles and Networks** — connecting and managing social media accounts across LinkedIn, X (Twitter), Facebook, Instagram, TikTok, YouTube, and Pinterest
- **Messages and Scheduling** — composing, scheduling, and publishing social media posts with media attachments
- **Analytics and Reporting** — engagement metrics, post performance, and profile analytics
- **Teams and Collaboration** — members, roles, permissions, organizations, and approval workflows
- **Inbox and Engagement** — monitoring streams, conversation threads, and message assignment
- **Content Library** — reusable assets and drafts

## Types Reference

### Core Messaging Types

| Type | Description |
|------|-------------|
| `Message` | Base message object with scheduling and publication state |
| `MessageDetails` | Extended message metadata including network-specific fields |
| `MessageStatus` | Enum for draft, scheduled, sent, failed, pending review |
| `MessageType` | Enum for text, image, video, link, carousel post types |
| `PostMessage` | Input for creating or updating a message |
| `ScheduledMessage` | A message queued for future publication |
| `SentMessage` | A message that has been published to a social network |
| `MessageReview` | Review record attached to a message pending approval |
| `ReviewStatus` | Enum for pending, approved, rejected states |
| `ReviewFlow` | Approval workflow configuration for a team or organization |
| `MessageMedia` | Media attachment (image, video, GIF) for a message |

### Social Network and Profile Types

| Type | Description |
|------|-------------|
| `SocialNetwork` | Enum for supported networks: TWITTER, FACEBOOK, INSTAGRAM, LINKEDIN, TIKTOK, YOUTUBE, PINTEREST |
| `SocialNetworkProfile` | A connected social media account/page |
| `ProfileDetails` | Extended profile information including follower counts and network-specific metadata |
| `ProfileAuth` | OAuth credential state for a connected profile |
| `ProfileType` | Enum for personal, page, group, business account types |

### Stream and Monitoring Types

| Type | Description |
|------|-------------|
| `Stream` | A real-time social monitoring stream |
| `StreamDetails` | Configuration and filter settings for a stream |
| `StreamKeyword` | Keyword or hashtag tracked within a stream |
| `StreamFilter` | Filter criteria applied to a stream (language, location, sentiment) |

### Publishing and Draft Types

| Type | Description |
|------|-------------|
| `PublishingRequest` | A request to publish content to one or more profiles |
| `ContentDetails` | Content body, links, and formatting for a post |
| `BulkPublish` | Batch operation to schedule multiple messages at once |
| `Draft` | Unsaved or work-in-progress message |
| `DraftDetails` | Extended draft metadata including version history |

### Team and Organization Types

| Type | Description |
|------|-------------|
| `Member` | A Hootsuite user within an organization |
| `MemberDetails` | Extended member profile including role and permissions |
| `MemberRole` | Enum for super admin, admin, editor, restricted editor roles |
| `Permission` | Granular permission grant for a member |
| `Organization` | A Hootsuite account organization |
| `OrganizationPlan` | Plan details including feature entitlements and limits |
| `Team` | A sub-group within an organization |
| `TeamMember` | Association between a member and a team |
| `Invitation` | Pending invitation to join an organization or team |

### Analytics and Reporting Types

| Type | Description |
|------|-------------|
| `Analytics` | Top-level analytics object for a date range and profile set |
| `ProfileAnalytics` | Aggregate metrics for a social profile over a period |
| `PostAnalytics` | Engagement and reach metrics for a single post |
| `EngagementReport` | Compiled report of interactions and audience response |
| `Metrics` | Raw numeric metric values (impressions, clicks, likes, shares) |
| `AnalyticsFilter` | Filter parameters for analytics queries |
| `Report` | A saved or exported analytics report |
| `ReportDetails` | Configuration and scheduling details for a report |

### Inbox and Assignment Types

| Type | Description |
|------|-------------|
| `Assignment` | Assignment of an inbox thread to a team member |
| `InboxThread` | A conversation thread in the Hootsuite inbox |
| `ThreadMessage` | An individual message within an inbox thread |
| `MessageAttachment` | Attachment (media, link preview) on a thread message |

### Campaign and Content Types

| Type | Description |
|------|-------------|
| `Campaign` | A grouped set of messages and posts for a marketing campaign |
| `CampaignDetails` | Dates, goals, tags, and associated profiles for a campaign |
| `ContentLibrary` | Organization-level library of reusable media and content |
| `LibraryItem` | An individual asset in the content library |

### Approval Types

| Type | Description |
|------|-------------|
| `Approval` | An approval decision on a message or campaign |
| `ApprovalFlow` | Multi-step approval workflow configuration |

### Auth and Integration Types

| Type | Description |
|------|-------------|
| `APIKey` | An API key issued for programmatic access |
| `Token` | OAuth 2.0 access or refresh token |
| `Webhook` | Webhook subscription for platform events |

## Queries

- `messages` — list messages with status and schedule filters
- `message(id)` — fetch a single message by ID
- `scheduledMessages` — list upcoming scheduled messages
- `sentMessages` — retrieve published messages with analytics
- `profiles` — list connected social network profiles
- `profile(id)` — fetch a single profile with details
- `streams` — list monitoring streams
- `members` — list organization members
- `analytics` — fetch analytics for profiles and date ranges
- `inboxThreads` — list inbox conversation threads
- `campaigns` — list marketing campaigns
- `contentLibrary` — browse library assets

## Mutations

- `createMessage` — compose and schedule a message
- `updateMessage` — modify a scheduled message
- `deleteMessage` — remove a scheduled or draft message
- `publishMessage` — immediately publish a message
- `createDraft` — save a draft message
- `approveMessage` — approve a pending message in a review flow
- `rejectMessage` — reject a pending message with feedback
- `assignThread` — assign an inbox thread to a member
- `createWebhook` — register a new webhook endpoint
- `deleteWebhook` — remove a webhook subscription
- `inviteMember` — send an organization or team invitation

## Authentication

The Hootsuite REST API and this conceptual GraphQL surface use OAuth 2.0 authorization code flow. Clients obtain an access token via `https://platform.hootsuite.com/oauth2/token` and include it as a Bearer token in the `Authorization` header. Refresh tokens are supported for long-lived integrations.

## References

- Developer Portal: https://developer.hootsuite.com
- REST API Documentation: https://developer.hootsuite.com/docs/api
- API Reference: https://platform.hootsuite.com/docs/api/index.html
- Authentication Guide: https://developer.hootsuite.com/docs/api-authentication
- GitHub Organization: https://github.com/hootsuite
