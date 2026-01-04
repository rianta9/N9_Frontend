# API Integration Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Frontend Architecture Team |
| Review Cycle | Quarterly |
| Related Documents | 04_STATE_MANAGEMENT.md, 10_SECURITY_COMPLIANCE.md |

---

## 2. API Architecture Overview

### 2.1 Integration Layers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         API INTEGRATION LAYERS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ PRESENTATION LAYER                                                      ││
│  │ • React Components                                                      ││
│  │ • useQuery hooks                                                        ││
│  │ • useMutation hooks                                                     ││
│  └───────────────────────────────────┬─────────────────────────────────────┘│
│                                      │                                       │
│  ┌───────────────────────────────────▼─────────────────────────────────────┐│
│  │ REACT QUERY LAYER                                                       ││
│  │ • Query/Mutation definitions                                            ││
│  │ • Caching & Invalidation                                                ││
│  │ • Optimistic updates                                                    ││
│  └───────────────────────────────────┬─────────────────────────────────────┘│
│                                      │                                       │
│  ┌───────────────────────────────────▼─────────────────────────────────────┐│
│  │ API SERVICE LAYER                                                       ││
│  │ • Domain-specific API classes                                           ││
│  │ • Request transformation                                                ││
│  │ • Response normalization                                                ││
│  └───────────────────────────────────┬─────────────────────────────────────┘│
│                                      │                                       │
│  ┌───────────────────────────────────▼─────────────────────────────────────┐│
│  │ HTTP CLIENT LAYER                                                       ││
│  │ • Axios instance                                                        ││
│  │ • Interceptors (auth, refresh, retry)                                   ││
│  │ • Error transformation                                                  ││
│  └───────────────────────────────────┬─────────────────────────────────────┘│
│                                      │                                       │
│  ┌───────────────────────────────────▼─────────────────────────────────────┐│
│  │ NETWORK LAYER                                                           ││
│  │ • HTTP/HTTPS requests                                                   ││
│  │ • WebSocket connections                                                 ││
│  │ • Server-Sent Events                                                    ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 API Base Configuration

| Environment | Base URL | Timeout |
|-------------|----------|---------|
| Production | `https://api.n9.example.com/v1` | 30s |
| Staging | `https://api.staging.n9.example.com/v1` | 30s |
| Development | `http://localhost:8080/v1` | 60s |

---

## 3. HTTP Client Configuration

### 3.1 Axios Instance Setup

```typescript
// lib/api/client.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosError } from 'axios';
import { useAuthStore } from '@/stores/authStore';

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8080/v1';
const API_TIMEOUT = Number(import.meta.env.VITE_API_TIMEOUT) || 30000;

// Create axios instance
export const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  timeout: API_TIMEOUT,
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
  withCredentials: true,
});

// Request interceptor for auth token
apiClient.interceptors.request.use(
  (config) => {
    const token = useAuthStore.getState().accessToken;
    
    if (token && config.headers) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    
    // Add request ID for tracing
    config.headers['X-Request-ID'] = crypto.randomUUID();
    
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor for error handling
apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as AxiosRequestConfig & { _retry?: boolean };
    
    // Handle 401 Unauthorized - attempt token refresh
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      
      try {
        const newToken = await refreshAccessToken();
        if (newToken && originalRequest.headers) {
          originalRequest.headers.Authorization = `Bearer ${newToken}`;
          return apiClient(originalRequest);
        }
      } catch (refreshError) {
        // Refresh failed, logout user
        useAuthStore.getState().logout();
        window.location.href = '/login';
      }
    }
    
    // Transform error to ApiError
    throw transformError(error);
  }
);
```

### 3.2 Token Refresh Logic

