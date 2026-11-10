# Data Model: User Story Intake and Admin Dashboard

## Trust and identity

User accounts and role assignment belong to the existing authentication
system. The API authentication adapter supplies an immutable request principal:

| Field | Type | Constraints |
|---|---|---|
| `userId` | string | Required stable authenticated subject |
| `role` | enum | Exactly `CONTRIBUTOR` or `ADMIN` |

The application does not persist or trust a user-supplied role or owner. A
demo/testing role chooser can change only the requested interface view; API
authorization uses this principal.

## UserStory

One record represents the current story content and workflow state.

| Field | Type | Constraints |
|---|---|---|
| `id` | UUID string | Server-generated, unique, immutable |
| `ownerId` | string | From authenticated principal; immutable |
| `title` | string | Trimmed, 1-120 characters |
| `persona` | string | Trimmed, 1-120 characters |
| `requirement` | string | Trimmed, 1-1000 characters |
| `businessBenefit` | string | Trimmed, 1-1000 characters |
| `description` | string | Optional/empty; maximum 5000 characters |
| `acceptanceCriteria` | string array | 1-20 trimmed items; each 1-1000 characters; order preserved |
| `priority` | enum | `LOW`, `MEDIUM`, `HIGH` |
| `status` | enum | `SUBMITTED`, `IN_REVIEW`, `NEEDS_CHANGES`, `APPROVED`, `REJECTED` |
| `createdAt` | UTC datetime | Server-generated and immutable |
| `latestSubmittedAt` | UTC datetime | Server-generated; updated on contributor resubmission |
| `version` | integer | Server-managed optimistic concurrency version |
| `idempotencyKey` | string | Creation-only, scoped to owner; not exposed in responses |
| `contentFingerprint` | string | Server-generated hash of normalized creation content; not exposed |

Unique constraints: `id` and the compound key `(ownerId, idempotencyKey)`.
The latter ensures concurrent retries cannot create duplicate stories for one
contributor and request key. Index `ownerId`; support admin filtering and
stable listing by `(status, priority, latestSubmittedAt desc, id asc)`. Search
text is escaped and treated as a literal case-insensitive substring.

## ReviewComment

Comments are append-only and retained across edits and status changes.

| Field | Type | Constraints |
|---|---|---|
| `id` | UUID string | Server-generated |
| `storyId` | UUID string | References `UserStory.id` |
| `authorId` | string | From authenticated principal |
| `text` | string | Trimmed, 1-2000 characters |
| `createdAt` | UTC datetime | Server-generated |

Read comments oldest first, using `(storyId, createdAt asc, id asc)` for stable
ordering.

## StatusEvent

Status events are append-only, providing an auditable history independent of
the current status.

| Field | Type | Constraints |
|---|---|---|
| `id` | UUID string | Server-generated |
| `storyId` | UUID string | References `UserStory.id` |
| `fromStatus` | status enum or null | Null only for initial submission |
| `toStatus` | status enum | Resulting status |
| `actorId` | string | Authenticated contributor or admin |
| `createdAt` | UTC datetime | Server-generated |

Every successful status change and resubmission records an event. Initial
creation records `null → SUBMITTED`. Read history oldest first by
`(storyId, createdAt asc, id asc)`.

## Relationships and atomicity

- One authenticated owner has zero or more stories.
- Each story has zero or more comments and one or more status events.
- Story content is updated only by its owner while status is `NEEDS_CHANGES`.
- Status and its required `NEEDS_CHANGES`/`REJECTED` comment must be committed
  together with the status event. Use a MongoDB transaction for status changes
  that include related records. The MongoDB deployment must support
  transactions (replica set or sharded cluster).
- Status, comment, and content writes use server-controlled concurrency
  checks; stale version preconditions return conflict without partial writes.

## Lifecycle

| Current status | Actor | Allowed next status |
|---|---|---|
| New | CONTRIBUTOR | `SUBMITTED` |
| `SUBMITTED` | ADMIN | `IN_REVIEW`, `NEEDS_CHANGES`, `APPROVED`, `REJECTED` |
| `IN_REVIEW` | ADMIN | `NEEDS_CHANGES`, `APPROVED`, `REJECTED` |
| `NEEDS_CHANGES` | Owner CONTRIBUTOR on resubmit | `SUBMITTED` |
| `APPROVED` | Any | None |
| `REJECTED` | Any | None |

No-op transitions are invalid. Transition to `NEEDS_CHANGES` or `REJECTED`
requires an explanatory comment. Every other transition may include an
optional comment.

## Consistency rules

- Normalize accepted text by trimming leading and trailing whitespace before
  validation and storage. Normalize arrays item-by-item without changing order.
- Fingerprint the normalized creation fields, excluding owner, server metadata,
  status, and request identifier. A duplicate key with the same fingerprint
  returns the existing story; a different fingerprint returns conflict.
- All timestamps are UTC. `createdAt` never changes; `latestSubmittedAt`
  changes on initial submission and successful resubmission.
- Summary counts group persisted stories by current status and priority and
  include zero-valued categories in the response.
- Comments and status history are separate records to avoid unbounded arrays
  in the story document. Coupled status/comment/event updates use a
  transaction.
