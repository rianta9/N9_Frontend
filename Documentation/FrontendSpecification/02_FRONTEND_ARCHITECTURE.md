# Frontend Architecture Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Frontend Architecture Team |
| Review Cycle | Quarterly |
| Related Documents | 01_SYSTEM_OVERVIEW.md, Frontend Design Architecture |

---

## 2. Architecture Overview

### 2.1 Design Principles

| Principle | Description | Implementation |
|-----------|-------------|----------------|
| **Component-Based** | Build UI as composable, reusable components | React functional components |
| **Separation of Concerns** | Distinct layers for UI, logic, and data | Feature-based architecture |
| **Type Safety** | Catch errors at compile time | TypeScript throughout |
| **Performance First** | Optimize for Core Web Vitals | Code splitting, lazy loading |
| **Accessibility** | Inclusive by design | ARIA, semantic HTML |
| **Testability** | Easy to test all layers | Dependency injection, mocking |

### 2.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        N9 FRONTEND ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         PRESENTATION LAYER                              │ │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐  │ │
│  │  │    Pages     │ │   Layouts    │ │  Components  │ │    Styles    │  │ │
│  │  │              │ │              │ │              │ │              │  │ │
│  │  │ • HomePage   │ │ • MainLayout │ │ • StoryCard  │ │ • Tailwind   │  │ │
│  │  │ • ReaderPage │ │ • AuthLayout │ │ • Button     │ │ • CSS Vars   │  │ │
│  │  │ • AuthorPage │ │ • AdminLayout│ │ • Modal      │ │ • Themes     │  │ │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         APPLICATION LAYER                               │ │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐  │ │
│  │  │    Hooks     │ │    State     │ │    Forms     │ │   Business   │  │ │
│  │  │              │ │              │ │              │ │    Logic     │  │ │
│  │  │ • useAuth    │ │ • AuthStore  │ │ • LoginForm  │ │ • Validators │  │ │
│  │  │ • useStory   │ │ • ThemeStore │ │ • StoryForm  │ │ • Formatters │  │ │
│  │  │ • useReader  │ │ • UIStore    │ │ • ChapterForm│ │ • Calculators│  │ │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                           DOMAIN LAYER                                  │ │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐  │ │
│  │  │    Types     │ │  Interfaces  │ │   Schemas    │ │   Constants  │  │ │
│  │  │              │ │              │ │              │ │              │  │ │
│  │  │ • Story      │ │ • IUser      │ │ • LoginSchema│ │ • Roles      │  │ │
│  │  │ • Chapter    │ │ • IStory     │ │ • StorySchema│ │ • Statuses   │  │ │
│  │  │ • User       │ │ • IChapter   │ │ • UserSchema │ │ • Routes     │  │ │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                       INFRASTRUCTURE LAYER                              │ │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐  │ │
│  │  │  API Client  │ │  WebSocket   │ │   Storage    │ │   Services   │  │ │
│  │  │              │ │              │ │              │ │              │  │ │
│  │  │ • Axios      │ │ • Socket.io  │ │ • LocalStore │ │ • Analytics  │  │ │
│  │  │ • Interceptors│ │ • Events    │ │ • IndexedDB  │ │ • Sentry     │  │ │
│  │  │ • Retry      │ │ • Reconnect  │ │ • Cache      │ │ • Push       │  │ │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Technology Stack

### 3.1 Core Technologies

| Category | Technology | Version | Purpose |
|----------|------------|---------|---------|
| **Runtime** | Node.js | 20 LTS | Development & Build |
| **Framework** | React | 18.3+ | UI Library |
| **Language** | TypeScript | 5.3+ | Type Safety |
| **Build Tool** | Vite | 5.x | Build & Dev Server |
| **Package Manager** | pnpm | 8.x | Dependency Management |

### 3.2 State Management Stack

| Technology | Purpose | Use Case |
|------------|---------|----------|
| **TanStack Query** | Server State | API data fetching, caching, synchronization |
| **Zustand** | Client State | Global UI state, user preferences |
| **React Hook Form** | Form State | Form handling, validation |
| **URL State** | Navigation State | Filters, pagination, search params |

### 3.3 UI & Styling Stack

