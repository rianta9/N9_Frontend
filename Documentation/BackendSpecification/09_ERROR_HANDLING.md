# Error Handling Specification

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
This document defines the **error handling strategy, error codes, and exception management** for the N9 platform to ensure consistent, informative, and secure error responses.

### 2.2 Error Handling Principles

| Principle | Description |
|-----------|-------------|
| **Consistency** | All errors follow the same response format |
| **Clarity** | Error messages are clear and actionable |
| **Security** | No sensitive information leaked in errors |
| **Traceability** | Every error includes a request ID |
| **Localization** | Error messages support i18n |
| **Logging** | All errors logged with context |

---

## 3. Error Response Format

### 3.1 Standard Error Response

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": [
      {
        "field": "fieldName",
        "code": "FIELD_ERROR_CODE",
        "message": "Field-specific error message",
        "rejectedValue": "invalid-value"
      }
    ],
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/resource",
    "method": "POST",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### 3.2 Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string | Yes | Machine-readable error code |
| `message` | string | Yes | Human-readable description |
| `details` | array | No | Field-level errors |
| `timestamp` | string | Yes | ISO 8601 timestamp |
| `path` | string | Yes | Request path |
| `method` | string | Yes | HTTP method |
| `requestId` | string | Yes | Unique request identifier |

### 3.3 Detail Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `field` | string | Yes | Field name (dot notation for nested) |
| `code` | string | Yes | Validation code |
| `message` | string | Yes | Error description |
| `rejectedValue` | any | No | Invalid value (sanitized) |

---

## 4. Error Code Taxonomy

### 4.1 Error Code Structure

```
{CATEGORY}_{SPECIFIC_ERROR}
```

Examples:
- `AUTH_TOKEN_EXPIRED`
- `VALIDATION_REQUIRED_FIELD`
- `RESOURCE_NOT_FOUND`
- `PAYMENT_INSUFFICIENT_FUNDS`

### 4.2 Error Categories

| Category | Prefix | HTTP Range | Description |
|----------|--------|------------|-------------|
| Authentication | `AUTH_` | 401 | Authentication failures |
| Authorization | `AUTHZ_` | 403 | Permission failures |
| Validation | `VALIDATION_` | 400, 422 | Input validation |
| Resource | `RESOURCE_` | 404, 409 | Resource state |
| Business | `BIZ_` | 400, 422 | Business rule violations |
| Rate Limit | `RATE_` | 429 | Throttling |
| Payment | `PAYMENT_` | 400, 402 | Payment failures |
| External | `EXT_` | 502, 503 | External service errors |
| Internal | `INTERNAL_` | 500 | Server errors |

---

## 5. Authentication Errors (401)

### 5.1 Error Codes

| Code | Message | Cause | Resolution |
|------|---------|-------|------------|
| `AUTH_REQUIRED` | Authentication required | No token provided | Include Authorization header |
| `AUTH_TOKEN_INVALID` | Invalid authentication token | Malformed token | Re-authenticate |
| `AUTH_TOKEN_EXPIRED` | Authentication token expired | Token TTL exceeded | Refresh token |
| `AUTH_TOKEN_REVOKED` | Authentication token revoked | User logged out elsewhere | Re-authenticate |
| `AUTH_REFRESH_INVALID` | Invalid refresh token | Refresh token invalid | Re-authenticate |
| `AUTH_REFRESH_EXPIRED` | Refresh token expired | Refresh TTL exceeded | Re-authenticate |
| `AUTH_SESSION_INVALID` | Session invalidated | Session terminated | Re-authenticate |
| `AUTH_DEVICE_UNRECOGNIZED` | Unrecognized device | New device login | Verify device |
| `AUTH_MFA_REQUIRED` | Multi-factor authentication required | MFA enabled | Complete MFA |
| `AUTH_MFA_INVALID` | Invalid MFA code | Wrong OTP | Retry with correct code |

### 5.2 Example Response

```json
{
  "error": {
    "code": "AUTH_TOKEN_EXPIRED",
    "message": "Your session has expired. Please log in again.",
    "details": {
      "expiredAt": "2025-12-31T09:00:00Z"
    },
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/stories",
    "method": "GET",
    "requestId": "uuid"
  }
}
```

---

## 6. Authorization Errors (403)

### 6.1 Error Codes

