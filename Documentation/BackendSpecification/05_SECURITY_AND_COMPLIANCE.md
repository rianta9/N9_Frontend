# Security & Compliance Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2025-12-31 |
| Status | Approved |
| Owner | Security Team |
| Review Cycle | Quarterly |
| Classification | Internal |

---

## 2. Overview

### 2.1 Purpose
This document defines the **security architecture**, **controls**, and **compliance requirements** for the N9 platform, ensuring protection of user data, financial transactions, and content integrity.

### 2.2 Security Objectives

| Objective | Description |
|-----------|-------------|
| **Confidentiality** | Protect sensitive data from unauthorized access |
| **Integrity** | Ensure data accuracy and prevent tampering |
| **Availability** | Maintain system uptime and resilience |
| **Non-repudiation** | Ensure accountability through audit trails |
| **Privacy** | Protect user personal information |

### 2.3 Security Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SECURITY ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    PERIMETER SECURITY                        │   │
│  │  CDN (DDoS) → WAF → Load Balancer → Rate Limiter            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  APPLICATION SECURITY                        │   │
│  │  Authentication → Authorization → Input Validation           │   │
│  │  Session Management → CSRF Protection → Output Encoding      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     DATA SECURITY                            │   │
│  │  Encryption at Rest → Encryption in Transit → Key Mgmt      │   │
│  │  Data Masking → Access Control → Backup Encryption          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  MONITORING & RESPONSE                       │   │
│  │  Audit Logging → Threat Detection → Incident Response       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Authentication

### 3.1 Authentication Methods

| Method | Support | Notes |
|--------|---------|-------|
| **Username/Password** | Required | Primary authentication |
| **Email/Password** | Required | Alternative to username |
| **OAuth 2.0 Social** | Future | Google, Facebook, Apple |
| **TOTP MFA** | Optional | For authors/admins |
| **Passwordless** | Future | Email magic link |

### 3.2 Password Requirements

| Requirement | Value | Rationale |
|-------------|-------|-----------|
| Minimum Length | 8 characters | NIST SP 800-63B |
| Maximum Length | 128 characters | Prevent DoS |
| Complexity | 1 uppercase, 1 lowercase, 1 digit | Balance security/usability |
| Character Set | Unicode allowed | Passphrase support |
| Blocklist | Common passwords | HaveIBeenPwned top 100K |
| Breach Check | API check on registration | Compromised credential detection |
| History | Last 5 passwords | Prevent reuse |
| Expiry | None | NIST recommendation |
| Lockout | 5 failures → 15 min lockout | Brute force protection |

### 3.3 Password Storage