| Technology | Purpose | Rationale |
|------------|---------|-----------|
| **Tailwind CSS** | Utility Styling | Rapid development, consistent design |
| **shadcn/ui** | Component Base | Accessible, customizable components |
| **Framer Motion** | Animations | Smooth, performant animations |
| **Lucide Icons** | Iconography | Consistent, tree-shakable icons |
| **Recharts** | Data Visualization | Responsive charts for analytics |

### 3.4 Testing Stack

| Technology | Purpose | Coverage Target |
|------------|---------|-----------------|
| **Vitest** | Unit Testing | 80% business logic |
| **React Testing Library** | Component Testing | Key user interactions |
| **Playwright** | E2E Testing | Critical user flows |
| **MSW** | API Mocking | Isolated component tests |

### 3.5 Development Tools

| Tool | Purpose |
|------|---------|
| **ESLint** | Code linting & quality |
| **Prettier** | Code formatting |
| **Husky** | Git hooks |
| **lint-staged** | Pre-commit linting |
| **TypeScript ESLint** | Type-aware linting |

---

## 4. Project Structure

### 4.1 Directory Organization

```
src/
├── app/                          # Application setup
│   ├── App.tsx                   # Root component
│   ├── providers.tsx             # Provider composition
│   ├── router.tsx                # Route configuration
│   └── main.tsx                  # Entry point
│
├── pages/                        # Page components (route-level)
│   ├── auth/                     # Authentication pages
│   │   ├── LoginPage.tsx
│   │   ├── RegisterPage.tsx
│   │   ├── ForgotPasswordPage.tsx
│   │   └── index.ts
│   ├── home/                     # Home & discovery
│   │   ├── HomePage.tsx
│   │   ├── LandingPage.tsx
│   │   └── index.ts
│   ├── browse/                   # Browse & search
│   │   ├── BrowsePage.tsx
│   │   ├── SearchPage.tsx
│   │   ├── RankingsPage.tsx
│   │   └── index.ts
│   ├── story/                    # Story pages
│   │   ├── StoryDetailPage.tsx
│   │   ├── ChapterListPage.tsx
│   │   └── index.ts
│   ├── reader/                   # Reading experience
│   │   ├── ReaderPage.tsx
│   │   ├── LibraryPage.tsx
│   │   ├── BookmarksPage.tsx
│   │   └── index.ts
│   ├── author/                   # Author dashboard
│   │   ├── DashboardPage.tsx
│   │   ├── StoryEditorPage.tsx
│   │   ├── ChapterEditorPage.tsx
│   │   ├── AnalyticsPage.tsx
│   │   ├── EarningsPage.tsx
│   │   └── index.ts
│   ├── profile/                  # User profile
│   │   ├── ProfilePage.tsx
│   │   ├── SettingsPage.tsx
│   │   └── index.ts
│   ├── payment/                  # Payments
│   │   ├── WalletPage.tsx
│   │   ├── TopUpPage.tsx
│   │   └── index.ts
│   ├── admin/                    # Admin portal
│   │   ├── DashboardPage.tsx
│   │   ├── ModerationPage.tsx
│   │   ├── UsersPage.tsx
│   │   └── index.ts
│   └── errors/                   # Error pages
│       ├── NotFoundPage.tsx
│       ├── ErrorPage.tsx
│       └── index.ts
│
├── features/                     # Feature modules
│   ├── auth/                     # Authentication feature
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   ├── RegisterForm.tsx
│   │   │   └── OAuthButtons.tsx
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   ├── useLogin.ts
│   │   │   └── useLogout.ts
│   │   ├── api/
│   │   │   ├── authApi.ts
│   │   │   └── authQueries.ts
│   │   ├── stores/
│   │   │   └── authStore.ts
│   │   ├── types.ts
│   │   ├── schemas.ts
│   │   └── index.ts
│   │
│   ├── stories/                  # Stories feature
│   │   ├── components/
│   │   │   ├── StoryCard.tsx
│   │   │   ├── StoryGrid.tsx
│   │   │   ├── StoryFilters.tsx
│   │   │   ├── StoryMeta.tsx
│   │   │   └── ChapterList.tsx
│   │   ├── hooks/
│   │   │   ├── useStoryQuery.ts
│   │   │   ├── useStoryMutation.ts
│   │   │   └── useStoryFilters.ts
│   │   ├── api/
│   │   │   ├── storyApi.ts
│   │   │   └── storyQueries.ts
│   │   ├── types.ts
│   │   └── index.ts
│   │
│   ├── reader/                   # Reader feature
│   │   ├── components/
│   │   │   ├── ChapterContent.tsx
│   │   │   ├── ReaderHeader.tsx
│   │   │   ├── ReaderFooter.tsx
│   │   │   ├── ReaderSettings.tsx
│   │   │   ├── TableOfContents.tsx
│   │   │   └── ProgressBar.tsx
│   │   ├── hooks/
│   │   │   ├── useReaderSettings.ts
│   │   │   ├── useReadingProgress.ts
│   │   │   └── useChapterNavigation.ts
│   │   ├── stores/
│   │   │   └── readerSettingsStore.ts
│   │   └── index.ts
│   │
│   ├── interactions/             # Social interactions
│   │   ├── components/
│   │   │   ├── LikeButton.tsx
│   │   │   ├── FollowButton.tsx
│   │   │   ├── ReviewCard.tsx
│   │   │   ├── CommentThread.tsx
│   │   │   └── RatingStars.tsx
│   │   ├── hooks/
│   │   │   ├── useLike.ts
│   │   │   ├── useFollow.ts
│   │   │   └── useComments.ts
│   │   └── index.ts
│   │
│   ├── payments/                 # Payment feature
│   │   ├── components/
│   │   │   ├── WalletCard.tsx
│   │   │   ├── CoinPackages.tsx
│   │   │   ├── TransactionList.tsx
│   │   │   └── UnlockModal.tsx
│   │   ├── hooks/
│   │   │   ├── useWallet.ts
│   │   │   └── useUnlock.ts
│   │   └── index.ts
│   │
│   ├── notifications/            # Notifications feature
│   │   ├── components/
│   │   │   ├── NotificationBell.tsx
│   │   │   ├── NotificationList.tsx
│   │   │   └── NotificationItem.tsx
│   │   ├── hooks/
│   │   │   └── useNotifications.ts
│   │   └── index.ts
│   │
│   ├── search/                   # Search feature
│   │   ├── components/
│   │   │   ├── SearchBar.tsx
│   │   │   ├── SearchResults.tsx
│   │   │   └── SearchFilters.tsx
│   │   ├── hooks/
│   │   │   └── useSearch.ts
│   │   └── index.ts
│   │
│   └── admin/                    # Admin feature
│       ├── components/
│       ├── hooks/
│       └── index.ts
│
├── components/                   # Shared components
│   ├── ui/                       # Base UI (shadcn)
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── card.tsx
│   │   ├── modal.tsx
│   │   ├── dropdown.tsx
│   │   ├── toast.tsx
│   │   ├── skeleton.tsx
│   │   └── index.ts
│   │
│   ├── layout/                   # Layout components
│   │   ├── Header.tsx
│   │   ├── Footer.tsx
│   │   ├── Sidebar.tsx
│   │   ├── MainLayout.tsx
│   │   ├── AuthLayout.tsx
│   │   ├── ReaderLayout.tsx
│   │   ├── AuthorLayout.tsx
│   │   └── AdminLayout.tsx
│   │
│   ├── common/                   # Common components
│   │   ├── Logo.tsx
│   │   ├── Avatar.tsx
│   │   ├── Badge.tsx
│   │   ├── Spinner.tsx
│   │   ├── ErrorBoundary.tsx
│   │   ├── LoadingState.tsx
│   │   ├── EmptyState.tsx
│   │   ├── Pagination.tsx
│   │   ├── InfiniteScroll.tsx
│   │   └── SEOHead.tsx
│   │
│   └── forms/                    # Form components
│       ├── FormField.tsx
│       ├── FormError.tsx
│       ├── FormLabel.tsx
│       ├── ImageUpload.tsx
│       └── RichTextEditor.tsx
│
├── hooks/                        # Shared hooks
│   ├── useDebounce.ts
│   ├── useMediaQuery.ts
│   ├── useInfiniteScroll.ts
│   ├── useLocalStorage.ts
│   ├── useOnClickOutside.ts
│   ├── useKeyPress.ts
│   ├── usePrevious.ts
│   └── index.ts
│
├── lib/                          # Core utilities
│   ├── api/                      # API client
│   │   ├── client.ts
│   │   ├── interceptors.ts
│   │   ├── endpoints.ts
│   │   └── types.ts
│   │
│   ├── websocket/                # WebSocket client
│   │   ├── client.ts
│   │   ├── events.ts
│   │   └── handlers.ts
│   │
│   ├── storage/                  # Storage utilities
│   │   ├── localStorage.ts
│   │   ├── sessionStorage.ts
│   │   └── indexedDB.ts
│   │
│   ├── analytics/                # Analytics
│   │   ├── tracker.ts
│   │   └── events.ts
│   │
│   └── utils/                    # Utility functions
│       ├── format.ts
│       ├── validation.ts
│       ├── cn.ts
│       ├── date.ts
│       ├── number.ts
│       └── string.ts
│
├── stores/                       # Global stores
│   ├── authStore.ts
│   ├── themeStore.ts
│   ├── uiStore.ts
│   └── index.ts
│
├── types/                        # TypeScript types
│   ├── api.ts
│   ├── models.ts
│   ├── common.ts
│   └── index.ts
│
├── styles/                       # Global styles
│   ├── globals.css
│   ├── themes.css
│   └── typography.css
│
├── assets/                       # Static assets
│   ├── images/
│   ├── fonts/
│   └── icons/
│
├── config/                       # Configuration
│   ├── constants.ts
│   ├── env.ts
│   ├── routes.ts
│   └── queryClient.ts
│
└── test/                         # Test utilities
    ├── setup.ts
    ├── mocks/
    ├── factories/
    └── utils.ts
```

