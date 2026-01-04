# API Catalog Specification

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
This document provides a **comprehensive catalog** of all API endpoints available in the N9 platform, organized by domain and functionality.

### 2.2 API Base URL

| Environment | Base URL |
|-------------|----------|
| Production | `https://api.n9.example.com/v1` |
| Staging | `https://api.staging.n9.example.com/v1` |
| Development | `http://localhost:8080/v1` |

### 2.3 API Summary

| Domain | Endpoints | Description |
|--------|-----------|-------------|
| Authentication | 12 | User authentication & sessions |
| Users | 18 | User profiles & social |
| Stories | 22 | Story management |
| Chapters | 15 | Chapter management |
| Interactions | 20 | Comments, ratings, bookmarks |
| Payments | 16 | Wallet, donations, subscriptions |
| Search | 6 | Full-text search |
| Notifications | 8 | Push & in-app notifications |
| Admin | 25 | Administration & moderation |

---

## 3. Authentication APIs

### 3.1 Endpoints

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `POST` | `/auth/register` | Register new user | No |
| `POST` | `/auth/login` | Login with credentials | No |
| `POST` | `/auth/logout` | Logout current session | Yes |
| `POST` | `/auth/refresh` | Refresh access token | No* |
| `POST` | `/auth/forgot-password` | Request password reset | No |
| `POST` | `/auth/reset-password` | Reset password with token | No |
| `POST` | `/auth/verify-email` | Verify email address | No |
| `POST` | `/auth/resend-verification` | Resend verification email | No |
| `POST` | `/auth/mfa/enable` | Enable MFA | Yes |
| `POST` | `/auth/mfa/disable` | Disable MFA | Yes |
| `POST` | `/auth/mfa/verify` | Verify MFA code | No* |
| `GET` | `/auth/sessions` | List active sessions | Yes |
| `DELETE` | `/auth/sessions/{id}` | Revoke specific session | Yes |

*Requires refresh token

### 3.2 Register

```http
POST /v1/auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "displayName": "John Doe",
  "acceptTerms": true
}
```

**Response (201):**
```json
{
  "data": {
    "id": "uuid",
    "email": "user@example.com",
    "displayName": "John Doe",
    "emailVerified": false,
    "createdAt": "2025-12-31T10:00:00Z"
  },
  "meta": {
    "message": "Verification email sent"
  }
}
```

### 3.3 Login

```http
POST /v1/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "deviceId": "device-uuid",
  "deviceName": "Chrome on Windows"
}
```

**Response (200):**
```json
{
  "data": {
    "accessToken": "eyJhbG...",
    "refreshToken": "eyJhbG...",
    "expiresIn": 3600,
    "tokenType": "Bearer",
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "displayName": "John Doe",
      "roles": ["USER"]
    }
  }
}
```

---

## 4. Users APIs

### 4.1 Endpoints

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/users/{id}` | Get user profile | No |
| `GET` | `/users/{id}/stories` | Get user's stories | No |
| `GET` | `/users/{id}/followers` | Get followers | No |
| `GET` | `/users/{id}/following` | Get following | No |
| `GET` | `/me` | Get current user | Yes |
| `PUT` | `/me` | Update current user | Yes |
| `DELETE` | `/me` | Delete account | Yes |
| `PUT` | `/me/password` | Change password | Yes |
| `PUT` | `/me/avatar` | Update avatar | Yes |
| `GET` | `/me/preferences` | Get preferences | Yes |
| `PUT` | `/me/preferences` | Update preferences | Yes |
| `POST` | `/users/{id}/follow` | Follow user | Yes |
| `DELETE` | `/users/{id}/follow` | Unfollow user | Yes |
| `POST` | `/users/{id}/block` | Block user | Yes |
| `DELETE` | `/users/{id}/block` | Unblock user | Yes |
| `GET` | `/me/blocked` | Get blocked users | Yes |
| `POST` | `/me/author-application` | Apply for author | Yes |
| `GET` | `/me/author-application` | Check application status | Yes |

### 4.2 Get User Profile

```http
GET /v1/users/{id}
```

**Response (200):**
```json
{
  "data": {
    "id": "uuid",
    "type": "user",
    "attributes": {
      "displayName": "John Doe",
      "username": "johndoe",
      "bio": "Writer and reader",
      "avatarUrl": "https://cdn.n9.example.com/avatars/uuid.jpg",
      "isAuthor": true,
      "isVerified": true,
      "joinedAt": "2024-01-15T10:00:00Z",
      "stats": {
        "stories": 15,
        "followers": 1250,
        "following": 45
      }
    }
  }
}
```

---

## 5. Stories APIs

### 5.1 Endpoints

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/stories` | List stories | No |
| `GET` | `/stories/{id}` | Get story details | No |
| `POST` | `/stories` | Create story | Yes |
| `PUT` | `/stories/{id}` | Update story | Yes |
| `DELETE` | `/stories/{id}` | Delete story | Yes |
| `POST` | `/stories/{id}/publish` | Publish story | Yes |
| `POST` | `/stories/{id}/unpublish` | Unpublish story | Yes |
| `GET` | `/stories/{id}/chapters` | List chapters | No |
| `GET` | `/stories/{id}/stats` | Get story statistics | Yes |
| `GET` | `/stories/{id}/comments` | Get story comments | No |
| `GET` | `/stories/{id}/ratings` | Get story ratings | No |
| `POST` | `/stories/{id}/rate` | Rate story | Yes |
| `DELETE` | `/stories/{id}/rate` | Remove rating | Yes |
| `POST` | `/stories/{id}/bookmark` | Bookmark story | Yes |
| `DELETE` | `/stories/{id}/bookmark` | Remove bookmark | Yes |
| `POST` | `/stories/{id}/like` | Like story | Yes |
| `DELETE` | `/stories/{id}/like` | Unlike story | Yes |
| `POST` | `/stories/{id}/report` | Report story | Yes |
| `GET` | `/stories/trending` | Get trending stories | No |
| `GET` | `/stories/featured` | Get featured stories | No |
| `GET` | `/stories/recommended` | Get recommendations | Yes |
| `GET` | `/stories/new-releases` | Get new releases | No |

