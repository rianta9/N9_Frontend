# State Management & Routing

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
This document defines **state management patterns** and **routing architecture** for the N9 platform, ensuring consistent data flow, optimal caching, and seamless navigation across the application.

### 2.2 State Management Philosophy

```
┌─────────────────────────────────────────────────────────────────┐
│                    STATE MANAGEMENT LAYERS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Server State (React Query)                                     │
│   ├── Remote data from APIs                                      │
│   ├── Automatic caching & invalidation                           │
│   ├── Background refetching                                      │
│   └── Optimistic updates                                         │
│                                                                  │
│   Client State (Zustand)                                         │
│   ├── UI state (modals, sidebars)                               │
│   ├── User preferences (theme, settings)                        │
│   ├── Authentication state                                       │
│   └── Notification queue                                         │
│                                                                  │
│   URL State (React Router)                                       │
│   ├── Page navigation                                            │
│   ├── Filter/sort parameters                                     │
│   ├── Search queries                                             │
│   └── Pagination state                                           │
│                                                                  │
│   Form State (React Hook Form)                                   │
│   ├── Field values & validation                                  │
│   ├── Submission state                                           │
│   └── Error handling                                             │
│                                                                  │
│   Local State (useState/useReducer)                              │
│   ├── Component-specific state                                   │
│   └── Ephemeral UI interactions                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Server State (React Query)

### 3.1 Configuration

```typescript
// lib/query/client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,        // 5 minutes
      gcTime: 30 * 60 * 1000,          // 30 minutes (formerly cacheTime)
      retry: 3,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
    },
    mutations: {
      retry: 1,
      onError: (error) => {
        // Global error handling
        console.error('Mutation error:', error);
      },
    },
  },
});
```

### 3.2 Query Key Factory Pattern

```typescript
// lib/query/keys.ts

// Stories
export const storyKeys = {
  all: ['stories'] as const,
  lists: () => [...storyKeys.all, 'list'] as const,
  list: (filters: StoryFilters) => [...storyKeys.lists(), filters] as const,
  details: () => [...storyKeys.all, 'detail'] as const,
  detail: (id: string) => [...storyKeys.details(), id] as const,
  chapters: (storyId: string) => [...storyKeys.detail(storyId), 'chapters'] as const,
  reviews: (storyId: string) => [...storyKeys.detail(storyId), 'reviews'] as const,
};

// Users
export const userKeys = {
  all: ['users'] as const,
  current: () => [...userKeys.all, 'current'] as const,
  profile: (id: string) => [...userKeys.all, 'profile', id] as const,
  followers: (id: string) => [...userKeys.profile(id), 'followers'] as const,
  following: (id: string) => [...userKeys.profile(id), 'following'] as const,
};

// Chapters
export const chapterKeys = {
  all: ['chapters'] as const,
  detail: (id: string) => [...chapterKeys.all, id] as const,
  content: (id: string) => [...chapterKeys.detail(id), 'content'] as const,
  comments: (id: string) => [...chapterKeys.detail(id), 'comments'] as const,
};

// Notifications
export const notificationKeys = {
  all: ['notifications'] as const,
  list: (filters?: NotificationFilters) => [...notificationKeys.all, filters] as const,
  unreadCount: () => [...notificationKeys.all, 'unread-count'] as const,
};

