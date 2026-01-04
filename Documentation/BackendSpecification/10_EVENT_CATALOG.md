# Event Catalog (Target)

## 1. Event Envelope
All events use a common envelope for observability, idempotency, and replay.

```json
{
  "id": "uuid",
  "type": "chapter.published",
  "occurredAt": "2025-12-14T12:00:00Z",
  "version": 1,
  "producer": "n9-api",
  "traceId": "...",
  "actor": { "accountId": "...", "role": "AUTHOR" },
  "data": { }
}
```

## 2. Delivery & Ordering
- **Delivery**: at-least-once.
- **Consumer idempotency**: dedupe on `id` (and/or natural keys per event).
- **Ordering**: best-effort per aggregate id (e.g., `storyId`) where supported.

## 3. Core Events

### 3.1 Content
#### `story.created` (v1)
```json
{ "storyId": "...", "authorId": "...", "title": "...", "status": "DRAFT" }
```

#### `story.updated` (v1)
```json
{ "storyId": "...", "fields": ["title","description","cover"], "updatedAt": "..." }
```

#### `chapter.published` (v1)
```json
{ "storyId": "...", "chapterId": "...", "index": 12, "publishedAt": "..." }
```

### 3.2 Interactions
#### `interaction.followed` (v1)
```json
{ "storyId": "...", "accountId": "..." }
```

#### `interaction.liked` (v1)
```json
{ "targetType": "STORY|CHAPTER", "targetId": "...", "accountId": "..." }
```

#### `review.created` (v1)
```json
{ "storyId": "...", "reviewId": "...", "accountId": "...", "rating": 4.5 }
```

#### `comment.created` (v1)
```json
{ "chapterId": "...", "commentId": "...", "accountId": "...", "parentId": null }
```

### 3.3 Reading
#### `reading.progress.updated` (v1)
```json
{ "accountId": "...", "storyId": "...", "chapterId": "...", "chapterIndex": 12 }
```

### 3.4 Wallet / Payments
#### `payment.succeeded` (v1)
```json
{ "paymentId": "...", "accountId": "...", "amount": 10.0, "currency": "USD", "provider": "..." }
```

#### `wallet.credited` (v1)
```json
{ "accountId": "...", "amount": 10.0, "reason": "DEPOSIT|REFUND" , "refId": "paymentId" }
```

#### `wallet.debited` (v1)
```json
{ "accountId": "...", "amount": 2.0, "reason": "DONATION|UNLOCK" , "refId": "storyDonateId|unlockId" }
```

#### `payout.requested` (v1)
```json
{ "payoutId": "...", "authorId": "...", "amount": 50.0, "currency": "USD" }
```

#### `payout.completed` (v1)
```json
{ "payoutId": "...", "authorId": "...", "providerTxnId": "...", "completedAt": "..." }
```

### 3.5 Moderation
#### `moderation.reported` (v1)
```json
{ "reportType": "STORY|CHAPTER|COMMENT", "targetId": "...", "reportId": "...", "reasonCode": "..." }
```

#### `moderation.actioned` (v1)
```json
{ "action": "HIDE|DELETE|WARN|BAN", "targetType": "STORY|CHAPTER|COMMENT|ACCOUNT", "targetId": "..." }
```

### 3.6 AI Automation
#### `content.submitted` (v1)
```json
{ "submissionId": "...", "targetType": "STORY|CHAPTER", "targetId": "...", "authorId": "...", "checksum": "..." }
```

#### `ai.review.completed` (v1)
```json
{ "submissionId": "...", "decision": "PASS|NEEDS_REVIEW|BLOCK", "riskScore": 0.12, "labels": ["SPAM"], "policyVersion": "policy-key@v3" }
```

#### `moderation.approved` (v1)
```json
{ "submissionId": "...", "targetType": "STORY|CHAPTER", "targetId": "...", "approvedBy": "accountId", "policyVersion": "policy-key@v3" }
```

#### `moderation.rejected` (v1)
```json
{ "submissionId": "...", "targetType": "STORY|CHAPTER", "targetId": "...", "rejectedBy": "accountId", "reason": "..." }
```

#### `translation.requested` (v1)
```json
{ "jobId": "...", "targetType": "STORY|CHAPTER", "targetId": "...", "languages": ["en","vi"], "requestedBy": "accountId" }
```

#### `translation.completed` (v1)
```json
{ "jobId": "...", "targetType": "STORY|CHAPTER", "targetId": "...", "language": "en", "artifactId": "..." }
```

#### `translation.failed` (v1)
```json
{ "jobId": "...", "targetType": "STORY|CHAPTER", "targetId": "...", "errorCode": "..." }
```

## 4. Suggested Producers / Consumers
- Stories: produces `story.*`, `chapter.published`.
- Interactions: produces `interaction.*`, `review.created`, `comment.created`, `moderation.reported`.
- Readings: produces `reading.progress.updated`.
- Payments: produces `payment.*`, `wallet.*`, `payout.*`.
- Notifications: consumes most social + payout + system events.
- Statistics/Rankings: consumes content, interactions, readings, payments.
- Recommendation: consumes readings + interactions.

## 5. Outbox Storage
If using outbox:
- Table includes `id`, `type`, `occurredAt`, `aggregateType`, `aggregateId`, `payload`, `status`, `attempts`.
- Worker publishes and marks as delivered.