| Code | Message | Cause |
|------|---------|-------|
| `AUTHZ_FORBIDDEN` | You don't have permission to perform this action | General permission denied |
| `AUTHZ_ROLE_REQUIRED` | Required role not assigned | Missing role |
| `AUTHZ_PERMISSION_REQUIRED` | Required permission not granted | Missing specific permission |
| `AUTHZ_RESOURCE_OWNER` | You can only modify your own resources | Ownership violation |
| `AUTHZ_ACCOUNT_SUSPENDED` | Account suspended | User suspended |
| `AUTHZ_ACCOUNT_BANNED` | Account permanently banned | User banned |
| `AUTHZ_EMAIL_UNVERIFIED` | Email verification required | Email not verified |
| `AUTHZ_PREMIUM_REQUIRED` | Premium subscription required | Feature requires premium |
| `AUTHZ_AUTHOR_REQUIRED` | Author status required | Not an author |
| `AUTHZ_AGE_RESTRICTED` | Content restricted by age | Age verification failed |

### 6.2 Example Response

```json
{
  "error": {
    "code": "AUTHZ_PREMIUM_REQUIRED",
    "message": "This feature requires a premium subscription.",
    "details": {
      "feature": "offline_reading",
      "currentPlan": "FREE",
      "requiredPlan": "PREMIUM"
    },
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/chapters/uuid/download",
    "method": "POST",
    "requestId": "uuid"
  }
}
```

---

## 7. Validation Errors (400, 422)

### 7.1 Field Validation Codes

| Code | Message Template | Applies To |
|------|------------------|------------|
| `REQUIRED` | {field} is required | All fields |
| `SIZE` | {field} must be between {min} and {max} | String, Array |
| `MIN` | {field} must be at least {min} | Number |
| `MAX` | {field} must be at most {max} | Number |
| `PATTERN` | {field} format is invalid | String |
| `EMAIL` | Invalid email format | Email |
| `URL` | Invalid URL format | URL |
| `UUID` | Invalid UUID format | UUID |
| `ENUM` | {field} must be one of: {values} | Enum |
| `UNIQUE` | {field} already exists | Unique constraint |
| `PAST` | {field} must be in the past | Date |
| `FUTURE` | {field} must be in the future | Date |

### 7.2 Business Validation Codes

| Code | Message | Context |
|------|---------|---------|
| `VALIDATION_DUPLICATE` | Resource already exists | Duplicate entry |
| `VALIDATION_REFERENCE` | Referenced resource not found | Foreign key |
| `VALIDATION_STATE` | Invalid state transition | State machine |
| `VALIDATION_LIMIT` | Limit exceeded | Quota |
| `VALIDATION_FORMAT` | Invalid format | File/content type |
| `VALIDATION_CONTENT` | Content policy violation | Moderation |

