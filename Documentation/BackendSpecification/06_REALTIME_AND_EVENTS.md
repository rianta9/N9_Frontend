# Real-time & Events Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2025-12-31 |
| Status | Approved |
| Owner | Platform Architecture Team |
| Review Cycle | Quarterly |

---

## 2. Overview

### 2.1 Purpose
This document defines the **event-driven architecture** and **real-time communication** patterns for the N9 platform, enabling decoupled, scalable, and responsive user experiences.

### 2.2 Design Goals

| Goal | Description |
|------|-------------|
| **Decoupling** | Separate write paths from derived data updates |
| **Scalability** | Handle millions of events per day |
| **Reliability** | Guarantee at-least-once delivery |
| **Latency** | Sub-second real-time updates where needed |
| **Observability** | Full traceability of event flows |

### 2.3 Event-Driven Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EVENT-DRIVEN ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐ │
│  │   API       │    │  Domain     │    │      Event Store        │ │
│  │   Request   │───▶│  Service    │───▶│   (Outbox Table)        │ │
│  └─────────────┘    └─────────────┘    └───────────┬─────────────┘ │
│                                                     │               │
│                                        ┌────────────▼────────────┐ │
│                                        │    Outbox Publisher     │ │
│                                        │    (Polling Worker)     │ │
│                                        └────────────┬────────────┘ │
│                                                     │               │
│  ┌──────────────────────────────────────────────────▼──────────────┐│
│  │                      EVENT BUS                                  ││
│  │   (Redis Streams / Kafka / In-Memory for Phase 1)               ││
│  └──────────────────────────────────────────────────┬──────────────┘│
│                                                     │               │
│         ┌───────────────┬───────────────┬───────────┼───────────┐  │
│         │               │               │           │           │  │
│         ▼               ▼               ▼           ▼           ▼  │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────┐ ┌─────┐│
│  │Notification│ │ Statistics │ │  Search    │ │ Ranking  │ │ RT  ││
│  │  Consumer  │ │  Consumer  │ │  Indexer   │ │ Consumer │ │ Push││
│  └────────────┘ └────────────┘ └────────────┘ └──────────┘ └─────┘│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Event Categories

### 3.1 Event Classification

| Category | Characteristics | Examples |
|----------|-----------------|----------|
| **Domain Events** | Business state changes | `story.published`, `payment.completed` |
| **Integration Events** | Cross-component communication | `notification.send`, `search.index` |
| **System Events** | Infrastructure/operational | `health.degraded`, `backup.completed` |
| **Audit Events** | Security/compliance | `user.login`, `permission.changed` |

### 3.2 Event Priority Levels

| Priority | Description | Processing SLA | Examples |
|----------|-------------|----------------|----------|
| **P0 Critical** | Financial, security | < 1 second | `wallet.debited`, `payment.completed` |
| **P1 High** | User-facing real-time | < 5 seconds | `chapter.published`, `comment.created` |
| **P2 Normal** | Standard async | < 60 seconds | `follow.created`, `view.recorded` |
| **P3 Low** | Background/batch | < 5 minutes | `statistics.aggregate`, `ranking.refresh` |

---

## 4. Complete Event Catalog

### 4.1 Stories Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `story.created` | Author creates story | `{storyId, authorId, title, status}` | Search, Statistics |
| `story.updated` | Author updates metadata | `{storyId, changes[], version}` | Search, Cache |
| `story.published` | Story visibility → VISIBLE | `{storyId, authorId, publishedAt}` | Notification, Search, Ranking |
| `story.completed` | Status → COMPLETED | `{storyId, authorId, completedAt}` | Notification, Statistics |
| `story.deleted` | Soft delete | `{storyId, authorId, deletedAt}` | Search, Statistics, Cache |

