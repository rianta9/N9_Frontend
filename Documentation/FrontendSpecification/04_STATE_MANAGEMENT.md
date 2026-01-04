# State Management Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Frontend Architecture Team |
| Review Cycle | Quarterly |
| Related Documents | 02_FRONTEND_ARCHITECTURE.md, 06_API_INTEGRATION.md |

---

## 2. State Management Philosophy

### 2.1 Core Principles

| Principle | Description | Implementation |
|-----------|-------------|----------------|
| **Single Source of Truth** | Each piece of state has one owner | Avoid state duplication |
| **Minimal State** | Store only what's necessary | Derive computed values |
| **Colocation** | State close to where it's used | Feature-based organization |
| **Predictability** | State changes are traceable | Immutable updates |
| **Performance** | Selective re-renders | Granular subscriptions |

### 2.2 State Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           STATE CATEGORIES                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ SERVER STATE (TanStack Query)                                           ││
│  │ • Data fetched from API                                                 ││
│  │ • Stories, chapters, users, comments                                    ││
│  │ • Paginated lists, search results                                       ││
│  │ • Wallet balance, transactions                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ CLIENT STATE (Zustand)                                                  ││
│  │ • UI state (modals, sidebars)                                           ││
│  │ • User preferences (theme, reader settings)                             ││
│  │ • Authentication state                                                  ││
│  │ • Notification state                                                    ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ URL STATE (React Router)                                                ││
│  │ • Current route                                                         ││
│  │ • Search parameters                                                     ││
│  │ • Filters, pagination                                                   ││
│  │ • Tab selection                                                         ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ FORM STATE (React Hook Form)                                            ││
│  │ • Form input values                                                     ││
│  │ • Validation errors                                                     ││
│  │ • Form submission state                                                 ││
│  │ • Dirty/touched tracking                                                ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ COMPONENT STATE (useState/useReducer)                                   ││
│  │ • Local UI toggles                                                      ││
│  │ • Temporary values                                                      ││
│  │ • Animation states                                                      ││
│  │ • Ref-based state                                                       ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Server State (TanStack Query)

### 3.1 Configuration

```typescript
// config/queryClient.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Stale time - data is fresh for 5 minutes
      staleTime: 5 * 60 * 1000,
      
      // Cache time - unused data kept for 30 minutes
      gcTime: 30 * 60 * 1000,
      
      // Retry configuration
      retry: (failureCount, error) => {
        // Don't retry on 4xx errors
        if (error instanceof ApiError && error.status >= 400 && error.status < 500) {
          return false;
        }
        return failureCount < 3;
      },
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
      
      // Refetch behavior
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
      refetchOnMount: true,
    },
    mutations: {
      // Global error handling
      onError: (error) => {
        console.error('Mutation error:', error);
      },
    },
  },
});
```

### 3.2 Query Key Factory

```typescript
// lib/queryKeys.ts

export const queryKeys = {
  // ═══════════════════════════════════════════════════════════════════════════
  // STORIES
  // ═══════════════════════════════════════════════════════════════════════════
  stories: {
    all: ['stories'] as const,
    lists: () => [...queryKeys.stories.all, 'list'] as const,
    list: (filters: StoryFilters) => [...queryKeys.stories.lists(), filters] as const,
    details: () => [...queryKeys.stories.all, 'detail'] as const,
    detail: (slug: string) => [...queryKeys.stories.details(), slug] as const,
    chapters: (storySlug: string) => [...queryKeys.stories.detail(storySlug), 'chapters'] as const,
    reviews: (storySlug: string) => [...queryKeys.stories.detail(storySlug), 'reviews'] as const,
    similar: (storySlug: string) => [...queryKeys.stories.detail(storySlug), 'similar'] as const,
    trending: () => [...queryKeys.stories.all, 'trending'] as const,
    featured: () => [...queryKeys.stories.all, 'featured'] as const,
    recommended: () => [...queryKeys.stories.all, 'recommended'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // CHAPTERS
  // ═══════════════════════════════════════════════════════════════════════════
  chapters: {
    all: ['chapters'] as const,
    detail: (chapterId: string) => [...queryKeys.chapters.all, chapterId] as const,
    content: (chapterId: string) => [...queryKeys.chapters.detail(chapterId), 'content'] as const,
    comments: (chapterId: string) => [...queryKeys.chapters.detail(chapterId), 'comments'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // USERS
  // ═══════════════════════════════════════════════════════════════════════════
  users: {
    all: ['users'] as const,
    current: () => [...queryKeys.users.all, 'current'] as const,
    profile: (userId: string) => [...queryKeys.users.all, userId] as const,
    stories: (userId: string) => [...queryKeys.users.profile(userId), 'stories'] as const,
    followers: (userId: string) => [...queryKeys.users.profile(userId), 'followers'] as const,
    following: (userId: string) => [...queryKeys.users.profile(userId), 'following'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // LIBRARY
  // ═══════════════════════════════════════════════════════════════════════════
  library: {
    all: ['library'] as const,
    stories: () => [...queryKeys.library.all, 'stories'] as const,
    history: () => [...queryKeys.library.all, 'history'] as const,
    bookmarks: () => [...queryKeys.library.all, 'bookmarks'] as const,
    collections: () => [...queryKeys.library.all, 'collections'] as const,
    collection: (id: string) => [...queryKeys.library.collections(), id] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // PAYMENTS
  // ═══════════════════════════════════════════════════════════════════════════
  payments: {
    all: ['payments'] as const,
    wallet: () => [...queryKeys.payments.all, 'wallet'] as const,
    transactions: (filters?: TransactionFilters) => 
      [...queryKeys.payments.all, 'transactions', filters] as const,
    packages: () => [...queryKeys.payments.all, 'packages'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // NOTIFICATIONS
  // ═══════════════════════════════════════════════════════════════════════════
  notifications: {
    all: ['notifications'] as const,
    list: () => [...queryKeys.notifications.all, 'list'] as const,
    unreadCount: () => [...queryKeys.notifications.all, 'unread'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // AUTHOR
  // ═══════════════════════════════════════════════════════════════════════════
  author: {
    all: ['author'] as const,
    dashboard: () => [...queryKeys.author.all, 'dashboard'] as const,
    stories: () => [...queryKeys.author.all, 'stories'] as const,
    story: (id: string) => [...queryKeys.author.stories(), id] as const,
    analytics: (storyId?: string) => 
      storyId 
        ? [...queryKeys.author.all, 'analytics', storyId] 
        : [...queryKeys.author.all, 'analytics'] as const,
    earnings: () => [...queryKeys.author.all, 'earnings'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // SEARCH
  // ═══════════════════════════════════════════════════════════════════════════
  search: {
    all: ['search'] as const,
    results: (query: string, filters?: SearchFilters) => 
      [...queryKeys.search.all, query, filters] as const,
    suggestions: (query: string) => 
      [...queryKeys.search.all, 'suggestions', query] as const,
  },
} as const;
```