// Wallet
export const walletKeys = {
  all: ['wallet'] as const,
  balance: () => [...walletKeys.all, 'balance'] as const,
  transactions: (filters?: TransactionFilters) => [...walletKeys.all, 'transactions', filters] as const,
};
```

### 3.3 Query Hook Patterns

```typescript
// features/stories/hooks/useStoryQuery.ts
import { useQuery, useInfiniteQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { storyKeys } from '@/lib/query/keys';
import { storyApi } from '../api/storyApi';

// Single story query
export function useStoryQuery(storyId: string) {
  return useQuery({
    queryKey: storyKeys.detail(storyId),
    queryFn: () => storyApi.getStory(storyId),
    enabled: !!storyId,
    staleTime: 5 * 60 * 1000,
  });
}

// Infinite story list
export function useStoriesInfiniteQuery(filters: StoryFilters) {
  return useInfiniteQuery({
    queryKey: storyKeys.list(filters),
    queryFn: ({ pageParam = 1 }) => 
      storyApi.getStories({ ...filters, page: pageParam }),
    getNextPageParam: (lastPage) => 
      lastPage.meta.pagination.hasNext 
        ? lastPage.meta.pagination.page + 1 
        : undefined,
    initialPageParam: 1,
  });
}

// Prefetch story
export function usePrefetchStory() {
  const queryClient = useQueryClient();
  
  return (storyId: string) => {
    queryClient.prefetchQuery({
      queryKey: storyKeys.detail(storyId),
      queryFn: () => storyApi.getStory(storyId),
      staleTime: 5 * 60 * 1000,
    });
  };
}
```

### 3.4 Mutation Patterns

```typescript
// features/stories/hooks/useStoryMutations.ts

// Create story with optimistic update
export function useCreateStoryMutation() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: storyApi.createStory,
    onMutate: async (newStory) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: storyKeys.lists() });
      
      // Snapshot previous value
      const previousStories = queryClient.getQueryData(storyKeys.lists());
      
      // Optimistically update
      queryClient.setQueryData(storyKeys.lists(), (old: any) => ({
        ...old,
        data: [{ ...newStory, id: 'temp-id', status: 'DRAFT' }, ...(old?.data || [])],
      }));
      
      return { previousStories };
    },
    onError: (err, newStory, context) => {
      // Rollback on error
      queryClient.setQueryData(storyKeys.lists(), context?.previousStories);
    },
    onSettled: () => {
      // Refetch after mutation
      queryClient.invalidateQueries({ queryKey: storyKeys.lists() });
    },
  });
}

// Update story
export function useUpdateStoryMutation() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: UpdateStoryRequest }) =>
      storyApi.updateStory(id, data),
    onSuccess: (updatedStory) => {
      // Update cache directly
      queryClient.setQueryData(
        storyKeys.detail(updatedStory.id),
        updatedStory
      );
      // Invalidate lists
      queryClient.invalidateQueries({ queryKey: storyKeys.lists() });
    },
  });
}

// Delete story
export function useDeleteStoryMutation() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: storyApi.deleteStory,
    onSuccess: (_, storyId) => {
      // Remove from cache
      queryClient.removeQueries({ queryKey: storyKeys.detail(storyId) });
      // Invalidate lists
      queryClient.invalidateQueries({ queryKey: storyKeys.lists() });
    },
  });
}
```

### 3.5 Caching Strategy

| Resource | Stale Time | GC Time | Refetch Strategy |
|----------|------------|---------|------------------|
| Story List | 5 min | 30 min | Window focus |
| Story Detail | 5 min | 30 min | Window focus |
| Chapter Content | 10 min | 1 hour | Manual |
| User Profile | 5 min | 30 min | Window focus |
| Notifications | 1 min | 10 min | Polling (30s) |
| Wallet Balance | 30 sec | 5 min | Window focus + polling |
| Search Results | 2 min | 10 min | None |

---

## 4. Client State (Zustand)

### 4.1 Store Organization

```typescript
// stores/index.ts
export { useAuthStore } from './authStore';
export { useThemeStore } from './themeStore';
export { useUIStore } from './uiStore';
export { useReadingStore } from './readingStore';
export { useNotificationStore } from './notificationStore';
```

### 4.2 Auth Store

```typescript
// stores/authStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface User {
  id: string;
  email: string;
  displayName: string;
  avatarUrl?: string;
  roles: string[];
}

interface AuthState {
  user: User | null;
  accessToken: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
}

interface AuthActions {
  login: (user: User, tokens: { accessToken: string; refreshToken: string }) => void;
  logout: () => void;
  updateUser: (updates: Partial<User>) => void;
  setTokens: (tokens: { accessToken: string; refreshToken: string }) => void;
  setLoading: (loading: boolean) => void;
}