```
┌─────────────────────────────────────────────────────────────┐
│                  PASSWORD HASHING FLOW                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Input Password                                             │
│        │                                                    │
│        ▼                                                    │
│  ┌─────────────┐                                           │
│  │ Normalize   │  Unicode NFKC normalization               │
│  └─────────────┘                                           │
│        │                                                    │
│        ▼                                                    │
│  ┌─────────────┐                                           │
│  │ BCrypt Hash │  Cost factor: 12 (configurable)           │
│  │ + Salt      │  Salt: 128-bit random per password        │
│  └─────────────┘                                           │
│        │                                                    │
│        ▼                                                    │
│  Store: $2a$12$[salt][hash]                                │
│                                                             │
│  Verification: timing-safe comparison                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 Multi-Factor Authentication

| Aspect | Specification |
|--------|---------------|
| **Algorithm** | TOTP (RFC 6238) |
| **Code Length** | 6 digits |
| **Time Step** | 30 seconds |
| **Window** | ±1 step (90 seconds) |
| **Backup Codes** | 10 single-use codes |
| **Recovery** | Email verification + support ticket |
| **Required For** | Admin role (mandatory), Author role (encouraged) |

---

## 4. Session Management

### 4.1 Token Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     TOKEN ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   ACCESS TOKEN                       │   │
│  │  Type: JWT (JWS)                                     │   │
│  │  Algorithm: RS256                                    │   │
│  │  TTL: 15 minutes                                     │   │
│  │  Storage: Memory only (client)                       │   │
│  │  Claims: sub, role, permissions, exp, iat, jti       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  REFRESH TOKEN                       │   │
│  │  Type: Opaque (random)                               │   │
│  │  Length: 256 bits                                    │   │
│  │  TTL: 7 days (30 days with "remember me")            │   │
│  │  Storage: HttpOnly, Secure, SameSite=Strict cookie   │   │
│  │  Server: Hashed in database                          │   │
│  │  Rotation: New token on each refresh                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Token Claims

| Claim | Type | Description |
|-------|------|-------------|
| `sub` | UUID | User ID |
| `role` | String | Primary role (USER, AUTHOR, MODERATOR, ADMIN) |
| `permissions` | String[] | Fine-grained permissions |
| `sessionId` | UUID | Session identifier |
| `iat` | Timestamp | Issued at |
| `exp` | Timestamp | Expiration |
| `jti` | UUID | Unique token ID (for revocation) |

### 4.3 Refresh Token Rotation

```
┌─────────────────────────────────────────────────────────────┐
│                REFRESH TOKEN ROTATION                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Client sends refresh token                              │
│  2. Server validates:                                       │
│     - Token exists and not expired                          │
│     - Token not revoked                                     │
│     - Family ID matches                                     │
│  3. Server issues:                                          │
│     - New access token                                      │
│     - New refresh token (incremented generation)            │
│  4. Server invalidates old refresh token                    │
│                                                             │
│  REUSE DETECTION:                                           │
│  If an old token is reused after rotation:                  │
│  → Revoke entire token family                               │
│  → Force re-authentication                                  │
│  → Log security event                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 Session Limits

| Limit | Value | Action |
|-------|-------|--------|
| **Max concurrent sessions** | 5 | Oldest session revoked |
| **Session inactivity timeout** | 30 days | Session expired |
| **Absolute session lifetime** | 90 days | Force re-authentication |
| **Single device logout** | Supported | Revoke specific session |
| **All devices logout** | Supported | Revoke all sessions |

---

## 5. Authorization

### 5.1 Role-Based Access Control (RBAC)