### 4.2 Chapters Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `chapter.created` | Author creates chapter | `{chapterId, storyId, index, status}` | Statistics |
| `chapter.published` | Chapter visibility → VISIBLE | `{chapterId, storyId, authorId, index, publishedAt}` | Notification, Search, Statistics |
| `chapter.updated` | Content changed | `{chapterId, storyId, contentHash, version}` | Search, AI (re-review) |
| `chapter.deleted` | Soft delete | `{chapterId, storyId, deletedAt}` | Search, Statistics |

### 4.3 Users Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `user.registered` | New account created | `{userId, username, email, role}` | Welcome email, Statistics |
| `user.verified` | Email verified | `{userId, verifiedAt}` | Account service |
| `user.login` | Successful login | `{userId, ip, userAgent, sessionId}` | Audit, Security |
| `user.logout` | Logout initiated | `{userId, sessionId}` | Session cleanup |
| `user.role.changed` | Role updated | `{userId, oldRole, newRole, changedBy}` | Audit, Permissions |
| `user.suspended` | Account suspended | `{userId, reason, duration, suspendedBy}` | Notification, Session |
| `user.banned` | Account banned | `{userId, reason, bannedBy}` | Session, Content |

### 4.4 Interactions Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `follow.created` | User follows story/author | `{followId, followerId, targetType, targetId}` | Notification, Statistics |
| `follow.deleted` | User unfollows | `{followId, followerId, targetType, targetId}` | Statistics |
| `like.created` | User likes content | `{likeId, userId, targetType, targetId}` | Statistics, Ranking |
| `like.deleted` | User unlikes | `{likeId, userId, targetType, targetId}` | Statistics, Ranking |
| `review.created` | User posts review | `{reviewId, userId, storyId, rating}` | Notification, Statistics, Ranking |
| `review.updated` | User edits review | `{reviewId, userId, storyId, oldRating, newRating}` | Statistics, Ranking |
| `review.deleted` | User deletes review | `{reviewId, userId, storyId}` | Statistics, Ranking |
| `comment.created` | User posts comment | `{commentId, userId, chapterId, parentId}` | Notification, Statistics |
| `comment.deleted` | Comment removed | `{commentId, userId, chapterId}` | Statistics |
| `report.created` | User reports content | `{reportId, reporterId, targetType, targetId, reason}` | Moderation queue |

### 4.5 Readings Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `reading.started` | User begins reading | `{userId, storyId, chapterId, startedAt}` | Statistics |
| `reading.progress` | Progress update | `{userId, storyId, chapterId, position, timestamp}` | Recommendation |
| `reading.completed` | Chapter finished | `{userId, storyId, chapterId, duration}` | Statistics, Recommendation |
| `reading.session.ended` | Session timeout/close | `{sessionId, userId, storyId, totalDuration}` | Statistics |
| `bookmark.created` | User creates bookmark | `{bookmarkId, userId, chapterId, position}` | - |

### 4.6 Payments Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `wallet.credited` | Coins added | `{walletId, userId, amount, source, txId}` | Notification |
| `wallet.debited` | Coins spent | `{walletId, userId, amount, purpose, txId}` | - |
| `payment.initiated` | Payment started | `{paymentId, userId, amount, provider}` | - |
| `payment.completed` | Payment successful | `{paymentId, userId, amount, coins, provider}` | Notification, Wallet |
| `payment.failed` | Payment failed | `{paymentId, userId, error, provider}` | Notification |
| `donation.sent` | User donates | `{donationId, donorId, recipientId, storyId, amount}` | Notification, Statistics |
| `chapter.unlocked` | Premium chapter unlocked | `{userId, chapterId, storyId, coins}` | Statistics, Wallet |
| `payout.requested` | Author requests payout | `{payoutId, authorId, amount}` | Admin notification |
| `payout.approved` | Admin approves payout | `{payoutId, authorId, amount, approvedBy}` | Notification |
| `payout.completed` | Payout processed | `{payoutId, authorId, amount, reference}` | Notification |
| `payout.rejected` | Payout rejected | `{payoutId, authorId, reason, rejectedBy}` | Notification |