```typescript
// lib/api/tokenRefresh.ts
import { apiClient } from './client';
import { useAuthStore } from '@/stores/authStore';

let isRefreshing = false;
let refreshSubscribers: ((token: string) => void)[] = [];

function subscribeTokenRefresh(callback: (token: string) => void) {
  refreshSubscribers.push(callback);
}

function onTokenRefreshed(token: string) {
  refreshSubscribers.forEach((callback) => callback(token));
  refreshSubscribers = [];
}

export async function refreshAccessToken(): Promise<string | null> {
  if (isRefreshing) {
    // Wait for ongoing refresh
    return new Promise((resolve) => {
      subscribeTokenRefresh((token) => resolve(token));
    });
  }
  
  isRefreshing = true;
  
  try {
    const response = await axios.post(
      `${import.meta.env.VITE_API_BASE_URL}/auth/refresh`,
      {},
      { withCredentials: true }
    );
    
    const { accessToken, user } = response.data.data;
    
    // Update store
    useAuthStore.getState().setAuth(user, accessToken);
    
    // Notify waiting requests
    onTokenRefreshed(accessToken);
    
    return accessToken;
  } catch (error) {
    refreshSubscribers = [];
    return null;
  } finally {
    isRefreshing = false;
  }
}
```

---

## 4. Error Handling

### 4.1 API Error Class

```typescript
// lib/api/errors.ts

export class ApiError extends Error {
  public readonly status: number;
  public readonly code: string;
  public readonly details?: Record<string, string[]>;
  public readonly traceId?: string;
  
  constructor(
    message: string,
    status: number,
    code: string,
    details?: Record<string, string[]>,
    traceId?: string
  ) {
    super(message);
    this.name = 'ApiError';
    this.status = status;
    this.code = code;
    this.details = details;
    this.traceId = traceId;
  }
  
  get isNetworkError(): boolean {
    return this.status === 0;
  }
  
  get isClientError(): boolean {
    return this.status >= 400 && this.status < 500;
  }
  
  get isServerError(): boolean {
    return this.status >= 500;
  }
  
  get isUnauthorized(): boolean {
    return this.status === 401;
  }
  
  get isForbidden(): boolean {
    return this.status === 403;
  }
  
  get isNotFound(): boolean {
    return this.status === 404;
  }
  
  get isValidationError(): boolean {
    return this.status === 422;
  }
  
  get isRateLimited(): boolean {
    return this.status === 429;
  }
}

// Error transformation
export function transformError(error: AxiosError): ApiError {
  if (error.response) {
    const data = error.response.data as any;
    return new ApiError(
      data?.error?.message || error.message,
      error.response.status,
      data?.error?.code || 'UNKNOWN_ERROR',
      data?.error?.details,
      data?.error?.traceId
    );
  }
  
  if (error.request) {
    return new ApiError(
      'Network error. Please check your connection.',
      0,
      'NETWORK_ERROR'
    );
  }
  
  return new ApiError(
    error.message || 'An unexpected error occurred',
    0,
    'UNKNOWN_ERROR'
  );
}
```

### 4.2 Error Code Mapping

```typescript
// lib/api/errorMessages.ts

export const ERROR_MESSAGES: Record<string, string> = {
  // Authentication
  'AUTH_INVALID_CREDENTIALS': 'Invalid email or password',
  'AUTH_TOKEN_EXPIRED': 'Your session has expired. Please log in again.',
  'AUTH_EMAIL_NOT_VERIFIED': 'Please verify your email address',
  'AUTH_ACCOUNT_SUSPENDED': 'Your account has been suspended',
  'AUTH_ACCOUNT_BANNED': 'Your account has been banned',
  
  // Users
  'USER_NOT_FOUND': 'User not found',
  'USER_ALREADY_EXISTS': 'An account with this email already exists',
  'USERNAME_TAKEN': 'This username is already taken',
  
  // Stories
  'STORY_NOT_FOUND': 'Story not found',
  'CHAPTER_NOT_FOUND': 'Chapter not found',
  'CHAPTER_LOCKED': 'This chapter is locked. Unlock it to continue reading.',
  
  // Payments
  'INSUFFICIENT_BALANCE': 'Insufficient balance. Please top up your wallet.',
  'PAYMENT_FAILED': 'Payment processing failed. Please try again.',
  'ALREADY_UNLOCKED': 'You have already unlocked this chapter',
  
  // Validation
  'VALIDATION_ERROR': 'Please check your input and try again',
  
  // Rate Limiting
  'RATE_LIMITED': 'Too many requests. Please try again later.',
  
  // Generic
  'NETWORK_ERROR': 'Network error. Please check your connection.',
  'SERVER_ERROR': 'Server error. Please try again later.',
  'UNKNOWN_ERROR': 'An unexpected error occurred',
};

export function getErrorMessage(code: string, fallback?: string): string {
  return ERROR_MESSAGES[code] || fallback || ERROR_MESSAGES['UNKNOWN_ERROR'];
}
```