```
┌─────────────────────────────────────────────────────────────┐
│                     ROLE HIERARCHY                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                      ┌─────────┐                            │
│                      │  ADMIN  │                            │
│                      └────┬────┘                            │
│                           │                                 │
│              ┌────────────┼────────────┐                    │
│              │            │            │                    │
│         ┌────▼────┐  ┌────▼────┐  ┌────▼────┐              │
│         │MODERATOR│  │ AUTHOR  │  │  USER   │              │
│         └────┬────┘  └────┬────┘  └────┬────┘              │
│              │            │            │                    │
│              └────────────┼────────────┘                    │
│                           │                                 │
│                    ┌──────▼──────┐                          │
│                    │  ANONYMOUS  │                          │
│                    └─────────────┘                          │
│                                                             │
│  Note: Roles are NOT hierarchical for permissions           │
│  Each role has explicit permission grants                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Permission Model

| Category | Permissions |
|----------|-------------|
| **Story** | `story:create`, `story:read`, `story:update`, `story:delete`, `story:publish` |
| **Chapter** | `chapter:create`, `chapter:read`, `chapter:update`, `chapter:delete`, `chapter:publish` |
| **Interaction** | `interaction:follow`, `interaction:like`, `interaction:review`, `interaction:comment`, `interaction:report` |
| **Wallet** | `wallet:view`, `wallet:deposit`, `wallet:withdraw`, `wallet:transfer` |
| **Moderation** | `moderation:view_reports`, `moderation:resolve_reports`, `moderation:enforce` |
| **Admin** | `admin:users`, `admin:config`, `admin:payouts`, `admin:ai_policy` |

### 5.3 Role-Permission Matrix

| Permission | Anonymous | User | Author | Moderator | Admin |
|------------|-----------|------|--------|-----------|-------|
| `story:read` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `story:create` | | | ✓ | | ✓ |
| `story:update` | | | Own | | ✓ |
| `story:delete` | | | Own | | ✓ |
| `interaction:follow` | | ✓ | ✓ | ✓ | ✓ |
| `interaction:review` | | ✓ | ✓ | ✓ | ✓ |
| `wallet:view` | | ✓ | ✓ | ✓ | ✓ |
| `wallet:withdraw` | | | ✓ | | ✓ |
| `moderation:view_reports` | | | | ✓ | ✓ |
| `moderation:enforce` | | | | ✓ | ✓ |
| `admin:users` | | | | | ✓ |
| `admin:config` | | | | | ✓ |

### 5.4 Object-Level Authorization

| Resource | Owner Check | Additional Rules |
|----------|-------------|------------------|
| **Story** | `author_id == currentUser.id` | Moderator can hide, Admin can delete |
| **Chapter** | Via story ownership | Same as story |
| **Review** | `account_id == currentUser.id` | Moderator can hide |
| **Comment** | `account_id == currentUser.id` | Moderator can hide/delete |
| **Wallet** | `account_id == currentUser.id` | Admin can view for support |
| **Payout Request** | `author_id == currentUser.id` | Admin can process |

---

## 6. Transport Security

### 6.1 TLS Configuration

| Aspect | Requirement |
|--------|-------------|
| **Minimum Version** | TLS 1.2 |
| **Preferred Version** | TLS 1.3 |
| **Certificate** | RSA 2048-bit or ECDSA P-256 minimum |
| **Certificate Authority** | Trusted public CA (Let's Encrypt, DigiCert) |
| **Certificate Transparency** | Required |
| **OCSP Stapling** | Enabled |
| **Certificate Renewal** | Automated, 30 days before expiry |

### 6.2 Cipher Suites (Priority Order)

```
TLS 1.3:
- TLS_AES_256_GCM_SHA384
- TLS_AES_128_GCM_SHA256
- TLS_CHACHA20_POLY1305_SHA256

TLS 1.2:
- TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
```

### 6.3 Security Headers

| Header | Value | Purpose |
|--------|-------|---------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | Force HTTPS |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking |
| `X-XSS-Protection` | `1; mode=block` | XSS filter |
| `Content-Security-Policy` | (See below) | Content restrictions |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Referrer control |
| `Permissions-Policy` | `geolocation=(), camera=(), microphone=()` | Feature restrictions |

### 6.4 Content Security Policy

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'unsafe-inline' https://cdn.example.com;
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  img-src 'self' data: https://cdn.example.com https://images.example.com;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com wss://ws.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
```

---

## 7. API Security

### 7.1 Rate Limiting

#### 7.1.1 Rate Limit Tiers

| Tier | Applies To | Limits |
|------|------------|--------|
| **Anonymous** | Unauthenticated requests | 60 req/min per IP |
| **Authenticated** | Logged-in users | 120 req/min per user |
| **Premium** | Verified authors, partners | 300 req/min per user |
| **Admin** | Admin users | 600 req/min per user |

#### 7.1.2 Endpoint-Specific Limits

| Endpoint Category | Limit | Window | Scope |
|-------------------|-------|--------|-------|
| `/api/auth/login` | 10 | 1 min | IP |
| `/api/auth/register` | 5 | 1 min | IP |
| `/api/auth/password-reset` | 3 | 1 min | IP |
| `/api/payments/*` | 10 | 1 min | User |
| `/api/stories` (POST) | 10 | 1 hour | User |
| `/api/chapters` (POST) | 30 | 1 hour | User |
| `/api/ai/translate` | 5 | 1 hour | User |
| `/api/files/upload` | 20 | 1 hour | User |