export const useAuthStore = create<AuthState & AuthActions>()(
  persist(
    immer((set) => ({
      // State
      user: null,
      accessToken: null,
      refreshToken: null,
      isAuthenticated: false,
      isLoading: true,

      // Actions
      login: (user, tokens) =>
        set((state) => {
          state.user = user;
          state.accessToken = tokens.accessToken;
          state.refreshToken = tokens.refreshToken;
          state.isAuthenticated = true;
          state.isLoading = false;
        }),

      logout: () =>
        set((state) => {
          state.user = null;
          state.accessToken = null;
          state.refreshToken = null;
          state.isAuthenticated = false;
        }),

      updateUser: (updates) =>
        set((state) => {
          if (state.user) {
            Object.assign(state.user, updates);
          }
        }),

      setTokens: (tokens) =>
        set((state) => {
          state.accessToken = tokens.accessToken;
          state.refreshToken = tokens.refreshToken;
        }),

      setLoading: (loading) =>
        set((state) => {
          state.isLoading = loading;
        }),
    })),
    {
      name: 'n9-auth',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({
        accessToken: state.accessToken,
        refreshToken: state.refreshToken,
      }),
    }
  )
);

// Selectors
export const selectUser = (state: AuthState) => state.user;
export const selectIsAuthenticated = (state: AuthState) => state.isAuthenticated;
export const selectIsAuthor = (state: AuthState) => 
  state.user?.roles.includes('AUTHOR') ?? false;
export const selectIsAdmin = (state: AuthState) => 
  state.user?.roles.includes('ADMIN') ?? false;
```

### 4.3 Theme Store

```typescript
// stores/themeStore.ts
type Theme = 'light' | 'dark' | 'sepia' | 'system';

interface ThemeState {
  theme: Theme;
  resolvedTheme: 'light' | 'dark' | 'sepia';
}

interface ThemeActions {
  setTheme: (theme: Theme) => void;
}

export const useThemeStore = create<ThemeState & ThemeActions>()(
  persist(
    (set, get) => ({
      theme: 'system',
      resolvedTheme: 'light',

      setTheme: (theme) => {
        const resolved = theme === 'system' 
          ? getSystemTheme() 
          : theme;
        
        set({ theme, resolvedTheme: resolved });
        applyTheme(resolved);
      },
    }),
    {
      name: 'n9-theme',
      onRehydrateStorage: () => (state) => {
        if (state) {
          const resolved = state.theme === 'system' 
            ? getSystemTheme() 
            : state.theme;
          state.resolvedTheme = resolved;
          applyTheme(resolved);
        }
      },
    }
  )
);

function getSystemTheme(): 'light' | 'dark' {
  return window.matchMedia('(prefers-color-scheme: dark)').matches 
    ? 'dark' 
    : 'light';
}

function applyTheme(theme: string) {
  document.documentElement.setAttribute('data-theme', theme);
}
```

### 4.4 Reading Store

```typescript
// stores/readingStore.ts
interface ReadingPreferences {
  fontSize: 'sm' | 'md' | 'lg' | 'xl';
  fontFamily: 'serif' | 'sans' | 'mono';
  lineHeight: 'compact' | 'normal' | 'relaxed';
  theme: 'light' | 'dark' | 'sepia';
  autoScroll: boolean;
  autoScrollSpeed: number;
}

interface ReadingProgress {
  storyId: string;
  chapterId: string;
  scrollPosition: number;
  lastReadAt: string;
}

interface ReadingState {
  preferences: ReadingPreferences;
  currentProgress: ReadingProgress | null;
  recentlyRead: ReadingProgress[];
}

interface ReadingActions {
  setPreference: <K extends keyof ReadingPreferences>(
    key: K,
    value: ReadingPreferences[K]
  ) => void;
  updateProgress: (progress: Partial<ReadingProgress>) => void;
  clearProgress: () => void;
}

