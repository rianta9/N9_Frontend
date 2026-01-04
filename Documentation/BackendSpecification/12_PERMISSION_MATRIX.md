# Permission Matrix Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2025-12-31 |
| Status | Approved |
| Owner | Platform Security Team |
| Review Cycle | Quarterly |

---

## 2. Overview

### 2.1 Purpose
This document defines the **complete permission matrix** for all roles, resources, and actions in the N9 platform, implementing Role-Based Access Control (RBAC) with attribute-based extensions.

### 2.2 Access Control Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ACCESS CONTROL MODEL                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                         User                                 │    │
│  │                          │                                   │    │
│  │           ┌──────────────┼──────────────┐                   │    │
│  │           │              │              │                   │    │
│  │           ▼              ▼              ▼                   │    │
│  │     ┌──────────┐  ┌──────────┐  ┌──────────┐              │    │
│  │     │  Role 1  │  │  Role 2  │  │  Role 3  │   RBAC       │    │
│  │     └────┬─────┘  └────┬─────┘  └────┬─────┘              │    │
│  │          │             │             │                     │    │
│  │          └─────────────┼─────────────┘                     │    │
│  │                        │                                   │    │
│  │                        ▼                                   │    │
│  │              ┌───────────────────┐                         │    │
│  │              │   Permissions     │                         │    │
│  │              │   Aggregation     │                         │    │
│  │              └─────────┬─────────┘                         │    │
│  │                        │                                   │    │
│  │                        ▼                                   │    │
│  │              ┌───────────────────┐                         │    │
│  │              │    Attribute      │                         │    │
│  │              │    Conditions     │   ABAC Extensions       │    │
│  │              │  (Owner, Premium) │                         │    │
│  │              └─────────┬─────────┘                         │    │
│  │                        │                                   │    │
│  │                        ▼                                   │    │
│  │              ┌───────────────────┐                         │    │
│  │              │  Access Decision  │                         │    │
│  │              │  ALLOW / DENY     │                         │    │
│  │              └───────────────────┘                         │    │
│  │                                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Role Definitions

### 3.1 User Roles

| Role | Code | Description | Auto-Assigned |
|------|------|-------------|---------------|
| **Guest** | `GUEST` | Unauthenticated user | Yes |
| **User** | `USER` | Registered user | On registration |
| **Premium User** | `PREMIUM` | Paid subscriber | On subscription |
| **Author** | `AUTHOR` | Content creator | On application |
| **Verified Author** | `VERIFIED_AUTHOR` | Verified creator | On verification |
| **Moderator** | `MODERATOR` | Content moderator | Admin assigned |
| **Admin** | `ADMIN` | System administrator | Admin assigned |
| **Super Admin** | `SUPER_ADMIN` | Full access | Manual only |

### 3.2 Role Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ROLE HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                      SUPER_ADMIN                                     │
│                           │                                          │
│                           ▼                                          │
│                        ADMIN                                         │
│                           │                                          │
│                           ▼                                          │
│                      MODERATOR                                       │
│                           │                                          │
│              ┌────────────┼────────────┐                            │
│              │            │            │                            │
│              ▼            ▼            ▼                            │
│      VERIFIED_AUTHOR   AUTHOR      PREMIUM                          │
│              │            │            │                            │
│              └────────────┼────────────┘                            │
│                           │                                          │
│                           ▼                                          │
│                         USER                                         │
│                           │                                          │
│                           ▼                                          │
│                        GUEST                                         │
│                                                                      │
│  Note: Higher roles inherit permissions from lower roles             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Role Capabilities Summary

| Role | Read Public | Create Content | Moderate | Admin Panel |
|------|-------------|----------------|----------|-------------|
| Guest | ✓ | ✗ | ✗ | ✗ |
| User | ✓ | Limited | ✗ | ✗ |
| Premium | ✓ | Limited | ✗ | ✗ |
| Author | ✓ | Full | ✗ | ✗ |
| Verified Author | ✓ | Full + Premium Features | ✗ | ✗ |
| Moderator | ✓ | Limited | ✓ | Limited |
| Admin | ✓ | ✓ | ✓ | ✓ |
| Super Admin | ✓ | ✓ | ✓ | Full |

---

## 4. Stories Permission Matrix

### 4.1 Story Actions

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **List (public)** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **View (public)** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **View (private)** | ✗ | Owner | Owner | Owner | ✓ | ✓ |
| **Search** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Create** | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ |
| **Update** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **Delete** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **Publish** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **Unpublish** | ✗ | ✗ | ✗ | Owner | ✓ | ✓ |
| **Feature** | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **Pin** | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **Report** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Hide (moderation)** | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |

### 4.2 Chapter Actions

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **List** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **View (free)** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **View (locked)** | ✗ | Purchased | ✓* | Owner | ✓ | ✓ |
| **Create** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **Update** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **Delete** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **Publish** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **Schedule** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **Reorder** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **Unlock (pay)** | ✗ | ✓ | ✓ | N/A | N/A | N/A |

*Premium users can access locked chapters if story allows premium access

---

## 5. User Permission Matrix

