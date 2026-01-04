# Frontend Architecture

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2025-12-31 |
| Status | Approved |
| Owner | Frontend Engineering Team |
| Review Cycle | Quarterly |

---

## 2. Overview

### 2.1 Purpose
This document defines the **frontend architecture** for the N9 platform, covering system structure, technology choices, code organization, and integration patterns with backend services.

### 2.2 Architecture Goals

| Goal | Description |
|------|-------------|
| **Performance** | Sub-2s initial load, instant navigation |
| **Scalability** | Support 100K+ concurrent users |
| **Maintainability** | Clear boundaries, testable code |
| **Accessibility** | WCAG 2.1 AA compliance |
| **Developer Experience** | Fast builds, hot reload, type safety |

### 2.3 Platform Targets

| Platform | Technology | Priority |
|----------|------------|----------|
| Web (Desktop) | React SPA | P0 |
| Web (Mobile) | Responsive React | P0 |
| iOS | React Native | P1 |
| Android | React Native | P1 |
| Admin Portal | React SPA | P0 |

---

## 3. System Architecture

### 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENT APPLICATIONS                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────────┐ │
│  │   Web App      │  │  Mobile App    │  │      Admin Portal          │ │
│  │   (React)      │  │ (React Native) │  │        (React)             │ │
│  │                │  │                │  │                            │ │
│  │  • Reader UI   │  │  • Reader UI   │  │  • Moderation Dashboard    │ │
│  │  • Author UI   │  │  • Reading     │  │  • User Management         │ │
│  │  • Discovery   │  │  • Library     │  │  • Content Management      │ │
│  │  • Payments    │  │  • Payments    │  │  • Analytics               │ │
│  └───────┬────────┘  └───────┬────────┘  └────────────┬───────────────┘ │
│          │                   │                        │                  │
└──────────┼───────────────────┼────────────────────────┼──────────────────┘
           │                   │                        │
           ▼                   ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY LAYER                               │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │  REST API   │  │  WebSocket  │  │     CDN     │  │  Auth Service  │  │