### 4.7 Notifications Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `notification.created` | Notification generated | `{notificationId, userId, type, data}` | Push delivery |
| `notification.sent` | Delivered to channel | `{notificationId, channel, sentAt}` | Analytics |
| `notification.read` | User reads notification | `{notificationId, userId, readAt}` | Badge update |
| `notification.batch.ready` | Digest ready to send | `{batchId, userId, count}` | Email delivery |

### 4.8 Search Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `search.index.story` | Story needs indexing | `{storyId, action: create/update/delete}` | Elasticsearch |
| `search.index.chapter` | Chapter needs indexing | `{chapterId, storyId, action}` | Elasticsearch |
| `search.query` | User searches | `{query, userId, filters, resultCount}` | Analytics |

### 4.9 Statistics Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `stats.view.recorded` | View counted | `{storyId, chapterId, userId, timestamp}` | Aggregator |
| `stats.aggregation.daily` | Daily rollup complete | `{date, storiesProcessed}` | - |
| `ranking.refresh.triggered` | Ranking update due | `{rankingType, period}` | Ranking service |
| `ranking.updated` | Rankings recalculated | `{rankingType, period, timestamp}` | Cache invalidation |

### 4.10 Moderation Component Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `moderation.report.assigned` | Report assigned to mod | `{reportId, moderatorId}` | Mod dashboard |
| `moderation.action.taken` | Enforcement applied | `{enforcementId, targetType, targetId, action}` | Notification, Audit |
| `moderation.appeal.created` | User appeals | `{appealId, enforcementId, userId}` | Admin queue |
| `moderation.appeal.decided` | Appeal resolved | `{appealId, decision, decidedBy}` | Notification |

### 4.11 AI Automation Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `content.submitted` | Content sent for AI review | `{submissionId, targetType, targetId, authorId}` | AI processor |
| `ai.job.created` | AI job queued | `{jobId, submissionId, jobType, provider}` | - |
| `ai.job.started` | AI processing begun | `{jobId, startedAt}` | Monitoring |
| `ai.job.completed` | AI processing done | `{jobId, submissionId, decision, confidence}` | Workflow |
| `ai.job.failed` | AI processing failed | `{jobId, submissionId, error, willRetry}` | Alerting |
| `ai.review.approved` | Auto/manual approval | `{submissionId, approvedBy, isAuto}` | Notification, Content |
| `ai.review.rejected` | Auto/manual rejection | `{submissionId, rejectedBy, reason, isAuto}` | Notification |
| `ai.override.recorded` | Human overrides AI | `{submissionId, originalDecision, newDecision, reason, moderatorId}` | Audit, ML feedback |
| `translation.requested` | Translation job created | `{jobId, targetType, targetId, languages[]}` | Translation processor |
| `translation.completed` | Translation done | `{jobId, targetType, targetId, language, quality}` | Content, Cache |
| `translation.failed` | Translation failed | `{jobId, language, error}` | Alerting |

### 4.12 Advertising Events

| Event Type | Trigger | Payload | Consumers |
|------------|---------|---------|-----------|
| `ad.impression` | Banner displayed | `{bannerId, campaignId, userId, placement}` | Analytics, Billing |
| `ad.click` | Banner clicked | `{bannerId, campaignId, userId, impressionId}` | Analytics, Billing |
| `campaign.budget.low` | Budget < 20% | `{campaignId, advertiserId, remaining}` | Alerting |
| `campaign.budget.exhausted` | Budget depleted | `{campaignId, advertiserId}` | Campaign pause |

---

## 5. Event Schema Standards