### 4.2 Feature Module Structure

```
features/<feature-name>/
├── components/               # Feature-specific components
│   ├── Component1.tsx
│   ├── Component2.tsx
│   └── index.ts
│
├── hooks/                    # Feature hooks
│   ├── useFeatureQuery.ts
│   ├── useFeatureMutation.ts
│   └── index.ts
│
├── api/                      # API integration
│   ├── featureApi.ts         # API functions
│   └── featureQueries.ts     # React Query hooks
│
├── stores/                   # Feature stores (if needed)
│   └── featureStore.ts
│
├── types.ts                  # Feature-specific types
├── schemas.ts                # Zod validation schemas
├── constants.ts              # Feature constants
└── index.ts                  # Public exports
```

---

## 5. Component Architecture

### 5.1 Component Classification

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        COMPONENT CLASSIFICATION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ PAGES (Route-Level)                                                     ││
│  │ • Connected to router                                                   ││
│  │ • Handle data fetching                                                  ││
│  │ • Compose layouts and features                                          ││
│  │ • Examples: HomePage, StoryDetailPage, ReaderPage                       ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                       │                                      │
│                                       ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ FEATURE COMPONENTS (Feature-Specific)                                   ││
│  │ • Domain-specific logic                                                 ││
│  │ • Connected to feature state                                            ││
│  │ • May contain business logic                                            ││
│  │ • Examples: StoryCard, ChapterList, ReviewForm                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                       │                                      │
│                                       ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ SHARED COMPONENTS (Reusable)                                            ││
│  │ • Pure presentation                                                     ││
│  │ • Props-driven                                                          ││
│  │ • Highly reusable                                                       ││
│  │ • Examples: Button, Card, Modal                                         ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                       │                                      │
│                                       ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ UI PRIMITIVES (Base)                                                    ││
│  │ • Atomic components                                                     ││
│  │ • Design system tokens                                                  ││
│  │ • Accessibility built-in                                                ││
│  │ • Examples: Input, Label, Icon                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Component Design Patterns

