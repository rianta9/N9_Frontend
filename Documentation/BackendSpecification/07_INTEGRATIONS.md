# External Integrations Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2025-12-31 |
| Status | Approved |
| Owner | Platform Engineering Team |
| Review Cycle | Quarterly |

---

## 2. Overview

### 2.1 Purpose
This document defines all **external service integrations** for the N9 platform, including APIs, protocols, security requirements, and operational considerations.

### 2.2 Integration Landscape

```
┌─────────────────────────────────────────────────────────────────────┐
│                      N9 INTEGRATION LANDSCAPE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                         ┌──────────────────┐                        │
│                         │    N9 Platform   │                        │
│                         └────────┬─────────┘                        │
│                                  │                                   │
│    ┌─────────────────────────────┼─────────────────────────────┐    │
│    │                             │                             │    │
│    │  ┌──────────┐  ┌──────────┐│┌──────────┐  ┌──────────┐  │    │
│    │  │ Payment  │  │  Object  │││   CDN    │  │  Email   │  │    │
│    │  │ Gateway  │  │ Storage  │││          │  │ Service  │  │    │
│    │  └──────────┘  └──────────┘│└──────────┘  └──────────┘  │    │
│    │                             │                             │    │
│    │  ┌──────────┐  ┌──────────┐│┌──────────┐  ┌──────────┐  │    │
│    │  │   AI     │  │  Push    │││  Search  │  │   APM    │  │    │
│    │  │ Provider │  │  Notif.  │││  Engine  │  │ Provider │  │    │
│    │  └──────────┘  └──────────┘│└──────────┘  └──────────┘  │    │
│    │                             │                             │    │
│    └─────────────────────────────┴─────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Integration Categories

| Category | Integrations | Criticality |
|----------|--------------|-------------|
| **Payment** | Stripe, PayPal, VNPay | Critical |
| **Storage** | AWS S3, Azure Blob, MinIO | Critical |
| **Delivery** | CloudFront, Cloudflare | High |
| **Communication** | SendGrid, FCM, APNs | High |
| **AI Services** | OpenAI, Azure AI, Google AI | Medium |
| **Observability** | Datadog, New Relic, Prometheus | Medium |

---

## 3. Payment Gateway Integration

### 3.1 Supported Providers

| Provider | Markets | Features | Priority |
|----------|---------|----------|----------|
| **Stripe** | Global | Cards, Wallets, Subscriptions | Primary |
| **PayPal** | Global | PayPal Balance, Cards | Secondary |
| **VNPay** | Vietnam | Bank Transfer, QR | Regional |
| **MoMo** | Vietnam | E-Wallet | Regional (Future) |

### 3.2 Payment Flow Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      PAYMENT FLOW                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Client              N9 Server           Payment Gateway             │
│    │                     │                      │                    │
│    │  1. Select package  │                      │                    │
│    │────────────────────▶│                      │                    │
│    │                     │                      │                    │
│    │                     │  2. Create Payment   │                    │
│    │                     │     Intent           │                    │
│    │                     │─────────────────────▶│                    │
│    │                     │                      │                    │
│    │                     │  3. Client Secret    │                    │
│    │                     │◀─────────────────────│                    │
│    │                     │                      │                    │
│    │  4. Redirect/SDK    │                      │                    │
│    │◀────────────────────│                      │                    │
│    │                     │                      │                    │
│    │  5. Complete Payment│                      │                    │
│    │─────────────────────────────────────────────▶│                   │
│    │                     │                      │                    │
│    │                     │  6. Webhook          │                    │
│    │                     │◀─────────────────────│                    │
│    │                     │                      │                    │
│    │                     │  7. Verify & Process │                    │
│    │                     │  8. Credit Wallet    │                    │
│    │                     │                      │                    │
│    │  9. Confirmation    │                      │                    │
│    │◀────────────────────│                      │                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Stripe Integration

#### 3.3.1 API Endpoints Used

| Endpoint | Purpose | Method |
|----------|---------|--------|
| `/v1/payment_intents` | Create payment | POST |
| `/v1/payment_intents/{id}` | Retrieve status | GET |
| `/v1/payment_intents/{id}/confirm` | Confirm payment | POST |
| `/v1/refunds` | Process refund | POST |
| `/v1/webhook_endpoints` | Manage webhooks | POST/GET |

#### 3.3.2 Webhook Events

| Event | Trigger | Action |
|-------|---------|--------|
| `payment_intent.succeeded` | Payment completed | Credit wallet |
| `payment_intent.payment_failed` | Payment failed | Update status, notify |
| `payment_intent.canceled` | Payment canceled | Release hold |
| `charge.refunded` | Refund processed | Debit wallet |
| `charge.dispute.created` | Dispute opened | Flag transaction |

#### 3.3.3 Configuration

```yaml
stripe:
  api-key: ${STRIPE_SECRET_KEY}
  publishable-key: ${STRIPE_PUBLISHABLE_KEY}
  webhook-secret: ${STRIPE_WEBHOOK_SECRET}
  api-version: "2024-12-18.acacia"
  connect-timeout: 30s
  read-timeout: 80s
  max-retries: 3
  retry-initial-delay: 500ms