### 5.1 Event Envelope Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["id", "type", "source", "time", "data"],
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique event identifier"
    },
    "type": {
      "type": "string",
      "pattern": "^[a-z]+\\.[a-z]+\\.[a-z]+$",
      "description": "Event type (domain.entity.action)"
    },
    "source": {
      "type": "string",
      "description": "Service that produced the event"
    },
    "time": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp"
    },
    "dataContentType": {
      "type": "string",
      "default": "application/json"
    },
    "dataSchema": {
      "type": "string",
      "format": "uri",
      "description": "Schema URI for data payload"
    },
    "correlationId": {
      "type": "string",
      "format": "uuid",
      "description": "Request correlation ID"
    },
    "causationId": {
      "type": "string",
      "format": "uuid",
      "description": "ID of event that caused this event"
    },
    "data": {
      "type": "object",
      "description": "Event-specific payload"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "version": { "type": "integer" },
        "userId": { "type": "string" },
        "sessionId": { "type": "string" },
        "traceId": { "type": "string" }
      }
    }
  }
}
```

### 5.2 Example Event

```json
{
  "id": "evt-550e8400-e29b-41d4-a716-446655440000",
  "type": "story.chapter.published",
  "source": "n9-api/stories-service",
  "time": "2025-12-31T12:00:00.000Z",
  "dataContentType": "application/json",
  "dataSchema": "https://n9.example.com/schemas/chapter-published/v1",
  "correlationId": "req-123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "chapterId": "chap-uuid",
    "storyId": "story-uuid",
    "authorId": "author-uuid",
    "title": "Chapter 10: The Journey Begins",
    "index": 10,
    "wordCount": 3500,
    "publishedAt": "2025-12-31T12:00:00.000Z"
  },
  "metadata": {
    "version": 1,
    "userId": "author-uuid",
    "traceId": "trace-abc123"
  }
}
```

### 5.3 Schema Evolution Rules

| Rule | Description |
|------|-------------|
| **Backward Compatible** | New fields are optional |
| **Forward Compatible** | Consumers ignore unknown fields |
| **Versioning** | Schema version in `dataSchema` URI |
| **Deprecation** | 6-month notice before field removal |
| **Breaking Changes** | New event type (e.g., `story.published.v2`) |

---

## 6. Event Transport

### 6.1 Outbox Pattern Implementation

```
┌─────────────────────────────────────────────────────────────┐
│                    OUTBOX PATTERN                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              TRANSACTION BOUNDARY                    │   │
│  │                                                      │   │
│  │  1. BEGIN TRANSACTION                                │   │
│  │  2. UPDATE domain_table SET ...                      │   │
│  │  3. INSERT INTO outbox_event (...)                   │   │
│  │  4. COMMIT                                           │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              OUTBOX PUBLISHER (Async)                │   │
│  │                                                      │   │
│  │  LOOP every 100ms:                                   │   │
│  │    5. SELECT * FROM outbox_event                     │   │
│  │       WHERE status = 'PENDING'                       │   │
│  │       ORDER BY created_at LIMIT 100                  │   │
│  │       FOR UPDATE SKIP LOCKED                         │   │
│  │                                                      │   │
│  │    6. Publish to event bus                           │   │
│  │                                                      │   │
│  │    7. UPDATE outbox_event                            │   │
│  │       SET status = 'PUBLISHED',                      │   │
│  │           published_at = NOW()                       │   │
│  │       WHERE id IN (...)                              │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Outbox Table Schema

```sql
CREATE TABLE outbox_event (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_type VARCHAR(50) NOT NULL,
  aggregate_id UUID NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  correlation_id UUID,
  causation_id UUID,
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at TIMESTAMPTZ,
  retry_count INT NOT NULL DEFAULT 0,
  next_retry_at TIMESTAMPTZ,
  error_message TEXT
);

CREATE INDEX idx_outbox_pending ON outbox_event(status, created_at)
  WHERE status = 'PENDING';
CREATE INDEX idx_outbox_retry ON outbox_event(status, next_retry_at)
  WHERE status = 'RETRY';
```

### 6.3 Transport Options by Phase

| Phase | Transport | Use Case |
|-------|-----------|----------|
| **Phase 1** | PostgreSQL Outbox + Polling | MVP, < 1000 events/sec |
| **Phase 2** | Redis Streams | Real-time, < 10K events/sec |
| **Phase 3** | Kafka/Pulsar | High volume, > 10K events/sec |