#### 5.2.1 Composition Pattern

```typescript
// Component composition example
interface PageLayoutProps {
  header?: React.ReactNode;
  sidebar?: React.ReactNode;
  children: React.ReactNode;
  footer?: React.ReactNode;
}

const PageLayout: React.FC<PageLayoutProps> = ({
  header,
  sidebar,
  children,
  footer,
}) => (
  <div className="min-h-screen flex flex-col">
    {header && <header className="sticky top-0 z-50">{header}</header>}
    <div className="flex-1 flex">
      {sidebar && <aside className="w-64 shrink-0">{sidebar}</aside>}
      <main className="flex-1">{children}</main>
    </div>
    {footer && <footer>{footer}</footer>}
  </div>
);
```

#### 5.2.2 Render Props Pattern

```typescript
// Render props for flexible rendering
interface DataListProps<T> {
  data: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  renderEmpty?: () => React.ReactNode;
  renderLoading?: () => React.ReactNode;
  isLoading?: boolean;
}

function DataList<T>({
  data,
  renderItem,
  renderEmpty,
  renderLoading,
  isLoading,
}: DataListProps<T>) {
  if (isLoading && renderLoading) return <>{renderLoading()}</>;
  if (data.length === 0 && renderEmpty) return <>{renderEmpty()}</>;
  return <>{data.map((item, index) => renderItem(item, index))}</>;
}
```