### 3.3 Query Hook Patterns

#### 3.3.1 Basic Query Hook

```typescript
// features/stories/hooks/useStoryQuery.ts
import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '@/lib/queryKeys';
import { storyApi } from '../api/storyApi';
import type { Story } from '../types';

interface UseStoryQueryOptions {
  enabled?: boolean;
}

export function useStoryQuery(slug: string, options?: UseStoryQueryOptions) {
  return useQuery({
    queryKey: queryKeys.stories.detail(slug),
    queryFn: () => storyApi.getBySlug(slug),
    enabled: options?.enabled ?? !!slug,
    staleTime: 10 * 60 * 1000, // Stories can be stale for 10 minutes
  });
}
```

#### 3.3.2 Paginated Query Hook

```typescript
// features/stories/hooks/useStoriesQuery.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { queryKeys } from '@/lib/queryKeys';
import { storyApi } from '../api/storyApi';
import type { StoryFilters, PaginatedResponse, Story } from '../types';

export function useStoriesQuery(filters: StoryFilters) {
  return useInfiniteQuery({
    queryKey: queryKeys.stories.list(filters),
    queryFn: ({ pageParam = 1 }) => 
      storyApi.list({ ...filters, page: pageParam }),
    initialPageParam: 1,
    getNextPageParam: (lastPage) => {
      if (lastPage.meta.pagination.page < lastPage.meta.pagination.totalPages) {
        return lastPage.meta.pagination.page + 1;
      }
      return undefined;
    },
    staleTime: 5 * 60 * 1000,
  });
}

// Flattened data helper
export function useStoriesList(filters: StoryFilters) {
  const query = useStoriesQuery(filters);
  
  const stories = useMemo(() => {
    return query.data?.pages.flatMap(page => page.data) ?? [];
  }, [query.data]);
  
  return {
    ...query,
    stories,
  };
}
```

#### 3.3.3 Mutation Hook with Optimistic Updates

```typescript
// features/interactions/hooks/useLikeMutation.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { queryKeys } from '@/lib/queryKeys';
import { interactionApi } from '../api/interactionApi';
import type { Story } from '@/features/stories/types';

interface LikeContext {
  previousStory?: Story;
}

export function useLikeMutation(storySlug: string) {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: () => interactionApi.likeStory(storySlug),
    
    // Optimistic update
    onMutate: async (): Promise<LikeContext> => {
      // Cancel any outgoing refetches
      await queryClient.cancelQueries({
        queryKey: queryKeys.stories.detail(storySlug),
      });
      
      // Snapshot previous value
      const previousStory = queryClient.getQueryData<Story>(
        queryKeys.stories.detail(storySlug)
      );
      
      // Optimistically update
      if (previousStory) {
        queryClient.setQueryData<Story>(
          queryKeys.stories.detail(storySlug),
          {
            ...previousStory,
            isLiked: !previousStory.isLiked,
            stats: {
              ...previousStory.stats,
              likes: previousStory.isLiked 
                ? previousStory.stats.likes - 1 
                : previousStory.stats.likes + 1,
            },
          }
        );
      }
      
      return { previousStory };
    },
    
    // Rollback on error
    onError: (error, variables, context) => {
      if (context?.previousStory) {
        queryClient.setQueryData(
          queryKeys.stories.detail(storySlug),
          context.previousStory
        );
      }
    },
    
    // Refetch after success or error
    onSettled: () => {
      queryClient.invalidateQueries({
        queryKey: queryKeys.stories.detail(storySlug),
      });
    },
  });
}
```

### 3.4 Cache Invalidation Strategies

```typescript
// lib/cacheInvalidation.ts
import { queryClient } from '@/config/queryClient';
import { queryKeys } from '@/lib/queryKeys';

export const invalidation = {
  // After publishing a chapter
  onChapterPublished: (storySlug: string) => {
    queryClient.invalidateQueries({
      queryKey: queryKeys.stories.detail(storySlug),
    });
    queryClient.invalidateQueries({
      queryKey: queryKeys.stories.chapters(storySlug),
    });
    queryClient.invalidateQueries({
      queryKey: queryKeys.stories.trending(),
    });
    queryClient.invalidateQueries({
      queryKey: queryKeys.author.dashboard(),
    });
  },
  
  // After wallet transaction
  onWalletTransaction: () => {
    queryClient.invalidateQueries({
      queryKey: queryKeys.payments.wallet(),
    });
    queryClient.invalidateQueries({
      queryKey: queryKeys.payments.transactions(),
    });
  },
  
  // After user profile update
  onProfileUpdate: (userId: string) => {
    queryClient.invalidateQueries({
      queryKey: queryKeys.users.profile(userId),
    });
    queryClient.invalidateQueries({
      queryKey: queryKeys.users.current(),
    });
  },
  
  // Full cache clear on logout
  onLogout: () => {
    queryClient.clear();
  },
};
```

---

## 4. Client State (Zustand)

### 4.1 Store Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ZUSTAND STORES                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ AUTH STORE                                                              ││
│  │ • User session (tokens, user info)                                      ││
│  │ • Authentication status                                                 ││
│  │ • Login/logout actions                                                  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ THEME STORE                                                             ││
│  │ • Theme preference (light/dark/sepia)                                   ││
│  │ • System preference detection                                           ││
│  │ • Persistent storage                                                    ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ READER SETTINGS STORE                                                   ││
│  │ • Font size, family, line height                                        ││
│  │ • Reading theme                                                         ││
│  │ • Layout preferences                                                    ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ UI STORE                                                                ││
│  │ • Modal states                                                          ││
│  │ • Sidebar visibility                                                    ││
│  │ • Toast notifications                                                   ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
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
  username: string;
  avatarUrl?: string;
  roles: string[];
  isAuthor: boolean;
  isVerified: boolean;
}

