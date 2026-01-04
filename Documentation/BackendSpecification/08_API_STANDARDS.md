# API Standards Specification

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
This document defines **API design standards, conventions, and best practices** for all N9 platform APIs to ensure consistency, usability, and maintainability.

### 2.2 API Types

| Type | Protocol | Use Case |
|------|----------|----------|
| **REST API** | HTTP/HTTPS | CRUD operations, resource management |
| **WebSocket** | WS/WSS | Real-time updates, notifications |
| **GraphQL** | HTTP/HTTPS | Complex queries (future consideration) |

---

## 3. REST API Design Principles

### 3.1 Resource Naming Conventions

| Rule | Example | Anti-pattern |
|------|---------|--------------|
| Use nouns, not verbs | `/stories` | `/getStories` |
| Use plural forms | `/chapters` | `/chapter` |
| Use kebab-case | `/reading-lists` | `/readingLists` |
| Nest for relationships | `/stories/{id}/chapters` | `/story-chapters` |
| Max 2 levels deep | `/stories/{id}/chapters` | `/stories/{id}/chapters/{id}/comments/{id}/replies` |

### 3.2 URL Structure

```
https://api.n9.example.com/v1/{resource}/{id}/{sub-resource}?{query-params}
```

| Component | Description | Example |
|-----------|-------------|---------|
| Base URL | API domain | `api.n9.example.com` |
| Version | API version prefix | `/v1` |
| Resource | Primary resource | `/stories` |
| ID | Resource identifier | `/stories/uuid` |
| Sub-resource | Related resource | `/stories/uuid/chapters` |
| Query params | Filtering, pagination | `?page=1&size=20` |

### 3.3 HTTP Methods

| Method | Purpose | Idempotent | Safe | Request Body |
|--------|---------|------------|------|--------------|
| `GET` | Retrieve resource(s) | Yes | Yes | No |
| `POST` | Create resource | No | No | Yes |
| `PUT` | Replace resource | Yes | No | Yes |
| `PATCH` | Partial update | Yes | No | Yes |
| `DELETE` | Remove resource | Yes | No | No |

### 3.4 HTTP Status Codes

#### 3.4.1 Success Codes

| Code | Name | Use Case |
|------|------|----------|
| `200` | OK | Successful GET, PUT, PATCH, DELETE |
| `201` | Created | Successful POST (resource created) |
| `202` | Accepted | Async operation accepted |
| `204` | No Content | Successful DELETE (no body) |

#### 3.4.2 Client Error Codes

| Code | Name | Use Case |
|------|------|----------|
| `400` | Bad Request | Invalid request body/params |
| `401` | Unauthorized | Missing/invalid authentication |
| `403` | Forbidden | Insufficient permissions |
| `404` | Not Found | Resource doesn't exist |
| `409` | Conflict | Resource state conflict |
| `422` | Unprocessable Entity | Validation errors |
| `429` | Too Many Requests | Rate limit exceeded |

#### 3.4.3 Server Error Codes

| Code | Name | Use Case |
|------|------|----------|
| `500` | Internal Server Error | Unexpected server error |
| `502` | Bad Gateway | Upstream service failure |
| `503` | Service Unavailable | Maintenance/overload |
| `504` | Gateway Timeout | Upstream timeout |

---

## 4. Request Standards

### 4.1 Headers

#### 4.1.1 Required Headers

| Header | Description | Example |
|--------|-------------|---------|
| `Authorization` | Bearer token | `Bearer eyJhbG...` |
| `Content-Type` | Request body format | `application/json` |
| `Accept` | Expected response format | `application/json` |

#### 4.1.2 Optional Headers

| Header | Description | Example |
|--------|-------------|---------|
| `Accept-Language` | Preferred language | `vi, en;q=0.9` |
| `X-Request-ID` | Client request ID | `uuid` |
| `X-Idempotency-Key` | Idempotency key | `uuid` |
| `X-Client-Version` | Client app version | `1.2.3` |
| `X-Device-ID` | Device identifier | `uuid` |

### 4.2 Request Body Format

```json
{
  "title": "Story Title",
  "description": "Story description",
  "categoryId": "uuid",
  "tags": ["fantasy", "adventure"],
  "settings": {
    "visibility": "PUBLIC",
    "commentsEnabled": true
  }
}
```

#### 4.2.1 Naming Conventions