#### 5.2.3 Container/Presentational Pattern

```typescript
// Container: Handles logic and data
const StoryListContainer: React.FC = () => {
  const { data, isLoading, error } = useStoryListQuery();
  const filters = useStoryFilters();
  
  if (error) return <ErrorState error={error} />;
  
  return (
    <StoryList
      stories={data?.stories ?? []}
      isLoading={isLoading}
      filters={filters}
    />
  );
};

// Presentational: Pure UI rendering
interface StoryListProps {
  stories: Story[];
  isLoading: boolean;
  filters: StoryFilters;
}

const StoryList: React.FC<StoryListProps> = ({
  stories,
  isLoading,
  filters,
}) => (
  <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
    {isLoading
      ? <StoryListSkeleton count={6} />
      : stories.map(story => <StoryCard key={story.id} story={story} />)
    }
  </div>
);
```

### 5.3 Component Standards

#### 5.3.1 File Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Components | PascalCase | `StoryCard.tsx` |
| Hooks | camelCase with "use" prefix | `useStoryQuery.ts` |
| Utilities | camelCase | `formatDate.ts` |
| Types | PascalCase | `Story.ts` |
| Constants | UPPER_SNAKE_CASE | `API_ENDPOINTS.ts` |
| Styles | kebab-case | `story-card.css` |

#### 5.3.2 Component File Structure

```typescript
// ComponentName.tsx
import { type FC } from 'react';
// External imports
import { cn } from '@/lib/utils';
// Internal imports
import { ComponentProps } from './types';

// Types (if not in separate file)
interface ComponentNameProps {
  // Props definition
}

// Component
export const ComponentName: FC<ComponentNameProps> = ({
  prop1,
  prop2,
  ...rest
}) => {
  // Hooks
  // State
  // Effects
  // Handlers
  // Render
  return (
    <div className={cn('base-classes', className)} {...rest}>
      {/* Content */}
    </div>
  );
};

// Display name for debugging
ComponentName.displayName = 'ComponentName';

// Default export (optional)
export default ComponentName;
```

---

## 6. Data Flow Architecture

### 6.1 Unidirectional Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        UNIDIRECTIONAL DATA FLOW                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                          ┌─────────────────┐                                │
│                          │     ACTION      │                                │
│                          │  User clicks,   │                                │
│                          │  Form submits   │                                │
│                          └────────┬────────┘                                │
│                                   │                                          │
│                                   ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                        │  │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐   │  │
│  │  │   API Client    │───▶│   React Query   │───▶│     Store       │   │  │
│  │  │   (Mutation)    │    │     Cache       │    │   (Zustand)     │   │  │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────┘   │  │
│  │                                                                        │  │
│  │                         STATE MANAGEMENT                               │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                   │                                          │
│                                   ▼                                          │
│                          ┌─────────────────┐                                │
│                          │      VIEW       │                                │
│                          │   Components    │                                │
│                          │   re-render     │                                │
│                          └─────────────────┘                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 State Categories

| Category | Manager | Scope | Examples |
|----------|---------|-------|----------|
| **Server State** | TanStack Query | Global | API data, paginated lists |
| **Client State** | Zustand | Global | User preferences, UI state |
| **Form State** | React Hook Form | Component | Form inputs, validation |
| **URL State** | React Router | Global | Filters, pagination, search |
| **Component State** | useState | Component | Toggle states, local UI |

---

## 7. Layout System