---

## 5. API Service Layer

### 5.1 Base Service Class

```typescript
// lib/api/baseService.ts
import { apiClient } from './client';
import type { AxiosRequestConfig } from 'axios';

interface ApiResponse<T> {
  data: T;
  meta?: {
    pagination?: {
      page: number;
      size: number;
      totalItems: number;
      totalPages: number;
    };
    message?: string;
  };
}

export abstract class BaseService {
  protected basePath: string;
  
  constructor(basePath: string) {
    this.basePath = basePath;
  }
  
  protected async get<T>(
    path: string,
    config?: AxiosRequestConfig
  ): Promise<T> {
    const response = await apiClient.get<ApiResponse<T>>(
      `${this.basePath}${path}`,
      config
    );
    return response.data.data;
  }
  
  protected async getWithMeta<T>(
    path: string,
    config?: AxiosRequestConfig
  ): Promise<ApiResponse<T>> {
    const response = await apiClient.get<ApiResponse<T>>(
      `${this.basePath}${path}`,
      config
    );
    return response.data;
  }
  
  protected async post<T, D = unknown>(
    path: string,
    data?: D,
    config?: AxiosRequestConfig
  ): Promise<T> {
    const response = await apiClient.post<ApiResponse<T>>(
      `${this.basePath}${path}`,
      data,
      config
    );
    return response.data.data;
  }
  
  protected async put<T, D = unknown>(
    path: string,
    data?: D,
    config?: AxiosRequestConfig
  ): Promise<T> {
    const response = await apiClient.put<ApiResponse<T>>(
      `${this.basePath}${path}`,
      data,
      config
    );
    return response.data.data;
  }
  
  protected async patch<T, D = unknown>(
    path: string,
    data?: D,
    config?: AxiosRequestConfig
  ): Promise<T> {
    const response = await apiClient.patch<ApiResponse<T>>(
      `${this.basePath}${path}`,
      data,
      config
    );
    return response.data.data;
  }
  
  protected async delete<T = void>(
    path: string,
    config?: AxiosRequestConfig
  ): Promise<T> {
    const response = await apiClient.delete<ApiResponse<T>>(
      `${this.basePath}${path}`,
      config
    );
    return response.data.data;
  }
}
```

### 5.2 Domain Services

#### 5.2.1 Authentication Service

```typescript
// features/auth/api/authService.ts
import { BaseService } from '@/lib/api/baseService';
import type { User, LoginCredentials, RegisterData, AuthResponse } from '../types';

class AuthService extends BaseService {
  constructor() {
    super('/auth');
  }
  
  async login(credentials: LoginCredentials): Promise<AuthResponse> {
    return this.post<AuthResponse>('/login', {
      ...credentials,
      deviceId: this.getDeviceId(),
      deviceName: this.getDeviceName(),
    });
  }
  
  async register(data: RegisterData): Promise<User> {
    return this.post<User>('/register', data);
  }
  
  async logout(): Promise<void> {
    return this.post('/logout');
  }
  
  async refreshToken(): Promise<AuthResponse> {
    return this.post<AuthResponse>('/refresh');
  }
  
  async forgotPassword(email: string): Promise<void> {
    return this.post('/forgot-password', { email });
  }
  
  async resetPassword(token: string, password: string): Promise<void> {
    return this.post('/reset-password', { token, password });
  }
  
  async verifyEmail(token: string): Promise<void> {
    return this.post('/verify-email', { token });
  }
  
  async resendVerification(email: string): Promise<void> {
    return this.post('/resend-verification', { email });
  }
  
  private getDeviceId(): string {
    let deviceId = localStorage.getItem('n9-device-id');
    if (!deviceId) {
      deviceId = crypto.randomUUID();
      localStorage.setItem('n9-device-id', deviceId);
    }
    return deviceId;
  }
  
  private getDeviceName(): string {
    return navigator.userAgent;
  }
}

export const authService = new AuthService();
```

#### 5.2.2 Stories Service