### 6.4 Redis Streams Configuration

```yaml
redis:
  streams:
    consumer-group: n9-consumers
    streams:
      - name: events:stories
        consumers: 3
        block-timeout: 5000
      - name: events:payments
        consumers: 2
        block-timeout: 1000
      - name: events:notifications
        consumers: 5
        block-timeout: 2000
    pending:
      claim-after: 60000  # 1 minute
      max-retries: 5
```

---

## 7. Event Processing

### 7.1 Consumer Responsibilities

```
┌─────────────────────────────────────────────────────────────┐
│                  CONSUMER FLOW                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. RECEIVE event from transport                            │
│                                                             │
│  2. VALIDATE                                                │
│     - Schema validation                                     │
│     - Required fields present                               │
│                                                             │
│  3. IDEMPOTENCY CHECK                                       │
│     - Check processed_events table                          │
│     - If exists, ACK and skip                               │
│                                                             │
│  4. PROCESS                                                 │
│     - Execute business logic                                │
│     - Within transaction:                                   │
│       - Update state                                        │
│       - Record event ID in processed_events                 │
│                                                             │
│  5. ACK event                                               │
│                                                             │
│  ON FAILURE:                                                │
│  - Log error with correlation ID                            │
│  - Increment retry count                                    │
│  - If retries exhausted → DLQ                               │
│  - NACK for retry                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Idempotency Table

```sql
CREATE TABLE processed_event (
  event_id UUID PRIMARY KEY,
  consumer_group VARCHAR(50) NOT NULL,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT processed_event_uk UNIQUE (event_id, consumer_group)
);