### 7.1 Layout Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           LAYOUT SYSTEM                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                          ROOT LAYOUT                                    ││
│  │  • Provider wrapping (Query, Theme, Auth)                              ││
│  │  • Error boundaries                                                     ││
│  │  • Global modals and toasts                                            ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                       │                                      │
│           ┌───────────────────────────┼───────────────────────────┐         │
│           │                           │                           │         │
│           ▼                           ▼                           ▼         │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐ │
│  │   MAIN LAYOUT   │        │   AUTH LAYOUT   │        │  READER LAYOUT  │ │
│  │                 │        │                 │        │                 │ │
│  │  • Header       │        │  • Minimal      │        │  • Immersive    │ │
│  │  • Navigation   │        │  • Centered     │        │  • No chrome    │ │
│  │  • Footer       │        │  • Guest only   │        │  • Settings     │ │
│  │  • Sidebar?     │        │                 │        │  • TOC          │ │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘ │
│           │                                                     │           │
│           ▼                                                     ▼           │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                          PAGE LAYOUTS                                   ││
│  │                                                                         ││
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐││
│  │  │  Author   │ │   Admin   │ │  Profile  │ │  Payment  │ │  Browse   │││
│  │  │  Layout   │ │  Layout   │ │  Layout   │ │  Layout   │ │  Layout   │││
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘ └───────────┘││
│  │                                                                         ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Layout Components

#### 7.2.1 Main Layout

```typescript
// MainLayout.tsx
interface MainLayoutProps {
  children: React.ReactNode;
  showSidebar?: boolean;
}

const MainLayout: FC<MainLayoutProps> = ({ children, showSidebar = false }) => (
  <div className="min-h-screen bg-background">
    <Header />
    <div className="flex">
      {showSidebar && <Sidebar />}
      <main className="flex-1 container mx-auto px-4 py-8">
        {children}
      </main>
    </div>
    <Footer />
  </div>
);
```

#### 7.2.2 Reader Layout

```typescript
// ReaderLayout.tsx
interface ReaderLayoutProps {
  children: React.ReactNode;
  theme: 'light' | 'dark' | 'sepia';
}

const ReaderLayout: FC<ReaderLayoutProps> = ({ children, theme }) => (
  <div 
    className="min-h-screen"
    data-theme={theme}
    style={{
      '--font-size': 'var(--reader-font-size, 1.125rem)',
      '--line-height': 'var(--reader-line-height, 1.8)',
    } as React.CSSProperties}
  >
    <ReaderHeader />
    <main className="reader-content">
      {children}
    </main>
    <ReaderFooter />
  </div>
);
```

---

## 8. Error Handling Architecture

### 8.1 Error Boundary Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ERROR BOUNDARY HIERARCHY                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ ROOT ERROR BOUNDARY                                                     ││
│  │ • Catches fatal application errors                                      ││
│  │ • Shows full-page error with recovery options                           ││
│  │ • Reports to error tracking (Sentry)                                    ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                       │                                      │
│           ┌───────────────────────────┼───────────────────────────┐         │
│           │                           │                           │         │
│           ▼                           ▼                           ▼         │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐ │
│  │  LAYOUT ERROR   │        │   PAGE ERROR    │        │ FEATURE ERROR   │ │
│  │    BOUNDARY     │        │    BOUNDARY     │        │   BOUNDARY      │ │
│  │                 │        │                 │        │                 │ │
│  │ Layout-level    │        │ Page-level      │        │ Feature-level   │ │
│  │ fallback        │        │ fallback        │        │ fallback        │ │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Error Types

| Error Type | Handling | User Experience |
|------------|----------|-----------------|
| **Network Error** | Retry with backoff | Toast + retry button |
| **Auth Error (401)** | Refresh token | Silent refresh or redirect |
| **Forbidden (403)** | Show message | Access denied page |
| **Not Found (404)** | Show fallback | 404 page |
| **Validation Error (422)** | Show inline | Form field errors |
| **Server Error (500)** | Report + fallback | Error page with retry |
| **Client Error** | Report + recover | Error boundary fallback |

---

## 9. Performance Architecture

### 9.1 Code Splitting Strategy