interface AuthState {
  // State
  user: User | null;
  accessToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  
  // Actions
  setAuth: (user: User, accessToken: string) => void;
  updateUser: (updates: Partial<User>) => void;
  setLoading: (loading: boolean) => void;
  logout: () => void;
  
  // Computed (getters via selectors)
}

export const useAuthStore = create<AuthState>()(
  persist(
    immer((set, get) => ({
      // Initial state
      user: null,
      accessToken: null,
      isAuthenticated: false,
      isLoading: true,
      
      // Actions
      setAuth: (user, accessToken) => {
        set((state) => {
          state.user = user;
          state.accessToken = accessToken;
          state.isAuthenticated = true;
          state.isLoading = false;
        });
      },
      
      updateUser: (updates) => {
        set((state) => {
          if (state.user) {
            Object.assign(state.user, updates);
          }
        });
      },
      
      setLoading: (loading) => {
        set((state) => {
          state.isLoading = loading;
        });
      },
      
      logout: () => {
        set((state) => {
          state.user = null;
          state.accessToken = null;
          state.isAuthenticated = false;
          state.isLoading = false;
        });
      },
    })),
    {
      name: 'n9-auth',
      storage: createJSONStorage(() => sessionStorage),
      partialize: (state) => ({
        user: state.user,
        accessToken: state.accessToken,
        isAuthenticated: state.isAuthenticated,
      }),
    }
  )
);

// Selectors for derived state
export const selectIsAuthor = (state: AuthState) => state.user?.isAuthor ?? false;
export const selectUserRoles = (state: AuthState) => state.user?.roles ?? [];
export const selectHasRole = (role: string) => (state: AuthState) => 
  state.user?.roles?.includes(role) ?? false;
```

### 4.3 Theme Store

```typescript
// stores/themeStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

type Theme = 'light' | 'dark' | 'system';
type ResolvedTheme = 'light' | 'dark';

interface ThemeState {
  theme: Theme;
  resolvedTheme: ResolvedTheme;
  setTheme: (theme: Theme) => void;
}

const getSystemTheme = (): ResolvedTheme => {
  if (typeof window === 'undefined') return 'light';
  return window.matchMedia('(prefers-color-scheme: dark)').matches 
    ? 'dark' 
    : 'light';
};

const resolveTheme = (theme: Theme): ResolvedTheme => {
  if (theme === 'system') return getSystemTheme();
  return theme;
};

export const useThemeStore = create<ThemeState>()(
  persist(
    (set, get) => ({
      theme: 'system',
      resolvedTheme: getSystemTheme(),
      
      setTheme: (theme) => {
        const resolved = resolveTheme(theme);
        set({ theme, resolvedTheme: resolved });
        
        // Apply to document
        document.documentElement.setAttribute('data-theme', resolved);
      },
    }),
    {
      name: 'n9-theme',
      storage: createJSONStorage(() => localStorage),
      onRehydrateStorage: () => (state) => {
        if (state) {
          const resolved = resolveTheme(state.theme);
          document.documentElement.setAttribute('data-theme', resolved);
        }
      },
    }
  )
);

// Listen for system theme changes
if (typeof window !== 'undefined') {
  window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', () => {
    const { theme, setTheme } = useThemeStore.getState();
    if (theme === 'system') {
      setTheme('system'); // Re-resolve
    }
  });
}
```

### 4.4 Reader Settings Store

```typescript
// stores/readerSettingsStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

type ReaderTheme = 'light' | 'dark' | 'sepia';
type FontFamily = 'serif' | 'sans' | 'mono';

interface ReaderSettings {
  // Appearance
  theme: ReaderTheme;
  fontSize: number; // 14-24
  fontFamily: FontFamily;
  lineHeight: number; // 1.4-2.2
  
  // Layout
  maxWidth: 'narrow' | 'medium' | 'wide';
  textAlign: 'left' | 'justify';
  
  // Behavior
  autoScroll: boolean;
  scrollSpeed: number; // 1-10
  showProgress: boolean;
}

interface ReaderSettingsState extends ReaderSettings {
  // Actions
  setTheme: (theme: ReaderTheme) => void;
  setFontSize: (size: number) => void;
  increaseFontSize: () => void;
  decreaseFontSize: () => void;
  setFontFamily: (family: FontFamily) => void;
  setLineHeight: (height: number) => void;
  setMaxWidth: (width: ReaderSettings['maxWidth']) => void;
  setTextAlign: (align: ReaderSettings['textAlign']) => void;
  setAutoScroll: (enabled: boolean) => void;
  setScrollSpeed: (speed: number) => void;
  setShowProgress: (show: boolean) => void;
  resetToDefaults: () => void;
}

const DEFAULT_SETTINGS: ReaderSettings = {
  theme: 'light',
  fontSize: 18,
  fontFamily: 'serif',
  lineHeight: 1.8,
  maxWidth: 'medium',
  textAlign: 'left',
  autoScroll: false,
  scrollSpeed: 5,
  showProgress: true,
};

export const useReaderSettingsStore = create<ReaderSettingsState>()(
  persist(
    (set) => ({
      ...DEFAULT_SETTINGS,
      
      setTheme: (theme) => set({ theme }),
      
      setFontSize: (fontSize) => set({ 
        fontSize: Math.min(Math.max(fontSize, 14), 24) 
      }),
      
      increaseFontSize: () => set((state) => ({
        fontSize: Math.min(state.fontSize + 2, 24),
      })),
      
      decreaseFontSize: () => set((state) => ({
        fontSize: Math.max(state.fontSize - 2, 14),
      })),
      
      setFontFamily: (fontFamily) => set({ fontFamily }),
      
      setLineHeight: (lineHeight) => set({ 
        lineHeight: Math.min(Math.max(lineHeight, 1.4), 2.2) 
      }),
      
      setMaxWidth: (maxWidth) => set({ maxWidth }),
      
      setTextAlign: (textAlign) => set({ textAlign }),
      
      setAutoScroll: (autoScroll) => set({ autoScroll }),
      
      setScrollSpeed: (scrollSpeed) => set({ 
        scrollSpeed: Math.min(Math.max(scrollSpeed, 1), 10) 
      }),
      
      setShowProgress: (showProgress) => set({ showProgress }),
      
      resetToDefaults: () => set(DEFAULT_SETTINGS),
    }),
    {
      name: 'n9-reader-settings',
      storage: createJSONStorage(() => localStorage),
    }
  )
);
```

### 4.5 UI Store

```typescript
// stores/uiStore.ts
import { create } from 'zustand';