#### 7.1.3 Rate Limit Response Headers

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1704067200
Retry-After: 45
```

### 7.2 Input Validation

| Layer | Validation Type | Tools |
|-------|-----------------|-------|
| **Schema** | JSON Schema validation | OpenAPI + validation library |
| **Type** | Strong typing | Java type system |
| **Business** | Domain rules | Service layer validation |
| **Database** | Constraints | CHECK, NOT NULL, FK |

#### 7.2.1 Validation Rules

| Field Type | Validation |
|------------|------------|
| **String** | Max length, regex pattern, encoding |
| **Email** | RFC 5322 format, domain MX check |
| **URL** | Scheme whitelist, domain validation |
| **UUID** | Format validation |
| **Numeric** | Range, precision |
| **Date/Time** | ISO 8601 format, range |
| **Enum** | Allowed values whitelist |
| **HTML Content** | Sanitization (OWASP Java HTML Sanitizer) |

### 7.3 CORS Configuration

```yaml
# Production CORS configuration
cors:
  allowed-origins:
    - https://n9.example.com
    - https://www.n9.example.com
    - https://admin.n9.example.com
  allowed-methods:
    - GET
    - POST
    - PUT
    - DELETE
    - OPTIONS
  allowed-headers:
    - Authorization
    - Content-Type
    - X-Request-ID
  exposed-headers:
    - X-RateLimit-Limit
    - X-RateLimit-Remaining
    - X-RateLimit-Reset
  allow-credentials: true
  max-age: 3600
```

### 7.4 CSRF Protection

| Aspect | Implementation |
|--------|----------------|
| **Strategy** | Double Submit Cookie |
| **Token Generation** | Cryptographically random, 256-bit |
| **Token Storage** | Cookie: `__Host-csrf` (Secure, HttpOnly, SameSite=Strict) |
| **Token Validation** | Compare cookie with header `X-CSRF-Token` |
| **Exempt Endpoints** | Public read endpoints, webhooks (use signatures) |

---

## 8. Data Security

### 8.1 Encryption at Rest

| Data Type | Encryption | Key Management |
|-----------|------------|----------------|
| **Database** | PostgreSQL TDE or volume encryption | Cloud KMS |
| **Backups** | AES-256-GCM | Separate backup key |
| **File Storage** | Server-side encryption | Cloud provider KMS |
| **Sensitive Fields** | Application-level AES-256-GCM | Application key in vault |
| **Redis Cache** | TLS + AUTH | Password in vault |

### 8.2 Sensitive Data Classification

| Classification | Examples | Protection |
|----------------|----------|------------|
| **Secret** | Passwords, API keys, tokens | Hash/encrypt, never log |
| **Confidential** | Email, payment info, IP | Encrypt, mask in logs |
| **Internal** | User ID, story content | Standard protection |
| **Public** | Published stories, usernames | No special protection |

### 8.3 Data Masking

| Field | Log Format | Display Format |
|-------|------------|----------------|
| **Password** | `[REDACTED]` | `••••••••` |
| **Email** | `j***@e***.com` | `john***@example.com` |
| **IP Address** | `192.168.xxx.xxx` | Full (authorized only) |
| **Token** | `[REDACTED]` | Never displayed |
| **Card Number** | Never logged | `**** **** **** 1234` |
| **Bank Account** | Never logged | `****5678` |

### 8.4 Key Management

```
┌─────────────────────────────────────────────────────────────┐
│                    KEY HIERARCHY                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Master Key (Cloud KMS)                  │   │
│  │  - Hardware Security Module backed                   │   │
│  │  - Never exported                                    │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│          ┌───────────────┼───────────────┐                 │
│          │               │               │                 │
│  ┌───────▼───────┐ ┌────▼────┐ ┌────────▼────────┐       │
│  │ Data Key      │ │Token Key│ │ Backup Key       │       │
│  │ (DB fields)   │ │ (JWT)   │ │ (backups)        │       │
│  └───────────────┘ └─────────┘ └─────────────────┘       │
│                                                             │
│  Rotation Schedule:                                         │
│  - Master Key: Annual                                       │
│  - Data Keys: Quarterly                                     │
│  - Token Keys: Monthly                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. Payment Security