export const useReadingStore = create<ReadingState & ReadingActions>()(
  persist(
    immer((set) => ({
      preferences: {
        fontSize: 'md',
        fontFamily: 'serif',
        lineHeight: 'normal',
        theme: 'light',
        autoScroll: false,
        autoScrollSpeed: 50,
      },
      currentProgress: null,
      recentlyRead: [],

      setPreference: (key, value) =>
        set((state) => {
          state.preferences[key] = value;
        }),

      updateProgress: (progress) =>
        set((state) => {
          if (state.currentProgress) {
            Object.assign(state.currentProgress, progress);
          } else {
            state.currentProgress = progress as ReadingProgress;
          }
          // Update recently read
          const index = state.recentlyRead.findIndex(
            (r) => r.storyId === progress.storyId
          );
          if (index >= 0) {
            state.recentlyRead.splice(index, 1);
          }
          state.recentlyRead.unshift(state.currentProgress);
          state.recentlyRead = state.recentlyRead.slice(0, 20);
        }),

      clearProgress: () =>
        set((state) => {
          state.currentProgress = null;
        }),
    })),
    {
      name: 'n9-reading',
    }
  )
);
```

### 4.5 UI Store

```typescript
// stores/uiStore.ts
interface UIState {
  sidebarOpen: boolean;
  searchOpen: boolean;
  activeModal: string | null;
  modalData: Record<string, unknown>;
}

interface UIActions {
  toggleSidebar: () => void;
  setSidebarOpen: (open: boolean) => void;
  toggleSearch: () => void;
  openModal: (modalId: string, data?: Record<string, unknown>) => void;
  closeModal: () => void;
}

export const useUIStore = create<UIState & UIActions>((set) => ({
  sidebarOpen: false,
  searchOpen: false,
  activeModal: null,
  modalData: {},

  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
  setSidebarOpen: (open) => set({ sidebarOpen: open }),
  toggleSearch: () => set((state) => ({ searchOpen: !state.searchOpen })),
  openModal: (modalId, data = {}) => set({ activeModal: modalId, modalData: data }),
  closeModal: () => set({ activeModal: null, modalData: {} }),
}));
```

---

## 5. URL State (React Router)

### 5.1 Route Structure

```typescript
// config/routes.ts
export const routes = {
  // Public
  home: '/',
  browse: '/browse',
  search: '/search',
  storyDetail: '/stories/:storyId',
  authorProfile: '/author/:authorId',
  
  // Auth
  login: '/auth/login',
  register: '/auth/register',
  forgotPassword: '/auth/forgot-password',
  resetPassword: '/auth/reset-password',
  
  // Reader
  chapterReader: '/read/:storyId/:chapterId',
  
  // Protected - User
  library: '/library',
  notifications: '/notifications',
  wallet: '/wallet',
  settings: '/settings',
  profile: '/profile',
  
  // Protected - Author
  authorDashboard: '/author',
  authorStories: '/author/stories',
  storyEditor: '/author/stories/:storyId?',
  chapterEditor: '/author/stories/:storyId/chapters/:chapterId?',
  authorEarnings: '/author/earnings',
  
  // Protected - Admin
  adminDashboard: '/admin',
  adminUsers: '/admin/users',
  adminContent: '/admin/content',
  adminReports: '/admin/reports',
  adminPayouts: '/admin/payouts',
} as const;

// Route helpers
export function storyRoute(storyId: string) {
  return routes.storyDetail.replace(':storyId', storyId);
}

export function chapterRoute(storyId: string, chapterId: string) {
  return routes.chapterReader
    .replace(':storyId', storyId)
    .replace(':chapterId', chapterId);
}

export function authorRoute(authorId: string) {
  return routes.authorProfile.replace(':authorId', authorId);
}
```

### 5.2 URL State Hooks

```typescript
// hooks/useSearchParams.ts
import { useSearchParams as useRouterSearchParams } from 'react-router-dom';
import { useMemo, useCallback } from 'react';

export function useTypedSearchParams<T extends Record<string, string>>() {
  const [searchParams, setSearchParams] = useRouterSearchParams();

  const params = useMemo(() => {
    const result: Partial<T> = {};
    searchParams.forEach((value, key) => {
      (result as any)[key] = value;
    });
    return result;
  }, [searchParams]);

  const setParams = useCallback(
    (newParams: Partial<T>, options?: { replace?: boolean }) => {
      setSearchParams((prev) => {
        const updated = new URLSearchParams(prev);
        Object.entries(newParams).forEach(([key, value]) => {
          if (value === undefined || value === null || value === '') {
            updated.delete(key);
          } else {
            updated.set(key, String(value));
          }
        });
        return updated;
      }, options);
    },
    [setSearchParams]
  );

  return [params, setParams] as const;
}