```typescript
// features/stories/api/storyService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  Story, 
  StoryDetail, 
  StoryFilters, 
  CreateStoryData,
  UpdateStoryData,
  PaginatedResponse 
} from '../types';

class StoryService extends BaseService {
  constructor() {
    super('/stories');
  }
  
  async list(filters: StoryFilters): Promise<PaginatedResponse<Story>> {
    const params = this.buildQueryParams(filters);
    return this.getWithMeta<Story[]>('', { params });
  }
  
  async getBySlug(slug: string): Promise<StoryDetail> {
    return this.get<StoryDetail>(`/slug/${slug}`);
  }
  
  async getById(id: string): Promise<StoryDetail> {
    return this.get<StoryDetail>(`/${id}`);
  }
  
  async create(data: CreateStoryData): Promise<Story> {
    return this.post<Story>('', data);
  }
  
  async update(id: string, data: UpdateStoryData): Promise<Story> {
    return this.put<Story>(`/${id}`, data);
  }
  
  async delete(id: string): Promise<void> {
    return this.delete(`/${id}`);
  }
  
  async publish(id: string): Promise<Story> {
    return this.post<Story>(`/${id}/publish`);
  }
  
  async unpublish(id: string): Promise<Story> {
    return this.post<Story>(`/${id}/unpublish`);
  }
  
  async getTrending(): Promise<Story[]> {
    return this.get<Story[]>('/trending');
  }
  
  async getFeatured(): Promise<Story[]> {
    return this.get<Story[]>('/featured');
  }
  
  async getRecommended(): Promise<Story[]> {
    return this.get<Story[]>('/recommended');
  }
  
  async getNewReleases(): Promise<Story[]> {
    return this.get<Story[]>('/new-releases');
  }
  
  async getSimilar(storyId: string): Promise<Story[]> {
    return this.get<Story[]>(`/${storyId}/similar`);
  }
  
  async getStats(storyId: string): Promise<StoryStats> {
    return this.get<StoryStats>(`/${storyId}/stats`);
  }
  
  private buildQueryParams(filters: StoryFilters): Record<string, string> {
    const params: Record<string, string> = {};
    
    if (filters.page) params.page = String(filters.page);
    if (filters.size) params.size = String(filters.size);
    if (filters.sort) params.sort = filters.sort;
    if (filters.genre) params.genre = filters.genre;
    if (filters.status) params.status = filters.status;
    if (filters.rating) params.rating = filters.rating;
    
    return params;
  }
}

export const storyService = new StoryService();
```

#### 5.2.3 Chapters Service

```typescript
// features/chapters/api/chapterService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  Chapter, 
  ChapterContent, 
  CreateChapterData,
  UpdateChapterData 
} from '../types';

class ChapterService extends BaseService {
  constructor() {
    super('/chapters');
  }
  
  async getById(id: string): Promise<Chapter> {
    return this.get<Chapter>(`/${id}`);
  }
  
  async getContent(id: string): Promise<ChapterContent> {
    return this.get<ChapterContent>(`/${id}/content`);
  }
  
  async create(storyId: string, data: CreateChapterData): Promise<Chapter> {
    return this.post<Chapter>(`/stories/${storyId}/chapters`, data);
  }
  
  async update(id: string, data: UpdateChapterData): Promise<Chapter> {
    return this.put<Chapter>(`/${id}`, data);
  }
  
  async delete(id: string): Promise<void> {
    return this.delete(`/${id}`);
  }
  
  async publish(id: string): Promise<Chapter> {
    return this.post<Chapter>(`/${id}/publish`);
  }
  
  async schedule(id: string, publishAt: string): Promise<Chapter> {
    return this.post<Chapter>(`/${id}/schedule`, { publishAt });
  }
  
  async unlock(id: string): Promise<Chapter> {
    return this.post<Chapter>(`/${id}/unlock`);
  }
  
  async reorder(storyId: string, chapterIds: string[]): Promise<void> {
    return this.put(`/stories/${storyId}/chapters/reorder`, { chapterIds });
  }
}

export const chapterService = new ChapterService();
```

#### 5.2.4 Payments Service