### 5.2 List Stories

```http
GET /v1/stories?page=1&size=20&sort=createdAt,desc&filter=status:PUBLISHED
```

**Response (200):**
```json
{
  "data": [
    {
      "id": "uuid",
      "type": "story",
      "attributes": {
        "title": "Epic Fantasy Tale",
        "slug": "epic-fantasy-tale",
        "description": "An epic journey...",
        "coverUrl": "https://cdn.n9.example.com/covers/uuid.jpg",
        "status": "PUBLISHED",
        "chapterCount": 45,
        "wordCount": 125000,
        "stats": {
          "views": 15420,
          "likes": 892,
          "rating": 4.7
        }
      }
    }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "size": 20,
      "totalItems": 150,
      "totalPages": 8
    }
  }
}
```

---

## 6. Chapters APIs

### 6.1 Endpoints

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/chapters/{id}` | Get chapter details | No* |
| `GET` | `/chapters/{id}/content` | Get chapter content | No* |
| `POST` | `/stories/{storyId}/chapters` | Create chapter | Yes |
| `PUT` | `/chapters/{id}` | Update chapter | Yes |
| `DELETE` | `/chapters/{id}` | Delete chapter | Yes |
| `POST` | `/chapters/{id}/publish` | Publish chapter | Yes |
| `POST` | `/chapters/{id}/schedule` | Schedule publish | Yes |
| `PUT` | `/stories/{storyId}/chapters/reorder` | Reorder chapters | Yes |
| `POST` | `/chapters/{id}/unlock` | Unlock chapter (pay) | Yes |
| `GET` | `/chapters/{id}/comments` | Get chapter comments | No |
| `POST` | `/chapters/{id}/comments` | Add comment | Yes |
| `POST` | `/chapters/{id}/like` | Like chapter | Yes |
| `DELETE` | `/chapters/{id}/like` | Unlike chapter | Yes |
| `POST` | `/chapters/{id}/report` | Report chapter | Yes |
| `GET` | `/me/reading-progress/{storyId}` | Get reading progress | Yes |
| `PUT` | `/me/reading-progress/{storyId}` | Update reading progress | Yes |

*Some chapters may require authentication or payment

---

## 7. Interactions APIs

### 7.1 Comments

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/comments/{id}` | Get comment | No |
| `POST` | `/comments` | Create comment | Yes |
| `PUT` | `/comments/{id}` | Update comment | Yes |
| `DELETE` | `/comments/{id}` | Delete comment | Yes |
| `POST` | `/comments/{id}/like` | Like comment | Yes |
| `DELETE` | `/comments/{id}/like` | Unlike comment | Yes |
| `POST` | `/comments/{id}/reply` | Reply to comment | Yes |
| `GET` | `/comments/{id}/replies` | Get replies | No |
| `POST` | `/comments/{id}/report` | Report comment | Yes |