// Usage example
interface StoryFilters {
  category?: string;
  status?: string;
  sort?: string;
  page?: string;
}

function BrowsePage() {
  const [filters, setFilters] = useTypedSearchParams<StoryFilters>();
  
  const handleCategoryChange = (category: string) => {
    setFilters({ ...filters, category, page: '1' });
  };
}
```

### 5.3 Pagination with URL State

```typescript
// hooks/usePagination.ts
import { useSearchParams } from 'react-router-dom';
import { useCallback, useMemo } from 'react';

interface UsePaginationOptions {
  defaultPage?: number;
  defaultSize?: number;
}

export function usePagination(options: UsePaginationOptions = {}) {
  const { defaultPage = 1, defaultSize = 20 } = options;
  const [searchParams, setSearchParams] = useSearchParams();

  const page = useMemo(() => {
    const pageParam = searchParams.get('page');
    return pageParam ? parseInt(pageParam, 10) : defaultPage;
  }, [searchParams, defaultPage]);

  const size = useMemo(() => {
    const sizeParam = searchParams.get('size');
    return sizeParam ? parseInt(sizeParam, 10) : defaultSize;
  }, [searchParams, defaultSize]);

  const setPage = useCallback(
    (newPage: number) => {
      setSearchParams((prev) => {
        const updated = new URLSearchParams(prev);
        if (newPage === defaultPage) {
          updated.delete('page');
        } else {
          updated.set('page', String(newPage));
        }
        return updated;
      });
    },
    [setSearchParams, defaultPage]
  );

  const setSize = useCallback(
    (newSize: number) => {
      setSearchParams((prev) => {
        const updated = new URLSearchParams(prev);
        updated.set('size', String(newSize));
        updated.delete('page'); // Reset to first page
        return updated;
      });
    },
    [setSearchParams]
  );

  return { page, size, setPage, setSize };
}
```

### 5.4 Route Guards

```typescript
// features/auth/components/AuthGuard.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuthStore } from '@/stores/authStore';
import { routes } from '@/config/routes';

interface AuthGuardProps {
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export function AuthGuard({ children, fallback }: AuthGuardProps) {
  const { isAuthenticated, isLoading } = useAuthStore();
  const location = useLocation();

  if (isLoading) {
    return fallback ?? <LoadingScreen />;
  }

  if (!isAuthenticated) {
    return <Navigate to={routes.login} state={{ from: location }} replace />;
  }

  return <>{children}</>;
}

// features/auth/components/RoleGuard.tsx
interface RoleGuardProps {
  children: React.ReactNode;
  roles: string[];
  fallback?: React.ReactNode;
}

export function RoleGuard({ children, roles, fallback }: RoleGuardProps) {
  const user = useAuthStore((state) => state.user);

  if (!user) {
    return <Navigate to={routes.login} replace />;
  }

  const hasRole = roles.some((role) => user.roles.includes(role));
  
  if (!hasRole) {
    return fallback ?? <Navigate to="/403" replace />;
  }

  return <>{children}</>;
}

// features/auth/components/GuestGuard.tsx
export function GuestGuard({ children }: { children: React.ReactNode }) {
  const { isAuthenticated, isLoading } = useAuthStore();
  const location = useLocation();

  if (isLoading) {
    return <LoadingScreen />;
  }

  if (isAuthenticated) {
    const from = (location.state as any)?.from?.pathname || routes.home;
    return <Navigate to={from} replace />;
  }

  return <>{children}</>;
}
```

---

## 6. Form State (React Hook Form)

### 6.1 Form Configuration

```typescript
// lib/forms/config.ts
import { zodResolver } from '@hookform/resolvers/zod';
import { UseFormProps } from 'react-hook-form';

export function createFormConfig<T extends z.ZodType>(
  schema: T,
  defaultValues?: Partial<z.infer<T>>
): UseFormProps<z.infer<T>> {
  return {
    resolver: zodResolver(schema),
    defaultValues: defaultValues as any,
    mode: 'onBlur',
    reValidateMode: 'onChange',
  };
}
```

### 6.2 Form Schemas

```typescript
// features/auth/schemas.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(1, 'Password is required'),
  rememberMe: z.boolean().optional(),
});