### 9.1 PCI DSS Compliance

| Requirement | Implementation |
|-------------|----------------|
| **Card Data Storage** | Never store (tokenization via gateway) |
| **Card Data Transmission** | Direct to payment provider (iframe/redirect) |
| **PCI Scope** | SAQ A (redirect) or SAQ A-EP (iframe) |
| **Penetration Testing** | Annual by qualified assessor |
| **Vulnerability Scanning** | Quarterly ASV scans |

### 9.2 Payment Processing Flow

```
┌─────────────────────────────────────────────────────────────┐
│                 PAYMENT PROCESSING FLOW                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Client                 N9 Server              Payment GW   │
│    │                        │                       │       │
│    │  1. Select package     │                       │       │
│    │───────────────────────>│                       │       │
│    │                        │                       │       │
│    │  2. Create order       │                       │       │
│    │<───────────────────────│                       │       │
│    │    (order_id, amount)  │                       │       │
│    │                        │                       │       │
│    │  3. Redirect to GW     │                       │       │
│    │──────────────────────────────────────────────>│       │
│    │    (order_id, amount, return_url)             │       │
│    │                        │                       │       │
│    │  4. Enter payment      │                       │       │
│    │<──────────────────────────────────────────────│       │
│    │                        │                       │       │
│    │  5. Payment complete   │                       │       │
│    │──────────────────────>│                       │       │
│    │                        │                       │       │
│    │                        │  6. Webhook           │       │
│    │                        │<──────────────────────│       │
│    │                        │  (signed payload)     │       │
│    │                        │                       │       │
│    │                        │  7. Verify signature  │       │
│    │                        │  8. Credit wallet     │       │
│    │                        │  9. Acknowledge       │       │
│    │                        │───────────────────────>│       │
│    │                        │                       │       │
│    │  10. Confirmation      │                       │       │
│    │<───────────────────────│                       │       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 Webhook Security

| Control | Implementation |
|---------|----------------|
| **Signature Verification** | HMAC-SHA256 with shared secret |
| **Timestamp Validation** | Reject if > 5 minutes old |
| **Replay Prevention** | Store processed event IDs |
| **Idempotency** | Process each event once via idempotency key |
| **IP Whitelist** | Optional additional layer |

### 9.4 Financial Integrity Controls

| Control | Implementation |
|---------|----------------|
| **Double-Entry Ledger** | Every transaction has debit and credit |
| **Immutable Transactions** | No updates, only new entries |
| **Balance Reconciliation** | Daily automated checks |
| **Idempotency Keys** | Prevent duplicate processing |
| **Optimistic Locking** | Prevent concurrent balance updates |

---

## 10. Audit & Logging

### 10.1 Audit Event Categories

| Category | Events | Retention |
|----------|--------|-----------|
| **Authentication** | Login, logout, password change, MFA events | 1 year |
| **Authorization** | Role change, permission grant/revoke | 3 years |
| **Financial** | Payments, withdrawals, transfers | 7 years |
| **Moderation** | Reports, enforcement, appeals | 3 years |
| **Content** | Create, update, delete, publish | 1 year |
| **Admin** | Config changes, user management | 3 years |
| **AI** | Policy changes, overrides | 3 years |

### 10.2 Audit Log Schema

```json
{
  "id": "uuid",
  "timestamp": "2025-12-31T12:00:00.000Z",
  "event_type": "user.role.changed",
  "actor": {
    "id": "user-uuid",
    "type": "USER|SYSTEM|ADMIN",
    "ip_address": "192.168.1.1",
    "user_agent": "Mozilla/5.0..."
  },
  "target": {
    "type": "USER",
    "id": "target-user-uuid"
  },
  "action": "UPDATE",
  "changes": {
    "before": { "role": "USER" },
    "after": { "role": "AUTHOR" }
  },
  "metadata": {
    "request_id": "req-uuid",
    "session_id": "session-uuid",
    "reason": "Author application approved"
  },
  "result": "SUCCESS|FAILURE",
  "error_code": null
}
```

### 10.3 Log Security

| Requirement | Implementation |
|-------------|----------------|
| **Integrity** | Write-once storage, checksums |
| **Confidentiality** | Encrypted at rest, access controlled |
| **Availability** | Replicated, geo-redundant |
| **Non-repudiation** | Tamper-evident logging |
| **Access Control** | Role-based, audit access is audited |

---

## 11. Privacy & Compliance

### 11.1 Data Subject Rights (GDPR)

| Right | Implementation | SLA |
|-------|----------------|-----|
| **Access** | Export profile, activity, content | 30 days |
| **Rectification** | Self-service profile edit | Immediate |
| **Erasure** | Account deletion + anonymization | 30 days |
| **Restriction** | Account suspension flag | Immediate |
| **Portability** | JSON/CSV export | 30 days |
| **Objection** | Marketing opt-out | Immediate |

### 11.2 Data Deletion Process

```
┌─────────────────────────────────────────────────────────────┐
│                  DATA DELETION PROCESS                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. IMMEDIATE (Account closure)                             │
│     - Revoke all sessions                                   │
│     - Disable login                                         │
│     - Hide profile from public                              │
│                                                             │
│  2. WITHIN 30 DAYS                                          │
│     - Delete personal identifiers                           │
│     - Delete profile data                                   │
│     - Delete notification preferences                       │
│     - Delete reading history                                │
│                                                             │
│  3. ANONYMIZE (Preserve for integrity)                      │
│     - Replace author name with "Deleted User"               │
│     - Retain stories if published (ownership anonymized)    │
│     - Retain financial records (anonymized)                 │
│     - Retain audit logs (anonymized)                        │
│                                                             │
│  4. PERMANENT DELETION                                      │
│     - After retention period expiry                         │
│     - Verified deletion from all systems                    │
│     - Deletion certificate generated                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 11.3 Consent Management