### 7.3 Example Response

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "title",
        "code": "SIZE",
        "message": "Title must be between 1 and 200 characters",
        "rejectedValue": ""
      },
      {
        "field": "categoryId",
        "code": "REQUIRED",
        "message": "Category is required"
      },
      {
        "field": "tags[2]",
        "code": "SIZE",
        "message": "Tag must be between 2 and 30 characters",
        "rejectedValue": "x"
      }
    ],
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/stories",
    "method": "POST",
    "requestId": "uuid"
  }
}
```

---

## 8. Resource Errors (404, 409)

### 8.1 Error Codes

| Code | HTTP | Message | Cause |
|------|------|---------|-------|
| `RESOURCE_NOT_FOUND` | 404 | Resource not found | Entity doesn't exist |
| `RESOURCE_DELETED` | 404 | Resource has been deleted | Soft-deleted |
| `RESOURCE_CONFLICT` | 409 | Resource conflict | Concurrent modification |
| `RESOURCE_LOCKED` | 409 | Resource is locked | Being edited |
| `RESOURCE_VERSION_MISMATCH` | 409 | Resource version mismatch | Optimistic lock failure |
| `RESOURCE_STATE_INVALID` | 409 | Invalid resource state for operation | State machine violation |

### 8.2 Example Responses

**Not Found:**
```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Story not found",
    "details": {
      "resourceType": "story",
      "resourceId": "uuid"
    },
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/stories/uuid",
    "method": "GET",
    "requestId": "uuid"
  }
}
```

**Conflict:**
```json
{
  "error": {
    "code": "RESOURCE_VERSION_MISMATCH",
    "message": "The resource has been modified by another request",
    "details": {
      "currentVersion": 5,
      "providedVersion": 3
    },
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/stories/uuid",
    "method": "PUT",
    "requestId": "uuid"
  }
}
```

---

## 9. Business Rule Errors (400, 422)

### 9.1 Story/Chapter Errors

| Code | Message | Cause |
|------|---------|-------|
| `BIZ_STORY_CHAPTER_LIMIT` | Story has reached maximum chapter limit | Max 1000 chapters |
| `BIZ_STORY_NOT_PUBLISHED` | Story must be published first | Attempting action on draft |
| `BIZ_CHAPTER_ORDER_INVALID` | Invalid chapter order | Order index conflict |
| `BIZ_CHAPTER_LOCKED` | Chapter is locked for premium | Non-premium accessing locked |

### 9.2 User/Account Errors

| Code | Message | Cause |
|------|---------|-------|
| `BIZ_USER_FOLLOW_SELF` | Cannot follow yourself | Self-follow attempt |
| `BIZ_USER_ALREADY_FOLLOWING` | Already following this user | Duplicate follow |
| `BIZ_USER_NOT_FOLLOWING` | Not following this user | Unfollow non-followed |
| `BIZ_READING_LIST_FULL` | Reading list is full | Max items reached |

### 9.3 Payment/Wallet Errors

| Code | Message | Cause |
|------|---------|-------|
| `BIZ_WALLET_INSUFFICIENT` | Insufficient wallet balance | Not enough coins |
| `BIZ_DONATION_BELOW_MIN` | Donation below minimum amount | Min threshold |
| `BIZ_DONATION_ABOVE_MAX` | Donation exceeds maximum amount | Max threshold |
| `BIZ_PAYOUT_BELOW_MIN` | Payout amount below minimum | Min payout threshold |

### 9.4 Example Response

```json
{
  "error": {
    "code": "BIZ_WALLET_INSUFFICIENT",
    "message": "Insufficient wallet balance",
    "details": {
      "required": 100,
      "available": 50,
      "currency": "COINS"
    },
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/donations",
    "method": "POST",
    "requestId": "uuid"
  }
}
```

---

## 10. Rate Limit Errors (429)

### 10.1 Error Codes

| Code | Message | Context |
|------|---------|---------|
| `RATE_LIMIT_EXCEEDED` | Rate limit exceeded | General rate limit |
| `RATE_LIMIT_USER` | User rate limit exceeded | Per-user limit |
| `RATE_LIMIT_IP` | IP rate limit exceeded | Per-IP limit |
| `RATE_LIMIT_ENDPOINT` | Endpoint rate limit exceeded | Specific endpoint |
| `RATE_LIMIT_GLOBAL` | Service rate limit exceeded | System-wide limit |

### 10.2 Example Response

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "details": {
      "limit": 60,
      "remaining": 0,
      "window": "1 minute",
      "resetAt": "2025-12-31T10:01:00Z",
      "retryAfter": 45
    },
    "timestamp": "2025-12-31T10:00:15Z",
    "path": "/v1/stories",
    "method": "GET",
    "requestId": "uuid"
  }
}
```

### 10.3 Rate Limit Headers

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1735689660
Retry-After: 45
```

---

## 11. Payment Errors (400, 402)

### 11.1 Error Codes

| Code | HTTP | Message | Cause |
|------|------|---------|-------|
| `PAYMENT_FAILED` | 400 | Payment processing failed | Generic failure |
| `PAYMENT_DECLINED` | 402 | Payment declined | Card declined |
| `PAYMENT_EXPIRED` | 400 | Payment session expired | Timeout |
| `PAYMENT_INSUFFICIENT_FUNDS` | 402 | Insufficient funds | Card has no funds |
| `PAYMENT_INVALID_CARD` | 400 | Invalid card details | Bad card number |
| `PAYMENT_FRAUD_DETECTED` | 400 | Payment flagged for review | Fraud detection |
| `PAYMENT_CURRENCY_UNSUPPORTED` | 400 | Currency not supported | Invalid currency |
| `PAYMENT_AMOUNT_INVALID` | 400 | Invalid payment amount | Amount out of range |

### 11.2 Example Response

```json
{
  "error": {
    "code": "PAYMENT_DECLINED",
    "message": "Your payment was declined. Please try a different payment method.",
    "details": {
      "declineCode": "insufficient_funds",
      "transactionId": "txn_uuid"
    },
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/payments",
    "method": "POST",
    "requestId": "uuid"
  }
}
```

---

## 12. External Service Errors (502, 503, 504)

### 12.1 Error Codes

| Code | HTTP | Message | Cause |
|------|------|---------|-------|
| `EXT_SERVICE_UNAVAILABLE` | 503 | External service temporarily unavailable | Service down |
| `EXT_SERVICE_TIMEOUT` | 504 | External service timeout | Slow response |
| `EXT_SERVICE_ERROR` | 502 | External service error | 5xx from service |
| `EXT_PAYMENT_UNAVAILABLE` | 503 | Payment service unavailable | Stripe/PayPal down |
| `EXT_EMAIL_UNAVAILABLE` | 503 | Email service unavailable | SendGrid down |
| `EXT_STORAGE_UNAVAILABLE` | 503 | Storage service unavailable | S3 down |
| `EXT_AI_UNAVAILABLE` | 503 | AI service unavailable | OpenAI down |

### 12.2 Example Response

```json
{
  "error": {
    "code": "EXT_PAYMENT_UNAVAILABLE",
    "message": "Payment service is temporarily unavailable. Please try again later.",
    "details": {
      "service": "stripe",
      "retryAfter": 300
    },
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/payments",
    "method": "POST",
    "requestId": "uuid"
  }
}
```

---

## 13. Internal Errors (500)

### 13.1 Error Codes

| Code | Message | Logging Level |
|------|---------|---------------|
| `INTERNAL_ERROR` | An unexpected error occurred | ERROR |
| `INTERNAL_DATABASE_ERROR` | Database operation failed | ERROR |
| `INTERNAL_CACHE_ERROR` | Cache operation failed | WARN |
| `INTERNAL_QUEUE_ERROR` | Queue operation failed | ERROR |

### 13.2 Example Response

```json
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred. Please try again later.",
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/stories",
    "method": "POST",
    "requestId": "uuid"
  }
}
```

> **Note:** Internal error details are never exposed to clients for security reasons. Full details are logged server-side with the request ID.

---

## 14. Exception Hierarchy

### 14.1 Base Exception Classes

```java
// Base exception
public abstract class N9Exception extends RuntimeException {
    private final String errorCode;
    private final Map<String, Object> details;
    private final HttpStatus httpStatus;
}