type ModalType = 
  | 'login'
  | 'register'
  | 'unlock-chapter'
  | 'gift'
  | 'report'
  | 'share'
  | 'confirm'
  | null;

interface Toast {
  id: string;
  type: 'success' | 'error' | 'warning' | 'info';
  title: string;
  message?: string;
  duration?: number;
}

interface UIState {
  // Modal
  activeModal: ModalType;
  modalData: Record<string, unknown> | null;
  
  // Sidebar
  sidebarOpen: boolean;
  
  // Toasts
  toasts: Toast[];
  
  // Actions
  openModal: (modal: ModalType, data?: Record<string, unknown>) => void;
  closeModal: () => void;
  toggleSidebar: () => void;
  setSidebarOpen: (open: boolean) => void;
  addToast: (toast: Omit<Toast, 'id'>) => void;
  removeToast: (id: string) => void;
  clearToasts: () => void;
}

export const useUIStore = create<UIState>((set, get) => ({
  // Initial state
  activeModal: null,
  modalData: null,
  sidebarOpen: false,
  toasts: [],
  
  // Modal actions
  openModal: (modal, data = null) => set({
    activeModal: modal,
    modalData: data,
  }),
  
  closeModal: () => set({
    activeModal: null,
    modalData: null,
  }),
  
  // Sidebar actions
  toggleSidebar: () => set((state) => ({
    sidebarOpen: !state.sidebarOpen,
  })),
  
  setSidebarOpen: (open) => set({ sidebarOpen: open }),
  
  // Toast actions
  addToast: (toast) => {
    const id = `toast-${Date.now()}-${Math.random().toString(36).slice(2)}`;
    const newToast = { ...toast, id };
    
    set((state) => ({
      toasts: [...state.toasts, newToast],
    }));
    
    // Auto-remove after duration
    const duration = toast.duration ?? 5000;
    if (duration > 0) {
      setTimeout(() => {
        get().removeToast(id);
      }, duration);
    }
  },
  
  removeToast: (id) => set((state) => ({
    toasts: state.toasts.filter((t) => t.id !== id),
  })),
  
  clearToasts: () => set({ toasts: [] }),
}));

// Helper hooks
export const useModal = () => {
  const { activeModal, modalData, openModal, closeModal } = useUIStore();
  return { activeModal, modalData, openModal, closeModal };
};

export const useToast = () => {
  const { addToast } = useUIStore();
  return {
    success: (title: string, message?: string) => 
      addToast({ type: 'success', title, message }),
    error: (title: string, message?: string) => 
      addToast({ type: 'error', title, message }),
    warning: (title: string, message?: string) => 
      addToast({ type: 'warning', title, message }),
    info: (title: string, message?: string) => 
      addToast({ type: 'info', title, message }),
  };
};
```

---

## 5. URL State (React Router)

### 5.1 URL State Patterns

```typescript
// hooks/useSearchParams.ts
import { useSearchParams as useRouterSearchParams } from 'react-router-dom';
import { useMemo, useCallback } from 'react';

export function useQueryParams<T extends Record<string, string>>() {
  const [searchParams, setSearchParams] = useRouterSearchParams();
  
  const params = useMemo(() => {
    const obj: Record<string, string> = {};
    searchParams.forEach((value, key) => {
      obj[key] = value;
    });
    return obj as T;
  }, [searchParams]);
  
  const setParam = useCallback((key: keyof T, value: string | null) => {
    setSearchParams((prev) => {
      if (value === null || value === '') {
        prev.delete(key as string);
      } else {
        prev.set(key as string, value);
      }
      return prev;
    });
  }, [setSearchParams]);
  
  const setParams = useCallback((updates: Partial<T>) => {
    setSearchParams((prev) => {
      Object.entries(updates).forEach(([key, value]) => {
        if (value === null || value === '' || value === undefined) {
          prev.delete(key);
        } else {
          prev.set(key, value as string);
        }
      });
      return prev;
    });
  }, [setSearchParams]);
  
  return { params, setParam, setParams, searchParams };
}
```

### 5.2 Filter State in URL

```typescript
// features/stories/hooks/useStoryFilters.ts
import { useQueryParams } from '@/hooks/useSearchParams';

interface StoryFilterParams {
  genre?: string;
  status?: string;
  sort?: string;
  page?: string;
  rating?: string;
}

export function useStoryFilters() {
  const { params, setParam, setParams } = useQueryParams<StoryFilterParams>();
  
  const filters = useMemo(() => ({
    genre: params.genre || undefined,
    status: params.status || undefined,
    sort: params.sort || 'trending',
    page: parseInt(params.page || '1', 10),
    rating: params.rating ? parseInt(params.rating, 10) : undefined,
  }), [params]);
  
  const setGenre = (genre: string | null) => setParam('genre', genre);
  const setStatus = (status: string | null) => setParam('status', status);
  const setSort = (sort: string) => setParam('sort', sort);
  const setPage = (page: number) => setParam('page', page.toString());
  const setRating = (rating: number | null) => 
    setParam('rating', rating?.toString() ?? null);
  
  const resetFilters = () => setParams({
    genre: null,
    status: null,
    sort: 'trending',
    page: '1',
    rating: null,
  });
  
  return {
    filters,
    setGenre,
    setStatus,
    setSort,
    setPage,
    setRating,
    resetFilters,
  };
}
```

---

## 6. Form State (React Hook Form)

### 6.1 Form Configuration

```typescript
// features/auth/components/LoginForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string()
    .min(1, 'Email is required')
    .email('Please enter a valid email'),
  password: z.string()
    .min(1, 'Password is required'),
  rememberMe: z.boolean().optional(),
});

type LoginFormData = z.infer<typeof loginSchema>;

