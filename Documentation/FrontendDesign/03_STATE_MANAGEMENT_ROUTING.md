# State Management & Routing

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2026-01-05 |
| Status | Approved |
| Owner | Frontend Engineering Team |
| Review Cycle | Quarterly |

> **Specification Reference**: [04_STATE_MANAGEMENT.md](../FrontendSpecification/04_STATE_MANAGEMENT.md)

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
│   Server State (TanStack Query)                                  │
│   ├── Remote data from 142+ API endpoints                        │
│   ├── Automatic caching & invalidation                           │
│   ├── Background refetching                                      │
│   ├── Optimistic updates                                         │
│   └── Offline support with persistence                           │
│                                                                  │
│   Client State (Zustand)                                         │
│   ├── UI state (modals, sidebars)                               │
│   ├── User preferences (theme, reading settings)                │
│   ├── Authentication state                                       │
│   ├── WebSocket connection management                            │
│   ├── Notification queue & badge                                 │
│   └── Chapter editor drafts                                      │
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

### 3.2 Query Key Factory Pattern (Extended)

> **Backend Reference**: [13_API_CATALOG.md](../BackendSpecification/13_API_CATALOG.md) - 142+ endpoints across 9 domains

```typescript
// lib/query/keys.ts

// ============================================
// STORIES DOMAIN
// ============================================
export const storyKeys = {
  all: ['stories'] as const,
  lists: () => [...storyKeys.all, 'list'] as const,
  list: (filters: StoryFilters) => [...storyKeys.lists(), filters] as const,
  details: () => [...storyKeys.all, 'detail'] as const,
  detail: (id: string) => [...storyKeys.details(), id] as const,
  chapters: (storyId: string) => [...storyKeys.detail(storyId), 'chapters'] as const,
  reviews: (storyId: string) => [...storyKeys.detail(storyId), 'reviews'] as const,
  recommendations: (storyId: string) => [...storyKeys.detail(storyId), 'recommendations'] as const,
};

// ============================================
// CHAPTERS DOMAIN
// ============================================
export const chapterKeys = {
  all: ['chapters'] as const,
  detail: (id: string) => [...chapterKeys.all, id] as const,
  content: (id: string) => [...chapterKeys.detail(id), 'content'] as const,
  comments: (chapterId: string, filters?: CommentFilters) => 
    [...chapterKeys.detail(chapterId), 'comments', filters] as const,
  bookmarks: (chapterId: string) => [...chapterKeys.detail(chapterId), 'bookmarks'] as const,
};

// ============================================
// USERS DOMAIN
// ============================================
export const userKeys = {
  all: ['users'] as const,
  current: () => [...userKeys.all, 'current'] as const,
  preferences: () => [...userKeys.current(), 'preferences'] as const,
  profile: (username: string) => [...userKeys.all, 'profile', username] as const,
  followers: (userId: string) => [...userKeys.profile(userId), 'followers'] as const,
  following: (userId: string) => [...userKeys.profile(userId), 'following'] as const,
  stories: (userId: string) => [...userKeys.profile(userId), 'stories'] as const,
  readingLists: (userId: string) => [...userKeys.profile(userId), 'reading-lists'] as const,
};

// ============================================
// LIBRARY DOMAIN (Reading Progress)
// ============================================
export const libraryKeys = {
  all: ['library'] as const,
  continueReading: () => [...libraryKeys.all, 'continue'] as const,
  history: (filters?: HistoryFilters) => [...libraryKeys.all, 'history', filters] as const,
  bookmarks: () => [...libraryKeys.all, 'bookmarks'] as const,
  progress: (storyId: string) => [...libraryKeys.all, 'progress', storyId] as const,
};

// ============================================
// INTERACTIONS DOMAIN
// ============================================
export const interactionKeys = {
  all: ['interactions'] as const,
  follows: {
    story: (storyId: string) => [...interactionKeys.all, 'follows', 'story', storyId] as const,
    author: (authorId: string) => [...interactionKeys.all, 'follows', 'author', authorId] as const,
    check: (targetType: string, targetId: string) => 
      [...interactionKeys.all, 'follows', 'check', targetType, targetId] as const,
  },
  likes: {
    story: (storyId: string) => [...interactionKeys.all, 'likes', 'story', storyId] as const,
    chapter: (chapterId: string) => [...interactionKeys.all, 'likes', 'chapter', chapterId] as const,
    comment: (commentId: string) => [...interactionKeys.all, 'likes', 'comment', commentId] as const,
  },
  reviews: {
    list: (storyId: string, filters?: ReviewFilters) => 
      [...interactionKeys.all, 'reviews', storyId, filters] as const,
    detail: (reviewId: string) => [...interactionKeys.all, 'reviews', 'detail', reviewId] as const,
    userReview: (storyId: string) => [...interactionKeys.all, 'reviews', 'user', storyId] as const,
  },
  readingLists: {
    all: () => [...interactionKeys.all, 'reading-lists'] as const,
    list: (filters?: ListFilters) => [...interactionKeys.all, 'reading-lists', 'list', filters] as const,
    detail: (listId: string) => [...interactionKeys.all, 'reading-lists', listId] as const,
    items: (listId: string) => [...interactionKeys.all, 'reading-lists', listId, 'items'] as const,
  },
};

// ============================================
// PAYMENTS DOMAIN
// ============================================
export const paymentKeys = {
  all: ['payments'] as const,
  wallet: () => [...paymentKeys.all, 'wallet'] as const,
  balance: () => [...paymentKeys.wallet(), 'balance'] as const,
  transactions: (filters?: TransactionFilters) => 
    [...paymentKeys.wallet(), 'transactions', filters] as const,
  packages: () => [...paymentKeys.all, 'packages'] as const,
  unlocks: {
    chapter: (chapterId: string) => [...paymentKeys.all, 'unlocks', 'chapter', chapterId] as const,
    story: (storyId: string) => [...paymentKeys.all, 'unlocks', 'story', storyId] as const,
  },
};

// ============================================
// NOTIFICATIONS DOMAIN
// ============================================
export const notificationKeys = {
  all: ['notifications'] as const,
  list: (filters?: NotificationFilters) => [...notificationKeys.all, 'list', filters] as const,
  unreadCount: () => [...notificationKeys.all, 'unread-count'] as const,
  preferences: () => [...notificationKeys.all, 'preferences'] as const,
};

// ============================================
// AUTHOR DOMAIN
// ============================================
export const authorKeys = {
  all: ['author'] as const,
  dashboard: () => [...authorKeys.all, 'dashboard'] as const,
  stories: (filters?: AuthorStoryFilters) => [...authorKeys.all, 'stories', filters] as const,
  analytics: {
    overview: (period?: string) => [...authorKeys.all, 'analytics', 'overview', period] as const,
    story: (storyId: string, period?: string) => 
      [...authorKeys.all, 'analytics', 'story', storyId, period] as const,
  },
  earnings: {
    summary: (period?: string) => [...authorKeys.all, 'earnings', 'summary', period] as const,
    history: (filters?: EarningsFilters) => [...authorKeys.all, 'earnings', 'history', filters] as const,
    payouts: () => [...authorKeys.all, 'earnings', 'payouts'] as const,
  },
};

// ============================================
// SEARCH DOMAIN
// ============================================
export const searchKeys = {
  all: ['search'] as const,
  global: (query: string, filters?: SearchFilters) => 
    [...searchKeys.all, 'global', query, filters] as const,
  stories: (query: string, filters?: StorySearchFilters) => 
    [...searchKeys.all, 'stories', query, filters] as const,
  authors: (query: string) => [...searchKeys.all, 'authors', query] as const,
  suggestions: (query: string) => [...searchKeys.all, 'suggestions', query] as const,
};

// ============================================
// ADMIN DOMAIN
// ============================================
export const adminKeys = {
  all: ['admin'] as const,
  dashboard: () => [...adminKeys.all, 'dashboard'] as const,
  users: (filters?: AdminUserFilters) => [...adminKeys.all, 'users', filters] as const,
  reports: (filters?: ReportFilters) => [...adminKeys.all, 'reports', filters] as const,
  moderation: {
    queue: (filters?: ModerationFilters) => [...adminKeys.all, 'moderation', 'queue', filters] as const,
    history: (filters?: ModerationFilters) => [...adminKeys.all, 'moderation', 'history', filters] as const,
  },
  applications: (status?: string) => [...adminKeys.all, 'applications', status] as const,
  payouts: (status?: string) => [...adminKeys.all, 'payouts', status] as const,
  analytics: (period?: string) => [...adminKeys.all, 'analytics', period] as const,
};

// ============================================
// READING PROGRESS DOMAIN
// ============================================
export const readingProgressKeys = {
  all: ['reading-progress'] as const,
  story: (storyId: string) => [...readingProgressKeys.all, 'story', storyId] as const,
  chapter: (chapterId: string) => [...readingProgressKeys.all, 'chapter', chapterId] as const,
  streak: () => [...readingProgressKeys.all, 'streak'] as const,
  goals: () => [...readingProgressKeys.all, 'goals'] as const,
  statistics: (period?: string) => [...readingProgressKeys.all, 'statistics', period] as const,
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
export { useWebSocketStore } from './webSocketStore';
export { useReadingProgressStore } from './readingProgressStore';
export { useChapterEditorStore } from './chapterEditorStore';
```