export const registerSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain an uppercase letter')
    .regex(/[a-z]/, 'Password must contain a lowercase letter')
    .regex(/[0-9]/, 'Password must contain a number'),
  confirmPassword: z.string(),
  displayName: z
    .string()
    .min(2, 'Display name must be at least 2 characters')
    .max(50, 'Display name must be at most 50 characters'),
  acceptTerms: z.literal(true, {
    errorMap: () => ({ message: 'You must accept the terms' }),
  }),
}).refine((data) => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],
});

// features/stories/schemas.ts
export const storySchema = z.object({
  title: z
    .string()
    .min(1, 'Title is required')
    .max(200, 'Title must be at most 200 characters'),
  description: z
    .string()
    .max(5000, 'Description must be at most 5000 characters')
    .optional(),
  categories: z
    .array(z.string())
    .min(1, 'Select at least one category')
    .max(3, 'Select at most 3 categories'),
  tags: z.array(z.string()).max(10, 'Maximum 10 tags allowed').optional(),
  contentRating: z.enum(['GENERAL', 'TEEN', 'MATURE']),
  status: z.enum(['DRAFT', 'ONGOING', 'COMPLETED', 'HIATUS']),
});

export const chapterSchema = z.object({
  title: z
    .string()
    .min(1, 'Title is required')
    .max(200, 'Title must be at most 200 characters'),
  content: z
    .string()
    .min(100, 'Chapter must be at least 100 characters')
    .max(100000, 'Chapter must be at most 100,000 characters'),
  authorNote: z
    .string()
    .max(2000, 'Author note must be at most 2000 characters')
    .optional(),
  isFree: z.boolean(),
  price: z.number().min(0).max(100).optional(),
  scheduledPublishAt: z.date().optional(),
});

export type LoginFormData = z.infer<typeof loginSchema>;
export type RegisterFormData = z.infer<typeof registerSchema>;
export type StoryFormData = z.infer<typeof storySchema>;
export type ChapterFormData = z.infer<typeof chapterSchema>;
```

### 6.3 Form Component Pattern

```typescript
// features/auth/components/LoginForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { loginSchema, LoginFormData } from '../schemas';
import { useLoginMutation } from '../hooks/useAuthMutations';

export function LoginForm() {
  const loginMutation = useLoginMutation();
  
  const form = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
      rememberMe: false,
    },
  });

  const onSubmit = async (data: LoginFormData) => {
    try {
      await loginMutation.mutateAsync(data);
    } catch (error) {
      // Handle specific errors
      if (error instanceof ApiError && error.code === 'AUTH_INVALID_CREDENTIALS') {
        form.setError('root', { message: 'Invalid email or password' });
      }
    }
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
      <FormField
        control={form.control}
        name="email"
        render={({ field }) => (
          <FormItem>
            <FormLabel>Email</FormLabel>
            <FormControl>
              <Input
                type="email"
                placeholder="you@example.com"
                {...field}
              />
            </FormControl>
            <FormMessage />
          </FormItem>
        )}
      />

      <FormField
        control={form.control}
        name="password"
        render={({ field }) => (
          <FormItem>
            <FormLabel>Password</FormLabel>
            <FormControl>
              <Input type="password" {...field} />
            </FormControl>
            <FormMessage />
          </FormItem>
        )}
      />

      <FormField
        control={form.control}
        name="rememberMe"
        render={({ field }) => (
          <FormItem className="flex items-center gap-2">
            <FormControl>
              <Checkbox
                checked={field.value}
                onCheckedChange={field.onChange}
              />
            </FormControl>
            <FormLabel className="!mt-0">Remember me</FormLabel>
          </FormItem>
        )}
      />

      {form.formState.errors.root && (
        <Alert variant="destructive">
          {form.formState.errors.root.message}
        </Alert>
      )}

      <Button
        type="submit"
        className="w-full"
        disabled={loginMutation.isPending}
      >
        {loginMutation.isPending ? 'Signing in...' : 'Sign In'}
      </Button>
    </form>
  );
}
```

---

## 7. Data Synchronization

### 7.1 Real-time Updates via WebSocket

```typescript
// hooks/useRealtimeUpdates.ts
import { useEffect } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { wsClient } from '@/lib/websocket/client';
import { storyKeys, notificationKeys } from '@/lib/query/keys';