│  │  /api/v1/*  │  │   /ws/*     │  │   Assets    │  │    JWT/OAuth   │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Application Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                        │
│  Pages • Layouts • Components • Styles                          │
├─────────────────────────────────────────────────────────────────┤
│                        APPLICATION LAYER                         │
│  Hooks • State Management • Business Logic • Form Handling       │
├─────────────────────────────────────────────────────────────────┤
│                          DOMAIN LAYER                            │
│  Types • Interfaces • Validation Schemas • Domain Models         │
├─────────────────────────────────────────────────────────────────┤
│                       INFRASTRUCTURE LAYER                       │
│  API Client • WebSocket • Storage • Analytics • Error Tracking   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Technology Stack

### 4.1 Core Technologies

| Category | Technology | Version | Purpose |
|----------|------------|---------|---------|
| **Runtime** | Node.js | 20 LTS | Development environment |
| **Framework** | React | 18.x | UI library |
| **Language** | TypeScript | 5.x | Type safety |
| **Build** | Vite | 5.x | Fast development & builds |
| **Package Manager** | pnpm | 8.x | Efficient dependency management |

### 4.2 State & Data

| Category | Technology | Purpose |
|----------|------------|---------|
| **Server State** | TanStack Query (React Query) | API caching, sync |
| **Client State** | Zustand | Global UI state |
| **Form State** | React Hook Form | Form management |
| **Validation** | Zod | Schema validation |
| **Persistence** | localForage | Offline storage |

### 4.3 UI & Styling

| Category | Technology | Purpose |
|----------|------------|---------|
| **CSS Framework** | Tailwind CSS | Utility-first styling |
| **Component Library** | shadcn/ui | Accessible components |
| **Icons** | Lucide React | Icon system |
| **Animations** | Framer Motion | Micro-interactions |
| **Charts** | Recharts | Data visualization |

### 4.4 Routing & Navigation

| Category | Technology | Purpose |
|----------|------------|---------|
| **Routing** | React Router 6 | Client-side routing |
| **Navigation** | Custom | Tab/stack navigation |
| **Deep Linking** | React Router | URL state management |

### 4.5 Testing & Quality

| Category | Technology | Purpose |
|----------|------------|---------|
| **Unit Testing** | Vitest | Fast unit tests |
| **Component Testing** | Testing Library | DOM testing |
| **E2E Testing** | Playwright | End-to-end tests |
| **Linting** | ESLint | Code quality |
| **Formatting** | Prettier | Code formatting |
| **Type Checking** | TypeScript | Static analysis |

### 4.6 DevOps & Monitoring

| Category | Technology | Purpose |
|----------|------------|---------|
| **CI/CD** | GitHub Actions | Automated pipelines |
| **Hosting** | AWS CloudFront + S3 | Static hosting |
| **Error Tracking** | Sentry | Error monitoring |
| **Analytics** | Mixpanel | User analytics |
| **Performance** | Web Vitals | Core metrics |

---

## 5. Project Structure

### 5.1 Directory Organization

```
src/
├── app/                      # Application setup
│   ├── App.tsx              # Root component
│   ├── providers.tsx        # Provider composition
│   └── router.tsx           # Route configuration
│
├── pages/                    # Page components (route-level)
│   ├── auth/                # Authentication pages
│   │   ├── LoginPage.tsx
│   │   ├── RegisterPage.tsx
│   │   └── ForgotPasswordPage.tsx
│   ├── home/                # Home & discovery
│   │   ├── HomePage.tsx
│   │   └── BrowsePage.tsx
│   ├── story/               # Story-related pages
│   │   ├── StoryDetailPage.tsx
│   │   └── StoryListPage.tsx
│   ├── reader/              # Reading experience
│   │   ├── ChapterReaderPage.tsx
│   │   └── LibraryPage.tsx
│   ├── author/              # Author dashboard
│   │   ├── AuthorDashboardPage.tsx
│   │   ├── StoryEditorPage.tsx
│   │   └── ChapterEditorPage.tsx
│   ├── profile/             # User profile
│   │   ├── ProfilePage.tsx
│   │   └── SettingsPage.tsx
│   ├── payment/             # Payments
│   │   ├── WalletPage.tsx
│   │   └── TopUpPage.tsx
│   └── admin/               # Admin portal
│       ├── DashboardPage.tsx
│       ├── ModerationPage.tsx
│       └── UsersPage.tsx
│
├── features/                 # Feature modules
│   ├── auth/                # Authentication feature
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── stores/
│   │   ├── api/
│   │   └── types.ts
│   ├── stories/             # Stories feature
│   ├── chapters/            # Chapters feature
│   ├── reader/              # Reader feature
│   ├── interactions/        # Social features
│   ├── payments/            # Payment feature
│   ├── notifications/       # Notifications feature
│   ├── search/              # Search feature
│   └── admin/               # Admin feature
│
├── components/               # Shared components
│   ├── ui/                  # Base UI components (shadcn)
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── modal.tsx
│   │   └── ...
│   ├── layout/              # Layout components
│   │   ├── Header.tsx
│   │   ├── Footer.tsx
│   │   ├── Sidebar.tsx
│   │   └── PageLayout.tsx
│   ├── common/              # Common components
│   │   ├── LoadingSpinner.tsx
│   │   ├── ErrorBoundary.tsx
│   │   ├── Avatar.tsx
│   │   └── Pagination.tsx
│   └── forms/               # Form components
│       ├── FormField.tsx
│       ├── FormError.tsx
│       └── FormSubmit.tsx
│
├── hooks/                    # Shared hooks
│   ├── useMediaQuery.ts
│   ├── useDebounce.ts
│   ├── useInfiniteScroll.ts
│   └── useLocalStorage.ts
│
├── lib/                      # Core utilities
│   ├── api/                 # API client
│   │   ├── client.ts
│   │   ├── interceptors.ts
│   │   └── endpoints.ts
│   ├── websocket/           # WebSocket client
│   │   ├── client.ts
│   │   └── events.ts
│   ├── storage/             # Storage utilities
│   │   ├── localStorage.ts
│   │   └── sessionStorage.ts
│   └── utils/               # Utility functions
│       ├── format.ts
│       ├── validation.ts
│       └── cn.ts
│
├── stores/                   # Global stores
│   ├── authStore.ts
│   ├── themeStore.ts
│   └── notificationStore.ts
│
├── types/                    # TypeScript types
│   ├── api.ts
│   ├── models.ts
│   └── common.ts
│
├── styles/                   # Global styles
│   ├── globals.css
│   └── themes.css
│
├── assets/                   # Static assets
│   ├── images/
│   ├── fonts/
│   └── icons/
│
└── config/                   # Configuration
    ├── constants.ts
    ├── env.ts
    └── routes.ts
```

### 5.2 Feature Module Structure

```
features/stories/
├── components/               # Feature-specific components
│   ├── StoryCard.tsx
│   ├── StoryGrid.tsx
│   ├── StoryFilters.tsx
│   └── ChapterList.tsx
│
├── hooks/                    # Feature hooks
│   ├── useStoryQuery.ts
│   ├── useStoryMutation.ts
│   └── useStoryFilters.ts
│
├── api/                      # API integration
│   ├── storyApi.ts
│   └── storyQueries.ts
│
├── stores/                   # Feature stores (if needed)
│   └── storyFilterStore.ts
│
├── types.ts                  # Feature types
└── index.ts                  # Public exports
```

---

## 6. API Integration

### 6.1 API Client Architecture

```typescript
// lib/api/client.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import { getAccessToken, refreshToken } from '@/features/auth';

class ApiClient {
  private client: AxiosInstance;
  private refreshPromise: Promise<string> | null = null;

  constructor() {
    this.client = axios.create({
      baseURL: import.meta.env.VITE_API_URL,
      timeout: 30000,
      headers: {
        'Content-Type': 'application/json',
      },
    });

    this.setupInterceptors();
  }

  private setupInterceptors() {
    // Request interceptor - add auth token
    this.client.interceptors.request.use((config) => {
      const token = getAccessToken();
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });

    // Response interceptor - handle 401
    this.client.interceptors.response.use(
      (response) => response,
      async (error) => {
        if (error.response?.status === 401) {
          return this.handle401(error);
        }
        return Promise.reject(error);
      }
    );
  }

  private async handle401(error: any) {
    // Single refresh attempt with deduplication
    if (!this.refreshPromise) {
      this.refreshPromise = refreshToken();
    }
    
    try {
      const newToken = await this.refreshPromise;
      error.config.headers.Authorization = `Bearer ${newToken}`;
      return this.client.request(error.config);
    } finally {
      this.refreshPromise = null;
    }
  }

  async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.get<T>(url, config);
    return response.data;
  }

  async post<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.post<T>(url, data, config);
    return response.data;
  }

  async put<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.put<T>(url, data, config);
    return response.data;
  }

  async delete<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.delete<T>(url, config);
    return response.data;
  }
}

export const apiClient = new ApiClient();
```

### 6.2 React Query Integration

```typescript
// features/stories/api/storyQueries.ts
import { useQuery, useMutation, useInfiniteQuery } from '@tanstack/react-query';
import { apiClient } from '@/lib/api/client';
import { Story, StoryListParams, CreateStoryRequest } from '../types';

// Query Keys Factory
export const storyKeys = {
  all: ['stories'] as const,
  lists: () => [...storyKeys.all, 'list'] as const,
  list: (params: StoryListParams) => [...storyKeys.lists(), params] as const,
  details: () => [...storyKeys.all, 'detail'] as const,
  detail: (id: string) => [...storyKeys.details(), id] as const,
};

// Queries
export function useStoriesQuery(params: StoryListParams) {
  return useInfiniteQuery({
    queryKey: storyKeys.list(params),
    queryFn: ({ pageParam = 1 }) => 
      apiClient.get<PageResponse<Story>>('/stories', {
        params: { ...params, page: pageParam },
      }),
    getNextPageParam: (lastPage) => 
      lastPage.meta.pagination.hasNext 
        ? lastPage.meta.pagination.page + 1 
        : undefined,
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}

export function useStoryQuery(id: string) {
  return useQuery({
    queryKey: storyKeys.detail(id),
    queryFn: () => apiClient.get<Story>(`/stories/${id}`),
    staleTime: 5 * 60 * 1000,
    enabled: !!id,
  });
}

// Mutations
export function useCreateStoryMutation() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (data: CreateStoryRequest) => 
      apiClient.post<Story>('/stories', data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: storyKeys.lists() });
    },
  });
}
```

### 6.3 WebSocket Integration

```typescript
// lib/websocket/client.ts
import { io, Socket } from 'socket.io-client';
import { getAccessToken } from '@/features/auth';

class WebSocketClient {
  private socket: Socket | null = null;
  private listeners: Map<string, Set<Function>> = new Map();

  connect() {
    if (this.socket?.connected) return;

    this.socket = io(import.meta.env.VITE_WS_URL, {
      auth: { token: getAccessToken() },
      transports: ['websocket'],
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 1000,
    });

    this.socket.on('connect', () => {
      console.log('WebSocket connected');
    });

    this.socket.on('disconnect', (reason) => {
      console.log('WebSocket disconnected:', reason);
    });

    // Route events to listeners
    this.socket.onAny((event, data) => {
      const handlers = this.listeners.get(event);
      handlers?.forEach((handler) => handler(data));
    });
  }

  subscribe<T>(event: string, handler: (data: T) => void) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);

    return () => {
      this.listeners.get(event)?.delete(handler);
    };
  }

  emit(event: string, data: any) {
    this.socket?.emit(event, data);
  }

  disconnect() {
    this.socket?.disconnect();
    this.socket = null;
  }
}

export const wsClient = new WebSocketClient();
```

---

## 7. State Management

### 7.1 State Categories

```
┌─────────────────────────────────────────────────────────────────┐
│                       STATE MANAGEMENT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐ │
│  │  Server State  │  │  Client State  │  │    Form State      │ │
│  │                │  │                │  │                    │ │
│  │  React Query   │  │    Zustand     │  │  React Hook Form   │ │
│  │                │  │                │  │                    │ │
│  │  • API data    │  │  • UI state    │  │  • Field values    │ │
│  │  • Caching     │  │  • Theme       │  │  • Validation      │ │
│  │  • Sync        │  │  • Auth        │  │  • Errors          │ │
│  │  • Mutations   │  │  • Toasts      │  │  • Submission      │ │
│  └────────────────┘  └────────────────┘  └────────────────────┘ │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐                        │
│  │   URL State    │  │  Local State   │                        │
│  │                │  │                │                        │
│  │  React Router  │  │    useState    │                        │
│  │                │  │                │                        │
│  │  • Filters     │  │  • Component   │                        │
│  │  • Pagination  │  │    specific    │                        │
│  │  • Search      │  │  • Ephemeral   │                        │
│  └────────────────┘  └────────────────┘                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Zustand Store Pattern

```typescript
// stores/authStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { User } from '@/types/models';

interface AuthState {
  user: User | null;
  accessToken: string | null;
  isAuthenticated: boolean;
  
  // Actions
  setAuth: (user: User, token: string) => void;
  clearAuth: () => void;
  updateUser: (updates: Partial<User>) => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      accessToken: null,
      isAuthenticated: false,

      setAuth: (user, accessToken) => 
        set({ user, accessToken, isAuthenticated: true }),
      
      clearAuth: () => 
        set({ user: null, accessToken: null, isAuthenticated: false }),
      
      updateUser: (updates) =>
        set((state) => ({
          user: state.user ? { ...state.user, ...updates } : null,
        })),
    }),
    {
      name: 'n9-auth',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({
        accessToken: state.accessToken,
      }),
    }
  )
);

// Selectors
export const selectUser = (state: AuthState) => state.user;
export const selectIsAuthenticated = (state: AuthState) => state.isAuthenticated;
```

---

## 8. Routing Architecture

### 8.1 Route Configuration

```typescript
// app/router.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { lazy, Suspense } from 'react';
import { MainLayout, AuthLayout, AdminLayout } from '@/components/layout';
import { AuthGuard, RoleGuard } from '@/features/auth';
import { LoadingPage } from '@/components/common';

// Lazy loaded pages
const HomePage = lazy(() => import('@/pages/home/HomePage'));
const StoryDetailPage = lazy(() => import('@/pages/story/StoryDetailPage'));
const ChapterReaderPage = lazy(() => import('@/pages/reader/ChapterReaderPage'));
const LoginPage = lazy(() => import('@/pages/auth/LoginPage'));
const AuthorDashboardPage = lazy(() => import('@/pages/author/AuthorDashboardPage'));
const AdminDashboardPage = lazy(() => import('@/pages/admin/DashboardPage'));

const router = createBrowserRouter([
  // Public routes
  {
    path: '/',
    element: <MainLayout />,
    children: [
      { index: true, element: <HomePage /> },
      { path: 'browse', element: <BrowsePage /> },
      { path: 'stories/:storyId', element: <StoryDetailPage /> },
      { path: 'search', element: <SearchPage /> },
      { path: 'author/:authorId', element: <AuthorProfilePage /> },
    ],
  },
  
  // Auth routes (no layout header/footer)
  {
    path: '/auth',
    element: <AuthLayout />,
    children: [
      { path: 'login', element: <LoginPage /> },
      { path: 'register', element: <RegisterPage /> },
      { path: 'forgot-password', element: <ForgotPasswordPage /> },
      { path: 'reset-password', element: <ResetPasswordPage /> },
    ],
  },
  
  // Reader routes (minimal UI)
  {
    path: '/read',
    element: <ReaderLayout />,
    children: [
      { path: ':storyId/:chapterId', element: <ChapterReaderPage /> },
    ],
  },
  
  // Protected routes (authenticated users)
  {
    path: '/',
    element: <AuthGuard><MainLayout /></AuthGuard>,
    children: [
      { path: 'library', element: <LibraryPage /> },
      { path: 'wallet', element: <WalletPage /> },
      { path: 'notifications', element: <NotificationsPage /> },
      { path: 'settings', element: <SettingsPage /> },
      { path: 'profile', element: <ProfilePage /> },
    ],
  },
  
  // Author routes (author role required)
  {
    path: '/author',
    element: <RoleGuard roles={['AUTHOR', 'ADMIN']}><AuthorLayout /></RoleGuard>,
    children: [
      { index: true, element: <AuthorDashboardPage /> },
      { path: 'stories', element: <AuthorStoriesPage /> },
      { path: 'stories/new', element: <StoryEditorPage /> },
      { path: 'stories/:storyId', element: <StoryEditorPage /> },
      { path: 'stories/:storyId/chapters/new', element: <ChapterEditorPage /> },
      { path: 'stories/:storyId/chapters/:chapterId', element: <ChapterEditorPage /> },
      { path: 'earnings', element: <EarningsPage /> },
      { path: 'analytics', element: <AnalyticsPage /> },
    ],
  },
  
  // Admin routes (admin role required)
  {
    path: '/admin',
    element: <RoleGuard roles={['MODERATOR', 'ADMIN']}><AdminLayout /></RoleGuard>,
    children: [
      { index: true, element: <AdminDashboardPage /> },
      { path: 'users', element: <UsersManagementPage /> },
      { path: 'content', element: <ContentModerationPage /> },
      { path: 'reports', element: <ReportsPage /> },
      { path: 'payouts', element: <PayoutsPage /> },
      { path: 'settings', element: <SystemSettingsPage /> },
    ],
  },
  
  // Error routes
  { path: '/404', element: <NotFoundPage /> },
  { path: '*', element: <NotFoundPage /> },
]);

export function AppRouter() {
  return (
    <Suspense fallback={<LoadingPage />}>
      <RouterProvider router={router} />
    </Suspense>
  );
}
```

### 8.2 Route Guards

```typescript
// features/auth/components/AuthGuard.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuthStore, selectIsAuthenticated } from '@/stores/authStore';

interface AuthGuardProps {
  children: React.ReactNode;
}

export function AuthGuard({ children }: AuthGuardProps) {
  const isAuthenticated = useAuthStore(selectIsAuthenticated);
  const location = useLocation();

  if (!isAuthenticated) {
    return <Navigate to="/auth/login" state={{ from: location }} replace />;
  }

  return <>{children}</>;
}

// features/auth/components/RoleGuard.tsx
interface RoleGuardProps {
  children: React.ReactNode;
  roles: string[];
}

export function RoleGuard({ children, roles }: RoleGuardProps) {
  const user = useAuthStore(selectUser);
  
  if (!user || !roles.some((role) => user.roles.includes(role))) {
    return <Navigate to="/403" replace />;
  }

  return <>{children}</>;
}
```

---

## 9. Performance Strategy

### 9.1 Bundle Optimization

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true, gzipSize: true }),
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Vendor chunks
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'query-vendor': ['@tanstack/react-query'],
          'ui-vendor': ['framer-motion', '@radix-ui/react-dialog'],
          
          // Feature chunks
          'auth': ['./src/features/auth/index.ts'],
          'editor': ['./src/features/editor/index.ts'],
          'admin': ['./src/features/admin/index.ts'],
        },
      },
    },
    chunkSizeWarningLimit: 500,
  },
});
```

### 9.2 Image Optimization

```typescript
// components/common/OptimizedImage.tsx
interface OptimizedImageProps {
  src: string;
  alt: string;
  width: number;
  height: number;
  priority?: boolean;
}

export function OptimizedImage({
  src,
  alt,
  width,
  height,
  priority = false,
}: OptimizedImageProps) {
  const cdnUrl = import.meta.env.VITE_CDN_URL;
  
  // Generate responsive srcset
  const sizes = [0.5, 1, 1.5, 2];
  const srcSet = sizes
    .map((scale) => {
      const w = Math.round(width * scale);
      return `${cdnUrl}/${src}?w=${w}&format=webp ${w}w`;
    })
    .join(', ');

  return (
    <picture>
      <source srcSet={srcSet} type="image/webp" />
      <img
        src={`${cdnUrl}/${src}?w=${width}`}
        alt={alt}
        width={width}
        height={height}
        loading={priority ? 'eager' : 'lazy'}
        decoding="async"
      />
    </picture>
  );
}
```

### 9.3 Virtualization for Long Lists

```typescript
// components/common/VirtualizedList.tsx
import { useVirtualizer } from '@tanstack/react-virtual';

interface VirtualizedListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  estimateSize: number;
  overscan?: number;
}

export function VirtualizedList<T>({
  items,
  renderItem,
  estimateSize,
  overscan = 5,
}: VirtualizedListProps<T>) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => estimateSize,
    overscan,
  });

  return (
    <div ref={parentRef} className="h-full overflow-auto">
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {renderItem(items[virtualRow.index], virtualRow.index)}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 10. Error Handling

### 10.1 Error Boundary

```typescript
// components/common/ErrorBoundary.tsx
import { Component, ErrorInfo, ReactNode } from 'react';
import * as Sentry from '@sentry/react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    Sentry.captureException(error, { extra: errorInfo });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <ErrorFallback error={this.state.error} />;
    }

    return this.props.children;
  }
}

function ErrorFallback({ error }: { error: Error | null }) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h1 className="text-2xl font-bold text-red-600">Something went wrong</h1>
      <p className="text-gray-600 mt-2">{error?.message}</p>
      <button
        onClick={() => window.location.reload()}
        className="mt-4 px-4 py-2 bg-primary text-white rounded"
      >
        Reload Page
      </button>
    </div>
  );
}
```

### 10.2 API Error Handling

```typescript
// lib/api/errors.ts
export class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    message: string,
    public details?: Record<string, string[]>
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

// Error handler hook
export function useApiErrorHandler() {
  const { toast } = useToast();
  const navigate = useNavigate();

  const handleError = useCallback((error: unknown) => {
    if (error instanceof ApiError) {
      switch (error.status) {
        case 401:
          navigate('/auth/login');
          break;
        case 403:
          toast({ title: 'Access Denied', variant: 'destructive' });
          break;
        case 404:
          navigate('/404');
          break;
        case 422:
          // Validation errors - usually handled by forms
          break;
        default:
          toast({ 
            title: 'Error', 
            description: error.message,
            variant: 'destructive',
          });
      }
    } else {
      toast({ 
        title: 'Unexpected Error', 
        description: 'Please try again later',
        variant: 'destructive',
      });
    }
  }, [navigate, toast]);

  return { handleError };
}
```

---

## 11. Testing Strategy

### 11.1 Test Organization

```
src/
├── features/stories/
│   ├── components/
│   │   ├── StoryCard.tsx
│   │   └── __tests__/
│   │       └── StoryCard.test.tsx
│   ├── hooks/
│   │   ├── useStoryQuery.ts
│   │   └── __tests__/
│   │       └── useStoryQuery.test.ts
│
├── __tests__/                # Integration tests
│   ├── integration/
│   │   ├── auth.test.tsx
│   │   └── story-flow.test.tsx
│   └── e2e/                  # E2E test specs
│       ├── auth.spec.ts
│       └── reading.spec.ts
```

### 11.2 Testing Utilities

```typescript
// test/utils.tsx
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';

const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });

interface WrapperProps {
  children: React.ReactNode;
}

function AllProviders({ children }: WrapperProps) {
  const queryClient = createTestQueryClient();
  
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        {children}
      </BrowserRouter>
    </QueryClientProvider>
  );
}

export function renderWithProviders(
  ui: React.ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) {
  return render(ui, { wrapper: AllProviders, ...options });
}

export * from '@testing-library/react';
```

---

## 12. References

### 12.1 Related Documents

| Document | Purpose |
|----------|---------|
| [00_FRONTEND_DOCUMENTATION_STANDARDS.md](00_FRONTEND_DOCUMENTATION_STANDARDS.md) | Documentation guidelines |
| [02_DESIGN_SYSTEM_GUIDELINES.md](02_DESIGN_SYSTEM_GUIDELINES.md) | UI/UX standards |
| [03_STATE_MANAGEMENT_ROUTING.md](03_STATE_MANAGEMENT_ROUTING.md) | State patterns |

### 12.2 Backend References

| Document | Purpose |
|----------|---------|
| [08_API_STANDARDS.md](../Specification/08_API_STANDARDS.md) | API conventions |
| [13_API_CATALOG.md](../Specification/13_API_CATALOG.md) | Endpoint reference |
