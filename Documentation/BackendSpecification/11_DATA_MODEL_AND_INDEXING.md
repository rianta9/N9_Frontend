# Data Model & Indexing Guide (Target)

This guide complements the ERD in `02_DATABASE_SCHEMA.md` with production-oriented indexing/partitioning and data ownership.

## 1. Data Ownership (Module → Tables)
- **Users & Identity**: `Account`, `Role`, `Permission`, `UserPermission`, `UserSetting`, `UserWallet`.
- **Stories**: `Story`, `Chapter`, `Category`, `StoryCategory`, `StoryTag`, `Title`, `Status`.
- **Interactions**: `UserFavourite`, `StoryRating`, `Review`, `Comment`, `StoryReport`, `ChapterReport`.
- **Readings**: `ReadingProgress`, `UserChapterRead`.
- **Payments**: `PaymentHistory`, `PaymentHistoryDetail`, `DiscountCode`, `StoryDonate`.
- **Notifications**: `UserNotification`.
- **Admin/Audit**: `AdminActivity`, `UserActivity`, `Event`.
- **Advertising**: `Banner`.
- **AI Automation (proposed)**: `ContentSubmission`, `AiJob`, `AiReviewResult`, `TranslationArtifact`, `AiPolicyVersion`, `AiPromptVersion`.

## 2. Mandatory Constraints (Examples)
- Prevent duplicates:
  - Follow: unique (`userId`, `storyId`) on `UserFavourite`.
  - Rating/review: unique (`userId`, `storyId`) on `StoryRating` / `Review` (as per business rule).
  - Reading progress: unique (`userId`, `storyId`) on `ReadingProgress`.

- AI automation (proposed):
  - Translation artifact: unique (`targetType`, `targetId`, `language`).
  - Submission dedupe: unique (`targetType`, `targetId`, `contentChecksum`) (or via partial unique on active submissions).

- Stories (performance + gating):
  - Additive columns recommended: `Story.visibility`, `Story.version`, `Chapter.visibility`, `Chapter.version`.
  - Public reads must filter `visibility=VISIBLE`.

## 3. Indexing Recommendations
### 3.1 Hot Read Paths
- Story browse: index on (`status`, `createdAt`), (`status`, `updatedAt`), and optionally (`status`, `viewCount`) via `StoryStatistic`.
- For approval gating + browse filtering, add composite index on (`visibility`, `status`, `updatedAt`).
- Chapter list: index on (`storyId`, `index`).
- If filtering hidden/pending chapters in lists, use (`storyId`, `visibility`, `index`).
- Comments: index on (`chapterId`, `createdAt`).
- Notifications: index on (`userId`, `isRead`, `createdAt`).

- AI submissions queue (proposed):
  - `ContentSubmission(status, updatedAt)` for moderation queue.
  - `ContentSubmission(targetType, targetId)` for lookup from story/chapter.
  - `AiJob(status, updatedAt)` for worker polling.

### 3.2 Search
- Start with `GIN`/`btree` indexes depending on query patterns (e.g., title prefix, trigram for contains).
- Add composite indexes for common filters: (`categoryId`, `status`), (`authorId`, `status`).

## 4. Partitioning (When Needed)
Consider partitioning by month once tables exceed tens of millions of rows:
- `UserActivity`, `Event`, `UserNotification`.

## 5. Auditing
- Financial tables must retain immutable history (append-only preferred).
- Admin actions must be auditable in `AdminActivity`.