| Rule | Apply To | Example |
|------|----------|---------|
| camelCase | Field names | `firstName`, `createdAt` |
| UPPER_SNAKE | Enums | `PUBLIC`, `IN_PROGRESS` |
| ISO 8601 | Dates/times | `2025-12-31T23:59:59Z` |
| UUID | Identifiers | `550e8400-e29b-41d4-a716-446655440000` |

### 4.3 Query Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `page` | integer | Page number (1-based) | `?page=2` |
| `size` | integer | Items per page | `?size=20` |
| `sort` | string | Sort field(s) | `?sort=createdAt,desc` |
| `filter` | string | Filter expression | `?filter=status:PUBLISHED` |
| `fields` | string | Sparse fieldset | `?fields=id,title,author` |
| `include` | string | Related resources | `?include=author,categories` |
| `q` | string | Search query | `?q=fantasy` |

---

## 5. Response Standards

### 5.1 Success Response Structure

#### 5.1.1 Single Resource

```json
{
  "data": {
    "id": "uuid",
    "type": "story",
    "attributes": {
      "title": "Story Title",
      "description": "Description...",
      "status": "PUBLISHED",
      "createdAt": "2025-12-31T10:00:00Z",
      "updatedAt": "2025-12-31T12:00:00Z"
    },
    "relationships": {
      "author": {
        "id": "uuid",
        "type": "user",
        "attributes": {
          "displayName": "Author Name"
        }
      }
    }
  },
  "meta": {
    "requestId": "uuid"
  }
}
```

#### 5.1.2 Collection Response

```json
{
  "data": [
    {
      "id": "uuid",
      "type": "story",
      "attributes": { ... }
    }
  ],
  "meta": {
    "requestId": "uuid",
    "pagination": {
      "page": 1,
      "size": 20,
      "totalItems": 150,
      "totalPages": 8,
      "hasNext": true,
      "hasPrevious": false
    }
  },
  "links": {
    "self": "/v1/stories?page=1&size=20",
    "first": "/v1/stories?page=1&size=20",
    "last": "/v1/stories?page=8&size=20",
    "next": "/v1/stories?page=2&size=20",
    "prev": null
  }
}
```