### 4.2 WebSocket Store

> **Backend Reference**: [06_REALTIME_AND_EVENTS.md](../BackendSpecification/06_REALTIME_AND_EVENTS.md)

```typescript
// stores/webSocketStore.ts
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';

type ConnectionStatus = 'disconnected' | 'connecting' | 'connected' | 'reconnecting';

interface WebSocketState {
  socket: WebSocket | null;
  status: ConnectionStatus;
  subscribedChannels: Set<string>;
  reconnectAttempts: number;
  lastError: string | null;
}

interface WebSocketActions {
  connect: (token: string) => void;
  disconnect: () => void;
  subscribe: (channel: string) => void;
  unsubscribe: (channel: string) => void;
  sendMessage: (type: string, payload: unknown) => void;
  setStatus: (status: ConnectionStatus) => void;
  incrementReconnect: () => void;
  resetReconnect: () => void;
}

export const useWebSocketStore = create<WebSocketState & WebSocketActions>()(
  subscribeWithSelector((set, get) => ({
    // State
    socket: null,
    status: 'disconnected',
    subscribedChannels: new Set(),
    reconnectAttempts: 0,
    lastError: null,

    // Actions
    connect: (token) => {
      const currentSocket = get().socket;
      if (currentSocket?.readyState === WebSocket.OPEN) return;

      set({ status: 'connecting' });

      const ws = new WebSocket(
        `${import.meta.env.VITE_WS_URL}/v1/connect?token=${token}`
      );

      ws.onopen = () => {
        set({ socket: ws, status: 'connected', lastError: null });
        get().resetReconnect();
        // Resubscribe to channels
        get().subscribedChannels.forEach((channel) => {
          ws.send(JSON.stringify({ type: 'SUBSCRIBE', channel }));
        });
      };

      ws.onclose = () => {
        set({ status: 'disconnected' });
        // Auto-reconnect logic
        const attempts = get().reconnectAttempts;
        if (attempts < 5) {
          setTimeout(() => {
            get().incrementReconnect();
            get().connect(token);
          }, Math.min(1000 * Math.pow(2, attempts), 30000));
        }
      };

      ws.onerror = (error) => {
        set({ lastError: 'WebSocket connection error' });
      };

      set({ socket: ws });
    },

    disconnect: () => {
      const { socket } = get();
      socket?.close();
      set({ socket: null, status: 'disconnected', subscribedChannels: new Set() });
    },

    subscribe: (channel) => {
      const { socket, subscribedChannels } = get();
      if (!subscribedChannels.has(channel)) {
        subscribedChannels.add(channel);
        set({ subscribedChannels: new Set(subscribedChannels) });
        if (socket?.readyState === WebSocket.OPEN) {
          socket.send(JSON.stringify({ type: 'SUBSCRIBE', channel }));
        }
      }
    },

    unsubscribe: (channel) => {
      const { socket, subscribedChannels } = get();
      if (subscribedChannels.has(channel)) {
        subscribedChannels.delete(channel);
        set({ subscribedChannels: new Set(subscribedChannels) });
        if (socket?.readyState === WebSocket.OPEN) {
          socket.send(JSON.stringify({ type: 'UNSUBSCRIBE', channel }));
        }
      }
    },

    sendMessage: (type, payload) => {
      const { socket } = get();
      if (socket?.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify({ type, payload }));
      }
    },

    setStatus: (status) => set({ status }),
    incrementReconnect: () => set((s) => ({ reconnectAttempts: s.reconnectAttempts + 1 })),
    resetReconnect: () => set({ reconnectAttempts: 0 }),
  }))
);

// Selectors
export const selectIsConnected = (state: WebSocketState) => state.status === 'connected';
export const selectConnectionStatus = (state: WebSocketState) => state.status;
```