```

### 3.4 Webhook Security

| Control | Implementation |
|---------|----------------|
| **Signature Verification** | HMAC-SHA256 with webhook secret |
| **Timestamp Validation** | Reject if > 5 minutes old |
| **Idempotency** | Store event ID, skip duplicates |
| **IP Whitelist** | Optional: Stripe IP ranges |
| **Payload Logging** | Store raw payload (encrypted) |

#### 3.4.1 Webhook Handler

```java
@PostMapping("/webhooks/stripe")
public ResponseEntity<String> handleStripeWebhook(
    @RequestBody String payload,
    @RequestHeader("Stripe-Signature") String signature) {
    
    // 1. Verify signature
    Event event = Webhook.constructEvent(
        payload, signature, webhookSecret);
    
    // 2. Check idempotency
    if (processedEventRepository.exists(event.getId())) {
        return ResponseEntity.ok("Already processed");
    }
    
    // 3. Process based on type
    switch (event.getType()) {
        case "payment_intent.succeeded":
            handlePaymentSuccess(event);
            break;
        case "payment_intent.payment_failed":
            handlePaymentFailure(event);
            break;
        // ... other events
    }
    
    // 4. Record as processed
    processedEventRepository.save(event.getId());
    
    return ResponseEntity.ok("Processed");
}
```

### 3.5 Payout Integration

| Provider | Method | Processing Time | Fees |
|----------|--------|-----------------|------|
| **Stripe Connect** | Bank transfer | 2-7 business days | Platform defined |
| **PayPal Payouts** | PayPal account | Instant to 1 day | $0.25 per payout |
| **Bank Transfer** | Direct transfer | 1-3 business days | Bank fees |

---

## 4. Object Storage Integration

### 4.1 Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OBJECT STORAGE ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                         S3-Compatible                          │ │
│  │                                                                 │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │ │
│  │  │   covers/    │  │   uploads/   │  │  backups/    │         │ │
│  │  │              │  │              │  │              │         │ │
│  │  │ Story covers │  │ User uploads │  │ DB backups   │         │ │
│  │  │ (public CDN) │  │ (temp, scan) │  │ (encrypted)  │         │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │ │
│  │                                                                 │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │ │
│  │  │   avatars/   │  │   exports/   │  │    logs/     │         │ │
│  │  │              │  │              │  │              │         │ │
│  │  │ User avatars │  │ Data exports │  │ Audit logs   │         │ │
│  │  │ (public CDN) │  │ (private)    │  │ (private)    │         │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │ │
│  │                                                                 │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Bucket Configuration

| Bucket | Access | Lifecycle | Encryption | CDN |
|--------|--------|-----------|------------|-----|
| `n9-covers` | Public read | Indefinite | SSE-S3 | Yes |
| `n9-avatars` | Public read | Indefinite | SSE-S3 | Yes |
| `n9-uploads` | Private | 24h temp | SSE-S3 | No |
| `n9-exports` | Private | 7 days | SSE-KMS | No |
| `n9-backups` | Private | 90 days | SSE-KMS | No |
| `n9-logs` | Private | 1 year | SSE-KMS | No |

### 4.3 Upload Flow (Pre-signed URL)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRESIGNED URL UPLOAD FLOW                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Client                N9 Server              S3                     │
│    │                       │                   │                     │
│    │  1. Request upload    │                   │                     │
│    │      {filename, type} │                   │                     │
│    │──────────────────────▶│                   │                     │
│    │                       │                   │                     │
│    │                       │  2. Generate      │                     │
│    │                       │     presigned URL │                     │
│    │                       │     (PUT, 15min)  │                     │
│    │                       │                   │                     │
│    │  3. Return {url, key} │                   │                     │
│    │◀──────────────────────│                   │                     │
│    │                       │                   │                     │
│    │  4. PUT file directly │                   │                     │
│    │───────────────────────────────────────────▶│                    │
│    │                       │                   │                     │
│    │  5. 200 OK            │                   │                     │
│    │◀──────────────────────────────────────────│                     │
│    │                       │                   │                     │
│    │  6. Confirm upload    │                   │                     │
│    │      {key}            │                   │                     │
│    │──────────────────────▶│                   │                     │
│    │                       │                   │                     │
│    │                       │  7. Verify exists │                     │
│    │                       │  8. Scan (async)  │                     │
│    │                       │  9. Move to final │                     │
│    │                       │                   │                     │
│    │  10. {cdnUrl}         │                   │                     │
│    │◀──────────────────────│                   │                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.4 Image Processing Pipeline

| Step | Input | Output | Tool |
|------|-------|--------|------|
| **Validation** | Raw upload | Validated | ImageMagick |
| **Virus Scan** | Validated | Scanned | ClamAV |
| **Optimization** | Scanned | Optimized | Sharp/ImageMagick |
| **Resize** | Optimized | Variants | Sharp |
| **CDN Upload** | Variants | CDN URLs | S3 + CloudFront |

### 4.5 Image Variants

| Variant | Dimensions | Format | Use Case |
|---------|------------|--------|----------|
| `thumbnail` | 150x200 | WebP, JPEG | List view |
| `card` | 300x400 | WebP, JPEG | Card view |
| `cover` | 600x800 | WebP, JPEG | Story detail |
| `original` | As uploaded | Original | Full view |
| `og` | 1200x630 | JPEG | Social sharing |

### 4.6 Storage Configuration

```yaml
storage:
  provider: s3  # s3, azure, minio
  s3:
    region: ap-southeast-1
    endpoint: https://s3.ap-southeast-1.amazonaws.com
    access-key: ${AWS_ACCESS_KEY_ID}
    secret-key: ${AWS_SECRET_ACCESS_KEY}
    buckets:
      covers: n9-covers
      avatars: n9-avatars
      uploads: n9-uploads
      exports: n9-exports
    presigned-url-expiry: 15m
    max-file-size: 10MB
    allowed-types:
      - image/jpeg
      - image/png
      - image/webp
      - image/gif