### 7.2 Bookmarks & Reading Lists

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/me/bookmarks` | Get bookmarks | Yes |
| `GET` | `/me/reading-lists` | Get reading lists | Yes |
| `POST` | `/me/reading-lists` | Create reading list | Yes |
| `GET` | `/reading-lists/{id}` | Get reading list | No* |
| `PUT` | `/reading-lists/{id}` | Update reading list | Yes |
| `DELETE` | `/reading-lists/{id}` | Delete reading list | Yes |
| `POST` | `/reading-lists/{id}/stories` | Add story to list | Yes |
| `DELETE` | `/reading-lists/{id}/stories/{storyId}` | Remove story | Yes |

---

## 8. Payments APIs

### 8.1 Wallet

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/me/wallet` | Get wallet balance | Yes |
| `GET` | `/me/wallet/transactions` | Get transactions | Yes |
| `POST` | `/wallet/topup` | Create top-up payment | Yes |
| `POST` | `/wallet/topup/confirm` | Confirm payment | Yes |

### 8.2 Donations

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `POST` | `/donations` | Send donation | Yes |
| `GET` | `/me/donations/sent` | Get sent donations | Yes |
| `GET` | `/me/donations/received` | Get received (author) | Yes |

### 8.3 Subscriptions

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/subscriptions/plans` | Get available plans | No |
| `POST` | `/subscriptions` | Subscribe to plan | Yes |
| `GET` | `/me/subscription` | Get current subscription | Yes |
| `POST` | `/me/subscription/cancel` | Cancel subscription | Yes |

### 8.4 Author Payouts

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/author/earnings` | Get earnings summary | Yes |
| `POST` | `/author/payout` | Request payout | Yes |
| `GET` | `/author/payouts` | Get payout history | Yes |

---

## 9. Search APIs

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/search` | Global search | No |
| `GET` | `/search/stories` | Search stories | No |
| `GET` | `/search/authors` | Search authors | No |
| `GET` | `/search/tags` | Search tags | No |
| `GET` | `/search/suggestions` | Get suggestions | No |
| `GET` | `/search/trending` | Get trending searches | No |

---

## 10. Notifications APIs

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/me/notifications` | Get notifications | Yes |
| `GET` | `/me/notifications/unread-count` | Get unread count | Yes |
| `PUT` | `/me/notifications/{id}/read` | Mark as read | Yes |
| `PUT` | `/me/notifications/read-all` | Mark all as read | Yes |
| `DELETE` | `/me/notifications/{id}` | Delete notification | Yes |
| `GET` | `/me/notification-settings` | Get settings | Yes |
| `PUT` | `/me/notification-settings` | Update settings | Yes |

---

## 11. Admin APIs

### 11.1 Moderation

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/admin/reports` | List reports | Mod+ |
| `GET` | `/admin/reports/{id}` | Get report details | Mod+ |
| `POST` | `/admin/reports/{id}/resolve` | Resolve report | Mod+ |
| `POST` | `/admin/content/{type}/{id}/hide` | Hide content | Mod+ |
| `DELETE` | `/admin/content/{type}/{id}` | Delete content | Admin+ |

### 11.2 User Management

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/admin/users` | List users | Admin+ |
| `POST` | `/admin/users/{id}/warn` | Warn user | Mod+ |
| `POST` | `/admin/users/{id}/suspend` | Suspend user | Mod+ |
| `POST` | `/admin/users/{id}/ban` | Ban user | Admin+ |
| `PUT` | `/admin/users/{id}/roles` | Update roles | Super+ |

### 11.3 System Management

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/admin/dashboard` | Get dashboard stats | Mod+ |
| `GET` | `/admin/analytics` | Get analytics | Admin+ |
| `GET` | `/admin/categories` | List categories | Admin+ |
| `POST` | `/admin/categories` | Create category | Admin+ |
| `GET` | `/admin/audit-logs` | Get audit logs | Admin+ |

### 11.4 Financial

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/admin/payouts/pending` | Get pending payouts | Admin+ |
| `POST` | `/admin/payouts/{id}/approve` | Approve payout | Admin+ |
| `POST` | `/admin/refunds` | Process refund | Admin+ |

---

## 12. WebSocket APIs

### 12.1 Connection

```
wss://ws.n9.example.com/v1/connect?token={jwt}
```

### 12.2 Channels

| Channel | Description |
|---------|-------------|
| `user:{userId}` | User notifications |
| `story:{storyId}` | Story updates |
| `chapter:{chapterId}` | Live comments |

### 12.3 Events

| Event | Description |
|-------|-------------|
| `NOTIFICATION` | New notification |
| `NEW_COMMENT` | New comment |
| `CHAPTER_PUBLISHED` | Chapter published |

---

## 13. References

### 13.1 Related Specifications
- [08_API_STANDARDS.md](08_API_STANDARDS.md)
- [09_ERROR_HANDLING.md](09_ERROR_HANDLING.md)
- [12_PERMISSION_MATRIX.md](12_PERMISSION_MATRIX.md)