### 4.3 Auth Store

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

### 4.6 UI Store

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

### 4.7 Notification Store (Extended)

> **Backend Reference**: [07_NOTIFICATIONS_COMPONENT.md](../BackendDesign/Components/07_NOTIFICATIONS_COMPONENT.md)

```typescript
// stores/notificationStore.ts
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

interface Notification {
  id: string;
  type: NotificationType;
  title: string;
  body: string;
  data?: Record<string, unknown>;
  read: boolean;
  createdAt: string;
}

type NotificationType = 
  | 'NEW_CHAPTER'
  | 'STORY_UPDATE'
  | 'COMMENT_REPLY'
  | 'COMMENT_MENTION'
  | 'NEW_FOLLOWER'
  | 'STORY_LIKE'
  | 'REVIEW_RECEIVED'
  | 'DONATION_RECEIVED'
  | 'MILESTONE_REACHED'
  | 'SYSTEM_ALERT'
  | 'PAYOUT_PROCESSED';

interface NotificationState {
  notifications: Notification[];
  unreadCount: number;
  hasNewNotifications: boolean;
  lastFetchedAt: string | null;
}

interface NotificationActions {
  setNotifications: (notifications: Notification[]) => void;
  addNotification: (notification: Notification) => void;
  markAsRead: (notificationId: string) => void;
  markAllAsRead: () => void;
  removeNotification: (notificationId: string) => void;
  setUnreadCount: (count: number) => void;
  incrementUnread: () => void;
  decrementUnread: () => void;
  clearHasNew: () => void;
}

export const useNotificationStore = create<NotificationState & NotificationActions>()(
  immer((set) => ({
    // State
    notifications: [],
    unreadCount: 0,
    hasNewNotifications: false,
    lastFetchedAt: null,

    // Actions
    setNotifications: (notifications) =>
      set((state) => {
        state.notifications = notifications;
        state.unreadCount = notifications.filter((n) => !n.read).length;
        state.lastFetchedAt = new Date().toISOString();
      }),

    addNotification: (notification) =>
      set((state) => {
        state.notifications.unshift(notification);
        if (!notification.read) {
          state.unreadCount += 1;
          state.hasNewNotifications = true;
        }
      }),

    markAsRead: (notificationId) =>
      set((state) => {
        const notification = state.notifications.find((n) => n.id === notificationId);
        if (notification && !notification.read) {
          notification.read = true;
          state.unreadCount = Math.max(0, state.unreadCount - 1);
        }
      }),

    markAllAsRead: () =>
      set((state) => {
        state.notifications.forEach((n) => {
          n.read = true;
        });
        state.unreadCount = 0;
      }),

    removeNotification: (notificationId) =>
      set((state) => {
        const index = state.notifications.findIndex((n) => n.id === notificationId);
        if (index !== -1) {
          const notification = state.notifications[index];
          if (!notification.read) {
            state.unreadCount = Math.max(0, state.unreadCount - 1);
          }
          state.notifications.splice(index, 1);
        }
      }),

    setUnreadCount: (count) => set((state) => { state.unreadCount = count; }),
    incrementUnread: () => set((state) => { state.unreadCount += 1; }),
    decrementUnread: () => set((state) => { state.unreadCount = Math.max(0, state.unreadCount - 1); }),
    clearHasNew: () => set((state) => { state.hasNewNotifications = false; }),
  }))
);

// Selectors
export const selectUnreadCount = (state: NotificationState) => state.unreadCount;
export const selectHasNewNotifications = (state: NotificationState) => state.hasNewNotifications;
```