| Consent Type | Method | Withdrawal |
|--------------|--------|------------|
| **Terms of Service** | Checkbox at registration | Account deletion |
| **Privacy Policy** | Checkbox at registration | Account deletion |
| **Marketing Email** | Opt-in toggle | One-click unsubscribe |
| **Push Notifications** | Browser permission + toggle | Settings or browser |
| **Analytics Cookies** | Cookie banner (opt-in for EU) | Cookie settings |
| **Personalization** | Settings toggle | Settings |

### 11.4 Data Retention Schedule

| Data Type | Retention | Legal Basis |
|-----------|-----------|-------------|
| **Account Data** | Account lifetime + 30 days | Contract |
| **Published Content** | Indefinite (unless deleted) | Legitimate interest |
| **Reading History** | 2 years | Legitimate interest |
| **Financial Records** | 7 years | Legal requirement |
| **Audit Logs** | 3 years | Legal requirement |
| **Support Tickets** | 3 years | Legitimate interest |
| **Marketing Data** | Until consent withdrawn | Consent |

---

## 12. AI Security

### 12.1 AI Data Handling

| Principle | Implementation |
|-----------|----------------|
| **Data Minimization** | Send only required content (no PII) |
| **Pseudonymization** | Strip author identity before AI processing |
| **Purpose Limitation** | Use only for specified tasks |
| **Retention Limitation** | Clear AI cache after processing |
| **Provider Isolation** | No cross-user data sharing |

### 12.2 Prompt Security

| Risk | Mitigation |
|------|------------|
| **Prompt Injection** | User content isolated from system prompts |
| **Data Leakage** | Output sanitization before storage |
| **Model Manipulation** | Rate limiting, anomaly detection |
| **Policy Bypass** | Multi-layer review (AI + rules + human) |

### 12.3 AI Governance