### 5.1 Profile Actions

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **View public** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **View private** | ✗ | Owner | Owner | Owner | ✓ | ✓ |
| **Update own** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Update other** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Delete own** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Delete other** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Upload avatar** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Change password** | ✗ | Owner | Owner | Owner | Owner | ✓ |
| **View email** | ✗ | Owner | Owner | Owner | ✓ | ✓ |

### 5.2 Social Actions

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **Follow user** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Unfollow user** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Block user** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **View followers** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **View following** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Report user** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |

### 5.3 Account Management

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **Suspend account** | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **Ban account** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Verify email** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Reset MFA** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Assign role** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **View audit log** | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |

---

## 6. Interactions Permission Matrix

### 6.1 Comments

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **View** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Create** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Update own** | ✗ | ✓* | ✓* | ✓* | ✓* | ✓ |
| **Delete own** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Delete any** | ✗ | ✗ | ✗ | Story Owner | ✓ | ✓ |
| **Reply** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Like** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Report** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Hide (mod)** | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **Pin** | ✗ | ✗ | ✗ | Story Owner | ✓ | ✓ |

*Within edit window (e.g., 30 minutes)

### 6.2 Ratings & Reviews

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **View** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Create** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Update own** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Delete own** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Delete any** | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **Report** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |

### 6.3 Bookmarks & Reading Lists

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **View own** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Create** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Update** | ✗ | Owner | Owner | Owner | ✗ | ✓ |
| **Delete** | ✗ | Owner | Owner | Owner | ✗ | ✓ |
| **Share (public list)** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **View shared** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Max lists** | 0 | 10 | 50 | 50 | 50 | ∞ |

---

## 7. Payment Permission Matrix

### 7.1 Wallet Operations

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **View balance** | ✗ | Owner | Owner | Owner | ✗ | ✓ |
| **View history** | ✗ | Owner | Owner | Owner | ✗ | ✓ |
| **Top up** | ✗ | ✓ | ✓ | ✓ | ✗ | ✓ |
| **Request payout** | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ |
| **Process payout** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Refund** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Adjust balance** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

### 7.2 Donations

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **Send donation** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Receive donation** | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ |
| **View received** | ✗ | ✗ | ✗ | Owner | ✗ | ✓ |
| **View sent** | ✗ | Owner | Owner | Owner | ✗ | ✓ |
| **Refund donation** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

### 7.3 Subscriptions

| Action | Guest | User | Premium | Author | Mod | Admin |
|--------|-------|------|---------|--------|-----|-------|
| **Subscribe** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Cancel** | ✗ | Owner | Owner | Owner | ✗ | ✓ |
| **View own** | ✗ | Owner | Owner | Owner | ✗ | ✓ |
| **View any** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Extend** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Revoke** | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

---

## 8. Admin Permission Matrix

### 8.1 Content Moderation

| Action | Mod | Admin | Super Admin |
|--------|-----|-------|-------------|
| **View reports** | ✓ | ✓ | ✓ |
| **Resolve reports** | ✓ | ✓ | ✓ |
| **Hide content** | ✓ | ✓ | ✓ |
| **Delete content** | ✗ | ✓ | ✓ |
| **Warn user** | ✓ | ✓ | ✓ |
| **Suspend user** | ✓ | ✓ | ✓ |
| **Ban user** | ✗ | ✓ | ✓ |
| **View mod log** | Own | All | All |

### 8.2 System Administration

| Action | Mod | Admin | Super Admin |
|--------|-----|-------|-------------|
| **View dashboard** | ✓ | ✓ | ✓ |
| **View analytics** | Limited | ✓ | ✓ |
| **Manage categories** | ✗ | ✓ | ✓ |
| **Manage tags** | ✓ | ✓ | ✓ |
| **Manage users** | ✗ | ✓ | ✓ |
| **Manage roles** | ✗ | ✗ | ✓ |
| **View audit logs** | Own | ✓ | ✓ |
| **Manage settings** | ✗ | ✓ | ✓ |
| **Manage API keys** | ✗ | ✗ | ✓ |
| **Database access** | ✗ | ✗ | ✓ |

### 8.3 Financial Administration

| Action | Mod | Admin | Super Admin |
|--------|-----|-------|-------------|
| **View transactions** | ✗ | ✓ | ✓ |
| **Process refunds** | ✗ | ✓ | ✓ |
| **Process payouts** | ✗ | ✓ | ✓ |
| **View revenue** | ✗ | ✓ | ✓ |
| **Adjust balances** | ✗ | ✗ | ✓ |
| **Configure pricing** | ✗ | ✗ | ✓ |

---

## 9. API Access Matrix

### 9.1 Public Endpoints

| Endpoint | Guest | Authenticated |
|----------|-------|---------------|
| `GET /v1/stories` | ✓ | ✓ |
| `GET /v1/stories/{id}` | ✓ | ✓ |
| `GET /v1/stories/{id}/chapters` | ✓ | ✓ |
| `GET /v1/categories` | ✓ | ✓ |
| `GET /v1/tags` | ✓ | ✓ |
| `GET /v1/users/{id}/profile` | ✓ | ✓ |
| `POST /v1/auth/login` | ✓ | ✗ |
| `POST /v1/auth/register` | ✓ | ✗ |