### 4.8 Reading Progress Store

> **Backend Reference**: [05_READINGS_COMPONENT.md](../BackendDesign/Components/05_READINGS_COMPONENT.md)

```typescript
// stores/readingProgressStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface ReadingProgressEntry {
  storyId: string;
  chapterId: string;
  chapterNumber: number;
  scrollPosition: number;
  progressPercent: number;
  lastReadAt: string;
}

interface PendingSync {
  storyId: string;
  chapterId: string;
  scrollPosition: number;
  timestamp: string;
}

interface ReadingStreakData {
  currentStreak: number;
  longestStreak: number;
  todayMinutes: number;
  weeklyMinutes: number;
  lastReadDate: string | null;
}

interface ReadingProgressState {
  // Current session
  currentProgress: ReadingProgressEntry | null;
  
  // Local cache for continue reading
  recentProgress: Map<string, ReadingProgressEntry>; // storyId -> progress
  
  // Offline support
  pendingSyncs: PendingSync[];
  isOnline: boolean;
  
  // Streak data (cached)
  streakData: ReadingStreakData | null;
  
  // Reading session tracking
  sessionStartTime: number | null;
  sessionMinutes: number;
}

interface ReadingProgressActions {
  // Progress tracking
  updateProgress: (progress: Partial<ReadingProgressEntry> & { storyId: string }) => void;
  setCurrentProgress: (progress: ReadingProgressEntry) => void;
  clearCurrentProgress: () => void;
  
  // Sync management
  addPendingSync: (sync: PendingSync) => void;
  removePendingSync: (storyId: string, chapterId: string) => void;
  clearPendingSyncs: () => void;
  setOnlineStatus: (isOnline: boolean) => void;
  
  // Streak tracking
  setStreakData: (data: ReadingStreakData) => void;
  
  // Session tracking
  startReadingSession: () => void;
  endReadingSession: () => void;
  updateSessionTime: (minutes: number) => void;
}

export const useReadingProgressStore = create<ReadingProgressState & ReadingProgressActions>()(
  persist(
    immer((set, get) => ({
      // State
      currentProgress: null,
      recentProgress: new Map(),
      pendingSyncs: [],
      isOnline: navigator.onLine,
      streakData: null,
      sessionStartTime: null,
      sessionMinutes: 0,

      // Actions
      updateProgress: (progress) =>
        set((state) => {
          const existing = state.recentProgress.get(progress.storyId);
          const updated = {
            ...existing,
            ...progress,
            lastReadAt: new Date().toISOString(),
          } as ReadingProgressEntry;
          
          state.recentProgress.set(progress.storyId, updated);
          state.currentProgress = updated;
          
          // Queue for sync if offline
          if (!state.isOnline && progress.chapterId && progress.scrollPosition !== undefined) {
            state.pendingSyncs.push({
              storyId: progress.storyId,
              chapterId: progress.chapterId,
              scrollPosition: progress.scrollPosition,
              timestamp: new Date().toISOString(),
            });
          }
        }),

      setCurrentProgress: (progress) =>
        set((state) => {
          state.currentProgress = progress;
          state.recentProgress.set(progress.storyId, progress);
        }),

      clearCurrentProgress: () =>
        set((state) => {
          state.currentProgress = null;
        }),

      addPendingSync: (sync) =>
        set((state) => {
          state.pendingSyncs.push(sync);
        }),

      removePendingSync: (storyId, chapterId) =>
        set((state) => {
          state.pendingSyncs = state.pendingSyncs.filter(
            (s) => !(s.storyId === storyId && s.chapterId === chapterId)
          );
        }),

      clearPendingSyncs: () =>
        set((state) => {
          state.pendingSyncs = [];
        }),

      setOnlineStatus: (isOnline) =>
        set((state) => {
          state.isOnline = isOnline;
        }),

      setStreakData: (data) =>
        set((state) => {
          state.streakData = data;
        }),

      startReadingSession: () =>
        set((state) => {
          state.sessionStartTime = Date.now();
        }),

      endReadingSession: () =>
        set((state) => {
          if (state.sessionStartTime) {
            const minutes = Math.round((Date.now() - state.sessionStartTime) / 60000);
            state.sessionMinutes += minutes;
            state.sessionStartTime = null;
          }
        }),

      updateSessionTime: (minutes) =>
        set((state) => {
          state.sessionMinutes = minutes;
        }),
    })),
    {
      name: 'n9-reading-progress',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({
        recentProgress: Array.from(state.recentProgress.entries()),
        pendingSyncs: state.pendingSyncs,
        streakData: state.streakData,
      }),
      merge: (persistedState: any, currentState) => ({
        ...currentState,
        ...persistedState,
        recentProgress: new Map(persistedState?.recentProgress || []),
      }),
    }
  )
);

// Selectors
export const selectCurrentProgress = (state: ReadingProgressState) => state.currentProgress;
export const selectPendingSyncs = (state: ReadingProgressState) => state.pendingSyncs;
export const selectStreakData = (state: ReadingProgressState) => state.streakData;
```