| Control | Implementation |
|---------|----------------|
| **Prompt Versioning** | All prompts version-controlled |
| **Policy Versioning** | All policies version-controlled |
| **Change Approval** | Admin-only with audit trail |
| **A/B Testing** | Controlled rollout of policy changes |
| **Override Logging** | All human overrides recorded |
| **Bias Monitoring** | Regular accuracy audits |

---

## 13. Incident Response

### 13.1 Security Incident Categories

| Severity | Description | Response Time |
|----------|-------------|---------------|
| **P1 Critical** | Active breach, data exposure | 15 minutes |
| **P2 High** | Vulnerability exploitation | 1 hour |
| **P3 Medium** | Suspicious activity | 4 hours |
| **P4 Low** | Policy violation | 24 hours |

### 13.2 Incident Response Phases

```
┌─────────────────────────────────────────────────────────────┐
│                INCIDENT RESPONSE PHASES                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. DETECTION                                               │
│     - Automated alerts                                      │
│     - User reports                                          │
│     - Security scanning                                     │
│                                                             │
│  2. TRIAGE                                                  │
│     - Severity assessment                                   │
│     - Assign incident commander                             │
│     - Initial scope determination                           │
│                                                             │
│  3. CONTAINMENT                                             │
│     - Isolate affected systems                              │
│     - Block malicious actors                                │
│     - Preserve evidence                                     │
│                                                             │
│  4. ERADICATION                                             │
│     - Remove threat                                         │
│     - Patch vulnerabilities                                 │
│     - Verify removal                                        │
│                                                             │
│  5. RECOVERY                                                │
│     - Restore systems                                       │
│     - Verify functionality                                  │
│     - Monitor for recurrence                                │
│                                                             │
│  6. POST-INCIDENT                                           │
│     - Root cause analysis                                   │
│     - Documentation                                         │
│     - Process improvements                                  │
│     - Stakeholder notification                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 13.3 Breach Notification

| Stakeholder | Notification Timeline | Method |
|-------------|----------------------|--------|
| **Affected Users** | 72 hours | Email + in-app |
| **Regulatory Authority** | 72 hours (GDPR) | Official submission |
| **Executive Team** | Immediate | Secure channel |
| **Legal Counsel** | Immediate | Secure channel |
| **Public** | As required | Press release |

---

## 14. Security Testing

### 14.1 Testing Requirements

| Test Type | Frequency | Scope |
|-----------|-----------|-------|
| **SAST** | Every build | All code changes |
| **DAST** | Weekly | Production-like environment |
| **Dependency Scan** | Daily | All dependencies |
| **Container Scan** | Every build | Container images |
| **Penetration Test** | Annual | Full application |
| **Red Team Exercise** | Annual | Organization-wide |

### 14.2 Vulnerability Management

| Severity | Remediation SLA |
|----------|-----------------|
| **Critical (CVSS 9.0+)** | 24 hours |
| **High (CVSS 7.0-8.9)** | 7 days |
| **Medium (CVSS 4.0-6.9)** | 30 days |
| **Low (CVSS 0.1-3.9)** | 90 days |

---

## 15. References

### 15.1 Design Documents
- [02_USERS_COMPONENT.md](../Design/Components/02_USERS_COMPONENT.md)
- [03_PAYMENTS_COMPONENT.md](../Design/Components/03_PAYMENTS_COMPONENT.md)
- [13_AI_AUTOMATION_COMPONENT.md](../Design/Components/13_AI_AUTOMATION_COMPONENT.md)

### 15.2 Related Specifications
- [03_NON_FUNCTIONAL_REQUIREMENTS.md](03_NON_FUNCTIONAL_REQUIREMENTS.md)
- [12_PERMISSION_MATRIX.md](12_PERMISSION_MATRIX.md)

### 15.3 External Standards
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [NIST SP 800-63B](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [GDPR](https://gdpr.eu/)
- [PCI DSS](https://www.pcisecuritystandards.org/)