export function useRealtimeUpdates() {
  const queryClient = useQueryClient();

  useEffect(() => {
    // Subscribe to story updates
    const unsubStory = wsClient.subscribe('story:updated', (data: any) => {
      queryClient.invalidateQueries({ 
        queryKey: storyKeys.detail(data.storyId) 
      });
    });

    // Subscribe to new chapter notifications
    const unsubChapter = wsClient.subscribe('chapter:published', (data: any) => {
      queryClient.invalidateQueries({ 
        queryKey: storyKeys.chapters(data.storyId) 
      });
      queryClient.invalidateQueries({ 
        queryKey: notificationKeys.all 
      });
    });

    // Subscribe to notification updates
    const unsubNotification = wsClient.subscribe('notification:new', () => {
      queryClient.invalidateQueries({ 
        queryKey: notificationKeys.unreadCount() 
      });
    });

    return () => {
      unsubStory();
      unsubChapter();
      unsubNotification();
    };
  }, [queryClient]);
}
```

### 7.2 Offline Support

```typescript
// lib/offline/offlineManager.ts
import { QueryClient } from '@tanstack/react-query';
import { persistQueryClient } from '@tanstack/react-query-persist-client';
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';

export function setupOfflineSupport(queryClient: QueryClient) {
  const persister = createSyncStoragePersister({
    storage: window.localStorage,
    key: 'n9-query-cache',
  });

  persistQueryClient({
    queryClient,
    persister,
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
    dehydrateOptions: {
      shouldDehydrateQuery: (query) => {
        // Only persist certain queries
        const persistKeys = ['stories', 'user', 'reading-progress'];
        return persistKeys.some((key) => 
          query.queryKey[0]?.toString().includes(key)
        );
      },
    },
  });
}
```

---

## 8. Performance Patterns

### 8.1 Selective Subscriptions

```typescript
// Only subscribe to specific state slices
function UserAvatar() {
  // Only re-renders when avatarUrl changes
  const avatarUrl = useAuthStore((state) => state.user?.avatarUrl);
  
  return <Avatar src={avatarUrl} />;
}

// Use shallow comparison for object slices
import { shallow } from 'zustand/shallow';

function UserInfo() {
  const { displayName, email } = useAuthStore(
    (state) => ({
      displayName: state.user?.displayName,
      email: state.user?.email,
    }),
    shallow
  );
  
  return (
    <div>
      <span>{displayName}</span>
      <span>{email}</span>
    </div>
  );
}
```

### 8.2 Query Deduplication

```typescript
// React Query automatically deduplicates identical queries
function StoryPage({ storyId }: { storyId: string }) {
  // Both components share the same query
  return (
    <>
      <StoryHeader storyId={storyId} />
      <StoryContent storyId={storyId} />
    </>
  );
}

function StoryHeader({ storyId }: { storyId: string }) {
  const { data: story } = useStoryQuery(storyId);
  return <h1>{story?.title}</h1>;
}

function StoryContent({ storyId }: { storyId: string }) {
  const { data: story } = useStoryQuery(storyId);
  return <p>{story?.description}</p>;
}
```

---

## 9. References

### 9.1 Related Documents

| Document | Purpose |
|----------|---------|
| [01_FRONTEND_ARCHITECTURE.md](01_FRONTEND_ARCHITECTURE.md) | System architecture |
| [02_DESIGN_SYSTEM_GUIDELINES.md](02_DESIGN_SYSTEM_GUIDELINES.md) | UI/UX standards |

### 9.2 External Resources

- [TanStack Query Documentation](https://tanstack.com/query/latest)
- [Zustand Documentation](https://docs.pmnd.rs/zustand)
- [React Router Documentation](https://reactrouter.com)
- [React Hook Form Documentation](https://react-hook-form.com)