```typescript
// features/payments/api/paymentService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  Wallet, 
  Transaction, 
  TopUpRequest, 
  PaymentResult,
  CoinPackage 
} from '../types';

class PaymentService extends BaseService {
  constructor() {
    super('');
  }
  
  // Wallet
  async getWallet(): Promise<Wallet> {
    return this.get<Wallet>('/me/wallet');
  }
  
  async getTransactions(filters?: TransactionFilters): Promise<PaginatedResponse<Transaction>> {
    return this.getWithMeta<Transaction[]>('/me/wallet/transactions', { 
      params: filters 
    });
  }
  
  // Top-up
  async getPackages(): Promise<CoinPackage[]> {
    return this.get<CoinPackage[]>('/wallet/packages');
  }
  
  async createTopUp(request: TopUpRequest): Promise<PaymentResult> {
    return this.post<PaymentResult>('/wallet/topup', request);
  }
  
  async confirmTopUp(paymentId: string): Promise<PaymentResult> {
    return this.post<PaymentResult>('/wallet/topup/confirm', { paymentId });
  }
  
  // Donations
  async sendDonation(authorId: string, amount: number, message?: string): Promise<void> {
    return this.post('/donations', { authorId, amount, message });
  }
  
  async getSentDonations(): Promise<Donation[]> {
    return this.get<Donation[]>('/me/donations/sent');
  }
  
  async getReceivedDonations(): Promise<Donation[]> {
    return this.get<Donation[]>('/me/donations/received');
  }
}

export const paymentService = new PaymentService();
```

---

## 6. Type Definitions

### 6.1 API Response Types

```typescript
// types/api.ts

// Standard API response wrapper
export interface ApiResponse<T> {
  data: T;
  meta?: ApiMeta;
}

export interface ApiMeta {
  pagination?: PaginationMeta;
  message?: string;
  timestamp?: string;
}

export interface PaginationMeta {
  page: number;
  size: number;
  totalItems: number;
  totalPages: number;
  hasNext: boolean;
  hasPrevious: boolean;
}

export interface PaginatedResponse<T> {
  data: T[];
  meta: {
    pagination: PaginationMeta;
  };
}

// Error response
export interface ApiErrorResponse {
  error: {
    code: string;
    message: string;
    details?: Record<string, string[]>;
    traceId?: string;
    timestamp?: string;
  };
}
```

### 6.2 Domain Types

```typescript
// features/stories/types/index.ts

export interface Story {
  id: string;
  title: string;
  slug: string;
  description: string;
  coverUrl: string;
  status: StoryStatus;
  chapterCount: number;
  wordCount: number;
  author: AuthorSummary;
  categories: Category[];
  tags: Tag[];
  stats: StoryStats;
  createdAt: string;
  updatedAt: string;
}

export interface StoryDetail extends Story {
  chapters: ChapterSummary[];
  isInLibrary: boolean;
  isLiked: boolean;
  userRating?: number;
  readingProgress?: ReadingProgress;
}

export type StoryStatus = 'DRAFT' | 'PUBLISHED' | 'COMPLETED' | 'HIATUS';

export interface StoryStats {
  views: number;
  likes: number;
  rating: number;
  ratingCount: number;
  bookmarks: number;
  comments: number;
}

export interface StoryFilters {
  page?: number;
  size?: number;
  sort?: string;
  genre?: string;
  status?: string;
  rating?: string;
  search?: string;
}
```

---

## 7. React Query Integration

### 7.1 Query Hooks Pattern