### 5.2 Error Response Structure

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "title",
        "code": "SIZE",
        "message": "Title must be between 1 and 200 characters"
      },
      {
        "field": "categoryId",
        "code": "NOT_FOUND",
        "message": "Category not found"
      }
    ],
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/stories",
    "requestId": "uuid"
  }
}
```

### 5.3 Error Code Catalog

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400/422 | Request validation failed |
| `INVALID_JSON` | 400 | Malformed JSON body |
| `AUTHENTICATION_REQUIRED` | 401 | Missing authentication |
| `INVALID_TOKEN` | 401 | Invalid/expired token |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `RESOURCE_NOT_FOUND` | 404 | Resource doesn't exist |
| `CONFLICT` | 409 | State conflict |
| `RATE_LIMITED` | 429 | Rate limit exceeded |
| `INTERNAL_ERROR` | 500 | Unexpected server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily down |

---

## 6. Pagination Standards

### 6.1 Offset-Based Pagination

```
GET /v1/stories?page=2&size=20
```

| Parameter | Default | Min | Max |
|-----------|---------|-----|-----|
| `page` | 1 | 1 | 1000 |
| `size` | 20 | 1 | 100 |

### 6.2 Cursor-Based Pagination

```
GET /v1/activities?cursor=eyJpZCI6MTIzfQ&size=20
```

| Parameter | Description |
|-----------|-------------|
| `cursor` | Opaque cursor token |
| `size` | Items per page |

**Response:**
```json
{
  "data": [...],
  "meta": {
    "pagination": {
      "size": 20,
      "hasMore": true,
      "nextCursor": "eyJpZCI6MTQzfQ"
    }
  }
}
```

### 6.3 When to Use Each

| Scenario | Pagination Type | Reason |
|----------|-----------------|--------|
| Stories list | Offset | Random page access needed |
| Activity feed | Cursor | Real-time, append-only |
| Search results | Offset | Jump to specific page |
| Notifications | Cursor | Chronological, no random access |
| Comments | Cursor | Frequently updated |

---

## 7. Filtering & Sorting

### 7.1 Filter Syntax

```
GET /v1/stories?filter=status:PUBLISHED,category:fantasy
```

| Operator | Syntax | Example |
|----------|--------|---------|
| Equals | `field:value` | `status:PUBLISHED` |
| Not equals | `field:!value` | `status:!DRAFT` |
| Greater than | `field:>value` | `rating:>4` |
| Less than | `field:<value` | `chapters:<10` |
| Range | `field:min..max` | `rating:3..5` |
| In list | `field:[a,b,c]` | `status:[PUBLISHED,COMPLETED]` |
| Like | `field:~value` | `title:~fantasy` |

### 7.2 Sort Syntax

```
GET /v1/stories?sort=createdAt,desc;rating,desc
```

| Format | Description |
|--------|-------------|
| `field,asc` | Ascending order |
| `field,desc` | Descending order |
| Multiple: `a,desc;b,asc` | Multi-field sort |

### 7.3 Sortable Fields by Resource

| Resource | Sortable Fields |
|----------|-----------------|
| Stories | `createdAt`, `updatedAt`, `title`, `viewCount`, `rating`, `chapterCount` |
| Chapters | `orderIndex`, `createdAt`, `wordCount` |
| Comments | `createdAt`, `likeCount` |
| Users | `createdAt`, `displayName` |

---

## 8. Versioning Strategy

### 8.1 URL Path Versioning

```
/v1/stories
/v2/stories
```

### 8.2 Version Lifecycle

| Phase | Duration | Support Level |
|-------|----------|---------------|
| Current | Active | Full support |
| Previous | 12 months | Security fixes only |
| Deprecated | 6 months | No fixes, migration required |
| Sunset | After deprecation | Removed |

### 8.3 Deprecation Headers

```http
Deprecation: true
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Link: </v2/stories>; rel="successor-version"
```

---

## 9. Rate Limiting

### 9.1 Rate Limit Tiers

| Tier | Requests/min | Requests/hour | Applies To |
|------|--------------|---------------|------------|
| Anonymous | 30 | 300 | Unauthenticated |
| Free | 60 | 1,000 | Free users |
| Premium | 120 | 5,000 | Premium users |
| Author | 180 | 10,000 | Active authors |
| Internal | 600 | 50,000 | Service accounts |

### 9.2 Rate Limit Headers

```http
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1735689600
Retry-After: 30
```

### 9.3 Rate Limit Response

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Try again in 30 seconds.",
    "details": {
      "limit": 60,
      "remaining": 0,
      "resetAt": "2025-12-31T10:01:00Z",
      "retryAfter": 30
    }
  }
}
```

---

## 10. Authentication & Authorization Headers

### 10.1 JWT Token Structure

```
Authorization: Bearer <token>
```

**Token Payload:**
```json
{
  "sub": "user-uuid",
  "type": "ACCESS",
  "roles": ["USER", "AUTHOR"],
  "permissions": ["story:write", "chapter:write"],
  "iat": 1735689600,
  "exp": 1735693200
}
```

### 10.2 API Key Authentication

```http
X-API-Key: api_key_value
```

Used for:
- Service-to-service communication
- Webhook callbacks
- Third-party integrations

---

## 11. Idempotency

### 11.1 Idempotent Operations

| Method | Naturally Idempotent | Idempotency Key Required |
|--------|---------------------|--------------------------|
| GET | Yes | No |
| PUT | Yes | No |
| DELETE | Yes | No |
| POST | No | Yes (for safe retry) |
| PATCH | Depends | Recommended |

### 11.2 Idempotency Key Usage

```http
POST /v1/payments
X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "amount": 100,
  "currency": "USD"
}
```

### 11.3 Idempotency Response

**First request:**
```http
HTTP/1.1 201 Created
X-Idempotency-Status: EXECUTED
```

**Duplicate request:**
```http
HTTP/1.1 200 OK
X-Idempotency-Status: CACHED
```

---

## 12. HATEOAS Links

### 12.1 Link Relations

```json
{
  "data": {
    "id": "uuid",
    "type": "story",
    "attributes": { ... }
  },
  "links": {
    "self": "/v1/stories/uuid",
    "chapters": "/v1/stories/uuid/chapters",
    "author": "/v1/users/author-uuid",
    "comments": "/v1/stories/uuid/comments",
    "like": "/v1/stories/uuid/like",
    "bookmark": "/v1/stories/uuid/bookmark"
  }
}
```

### 12.2 Standard Link Relations

| Relation | Description |
|----------|-------------|
| `self` | Current resource |
| `collection` | Parent collection |
| `next` | Next page |
| `prev` | Previous page |
| `first` | First page |
| `last` | Last page |
| `related` | Related resource |