### 4.9 Chapter Editor Store

```typescript
// stores/chapterEditorStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface ChapterDraft {
  storyId: string;
  chapterId?: string; // undefined for new chapters
  title: string;
  content: string;
  authorNotes?: string;
  isPremium: boolean;
  coinPrice?: number;
  scheduledAt?: string;
  lastSavedAt: string;
  wordCount: number;
}

interface ChapterEditorState {
  drafts: Map<string, ChapterDraft>; // key: `${storyId}-${chapterId || 'new'}`
  currentDraftKey: string | null;
  autoSaveEnabled: boolean;
  lastAutoSaveAt: string | null;
  hasUnsavedChanges: boolean;
}

interface ChapterEditorActions {
  // Draft management
  setDraft: (draft: ChapterDraft) => void;
  updateDraft: (key: string, updates: Partial<ChapterDraft>) => void;
  deleteDraft: (key: string) => void;
  clearAllDrafts: () => void;
  
  // Current draft
  setCurrentDraft: (key: string | null) => void;
  getCurrentDraft: () => ChapterDraft | undefined;
  
  // Auto-save
  setAutoSave: (enabled: boolean) => void;
  markAutoSaved: () => void;
  setHasUnsavedChanges: (hasChanges: boolean) => void;
}

const getDraftKey = (storyId: string, chapterId?: string) => 
  `${storyId}-${chapterId || 'new'}`;

export const useChapterEditorStore = create<ChapterEditorState & ChapterEditorActions>()(
  persist(
    immer((set, get) => ({
      // State
      drafts: new Map(),
      currentDraftKey: null,
      autoSaveEnabled: true,
      lastAutoSaveAt: null,
      hasUnsavedChanges: false,

      // Actions
      setDraft: (draft) =>
        set((state) => {
          const key = getDraftKey(draft.storyId, draft.chapterId);
          state.drafts.set(key, {
            ...draft,
            lastSavedAt: new Date().toISOString(),
            wordCount: draft.content.split(/\s+/).filter(Boolean).length,
          });
          state.currentDraftKey = key;
          state.hasUnsavedChanges = false;
        }),

      updateDraft: (key, updates) =>
        set((state) => {
          const draft = state.drafts.get(key);
          if (draft) {
            Object.assign(draft, updates);
            if (updates.content !== undefined) {
              draft.wordCount = updates.content.split(/\s+/).filter(Boolean).length;
            }
            draft.lastSavedAt = new Date().toISOString();
            state.hasUnsavedChanges = true;
          }
        }),

      deleteDraft: (key) =>
        set((state) => {
          state.drafts.delete(key);
          if (state.currentDraftKey === key) {
            state.currentDraftKey = null;
          }
        }),

      clearAllDrafts: () =>
        set((state) => {
          state.drafts.clear();
          state.currentDraftKey = null;
        }),

      setCurrentDraft: (key) =>
        set((state) => {
          state.currentDraftKey = key;
        }),

      getCurrentDraft: () => {
        const { drafts, currentDraftKey } = get();
        return currentDraftKey ? drafts.get(currentDraftKey) : undefined;
      },

      setAutoSave: (enabled) =>
        set((state) => {
          state.autoSaveEnabled = enabled;
        }),

      markAutoSaved: () =>
        set((state) => {
          state.lastAutoSaveAt = new Date().toISOString();
          state.hasUnsavedChanges = false;
        }),

      setHasUnsavedChanges: (hasChanges) =>
        set((state) => {
          state.hasUnsavedChanges = hasChanges;
        }),
    })),
    {
      name: 'n9-chapter-editor',
      partialize: (state) => ({
        drafts: Array.from(state.drafts.entries()),
        autoSaveEnabled: state.autoSaveEnabled,
      }),
      merge: (persistedState: any, currentState) => ({
        ...currentState,
        ...persistedState,
        drafts: new Map(persistedState?.drafts || []),
      }),
    }
  )
);

// Selectors
export const selectCurrentDraft = (state: ChapterEditorState) => 
  state.currentDraftKey ? state.drafts.get(state.currentDraftKey) : undefined;
export const selectHasUnsavedChanges = (state: ChapterEditorState) => state.hasUnsavedChanges;
export const selectDraftCount = (state: ChapterEditorState) => state.drafts.size;
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
| [04_STATE_MANAGEMENT.md](../FrontendSpecification/04_STATE_MANAGEMENT.md) | Frontend Specification |
| [06_REALTIME_AND_EVENTS.md](../BackendSpecification/06_REALTIME_AND_EVENTS.md) | WebSocket events |
| [13_API_CATALOG.md](../BackendSpecification/13_API_CATALOG.md) | API endpoints |

### 9.2 Backend Component References

| Component | Design Document |
|-----------|-----------------|
| Payments | [03_PAYMENTS_COMPONENT.md](../BackendDesign/Components/03_PAYMENTS_COMPONENT.md) |
| Interactions | [04_INTERACTIONS_COMPONENT.md](../BackendDesign/Components/04_INTERACTIONS_COMPONENT.md) |
| Readings | [05_READINGS_COMPONENT.md](../BackendDesign/Components/05_READINGS_COMPONENT.md) |
| Notifications | [07_NOTIFICATIONS_COMPONENT.md](../BackendDesign/Components/07_NOTIFICATIONS_COMPONENT.md) |

### 9.3 External Resources

- [TanStack Query Documentation](https://tanstack.com/query/latest)
- [Zustand Documentation](https://docs.pmnd.rs/zustand)
- [React Router Documentation](https://reactrouter.com)
- [React Hook Form Documentation](https://react-hook-form.com)

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-12-31 | Frontend Engineering Team | Initial release |
| 2.0 | 2026-01-05 | Frontend Engineering Team | Extended query key factory with 9 domains (142+ endpoints). Added WebSocket Store, extended Notification Store, Reading Progress Store with offline support, Chapter Editor Store with auto-save. Aligned with Backend Design documents. |