// Client errors (4xx)
public class ClientException extends N9Exception { }

// Authentication errors (401)
public class AuthenticationException extends ClientException { }

// Authorization errors (403)
public class AuthorizationException extends ClientException { }

// Validation errors (400, 422)
public class ValidationException extends ClientException {
    private final List<FieldError> fieldErrors;
}

// Resource errors (404, 409)
public class ResourceException extends ClientException { }

// Business rule errors (400, 422)
public class BusinessRuleException extends ClientException { }

// Rate limit errors (429)
public class RateLimitException extends ClientException { }

// External service errors (5xx)
public class ExternalServiceException extends N9Exception { }

// Internal errors (500)
public class InternalException extends N9Exception { }
```

### 14.2 Exception Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      EXCEPTION FLOW                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Service Layer          Controller        Exception Handler          │
│       │                     │                    │                   │
│       │  throws exception   │                    │                   │
│       │────────────────────▶│                    │                   │
│       │                     │                    │                   │
│       │                     │  exception thrown  │                   │
│       │                     │───────────────────▶│                   │
│       │                     │                    │                   │
│       │                     │                    │  1. Log error     │
│       │                     │                    │  2. Build response│
│       │                     │                    │  3. Add headers   │
│       │                     │                    │                   │
│       │                     │  error response    │                   │
│       │                     │◀───────────────────│                   │
│       │                     │                    │                   │
│       │  return to client   │                    │                   │
│       │◀────────────────────│                    │                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 15. Global Exception Handler

### 15.1 Handler Implementation

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            ValidationException ex, HttpServletRequest request) {
        log.warn("Validation error: {} - {}", ex.getErrorCode(), ex.getMessage());
        return buildResponse(ex, request, HttpStatus.UNPROCESSABLE_ENTITY);
    }

    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ErrorResponse> handleAuthentication(
            AuthenticationException ex, HttpServletRequest request) {
        log.warn("Authentication error: {} - {}", ex.getErrorCode(), ex.getMessage());
        return buildResponse(ex, request, HttpStatus.UNAUTHORIZED);
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        log.info("Resource not found: {}", ex.getMessage());
        return buildResponse(ex, request, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(
            Exception ex, HttpServletRequest request) {
        log.error("Unexpected error", ex);
        return buildResponse(
            new InternalException("INTERNAL_ERROR", "An unexpected error occurred"),
            request,
            HttpStatus.INTERNAL_SERVER_ERROR
        );
    }
}
```

---

## 16. Error Logging Standards

### 16.1 Logging Levels by Error Type

| Error Category | Log Level | Include Stack Trace |
|----------------|-----------|---------------------|
| Validation (4xx) | WARN | No |
| Authentication (401) | WARN | No |
| Authorization (403) | WARN | No |
| Not Found (404) | INFO | No |
| Conflict (409) | WARN | No |
| Rate Limit (429) | WARN | No |
| External (5xx) | ERROR | Yes |
| Internal (500) | ERROR | Yes |

### 16.2 Log Format