```typescript
// features/stories/hooks/useStoryQueries.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { queryKeys } from '@/lib/queryKeys';
import { storyService } from '../api/storyService';
import { interactionService } from '@/features/interactions/api/interactionService';
import type { StoryFilters, CreateStoryData, UpdateStoryData } from '../types';

// ═══════════════════════════════════════════════════════════════════════════
// QUERY HOOKS
// ═══════════════════════════════════════════════════════════════════════════

export function useStory(slug: string) {
  return useQuery({
    queryKey: queryKeys.stories.detail(slug),
    queryFn: () => storyService.getBySlug(slug),
    enabled: !!slug,
    staleTime: 10 * 60 * 1000, // 10 minutes
  });
}

export function useStories(filters: StoryFilters) {
  return useQuery({
    queryKey: queryKeys.stories.list(filters),
    queryFn: () => storyService.list(filters),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}

export function useTrendingStories() {
  return useQuery({
    queryKey: queryKeys.stories.trending(),
    queryFn: () => storyService.getTrending(),
    staleTime: 5 * 60 * 1000,
  });
}

export function useFeaturedStories() {
  return useQuery({
    queryKey: queryKeys.stories.featured(),
    queryFn: () => storyService.getFeatured(),
    staleTime: 15 * 60 * 1000, // 15 minutes - featured changes less often
  });
}

export function useRecommendedStories() {
  return useQuery({
    queryKey: queryKeys.stories.recommended(),
    queryFn: () => storyService.getRecommended(),
    staleTime: 5 * 60 * 1000,
  });
}

// ═══════════════════════════════════════════════════════════════════════════
// MUTATION HOOKS
// ═══════════════════════════════════════════════════════════════════════════

export function useCreateStory() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (data: CreateStoryData) => storyService.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({
        queryKey: queryKeys.author.stories(),
      });
    },
  });
}

export function useUpdateStory() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: UpdateStoryData }) => 
      storyService.update(id, data),
    onSuccess: (story) => {
      queryClient.invalidateQueries({
        queryKey: queryKeys.stories.detail(story.slug),
      });
      queryClient.invalidateQueries({
        queryKey: queryKeys.author.stories(),
      });
    },
  });
}

export function useLikeStory(slug: string) {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: () => interactionService.likeStory(slug),
    onMutate: async () => {
      await queryClient.cancelQueries({
        queryKey: queryKeys.stories.detail(slug),
      });
      
      const previousStory = queryClient.getQueryData(
        queryKeys.stories.detail(slug)
      );
      
      queryClient.setQueryData(
        queryKeys.stories.detail(slug),
        (old: any) => ({
          ...old,
          isLiked: !old.isLiked,
          stats: {
            ...old.stats,
            likes: old.isLiked ? old.stats.likes - 1 : old.stats.likes + 1,
          },
        })
      );
      
      return { previousStory };
    },
    onError: (err, vars, context) => {
      if (context?.previousStory) {
        queryClient.setQueryData(
          queryKeys.stories.detail(slug),
          context.previousStory
        );
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({
        queryKey: queryKeys.stories.detail(slug),
      });
    },
  });
}
```

### 7.2 Infinite Query Pattern

```typescript
// features/stories/hooks/useInfiniteStories.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { queryKeys } from '@/lib/queryKeys';
import { storyService } from '../api/storyService';
import type { StoryFilters } from '../types';

export function useInfiniteStories(filters: Omit<StoryFilters, 'page'>) {
  return useInfiniteQuery({
    queryKey: queryKeys.stories.list(filters),
    queryFn: ({ pageParam = 1 }) => 
      storyService.list({ ...filters, page: pageParam }),
    initialPageParam: 1,
    getNextPageParam: (lastPage) => {
      const { pagination } = lastPage.meta;
      return pagination.hasNext ? pagination.page + 1 : undefined;
    },
    getPreviousPageParam: (firstPage) => {
      const { pagination } = firstPage.meta;
      return pagination.hasPrevious ? pagination.page - 1 : undefined;
    },
    staleTime: 5 * 60 * 1000,
  });
}

// Helper hook to flatten pages
export function useStoryList(filters: Omit<StoryFilters, 'page'>) {
  const query = useInfiniteStories(filters);
  
  const stories = useMemo(() => {
    return query.data?.pages.flatMap((page) => page.data) ?? [];
  }, [query.data]);
  
  const totalCount = query.data?.pages[0]?.meta.pagination.totalItems ?? 0;
  
  return {
    ...query,
    stories,
    totalCount,
  };
}
```

---

## 8. Real-time Integration

### 8.1 WebSocket Client

```typescript
// lib/websocket/client.ts
import { io, Socket } from 'socket.io-client';
import { useAuthStore } from '@/stores/authStore';

const WS_URL = import.meta.env.VITE_WS_URL || 'ws://localhost:8080';

class WebSocketClient {
  private socket: Socket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  
  connect(): void {
    const token = useAuthStore.getState().accessToken;
    
    this.socket = io(WS_URL, {
      auth: { token },
      transports: ['websocket'],
      reconnection: true,
      reconnectionAttempts: this.maxReconnectAttempts,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
    });
    
    this.setupEventHandlers();
  }
  
  private setupEventHandlers(): void {
    if (!this.socket) return;
    
    this.socket.on('connect', () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;
    });
    
    this.socket.on('disconnect', (reason) => {
      console.log('WebSocket disconnected:', reason);
    });
    
    this.socket.on('connect_error', (error) => {
      console.error('WebSocket connection error:', error);
      this.reconnectAttempts++;
    });
  }
  
  subscribe(event: string, callback: (data: any) => void): () => void {
    this.socket?.on(event, callback);
    return () => this.socket?.off(event, callback);
  }
  
  emit(event: string, data?: any): void {
    this.socket?.emit(event, data);
  }
  
  disconnect(): void {
    this.socket?.disconnect();
    this.socket = null;
  }
}

export const wsClient = new WebSocketClient();
```