```

---

## 5. CDN Integration

### 5.1 CDN Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CDN ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User Request                                                        │
│       │                                                              │
│       ▼                                                              │
│  ┌────────────────┐                                                  │
│  │   CDN Edge     │  Cache HIT → Return cached                       │
│  │   (CloudFront) │  Cache MISS ↓                                    │
│  └───────┬────────┘                                                  │
│          │                                                           │
│          ▼                                                           │
│  ┌────────────────┐     ┌────────────────┐                          │
│  │  Origin Shield │────▶│   S3 Origin    │                          │
│  │  (Regional)    │     │                │                          │
│  └────────────────┘     └────────────────┘                          │
│                                                                      │
│  Cache Behavior:                                                     │
│  /covers/*     → Cache 1 year, versioned URLs                       │
│  /avatars/*    → Cache 1 year, versioned URLs                       │
│  /api/*        → No cache, pass to API origin                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Cache Configuration

| Path Pattern | TTL | Cache Key | Invalidation |
|--------------|-----|-----------|--------------|
| `/covers/*` | 1 year | Path only | Versioned URL |
| `/avatars/*` | 1 year | Path only | Versioned URL |
| `/assets/*` | 1 year | Path only | Deploy hash |
| `/api/public/*` | 5 min | Path + Query | TTL expiry |
| `/api/*` | No cache | N/A | N/A |

### 5.3 CDN Headers

| Header | Value | Purpose |
|--------|-------|---------|
| `Cache-Control` | `public, max-age=31536000, immutable` | Browser caching |
| `Content-Type` | Auto-detected | MIME type |
| `Content-Encoding` | `gzip`, `br` | Compression |
| `X-Content-Type-Options` | `nosniff` | Security |
| `Access-Control-Allow-Origin` | Domain whitelist | CORS |

---

## 6. Email Service Integration

### 6.1 Email Types

| Type | Template | Trigger | Priority |
|------|----------|---------|----------|
| **Verification** | `email-verify` | Registration | High |
| **Password Reset** | `password-reset` | Reset request | High |
| **Welcome** | `welcome` | Email verified | Medium |
| **New Chapter** | `new-chapter` | Chapter published | Low |
| **Donation Received** | `donation-received` | Donation made | Medium |
| **Payout Processed** | `payout-processed` | Payout completed | High |
| **Account Warning** | `account-warning` | Moderation action | High |
| **Weekly Digest** | `weekly-digest` | Scheduled | Low |

### 6.2 Email Provider Configuration

```yaml
email:
  provider: sendgrid  # sendgrid, ses, smtp
  sendgrid:
    api-key: ${SENDGRID_API_KEY}
    from-email: noreply@n9.example.com
    from-name: N9 Platform
    templates:
      email-verify: d-abc123...
      password-reset: d-def456...
      welcome: d-ghi789...
  rate-limits:
    per-user-per-hour: 10
    global-per-minute: 100
  retry:
    max-attempts: 3
    initial-delay: 1s
    multiplier: 2
```

### 6.3 Email Delivery Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EMAIL DELIVERY FLOW                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Event                Email Service           SendGrid               │
│    │                       │                      │                  │
│    │  1. Email requested   │                      │                  │
│    │──────────────────────▶│                      │                  │
│    │                       │                      │                  │
│    │                       │  2. Check rate limit │                  │
│    │                       │  3. Check preferences│                  │
│    │                       │  4. Render template  │                  │
│    │                       │                      │                  │
│    │                       │  5. Send via API     │                  │
│    │                       │─────────────────────▶│                  │
│    │                       │                      │                  │
│    │                       │  6. Queued/Sent      │                  │
│    │                       │◀─────────────────────│                  │
│    │                       │                      │                  │
│    │                       │  7. Webhook events   │                  │
│    │                       │  (delivered, opened, │                  │
│    │                       │   clicked, bounced)  │                  │
│    │                       │◀─────────────────────│                  │
│    │                       │                      │                  │
│    │                       │  8. Update metrics   │                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.4 Email Webhook Events

| Event | Action |
|-------|--------|
| `delivered` | Mark as delivered |
| `opened` | Track engagement |
| `clicked` | Track engagement |
| `bounced` | Mark email invalid |
| `spam_report` | Unsubscribe user |
| `unsubscribed` | Update preferences |

---

## 7. Push Notification Integration

### 7.1 Push Providers

| Platform | Provider | Protocol |
|----------|----------|----------|
| **Web** | Firebase Cloud Messaging | HTTP v1 |
| **Android** | Firebase Cloud Messaging | HTTP v1 |
| **iOS** | Apple Push Notification Service | HTTP/2 |

### 7.2 Push Notification Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                  PUSH NOTIFICATION FLOW                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. User registers device token                                      │
│     ┌────────┐     ┌──────────┐                                     │
│     │ Client │────▶│ N9 API   │────▶ Store token                    │
│     └────────┘     └──────────┘                                     │
│                                                                      │
│  2. Event triggers notification                                      │
│     ┌──────────┐     ┌──────────────┐     ┌─────────────┐          │
│     │  Event   │────▶│ Notification │────▶│ Push Queue  │          │
│     │  Bus     │     │   Service    │     │             │          │
│     └──────────┘     └──────────────┘     └──────┬──────┘          │
│                                                   │                  │
│  3. Deliver to provider                           │                  │
│     ┌──────────────┐     ┌─────────┐     ┌───────▼──────┐          │
│     │ FCM / APNs   │◀────│  Push   │◀────│ Push Worker  │          │
│     │              │     │  API    │     │              │          │
│     └──────┬───────┘     └─────────┘     └──────────────┘          │
│            │                                                         │
│  4. Deliver to device                                                │
│     ┌──────▼───────┐                                                │
│     │   Device     │                                                │
│     └──────────────┘                                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.3 Push Configuration

```yaml
push:
  fcm:
    project-id: ${FCM_PROJECT_ID}
    credentials-file: ${FCM_CREDENTIALS_PATH}
  apns:
    key-id: ${APNS_KEY_ID}
    team-id: ${APNS_TEAM_ID}
    bundle-id: com.n9.app
    key-file: ${APNS_KEY_PATH}
    production: true
  batch-size: 500
  retry:
    max-attempts: 3
    delay: 1s
```

### 7.4 Push Payload Format

```json
{
  "notification": {
    "title": "New Chapter Available",
    "body": "Story XYZ has a new chapter: Chapter 10"
  },
  "data": {
    "type": "NEW_CHAPTER",
    "storyId": "uuid",
    "chapterId": "uuid",
    "deepLink": "n9://story/uuid/chapter/uuid"
  },
  "android": {
    "priority": "high",
    "notification": {
      "channel_id": "chapters"
    }
  },
  "apns": {
    "headers": {
      "apns-priority": "10"
    },
    "payload": {
      "aps": {
        "badge": 1,
        "sound": "default"
      }
    }
  }
}
```

---

## 8. AI Provider Integration

### 8.1 AI Services Overview

| Service | Provider Options | Use Case |
|---------|------------------|----------|
| **Content Moderation** | OpenAI, Azure AI, Google | Policy compliance |
| **Translation** | DeepL, Google Translate, Azure | Multi-language |
| **Text Analysis** | OpenAI, Anthropic | Quality scoring |

### 8.2 AI Provider Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AI PROVIDER ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                       AI Service Layer                         │ │
│  │                                                                 │ │
│  │  ┌──────────────┐                                              │ │
│  │  │  AI Router   │  Load balance, fallback, circuit breaker     │ │
│  │  └──────┬───────┘                                              │ │
│  │         │                                                       │ │
│  │         ├───────────────┬───────────────┬───────────────┐      │ │
│  │         │               │               │               │      │ │
│  │         ▼               ▼               ▼               ▼      │ │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   │ │
│  │  │  OpenAI  │   │ Azure AI │   │ DeepL    │   │ Google   │   │ │
│  │  │ (Primary)│   │(Fallback)│   │(Transl.) │   │(Fallback)│   │ │
│  │  └──────────┘   └──────────┘   └──────────┘   └──────────┘   │ │
│  │                                                                 │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.3 Content Moderation Integration

#### 8.3.1 Request Format

```json
{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "You are a content moderator. Analyze the following content for policy violations..."
    },
    {
      "role": "user",
      "content": "{chapter_content}"
    }
  ],
  "temperature": 0,
  "max_tokens": 1000,
  "response_format": { "type": "json_object" }
}
```

#### 8.3.2 Response Format (Normalized)

```json
{
  "decision": "APPROVE|REVIEW|REJECT",
  "confidence": 0.95,
  "categories": [
    {
      "code": "VIOLENCE",
      "severity": "LOW",
      "confidence": 0.85
    }
  ],
  "issues": [
    {
      "type": "CONTENT_WARNING",
      "description": "Contains mild violence",
      "location": { "start": 1500, "end": 1600 }
    }
  ],
  "suggestions": [
    "Consider adding content warning for violence"
  ]
}
```

### 8.4 Translation Integration

#### 8.4.1 Configuration

```yaml
translation:
  provider: deepl  # deepl, google, azure
  deepl:
    api-key: ${DEEPL_API_KEY}
    endpoint: https://api.deepl.com/v2
  supported-languages:
    - en
    - vi
    - zh
    - ja
    - ko
  chunk-size: 5000  # characters
  rate-limit:
    requests-per-minute: 30
    characters-per-month: 500000
```

#### 8.4.2 Translation Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TRANSLATION FLOW                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Chunk large content                                              │
│     "Chapter content..." → ["chunk1", "chunk2", ...]                │
│                                                                      │
│  2. Parallel translate chunks                                        │
│     chunk1 ─────▶ DeepL ─────▶ translated1                          │
│     chunk2 ─────▶ DeepL ─────▶ translated2                          │
│     chunk3 ─────▶ DeepL ─────▶ translated3                          │
│                                                                      │
│  3. Merge and validate                                               │
│     [translated1, translated2, ...] → "Full translation"            │
│                                                                      │
│  4. Quality check                                                    │
│     - Character ratio validation                                     │
│     - Language detection verification                                │
│     - Formatting preservation check                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.5 AI Rate Limiting & Cost Control

| Control | Implementation |
|---------|----------------|
| **Per-User Limit** | Max 10 submissions/hour |
| **Per-IP Limit** | Max 20 requests/hour |
| **Global Budget** | Monthly token/cost cap |
| **Queue Priority** | Premium users processed first |
| **Circuit Breaker** | Open on 3 failures in 60s |

### 8.6 AI Provider Configuration

```yaml
ai:
  providers:
    openai:
      api-key: ${OPENAI_API_KEY}
      organization: ${OPENAI_ORG}
      model: gpt-4-turbo
      max-tokens: 4000
      temperature: 0
      timeout: 60s
      retry:
        max-attempts: 3
        delay: 2s
    azure:
      endpoint: ${AZURE_AI_ENDPOINT}
      api-key: ${AZURE_AI_KEY}
      deployment: gpt-4
      api-version: "2024-02-15-preview"
  routing:
    moderation:
      primary: openai
      fallback: azure
    translation:
      primary: deepl
      fallback: google
  cost-limits:
    daily-budget-usd: 100
    monthly-budget-usd: 2000
    alert-threshold: 0.8
```

---

## 9. Search Engine Integration

### 9.1 Elasticsearch Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                  ELASTICSEARCH ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Elasticsearch Cluster                      │   │
│  │                                                               │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │                     Indices                             │  │   │
│  │  │                                                         │  │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │   │
│  │  │  │   stories    │  │   chapters   │  │   authors    │  │  │   │
│  │  │  │   (3 shards) │  │   (5 shards) │  │   (1 shard)  │  │  │   │
│  │  │  └──────────────┘  └──────────────┘  └──────────────┘  │  │   │
│  │  │                                                         │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  │                                                               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Index Sync: Event-driven (outbox) + Periodic full reindex          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 Index Mappings

```json
{
  "stories": {
    "mappings": {
      "properties": {
        "id": { "type": "keyword" },
        "title": { 
          "type": "text",
          "analyzer": "standard",
          "fields": {
            "keyword": { "type": "keyword" },
            "autocomplete": { "type": "text", "analyzer": "autocomplete" }
          }
        },
        "description": { "type": "text" },
        "authorId": { "type": "keyword" },
        "authorName": { "type": "text" },
        "categories": { "type": "keyword" },
        "tags": { "type": "keyword" },
        "status": { "type": "keyword" },
        "visibility": { "type": "keyword" },
        "contentRating": { "type": "keyword" },
        "chapterCount": { "type": "integer" },
        "wordCount": { "type": "long" },
        "viewCount": { "type": "long" },
        "likeCount": { "type": "long" },
        "avgRating": { "type": "float" },
        "publishedAt": { "type": "date" },
        "updatedAt": { "type": "date" }
      }
    }
  }
}
```

### 9.3 Search Configuration

```yaml
elasticsearch:
  hosts:
    - https://es-node1.example.com:9200
    - https://es-node2.example.com:9200
  username: ${ES_USERNAME}
  password: ${ES_PASSWORD}
  connection-timeout: 5s
  socket-timeout: 30s
  indices:
    stories:
      name: n9-stories
      shards: 3
      replicas: 1
    chapters:
      name: n9-chapters
      shards: 5
      replicas: 1
```

---

## 10. Observability Integration

### 10.1 Metrics Export (Prometheus)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: n9-api
      environment: ${ENVIRONMENT}
```

### 10.2 APM Integration

| Provider | Protocol | Use Case |
|----------|----------|----------|
| **Datadog** | Agent | Full APM suite |
| **New Relic** | Agent | APM + infrastructure |
| **Jaeger** | OpenTelemetry | Distributed tracing |
| **Prometheus** | Pull | Metrics |
| **Grafana** | N/A | Dashboards |

### 10.3 Distributed Tracing

```yaml
tracing:
  enabled: true
  exporter: otlp
  otlp:
    endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT}
  sampling:
    probability: 0.1  # 10% sampling
    always-sample-errors: true
  propagation:
    - W3C
    - B3
```

---

## 11. Integration Security Summary

| Integration | Authentication | Encryption | Secrets Storage |
|-------------|----------------|------------|-----------------|
| **Stripe** | API Key | TLS 1.2+ | Vault |
| **S3** | IAM / Access Keys | TLS + SSE | Vault |
| **SendGrid** | API Key | TLS 1.2+ | Vault |
| **FCM** | Service Account | TLS 1.2+ | Vault |
| **OpenAI** | API Key | TLS 1.2+ | Vault |
| **Elasticsearch** | Basic Auth | TLS 1.2+ | Vault |

---

## 12. Integration Health Checks

| Integration | Health Check | Interval | Timeout |
|-------------|--------------|----------|---------|
| **PostgreSQL** | SELECT 1 | 15s | 5s |
| **Redis** | PING | 10s | 2s |
| **Elasticsearch** | Cluster health | 30s | 5s |
| **S3** | HEAD bucket | 60s | 10s |
| **Payment Gateway** | API status | 60s | 10s |
| **Email Service** | API status | 60s | 10s |
| **AI Provider** | Models list | 60s | 30s |

---

## 13. References

### 13.1 Design Documents
- [03_PAYMENTS_COMPONENT.md](../Design/Components/03_PAYMENTS_COMPONENT.md)
- [11_MEDIA_FILES_COMPONENT.md](../Design/Components/11_MEDIA_FILES_COMPONENT.md)
- [13_AI_AUTOMATION_COMPONENT.md](../Design/Components/13_AI_AUTOMATION_COMPONENT.md)

### 13.2 Related Specifications
- [05_SECURITY_AND_COMPLIANCE.md](05_SECURITY_AND_COMPLIANCE.md)
- [06_REALTIME_AND_EVENTS.md](06_REALTIME_AND_EVENTS.md)

### 13.3 External Documentation
- [Stripe API Documentation](https://stripe.com/docs/api)
- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [SendGrid API Documentation](https://docs.sendgrid.com/)
- [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)
- [OpenAI API Documentation](https://platform.openai.com/docs)