```typescript
// Route-based code splitting
const HomePage = lazy(() => import('@/pages/home/HomePage'));
const StoryDetailPage = lazy(() => import('@/pages/story/StoryDetailPage'));
const ReaderPage = lazy(() => import('@/pages/reader/ReaderPage'));
const AuthorDashboard = lazy(() => import('@/pages/author/DashboardPage'));
const AdminDashboard = lazy(() => import('@/pages/admin/DashboardPage'));

// Component-based code splitting
const RichTextEditor = lazy(() => import('@/components/forms/RichTextEditor'));
const Charts = lazy(() => import('@/features/analytics/components/Charts'));
```

### 9.2 Caching Strategy

| Layer | Strategy | TTL | Invalidation |
|-------|----------|-----|--------------|
| **Browser Cache** | Static assets | 1 year | Hash-based |
| **Service Worker** | App shell, fonts | Runtime | Version-based |
| **React Query** | API responses | 5 min | Mutation/Manual |
| **Local Storage** | User preferences | Persistent | Manual |
| **Memory** | Component state | Session | Navigation |

### 9.3 Rendering Optimization

```typescript
// Virtualization for long lists
import { useVirtualizer } from '@tanstack/react-virtual';

const ChapterList: FC<{ chapters: Chapter[] }> = ({ chapters }) => {
  const parentRef = useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: chapters.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
  });
  
  return (
    <div ref={parentRef} className="h-[600px] overflow-auto">
      <div style={{ height: `${virtualizer.getTotalSize()}px` }}>
        {virtualizer.getVirtualItems().map(virtualItem => (
          <ChapterItem 
            key={virtualItem.key}
            chapter={chapters[virtualItem.index]}
            style={{
              height: `${virtualItem.size}px`,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          />
        ))}
      </div>
    </div>
  );
};
```

---

## 10. Security Architecture

### 10.1 Security Layers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SECURITY ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ TRANSPORT SECURITY                                                      ││
│  │ • HTTPS only (HSTS)                                                     ││
│  │ • Certificate pinning (mobile)                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                       │                                      │
│                                       ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ CONTENT SECURITY                                                        ││
│  │ • Content Security Policy (CSP)                                         ││
│  │ • XSS prevention (sanitization)                                         ││
│  │ • CSRF tokens                                                           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                       │                                      │
│                                       ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ AUTHENTICATION                                                          ││
│  │ • JWT access tokens (short-lived)                                       ││
│  │ • Refresh tokens (HTTP-only cookie)                                     ││
│  │ • Secure token storage                                                  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                       │                                      │
│                                       ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ AUTHORIZATION                                                           ││
│  │ • Role-based access control (RBAC)                                      ││
│  │ • Route guards                                                          ││
│  │ • Component-level guards                                                ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Token Management

```typescript
// Secure token storage strategy
const tokenStorage = {
  // Access token in memory only
  accessToken: null as string | null,
  
  setAccessToken(token: string) {
    this.accessToken = token;
  },
  
  getAccessToken() {
    return this.accessToken;
  },
  
  clearAccessToken() {
    this.accessToken = null;
  },
};

// Refresh token handled via HTTP-only cookie (set by backend)
// No client-side access to refresh token
```

---

## 11. Build & Bundle Architecture

### 11.1 Build Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react-swc';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          query: ['@tanstack/react-query'],
          ui: ['framer-motion', '@radix-ui/react-*'],
        },
      },
    },
    sourcemap: true,
    target: 'es2020',
  },
});
```

### 11.2 Bundle Analysis

| Chunk | Target Size | Contents |
|-------|-------------|----------|
| **main** | < 50KB | App shell, routing |
| **vendor** | < 80KB | React, Router |
| **query** | < 30KB | TanStack Query |
| **ui** | < 40KB | UI components |
| **page-**** | < 30KB each | Page components |

---

## 12. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Frontend Architecture Team | Initial release |

---

## 13. Related Documents

- [01_SYSTEM_OVERVIEW.md](./01_SYSTEM_OVERVIEW.md) - System overview
- [03_DESIGN_SYSTEM.md](./03_DESIGN_SYSTEM.md) - Design system specification
- [04_STATE_MANAGEMENT.md](./04_STATE_MANAGEMENT.md) - State management patterns
- [Frontend Design - Architecture](../FrontendDesign/01_FRONTEND_ARCHITECTURE.md)