-- Cleanup old entries (keep 7 days)
CREATE INDEX idx_processed_event_cleanup ON processed_event(processed_at);
```

### 7.3 Consumer Groups

| Consumer Group | Events Consumed | Purpose |
|----------------|-----------------|---------|
| `notification-service` | `chapter.published`, `donation.sent`, `comment.created`, etc. | Generate notifications |
| `statistics-aggregator` | `reading.*`, `like.*`, `follow.*`, `view.*` | Update counters |
| `search-indexer` | `story.*`, `chapter.*` | Elasticsearch sync |
| `ranking-calculator` | `like.*`, `view.*`, `follow.*` | Recalculate rankings |
| `recommendation-engine` | `reading.*`, `like.*`, `follow.*` | Update user interests |
| `ai-processor` | `content.submitted` | Process AI jobs |
| `audit-logger` | `user.*`, `moderation.*`, `payment.*` | Audit trail |
| `cache-invalidator` | `story.updated`, `chapter.updated` | Clear stale cache |

### 7.4 Event Routing Matrix

| Event Type | notification | statistics | search | ranking | recommendation | ai | audit |
|------------|--------------|------------|--------|---------|----------------|-----|-------|
| `story.published` | ✓ | ✓ | ✓ | | | | |
| `chapter.published` | ✓ | ✓ | ✓ | | | | |
| `like.created` | | ✓ | | ✓ | ✓ | | |
| `follow.created` | ✓ | ✓ | | | ✓ | | |
| `donation.sent` | ✓ | ✓ | | | | | |
| `reading.completed` | | ✓ | | | ✓ | | |
| `content.submitted` | | | | | | ✓ | |
| `user.role.changed` | | | | | | | ✓ |
| `payment.completed` | ✓ | | | | | | ✓ |

---

## 8. Real-time Communication

### 8.1 WebSocket Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  WEBSOCKET ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐     ┌──────────────┐     ┌────────────────┐  │
│  │  Client  │◄───▶│   Gateway    │◄───▶│  WS Handler    │  │
│  │  (WS)    │     │   (Nginx)    │     │  (Spring)      │  │
│  └──────────┘     └──────────────┘     └───────┬────────┘  │
│                                                 │           │
│                                        ┌────────▼────────┐ │
│                                        │  Redis Pub/Sub  │ │
│                                        │  (Fan-out)      │ │
│                                        └────────┬────────┘ │
│                                                 │           │
│               ┌─────────────┬──────────────────┼──────┐    │
│               │             │                  │      │    │
│        ┌──────▼──────┐ ┌────▼─────┐ ┌─────────▼────┐ │    │
│        │ Notification│ │  Counter │ │  Presence    │ │    │
│        │   Service   │ │  Service │ │  Service     │ │    │
│        └─────────────┘ └──────────┘ └──────────────┘ │    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 WebSocket Connection Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│               WEBSOCKET CONNECTION LIFECYCLE                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CLIENT                              SERVER                 │
│    │                                    │                   │
│    │  1. Connect /ws?token=JWT          │                   │
│    │───────────────────────────────────▶│                   │
│    │                                    │                   │
│    │                           2. Validate token            │
│    │                           3. Register session          │
│    │                           4. Subscribe to channels     │
│    │                                    │                   │
│    │  5. Connected (session_id)         │                   │
│    │◀───────────────────────────────────│                   │
│    │                                    │                   │
│    │  6. Subscribe to topics            │                   │
│    │───────────────────────────────────▶│                   │
│    │                                    │                   │
│    │  7. Push: notification             │                   │
│    │◀───────────────────────────────────│                   │
│    │                                    │                   │
│    │  8. Push: counter update           │                   │
│    │◀───────────────────────────────────│                   │
│    │                                    │                   │
│    │  9. Ping (every 30s)               │                   │
│    │───────────────────────────────────▶│                   │
│    │  10. Pong                          │                   │
│    │◀───────────────────────────────────│                   │
│    │                                    │                   │
│    │  11. Disconnect                    │                   │
│    │───────────────────────────────────▶│                   │
│    │                           12. Cleanup session          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 WebSocket Message Types

| Type | Direction | Purpose | Example |
|------|-----------|---------|---------|
| `connect` | C→S | Initial connection | `{token: "jwt"}` |
| `connected` | S→C | Connection confirmed | `{sessionId: "uuid"}` |
| `subscribe` | C→S | Subscribe to topic | `{topic: "story:uuid"}` |
| `unsubscribe` | C→S | Unsubscribe | `{topic: "story:uuid"}` |
| `notification` | S→C | New notification | `{type, data}` |
| `counter` | S→C | Counter update | `{target, count}` |
| `presence` | S→C | User online status | `{userId, status}` |
| `ping` | C→S | Keepalive | `{}` |
| `pong` | S→C | Keepalive response | `{}` |
| `error` | S→C | Error message | `{code, message}` |

### 8.4 SSE (Server-Sent Events) Fallback

```
GET /api/events/stream
Authorization: Bearer {token}
Accept: text/event-stream

---

event: notification
id: evt-123
data: {"type":"NEW_CHAPTER","storyId":"uuid","title":"Chapter 10"}

event: counter
id: evt-124
data: {"target":"story:uuid","field":"likes","count":1502}

event: heartbeat
id: evt-125
data: {}
```

### 8.5 Channel Structure

| Channel Pattern | Description | Example |
|-----------------|-------------|---------|
| `user:{userId}` | User-specific notifications | `user:abc-123` |
| `story:{storyId}` | Story updates, comments | `story:xyz-456` |
| `chapter:{chapterId}` | Chapter comments | `chapter:def-789` |
| `admin:reports` | Moderation queue | `admin:reports` |
| `global:announcements` | System-wide messages | `global:announcements` |

---

## 9. Failure Handling

### 9.1 Retry Strategy

| Attempt | Delay | Total Time |
|---------|-------|------------|
| 1 | Immediate | 0 |
| 2 | 1 second | 1s |
| 3 | 5 seconds | 6s |
| 4 | 30 seconds | 36s |
| 5 | 2 minutes | 2m 36s |
| 6+ | DLQ | - |

### 9.2 Dead Letter Queue (DLQ)

```sql
CREATE TABLE dead_letter_event (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  original_event_id UUID NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  error_message TEXT NOT NULL,
  retry_count INT NOT NULL,
  consumer_group VARCHAR(50) NOT NULL,
  failed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  reprocessed_at TIMESTAMPTZ,
  reprocess_status VARCHAR(20)
);