export function LoginForm() {
  const form = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
      rememberMe: false,
    },
    mode: 'onBlur', // Validate on blur
  });
  
  const onSubmit = async (data: LoginFormData) => {
    // Handle submission
  };
  
  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* Form fields */}
    </form>
  );
}
```

### 6.2 Complex Form with Nested State

```typescript
// features/author/components/StoryForm.tsx
import { useForm, useFieldArray } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const storySchema = z.object({
  title: z.string().min(3).max(200),
  description: z.string().min(50).max(5000),
  coverUrl: z.string().url().optional(),
  categories: z.array(z.string()).min(1).max(3),
  tags: z.array(z.object({
    name: z.string().min(2).max(30),
  })).max(10),
  contentRating: z.enum(['GENERAL', 'TEEN', 'MATURE']),
  settings: z.object({
    enableComments: z.boolean(),
    enableDonations: z.boolean(),
    visibility: z.enum(['PUBLIC', 'FOLLOWERS', 'PRIVATE']),
  }),
});

type StoryFormData = z.infer<typeof storySchema>;

export function StoryForm({ defaultValues, onSubmit }) {
  const form = useForm<StoryFormData>({
    resolver: zodResolver(storySchema),
    defaultValues: defaultValues ?? {
      title: '',
      description: '',
      categories: [],
      tags: [],
      contentRating: 'GENERAL',
      settings: {
        enableComments: true,
        enableDonations: true,
        visibility: 'PUBLIC',
      },
    },
  });
  
  const { fields: tagFields, append: addTag, remove: removeTag } = 
    useFieldArray({
      control: form.control,
      name: 'tags',
    });
  
  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* Complex form with tags array and nested settings */}
    </form>
  );
}
```

---

## 7. State Flow Diagrams

### 7.1 Authentication Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AUTHENTICATION STATE FLOW                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐    Login Form Submit    ┌──────────────┐                  │
│  │ Unauthenticated│─────────────────────▶│   Loading    │                  │
│  │    State     │                         │    State     │                  │
│  └──────────────┘                         └──────┬───────┘                  │
│         ▲                                        │                          │
│         │                              ┌─────────┴─────────┐                │
│         │                              ▼                   ▼                │
│         │                       ┌────────────┐      ┌────────────┐          │
│         │                       │  Success   │      │   Error    │          │
│         │                       └──────┬─────┘      └──────┬─────┘          │
│         │                              │                   │                │
│         │                              ▼                   │                │
│         │                       ┌────────────┐             │                │
│         │                       │Authenticated│            │                │
│         │                       │   State    │◀────────────┘                │
│   Logout│                       └──────┬─────┘     Retry                    │
│         │                              │                                     │
│         │                   Token Refresh│                                   │
│         │                              │                                     │
│         │                              ▼                                     │
│         │                       ┌────────────┐                              │
│         └───────────────────────│  Refresh   │                              │
│                                 │   Token    │                              │
│                                 └────────────┘                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Data Fetching Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DATA FETCHING STATE FLOW                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Component Mount                                                             │
│        │                                                                     │
│        ▼                                                                     │
│  ┌─────────────┐                                                            │
│  │ Check Cache │                                                            │
│  └──────┬──────┘                                                            │
│         │                                                                    │
│    ┌────┴────┐                                                              │
│    ▼         ▼                                                              │
│ ┌──────┐  ┌──────┐                                                          │
│ │Fresh │  │Stale │                                                          │
│ │Data  │  │/None │                                                          │
│ └──┬───┘  └──┬───┘                                                          │
│    │         │                                                               │
│    │         ▼                                                               │
│    │    ┌─────────┐                                                         │
│    │    │ Fetch   │                                                         │
│    │    │ Loading │                                                         │
│    │    └────┬────┘                                                         │
│    │         │                                                               │
│    │    ┌────┴────┐                                                         │
│    │    ▼         ▼                                                         │
│    │ ┌──────┐  ┌──────┐                                                     │
│    │ │Success│  │Error │                                                    │
│    │ └──┬───┘  └──┬───┘                                                     │
│    │    │         │                                                          │
│    │    ▼         ▼                                                          │
│    │ ┌──────┐  ┌──────┐                                                     │
│    │ │Update │  │Retry │                                                    │
│    │ │Cache │  │Logic │                                                     │
│    │ └──┬───┘  └──────┘                                                     │
│    │    │                                                                    │
│    ▼    ▼                                                                    │
│  ┌──────────┐                                                               │
│  │  Render  │                                                               │
│  │Component │                                                               │
│  └──────────┘                                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Best Practices

### 8.1 Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Use React Query for server state | Store API data in Zustand |
| Colocate state with components | Create global state for local UI |
| Use URL for shareable state | Store filters in component state |
| Derive values with selectors | Duplicate state across stores |
| Use TypeScript for type safety | Use `any` types |
| Keep stores small and focused | Create monolithic stores |

### 8.2 Performance Guidelines

1. **Selective Subscriptions**: Only subscribe to needed state slices
2. **Memoization**: Use `useMemo` for derived values
3. **Batching**: Combine related state updates
4. **Lazy Loading**: Load stores only when needed
5. **Persistence**: Only persist necessary data

---

---

## 9. Extended Query Keys (Aligned with Backend API Catalog)

### 9.1 Complete Query Key Factory

```typescript
// lib/queryKeys.ts