---

## 13. Content Negotiation

### 13.1 Supported Media Types

| Media Type | Description | Use Case |
|------------|-------------|----------|
| `application/json` | JSON (default) | All APIs |
| `application/json; charset=utf-8` | JSON with encoding | Explicit encoding |
| `text/event-stream` | Server-Sent Events | Real-time streams |
| `multipart/form-data` | File uploads | Image uploads |

### 13.2 Accept Header Handling

```http
Accept: application/json
Accept: application/json; version=1
Accept: text/event-stream
```

---

## 14. Bulk Operations

### 14.1 Bulk Create

```http
POST /v1/chapters/bulk
Content-Type: application/json

{
  "items": [
    { "title": "Chapter 1", "content": "..." },
    { "title": "Chapter 2", "content": "..." }
  ]
}
```

**Response:**
```json
{
  "data": {
    "created": 2,
    "failed": 0,
    "items": [
      { "id": "uuid1", "status": "CREATED" },
      { "id": "uuid2", "status": "CREATED" }
    ]
  }
}
```

### 14.2 Bulk Delete

```http
DELETE /v1/notifications/bulk
Content-Type: application/json

{
  "ids": ["uuid1", "uuid2", "uuid3"]
}
```

### 14.3 Bulk Limits

| Operation | Max Items | Timeout |
|-----------|-----------|---------|
| Bulk Create | 100 | 30s |
| Bulk Update | 100 | 30s |
| Bulk Delete | 500 | 60s |

---

## 15. Field Selection (Sparse Fieldsets)

### 15.1 Syntax

```
GET /v1/stories?fields=id,title,author.displayName
```

### 15.2 Nested Fields

```
GET /v1/stories/uuid?fields=id,title,author(id,displayName),categories(id,name)
```

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "title": "Story Title",
    "author": {
      "id": "author-uuid",
      "displayName": "Author Name"
    },
    "categories": [
      { "id": "cat-uuid", "name": "Fantasy" }
    ]
  }
}
```

---

## 16. API Documentation Standards

### 16.1 OpenAPI Specification

All APIs documented using OpenAPI 3.1:

```yaml
openapi: 3.1.0
info:
  title: N9 Platform API
  version: 1.0.0
  description: Story reading platform API
servers:
  - url: https://api.n9.example.com/v1
    description: Production
  - url: https://api.staging.n9.example.com/v1
    description: Staging
```

### 16.2 Documentation Requirements

| Element | Required | Description |
|---------|----------|-------------|
| Summary | Yes | Brief endpoint description |
| Description | Yes | Detailed explanation |
| Parameters | Yes | All query/path params |
| Request Body | If applicable | Schema with examples |
| Responses | Yes | All possible responses |
| Security | Yes | Auth requirements |
| Tags | Yes | Grouping |
| Examples | Recommended | Request/response examples |

---

## 17. WebSocket API Standards

### 17.1 Connection

```
wss://ws.n9.example.com/v1/connect?token={jwt}
```

### 17.2 Message Format

```json
{
  "type": "MESSAGE_TYPE",
  "payload": { ... },
  "timestamp": "2025-12-31T10:00:00Z",
  "id": "message-uuid"
}
```

### 17.3 Message Types

| Type | Direction | Description |
|------|-----------|-------------|
| `PING` | Client → Server | Keep-alive |
| `PONG` | Server → Client | Keep-alive response |
| `SUBSCRIBE` | Client → Server | Subscribe to channel |
| `UNSUBSCRIBE` | Client → Server | Unsubscribe |
| `EVENT` | Server → Client | Event notification |
| `ERROR` | Server → Client | Error message |

---

## 18. References

### 18.1 Design Documents
- [00_PLATFORM_PERFORMANCE_ASYNC_DESIGN.md](../Design/00_PLATFORM_PERFORMANCE_ASYNC_DESIGN.md)

### 18.2 Related Specifications
- [05_SECURITY_AND_COMPLIANCE.md](05_SECURITY_AND_COMPLIANCE.md)
- [06_REALTIME_AND_EVENTS.md](06_REALTIME_AND_EVENTS.md)
- [09_ERROR_HANDLING.md](09_ERROR_HANDLING.md)

### 18.3 External Standards
- [RFC 7231 - HTTP/1.1 Semantics](https://tools.ietf.org/html/rfc7231)
- [JSON:API Specification](https://jsonapi.org/)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