CREATE INDEX idx_dlq_pending ON dead_letter_event(consumer_group, failed_at)
  WHERE reprocessed_at IS NULL;
```

### 9.3 DLQ Management

| Action | Description |
|--------|-------------|
| **Review** | Manual inspection of failed events |
| **Replay** | Re-process individual or batch events |
| **Discard** | Mark as permanently failed |
| **Alert** | Notify on DLQ threshold breach |

### 9.4 Backpressure Handling

```
┌─────────────────────────────────────────────────────────────┐
│                  BACKPRESSURE STRATEGY                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PRODUCER SIDE:                                             │
│  - Rate limit event production                              │
│  - Buffer in outbox (bounded)                               │
│  - Circuit breaker on bus failures                          │
│                                                             │
│  CONSUMER SIDE:                                             │
│  - Bounded consumer queue                                   │
│  - Pause consumption when behind                            │
│  - Scale consumers horizontally                             │
│                                                             │
│  MONITORING:                                                │
│  - Queue depth metrics                                      │
│  - Consumer lag alerts                                      │
│  - Processing rate vs publish rate                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Observability

### 10.1 Event Metrics

| Metric | Type | Labels |
|--------|------|--------|
| `events_published_total` | Counter | `event_type`, `source` |
| `events_consumed_total` | Counter | `event_type`, `consumer_group`, `status` |
| `event_processing_duration_seconds` | Histogram | `event_type`, `consumer_group` |
| `consumer_lag` | Gauge | `consumer_group`, `stream` |
| `dlq_size` | Gauge | `consumer_group` |
| `outbox_pending_count` | Gauge | - |
| `websocket_connections` | Gauge | - |
| `websocket_messages_total` | Counter | `type`, `direction` |

### 10.2 Event Tracing

| Attribute | Description |
|-----------|-------------|
| `event.id` | Event identifier |
| `event.type` | Event type |
| `event.correlation_id` | Request correlation |
| `event.causation_id` | Parent event |
| `consumer.group` | Processing consumer |
| `consumer.duration_ms` | Processing time |

### 10.3 Alerting Rules

| Alert | Condition | Severity |
|-------|-----------|----------|
| `EventPublishFailure` | Publish error rate > 1% | Critical |
| `ConsumerLagHigh` | Lag > 10,000 events | Warning |
| `ConsumerLagCritical` | Lag > 100,000 events | Critical |
| `DLQGrowing` | DLQ size increasing | Warning |
| `OutboxBacklog` | Pending > 10,000 | Warning |
| `WebSocketConnectionsDrop` | 50% drop in 5 min | Warning |

---

## 11. References

### 11.1 Design Documents
- [07_NOTIFICATIONS_COMPONENT.md](../Design/Components/07_NOTIFICATIONS_COMPONENT.md)
- [09_STATISTICS_COMPONENT.md](../Design/Components/09_STATISTICS_COMPONENT.md)
- [13_AI_AUTOMATION_COMPONENT.md](../Design/Components/13_AI_AUTOMATION_COMPONENT.md)

### 11.2 Related Specifications
- [10_EVENT_CATALOG.md](10_EVENT_CATALOG.md)
- [03_NON_FUNCTIONAL_REQUIREMENTS.md](03_NON_FUNCTIONAL_REQUIREMENTS.md)

### 11.3 External Standards
- [CloudEvents Specification](https://cloudevents.io/)
- [CQRS/Event Sourcing](https://martinfowler.com/bliki/CQRS.html)