### 8.2 Real-time Hooks

```typescript
// hooks/useRealtime.ts
import { useEffect } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { wsClient } from '@/lib/websocket/client';
import { queryKeys } from '@/lib/queryKeys';

// Hook for real-time notifications
export function useRealtimeNotifications() {
  const queryClient = useQueryClient();
  
  useEffect(() => {
    const unsubscribe = wsClient.subscribe('notification', (notification) => {
      // Add to notification list
      queryClient.setQueryData(
        queryKeys.notifications.list(),
        (old: any) => ({
          ...old,
          data: [notification, ...(old?.data || [])],
        })
      );
      
      // Update unread count
      queryClient.setQueryData(
        queryKeys.notifications.unreadCount(),
        (old: number) => (old ?? 0) + 1
      );
    });
    
    return unsubscribe;
  }, [queryClient]);
}

// Hook for real-time chapter comments
export function useRealtimeComments(chapterId: string) {
  const queryClient = useQueryClient();
  
  useEffect(() => {
    // Join chapter room
    wsClient.emit('join:chapter', { chapterId });
    
    const unsubscribe = wsClient.subscribe('comment:new', (comment) => {
      queryClient.setQueryData(
        queryKeys.chapters.comments(chapterId),
        (old: any) => ({
          ...old,
          data: [comment, ...(old?.data || [])],
        })
      );
    });
    
    return () => {
      wsClient.emit('leave:chapter', { chapterId });
      unsubscribe();
    };
  }, [chapterId, queryClient]);
}
```

---

## 9. API Request Patterns

### 9.1 Request/Response Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         REQUEST/RESPONSE FLOW                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Component                                                                   │
│      │                                                                       │
│      │ 1. Call query hook                                                   │
│      ▼                                                                       │
│  ┌─────────────┐                                                            │
│  │ React Query │                                                            │
│  │   Cache     │───────────────────────────────────────┐                    │
│  └─────┬───────┘                                       │                    │
│        │                                               │                    │
│        │ 2. Cache miss/stale                           │ 3. Cache hit       │
│        ▼                                               │                    │
│  ┌─────────────┐                                       │                    │
│  │ API Service │                                       │                    │
│  └─────┬───────┘                                       │                    │
│        │                                               │                    │
│        │ 4. Call API client                            │                    │
│        ▼                                               │                    │
│  ┌─────────────┐                                       │                    │
│  │   Axios     │                                       │                    │
│  │ Interceptors│                                       │                    │
│  └─────┬───────┘                                       │                    │
│        │                                               │                    │
│        │ 5. HTTP request                               │                    │
│        ▼                                               │                    │
│  ┌─────────────┐                                       │                    │
│  │   Server    │                                       │                    │
│  └─────┬───────┘                                       │                    │
│        │                                               │                    │
│        │ 6. HTTP response                              │                    │
│        ▼                                               │                    │
│  ┌─────────────┐                                       │                    │
│  │  Response   │                                       │                    │
│  │Interceptors │                                       │                    │
│  └─────┬───────┘                                       │                    │
│        │                                               │                    │
│        │ 7. Transform response                         │                    │
│        ▼                                               │                    │
│  ┌─────────────┐                                       │                    │
│  │ Update Cache│◀──────────────────────────────────────┘                    │
│  └─────┬───────┘                                                            │
│        │                                                                     │
│        │ 8. Return data                                                     │
│        ▼                                                                     │
│  Component Re-render                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Frontend Architecture Team | Initial release |

---

## 11. Related Documents

- [04_STATE_MANAGEMENT.md](./04_STATE_MANAGEMENT.md) - State management with React Query
- [05_ROUTING_NAVIGATION.md](./05_ROUTING_NAVIGATION.md) - Route definitions
- [10_SECURITY_COMPLIANCE.md](./10_SECURITY_COMPLIANCE.md) - Security patterns