### 9.2 Protected Endpoints

| Endpoint | User | Premium | Author | Admin |
|----------|------|---------|--------|-------|
| `POST /v1/stories` | ✗ | ✗ | ✓ | ✓ |
| `PUT /v1/stories/{id}` | ✗ | ✗ | Owner | ✓ |
| `DELETE /v1/stories/{id}` | ✗ | ✗ | Owner | ✓ |
| `POST /v1/comments` | ✓ | ✓ | ✓ | ✓ |
| `POST /v1/donations` | ✓ | ✓ | ✓ | ✓ |
| `GET /v1/me/wallet` | ✓ | ✓ | ✓ | ✓ |
| `POST /v1/wallet/topup` | ✓ | ✓ | ✓ | ✓ |
| `POST /v1/author/payout` | ✗ | ✗ | ✓ | ✓ |

### 9.3 Admin Endpoints

| Endpoint | Mod | Admin | Super Admin |
|----------|-----|-------|-------------|
| `GET /v1/admin/reports` | ✓ | ✓ | ✓ |
| `POST /v1/admin/reports/{id}/resolve` | ✓ | ✓ | ✓ |
| `POST /v1/admin/users/{id}/suspend` | ✓ | ✓ | ✓ |
| `POST /v1/admin/users/{id}/ban` | ✗ | ✓ | ✓ |
| `GET /v1/admin/analytics` | ✗ | ✓ | ✓ |
| `POST /v1/admin/payouts/process` | ✗ | ✓ | ✓ |
| `POST /v1/admin/roles/assign` | ✗ | ✗ | ✓ |

---

## 10. Attribute-Based Conditions

### 10.1 Common Conditions

| Condition | Syntax | Description |
|-----------|--------|-------------|
| **Owner** | `resource.ownerId == user.id` | User owns the resource |
| **Premium** | `user.hasPremium == true` | User has active premium |
| **Verified** | `user.emailVerified == true` | Email is verified |
| **Not Suspended** | `user.status != SUSPENDED` | Account not suspended |
| **Age Verified** | `user.ageVerified == true` | Age verified for mature |
| **Same User** | `target.id == user.id` | Target is current user |

### 10.2 Resource-Specific Conditions

| Resource | Condition | Use Case |
|----------|-----------|----------|
| **Story** | `story.visibility == PUBLIC` | Public visibility |
| **Story** | `story.authorId == user.id` | Author of story |
| **Chapter** | `chapter.price == 0 OR purchased` | Free or purchased |
| **Comment** | `comment.userId == user.id` | Comment author |
| **Comment** | `now < comment.createdAt + 30min` | Within edit window |

---

## 11. Rate Limits by Role

### 11.1 API Rate Limits

| Role | Requests/min | Requests/hour | Burst |
|------|--------------|---------------|-------|
| Guest | 30 | 300 | 50 |
| User | 60 | 1,000 | 100 |
| Premium | 120 | 5,000 | 200 |
| Author | 180 | 10,000 | 300 |
| Moderator | 300 | 20,000 | 500 |
| Admin | 600 | 50,000 | 1000 |

### 11.2 Action-Specific Limits

| Action | User | Premium | Author |
|--------|------|---------|--------|
| Comments/day | 50 | 100 | 100 |
| Donations/day | 10 | 50 | 50 |
| Reports/day | 5 | 10 | 10 |
| Stories/day | N/A | N/A | 5 |
| Chapters/day | N/A | N/A | 20 |

---

## 12. Audit Requirements

### 12.1 Audited Actions

| Category | Actions |
|----------|---------|
| **Authentication** | Login, logout, failed attempts, password change |
| **Authorization** | Permission denied, role changes |
| **Content** | Create, update, delete, publish, hide |
| **Moderation** | Reports, warnings, suspensions, bans |
| **Financial** | Payments, refunds, payouts, balance adjustments |
| **Admin** | Settings changes, role assignments |

### 12.2 Audit Log Format

```json
{
  "timestamp": "2025-12-31T10:00:00Z",
  "eventType": "CONTENT_UPDATE",
  "actor": {
    "userId": "uuid",
    "roles": ["AUTHOR"],
    "ipAddress": "192.168.1.1"
  },
  "resource": {
    "type": "STORY",
    "id": "uuid"
  },
  "action": "UPDATE",
  "result": "SUCCESS",
  "changes": {
    "title": {
      "old": "Old Title",
      "new": "New Title"
    }
  },
  "requestId": "uuid"
}
```

---

## 13. References

### 13.1 Related Specifications
- [05_SECURITY_AND_COMPLIANCE.md](05_SECURITY_AND_COMPLIANCE.md)
- [08_API_STANDARDS.md](08_API_STANDARDS.md)
- [09_ERROR_HANDLING.md](09_ERROR_HANDLING.md)

### 13.2 Design Documents
- [02_USERS_COMPONENT.md](../Design/Components/02_USERS_COMPONENT.md)
- [10_MODERATION_ADMIN_COMPONENT.md](../Design/Components/10_MODERATION_ADMIN_COMPONENT.md)