export const queryKeys = {
  // ═══════════════════════════════════════════════════════════════════════════
  // STORIES (Backend: 13_API_CATALOG - Stories APIs)
  // ═══════════════════════════════════════════════════════════════════════════
  stories: {
    all: ['stories'] as const,
    lists: () => [...queryKeys.stories.all, 'list'] as const,
    list: (filters: StoryFilters) => [...queryKeys.stories.lists(), filters] as const,
    details: () => [...queryKeys.stories.all, 'detail'] as const,
    detail: (slug: string) => [...queryKeys.stories.details(), slug] as const,
    chapters: (storySlug: string) => [...queryKeys.stories.detail(storySlug), 'chapters'] as const,
    reviews: (storySlug: string, filters?: ReviewFilters) => 
      [...queryKeys.stories.detail(storySlug), 'reviews', filters] as const,
    similar: (storySlug: string) => [...queryKeys.stories.detail(storySlug), 'similar'] as const,
    stats: (storyId: string) => [...queryKeys.stories.detail(storyId), 'stats'] as const,
    trending: () => [...queryKeys.stories.all, 'trending'] as const,
    featured: () => [...queryKeys.stories.all, 'featured'] as const,
    recommended: () => [...queryKeys.stories.all, 'recommended'] as const,
    newReleases: () => [...queryKeys.stories.all, 'new-releases'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // CHAPTERS (Backend: 13_API_CATALOG - Chapters APIs)
  // ═══════════════════════════════════════════════════════════════════════════
  chapters: {
    all: ['chapters'] as const,
    detail: (chapterId: string) => [...queryKeys.chapters.all, chapterId] as const,
    content: (chapterId: string) => [...queryKeys.chapters.detail(chapterId), 'content'] as const,
    comments: (chapterId: string, filters?: CommentFilters) => 
      [...queryKeys.chapters.detail(chapterId), 'comments', filters] as const,
    bookmarks: (chapterId: string) => [...queryKeys.chapters.detail(chapterId), 'bookmarks'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // USERS (Backend: 13_API_CATALOG - Users APIs)
  // ═══════════════════════════════════════════════════════════════════════════
  users: {
    all: ['users'] as const,
    current: () => [...queryKeys.users.all, 'current'] as const,
    preferences: () => [...queryKeys.users.all, 'preferences'] as const,
    profile: (userId: string) => [...queryKeys.users.all, userId] as const,
    stories: (userId: string, filters?: StoryFilters) => 
      [...queryKeys.users.profile(userId), 'stories', filters] as const,
    followers: (userId: string, page?: number) => 
      [...queryKeys.users.profile(userId), 'followers', page] as const,
    following: (userId: string, page?: number) => 
      [...queryKeys.users.profile(userId), 'following', page] as const,
    blocked: () => [...queryKeys.users.current(), 'blocked'] as const,
    authorApplication: () => [...queryKeys.users.current(), 'author-application'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // LIBRARY (Backend: 05_READINGS_COMPONENT)
  // ═══════════════════════════════════════════════════════════════════════════
  library: {
    all: ['library'] as const,
    history: (page?: number) => [...queryKeys.library.all, 'history', page] as const,
    bookmarks: (page?: number) => [...queryKeys.library.all, 'bookmarks', page] as const,
    readingLists: () => [...queryKeys.library.all, 'reading-lists'] as const,
    readingList: (id: string) => [...queryKeys.library.readingLists(), id] as const,
    readingProgress: (storyId: string) => 
      [...queryKeys.library.all, 'progress', storyId] as const,
    streak: () => [...queryKeys.library.all, 'streak'] as const,
    goals: () => [...queryKeys.library.all, 'goals'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // INTERACTIONS (Backend: 04_INTERACTIONS_COMPONENT)
  // ═══════════════════════════════════════════════════════════════════════════
  interactions: {
    all: ['interactions'] as const,
    reviews: () => [...queryKeys.interactions.all, 'reviews'] as const,
    myReview: (storyId: string) => [...queryKeys.interactions.reviews(), storyId] as const,
    comments: () => [...queryKeys.interactions.all, 'comments'] as const,
    commentReplies: (commentId: string) => 
      [...queryKeys.interactions.comments(), commentId, 'replies'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // PAYMENTS (Backend: 03_PAYMENTS_COMPONENT, 13_API_CATALOG - Payments APIs)
  // ═══════════════════════════════════════════════════════════════════════════
  payments: {
    all: ['payments'] as const,
    wallet: () => [...queryKeys.payments.all, 'wallet'] as const,
    transactions: (filters?: TransactionFilters) => 
      [...queryKeys.payments.all, 'transactions', filters] as const,
    packages: () => [...queryKeys.payments.all, 'packages'] as const,
    donationsSent: () => [...queryKeys.payments.all, 'donations-sent'] as const,
    donationsReceived: () => [...queryKeys.payments.all, 'donations-received'] as const,
    subscription: () => [...queryKeys.payments.all, 'subscription'] as const,
    subscriptionPlans: () => [...queryKeys.payments.all, 'plans'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // NOTIFICATIONS (Backend: 07_NOTIFICATIONS_COMPONENT)
  // ═══════════════════════════════════════════════════════════════════════════
  notifications: {
    all: ['notifications'] as const,
    list: (page?: number) => [...queryKeys.notifications.all, 'list', page] as const,
    unreadCount: () => [...queryKeys.notifications.all, 'unread-count'] as const,
    settings: () => [...queryKeys.notifications.all, 'settings'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // AUTHOR (Backend: 13_API_CATALOG - Author Payouts)
  // ═══════════════════════════════════════════════════════════════════════════
  author: {
    all: ['author'] as const,
    dashboard: () => [...queryKeys.author.all, 'dashboard'] as const,
    stories: (filters?: AuthorStoryFilters) => 
      [...queryKeys.author.all, 'stories', filters] as const,
    story: (id: string) => [...queryKeys.author.stories(), id] as const,
    analytics: (storyId?: string, period?: string) => 
      storyId 
        ? [...queryKeys.author.all, 'analytics', storyId, period] 
        : [...queryKeys.author.all, 'analytics', period] as const,
    earnings: () => [...queryKeys.author.all, 'earnings'] as const,
    payouts: () => [...queryKeys.author.all, 'payouts'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // SEARCH (Backend: 13_API_CATALOG - Search APIs)
  // ═══════════════════════════════════════════════════════════════════════════
  search: {
    all: ['search'] as const,
    results: (query: string, filters?: SearchFilters) => 
      [...queryKeys.search.all, 'results', query, filters] as const,
    stories: (query: string, filters?: SearchFilters) => 
      [...queryKeys.search.all, 'stories', query, filters] as const,
    authors: (query: string, page?: number) => 
      [...queryKeys.search.all, 'authors', query, page] as const,
    tags: (query: string) => [...queryKeys.search.all, 'tags', query] as const,
    suggestions: (query: string) => 
      [...queryKeys.search.all, 'suggestions', query] as const,
    trending: () => [...queryKeys.search.all, 'trending'] as const,
  },

  // ═══════════════════════════════════════════════════════════════════════════
  // ADMIN (Backend: 13_API_CATALOG - Admin APIs)
  // ═══════════════════════════════════════════════════════════════════════════
  admin: {
    all: ['admin'] as const,
    dashboard: () => [...queryKeys.admin.all, 'dashboard'] as const,
    analytics: (period?: string) => [...queryKeys.admin.all, 'analytics', period] as const,
    reports: (filters?: ReportFilters) => [...queryKeys.admin.all, 'reports', filters] as const,
    report: (id: string) => [...queryKeys.admin.reports(), id] as const,
    users: (filters?: UserFilters) => [...queryKeys.admin.all, 'users', filters] as const,
    pendingPayouts: () => [...queryKeys.admin.all, 'payouts-pending'] as const,
    categories: () => [...queryKeys.admin.all, 'categories'] as const,
    auditLogs: (filters?: AuditLogFilters) => [...queryKeys.admin.all, 'audit-logs', filters] as const,
  },
} as const;
```

---

## 10. Additional Client Stores

### 10.1 WebSocket Store (Backend: 06_REALTIME_AND_EVENTS)

```typescript
// stores/webSocketStore.ts
import { create } from 'zustand';
import { wsClient } from '@/lib/websocket/client';

type ConnectionStatus = 'disconnected' | 'connecting' | 'connected' | 'error';

interface ChannelSubscription {
  channel: 'user' | 'story' | 'chapter';
  id: string;
}

interface WebSocketState {
  // Connection state
  status: ConnectionStatus;
  error: string | null;
  
  // Active subscriptions
  subscriptions: ChannelSubscription[];
  
  // Actions
  connect: () => void;
  disconnect: () => void;
  setStatus: (status: ConnectionStatus) => void;
  setError: (error: string | null) => void;
  subscribe: (channel: ChannelSubscription['channel'], id: string) => void;
  unsubscribe: (channel: ChannelSubscription['channel'], id: string) => void;
}

export const useWebSocketStore = create<WebSocketState>((set, get) => ({
  status: 'disconnected',
  error: null,
  subscriptions: [],
  
  connect: () => {
    set({ status: 'connecting', error: null });
    
    wsClient.connect();
    
    wsClient.on('connect', () => {
      set({ status: 'connected' });
      
      // Resubscribe to channels after reconnect
      const { subscriptions } = get();
      subscriptions.forEach((sub) => {
        wsClient.emit('subscribe', { channel: sub.channel, id: sub.id });
      });
    });
    
    wsClient.on('disconnect', () => {
      set({ status: 'disconnected' });
    });
    
    wsClient.on('error', (error) => {
      set({ status: 'error', error: error.message });
    });
  },
  
  disconnect: () => {
    wsClient.disconnect();
    set({ status: 'disconnected', subscriptions: [] });
  },
  
  setStatus: (status) => set({ status }),
  
  setError: (error) => set({ error }),
  
  subscribe: (channel, id) => {
    const subscription = { channel, id };
    
    set((state) => ({
      subscriptions: [...state.subscriptions, subscription],
    }));
    
    if (get().status === 'connected') {
      wsClient.emit('subscribe', subscription);
    }
  },
  
  unsubscribe: (channel, id) => {
    set((state) => ({
      subscriptions: state.subscriptions.filter(
        (sub) => !(sub.channel === channel && sub.id === id)
      ),
    }));
    
    if (get().status === 'connected') {
      wsClient.emit('unsubscribe', { channel, id });
    }
  },
}));
```

### 10.2 Notification Store (Badge counts & real-time)

```typescript
// stores/notificationStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface NotificationState {
  // State
  unreadCount: number;
  lastFetchedAt: string | null;
  
  // Actions
  setUnreadCount: (count: number) => void;
  incrementUnread: () => void;
  decrementUnread: () => void;
  markAllRead: () => void;
  setLastFetched: () => void;
}

export const useNotificationStore = create<NotificationState>()(
  persist(
    (set) => ({
      unreadCount: 0,
      lastFetchedAt: null,
      
      setUnreadCount: (count) => set({ unreadCount: count }),
      
      incrementUnread: () => set((state) => ({ 
        unreadCount: state.unreadCount + 1 
      })),
      
      decrementUnread: () => set((state) => ({ 
        unreadCount: Math.max(0, state.unreadCount - 1) 
      })),
      
      markAllRead: () => set({ unreadCount: 0 }),
      
      setLastFetched: () => set({ 
        lastFetchedAt: new Date().toISOString() 
      }),
    }),
    {
      name: 'n9-notifications',
      storage: createJSONStorage(() => localStorage),
    }
  )
);
```

### 10.3 Reading Progress Store (Local cache for offline support)

```typescript
// stores/readingProgressStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface LocalProgress {
  storyId: string;
  chapterId: string;
  chapterIndex: number;
  scrollPosition: number;
  lastReadAt: string;
}

interface ReadingProgressState {
  // Local progress cache (for offline support)
  progress: Record<string, LocalProgress>;
  
  // Sync state
  pendingSync: string[]; // Story IDs pending sync
  
  // Actions
  updateLocalProgress: (storyId: string, data: Partial<LocalProgress>) => void;
  markForSync: (storyId: string) => void;
  clearPendingSync: (storyId: string) => void;
  getProgress: (storyId: string) => LocalProgress | undefined;
}

export const useReadingProgressStore = create<ReadingProgressState>()(
  persist(
    (set, get) => ({
      progress: {},
      pendingSync: [],
      
      updateLocalProgress: (storyId, data) => {
        set((state) => ({
          progress: {
            ...state.progress,
            [storyId]: {
              ...state.progress[storyId],
              ...data,
              storyId,
              lastReadAt: new Date().toISOString(),
            },
          },
          pendingSync: state.pendingSync.includes(storyId)
            ? state.pendingSync
            : [...state.pendingSync, storyId],
        }));
      },
      
      markForSync: (storyId) => {
        set((state) => ({
          pendingSync: state.pendingSync.includes(storyId)
            ? state.pendingSync
            : [...state.pendingSync, storyId],
        }));
      },
      
      clearPendingSync: (storyId) => {
        set((state) => ({
          pendingSync: state.pendingSync.filter((id) => id !== storyId),
        }));
      },
      
      getProgress: (storyId) => get().progress[storyId],
    }),
    {
      name: 'n9-reading-progress',
      storage: createJSONStorage(() => localStorage),
    }
  )
);
```

### 10.4 Chapter Editor Store (Auto-save state)

```typescript
// stores/chapterEditorStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import debounce from 'lodash/debounce';

interface EditorDraft {
  chapterId: string;
  title: string;
  content: string;
  wordCount: number;
  lastSavedAt: string;
}

interface ChapterEditorState {
  // Draft state
  drafts: Record<string, EditorDraft>;
  activeDraftId: string | null;
  
  // Editor state
  isDirty: boolean;
  isSaving: boolean;
  lastError: string | null;
  
  // Actions
  setActiveDraft: (chapterId: string) => void;
  updateDraft: (chapterId: string, updates: Partial<EditorDraft>) => void;
  saveDraft: (chapterId: string) => void;
  deleteDraft: (chapterId: string) => void;
  setDirty: (isDirty: boolean) => void;
  setSaving: (isSaving: boolean) => void;
  setError: (error: string | null) => void;
}

export const useChapterEditorStore = create<ChapterEditorState>()(
  persist(
    (set, get) => ({
      drafts: {},
      activeDraftId: null,
      isDirty: false,
      isSaving: false,
      lastError: null,
      
      setActiveDraft: (chapterId) => set({ 
        activeDraftId: chapterId,
        isDirty: false,
      }),
      
      updateDraft: (chapterId, updates) => {
        const wordCount = updates.content 
          ? updates.content.trim().split(/\s+/).filter(Boolean).length
          : get().drafts[chapterId]?.wordCount || 0;
        
        set((state) => ({
          drafts: {
            ...state.drafts,
            [chapterId]: {
              ...state.drafts[chapterId],
              ...updates,
              chapterId,
              wordCount,
              lastSavedAt: new Date().toISOString(),
            },
          },
          isDirty: true,
        }));
      },
      
      saveDraft: (chapterId) => {
        const draft = get().drafts[chapterId];
        if (draft) {
          // Save to localStorage is automatic via persist
          set({ isDirty: false });
        }
      },
      
      deleteDraft: (chapterId) => {
        set((state) => {
          const { [chapterId]: _, ...remaining } = state.drafts;
          return { 
            drafts: remaining,
            activeDraftId: state.activeDraftId === chapterId 
              ? null 
              : state.activeDraftId,
          };
        });
      },
      
      setDirty: (isDirty) => set({ isDirty }),
      setSaving: (isSaving) => set({ isSaving }),
      setError: (lastError) => set({ lastError }),
    }),
    {
      name: 'n9-chapter-drafts',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ drafts: state.drafts }),
    }
  )
);
```

---

## 11. State Synchronization Patterns

### 11.1 Server/Client State Sync

```typescript
// hooks/useSyncNotifications.ts
import { useEffect } from 'react';
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { queryKeys } from '@/lib/queryKeys';
import { notificationService } from '@/features/notifications/api';
import { useNotificationStore } from '@/stores/notificationStore';
import { useWebSocketStore } from '@/stores/webSocketStore';

export function useSyncNotifications() {
  const queryClient = useQueryClient();
  const { setUnreadCount, incrementUnread } = useNotificationStore();
  const wsStatus = useWebSocketStore((s) => s.status);
  
  // Fetch initial count
  const { data: countData } = useQuery({
    queryKey: queryKeys.notifications.unreadCount(),
    queryFn: () => notificationService.getUnreadCount(),
    refetchInterval: 60000, // Poll every minute as fallback
  });
  
  // Sync server state to client store
  useEffect(() => {
    if (countData?.count !== undefined) {
      setUnreadCount(countData.count);
    }
  }, [countData, setUnreadCount]);
  
  // Handle real-time notifications
  useEffect(() => {
    if (wsStatus !== 'connected') return;
    
    const unsubscribe = wsClient.subscribe('NOTIFICATION', (notification) => {
      // Update cache
      queryClient.setQueryData(
        queryKeys.notifications.list(),
        (old: any) => ({
          ...old,
          data: [notification, ...(old?.data || [])],
        })
      );
      
      // Update local store
      incrementUnread();
    });
    
    return unsubscribe;
  }, [wsStatus, queryClient, incrementUnread]);
}
```

### 11.2 Offline Progress Sync

```typescript
// hooks/useSyncReadingProgress.ts
import { useEffect } from 'react';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useOnlineStatus } from '@/hooks/useOnlineStatus';
import { useReadingProgressStore } from '@/stores/readingProgressStore';
import { readingProgressService } from '@/features/readings/api';
import { queryKeys } from '@/lib/queryKeys';

export function useSyncReadingProgress() {
  const isOnline = useOnlineStatus();
  const queryClient = useQueryClient();
  const { pendingSync, progress, clearPendingSync } = useReadingProgressStore();
  
  const syncMutation = useMutation({
    mutationFn: async (storyId: string) => {
      const localProgress = progress[storyId];
      if (!localProgress) return;
      
      return readingProgressService.updateProgress(storyId, {
        currentChapterId: localProgress.chapterId,
        scrollPosition: localProgress.scrollPosition,
      });
    },
    onSuccess: (_, storyId) => {
      clearPendingSync(storyId);
      queryClient.invalidateQueries({
        queryKey: queryKeys.library.readingProgress(storyId),
      });
    },
  });
  
  // Sync when coming online
  useEffect(() => {
    if (isOnline && pendingSync.length > 0) {
      pendingSync.forEach((storyId) => {
        syncMutation.mutate(storyId);
      });
    }
  }, [isOnline, pendingSync]);
  
  return { isSyncing: syncMutation.isPending };
}
```

---

## 12. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Frontend Architecture Team | Initial release |
| 2.0 | 2026-01-05 | Frontend Architecture Team | Added extended query keys aligned with Backend API, WebSocket store, notification store, reading progress store, chapter editor store |

---

## 13. Related Documents

- [02_FRONTEND_ARCHITECTURE.md](./02_FRONTEND_ARCHITECTURE.md) - Architecture overview
- [06_API_INTEGRATION.md](./06_API_INTEGRATION.md) - API integration patterns
- [09_PERFORMANCE_OPTIMIZATION.md](./09_PERFORMANCE_OPTIMIZATION.md) - Performance guidelines
- [Backend 06_REALTIME_AND_EVENTS.md](../../Backend/N9/Documentation/Specification/06_REALTIME_AND_EVENTS.md) - Event System Design
- [Backend 07_NOTIFICATIONS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/07_NOTIFICATIONS_COMPONENT.md) - Notifications Design
- [Backend 05_READINGS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/05_READINGS_COMPONENT.md) - Reading Progress Design