```json
{
  "timestamp": "2025-12-31T10:00:00Z",
  "level": "ERROR",
  "logger": "GlobalExceptionHandler",
  "message": "Internal error occurred",
  "context": {
    "requestId": "uuid",
    "userId": "user-uuid",
    "path": "/v1/stories",
    "method": "POST",
    "errorCode": "INTERNAL_ERROR"
  },
  "exception": {
    "type": "java.lang.NullPointerException",
    "message": "Object reference is null",
    "stackTrace": "..."
  }
}
```

### 16.3 Sensitive Data Handling

| Data Type | Logging Rule |
|-----------|--------------|
| Passwords | Never log |
| Tokens | Mask all but last 4 chars |
| Credit Cards | Mask all but last 4 digits |
| Personal Data | Log only IDs, not values |
| Request Bodies | Redact sensitive fields |

---

## 17. Retry Guidance

### 17.1 Retryable Errors

| Error Code | Retryable | Recommended Delay |
|------------|-----------|-------------------|
| `RATE_LIMIT_EXCEEDED` | Yes | Use Retry-After header |
| `EXT_SERVICE_UNAVAILABLE` | Yes | Exponential backoff |
| `EXT_SERVICE_TIMEOUT` | Yes | 1-5 seconds |
| `INTERNAL_ERROR` | Maybe | 1-3 attempts |
| `RESOURCE_VERSION_MISMATCH` | Yes | Fetch & retry immediately |
| `AUTH_TOKEN_EXPIRED` | Yes | Refresh & retry |
| All validation errors | No | Fix input |
| All authorization errors | No | Cannot retry |

### 17.2 Retry Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RETRY STRATEGY                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Attempt 1 ──▶ Fail ──▶ Wait 1s                                     │
│                          │                                           │
│  Attempt 2 ◀─────────────┘                                          │
│       │                                                              │
│       └──▶ Fail ──▶ Wait 2s                                         │
│                       │                                              │
│  Attempt 3 ◀──────────┘                                             │
│       │                                                              │
│       └──▶ Fail ──▶ Wait 4s                                         │
│                       │                                              │
│  Attempt 4 ◀──────────┘                                             │
│       │                                                              │
│       └──▶ Fail ──▶ Give up, return error                           │
│                                                                      │
│  Max retries: 3                                                      │
│  Max delay: 10s                                                      │
│  Jitter: ±20%                                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 18. Error Localization

### 18.1 Message Keys

```properties
# messages.properties (default - English)
error.auth.token_expired=Your session has expired. Please log in again.
error.validation.required={0} is required
error.validation.size={0} must be between {1} and {2} characters
error.resource.not_found={0} not found

# messages_vi.properties (Vietnamese)
error.auth.token_expired=Phiên đăng nhập đã hết hạn. Vui lòng đăng nhập lại.
error.validation.required={0} là bắt buộc
error.validation.size={0} phải có từ {1} đến {2} ký tự
error.resource.not_found=Không tìm thấy {0}
```

### 18.2 Localization Flow

1. Client sends `Accept-Language: vi, en;q=0.9`
2. Server resolves locale from header
3. Error messages resolved from appropriate bundle
4. Response includes localized message

---

## 19. Circuit Breaker Error Handling

### 19.1 Circuit States

| State | Behavior | Error Response |
|-------|----------|----------------|
| **Closed** | Normal operation | Normal errors |
| **Open** | Requests fail fast | `EXT_SERVICE_UNAVAILABLE` |
| **Half-Open** | Limited requests | Normal or open |

### 19.2 Circuit Breaker Response

```json
{
  "error": {
    "code": "EXT_SERVICE_UNAVAILABLE",
    "message": "Service temporarily unavailable due to high failure rate",
    "details": {
      "service": "payment-gateway",
      "circuitState": "OPEN",
      "retryAfter": 30
    },
    "timestamp": "2025-12-31T10:00:00Z",
    "path": "/v1/payments",
    "method": "POST",
    "requestId": "uuid"
  }
}
```

---

## 20. References

### 20.1 Related Specifications
- [08_API_STANDARDS.md](08_API_STANDARDS.md)
- [05_SECURITY_AND_COMPLIANCE.md](05_SECURITY_AND_COMPLIANCE.md)

### 20.2 Design Documents
- [00_PLATFORM_PERFORMANCE_ASYNC_DESIGN.md](../Design/00_PLATFORM_PERFORMANCE_ASYNC_DESIGN.md)

### 20.3 External References
- [RFC 7807 - Problem Details for HTTP APIs](https://tools.ietf.org/html/rfc7807)
- [HTTP Status Codes](https://httpstatuses.com/)
