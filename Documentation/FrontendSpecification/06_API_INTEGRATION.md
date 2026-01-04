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

---

## 10. Additional Domain Services

### 10.1 User Service

```typescript
// features/users/api/userService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  User, 
  UserProfile, 
  UserPreferences, 
  UpdateProfileData,
  AuthorApplication,
  PaginatedResponse 
} from '../types';

class UserService extends BaseService {
  constructor() {
    super('');
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // PROFILE APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getCurrentUser(): Promise<User> {
    return this.get<User>('/me');
  }
  
  async updateCurrentUser(data: UpdateProfileData): Promise<User> {
    return this.put<User>('/me', data);
  }
  
  async deleteAccount(): Promise<void> {
    return this.delete('/me');
  }
  
  async changePassword(currentPassword: string, newPassword: string): Promise<void> {
    return this.put('/me/password', { currentPassword, newPassword });
  }
  
  async updateAvatar(file: File): Promise<{ avatarUrl: string }> {
    const formData = new FormData();
    formData.append('avatar', file);
    return this.put('/me/avatar', formData, {
      headers: { 'Content-Type': 'multipart/form-data' }
    });
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // PUBLIC PROFILE APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getProfile(userId: string): Promise<UserProfile> {
    return this.get<UserProfile>(`/users/${userId}`);
  }
  
  async getUserStories(userId: string, filters?: StoryFilters): Promise<PaginatedResponse<Story>> {
    return this.getWithMeta<Story[]>(`/users/${userId}/stories`, { params: filters });
  }
  
  async getFollowers(userId: string, page?: number): Promise<PaginatedResponse<User>> {
    return this.getWithMeta<User[]>(`/users/${userId}/followers`, { params: { page } });
  }
  
  async getFollowing(userId: string, page?: number): Promise<PaginatedResponse<User>> {
    return this.getWithMeta<User[]>(`/users/${userId}/following`, { params: { page } });
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // PREFERENCES APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getPreferences(): Promise<UserPreferences> {
    return this.get<UserPreferences>('/me/preferences');
  }
  
  async updatePreferences(data: Partial<UserPreferences>): Promise<UserPreferences> {
    return this.put<UserPreferences>('/me/preferences', data);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // SOCIAL APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async followUser(userId: string): Promise<void> {
    return this.post(`/users/${userId}/follow`);
  }
  
  async unfollowUser(userId: string): Promise<void> {
    return this.delete(`/users/${userId}/follow`);
  }
  
  async blockUser(userId: string): Promise<void> {
    return this.post(`/users/${userId}/block`);
  }
  
  async unblockUser(userId: string): Promise<void> {
    return this.delete(`/users/${userId}/block`);
  }
  
  async getBlockedUsers(): Promise<User[]> {
    return this.get<User[]>('/me/blocked');
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // AUTHOR APPLICATION APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async applyForAuthor(data: AuthorApplication): Promise<void> {
    return this.post('/me/author-application', data);
  }
  
  async getAuthorApplicationStatus(): Promise<{ status: string; submittedAt?: string }> {
    return this.get('/me/author-application');
  }
}

export const userService = new UserService();
```

### 10.2 Interaction Service

```typescript
// features/interactions/api/interactionService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  Follow,
  Like,
  Review,
  ReviewVote,
  Comment,
  Report,
  PaginatedResponse 
} from '../types';

class InteractionService extends BaseService {
  constructor() {
    super('');
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // FOLLOWS APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async followStory(storyId: string): Promise<void> {
    return this.post(`/stories/${storyId}/follow`);
  }
  
  async unfollowStory(storyId: string): Promise<void> {
    return this.delete(`/stories/${storyId}/follow`);
  }
  
  async followAuthor(authorId: string): Promise<void> {
    return this.post(`/users/${authorId}/follow`);
  }
  
  async unfollowAuthor(authorId: string): Promise<void> {
    return this.delete(`/users/${authorId}/follow`);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // LIKES APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async likeStory(storyId: string): Promise<void> {
    return this.post(`/stories/${storyId}/like`);
  }
  
  async unlikeStory(storyId: string): Promise<void> {
    return this.delete(`/stories/${storyId}/like`);
  }
  
  async likeChapter(chapterId: string): Promise<void> {
    return this.post(`/chapters/${chapterId}/like`);
  }
  
  async unlikeChapter(chapterId: string): Promise<void> {
    return this.delete(`/chapters/${chapterId}/like`);
  }
  
  async likeComment(commentId: string): Promise<void> {
    return this.post(`/comments/${commentId}/like`);
  }
  
  async unlikeComment(commentId: string): Promise<void> {
    return this.delete(`/comments/${commentId}/like`);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // BOOKMARKS APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getBookmarks(): Promise<PaginatedResponse<Story>> {
    return this.getWithMeta<Story[]>('/me/bookmarks');
  }
  
  async bookmarkStory(storyId: string): Promise<void> {
    return this.post(`/stories/${storyId}/bookmark`);
  }
  
  async removeBookmark(storyId: string): Promise<void> {
    return this.delete(`/stories/${storyId}/bookmark`);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // REVIEWS APIs (Backend: 04_INTERACTIONS_COMPONENT)
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getStoryReviews(storyId: string, filters?: ReviewFilters): Promise<PaginatedResponse<Review>> {
    return this.getWithMeta<Review[]>(`/stories/${storyId}/reviews`, { params: filters });
  }
  
  async createReview(storyId: string, data: CreateReviewData): Promise<Review> {
    return this.post<Review>(`/stories/${storyId}/reviews`, data);
  }
  
  async updateReview(reviewId: string, data: UpdateReviewData): Promise<Review> {
    return this.put<Review>(`/reviews/${reviewId}`, data);
  }
  
  async deleteReview(reviewId: string): Promise<void> {
    return this.delete(`/reviews/${reviewId}`);
  }
  
  async voteReviewHelpful(reviewId: string, isHelpful: boolean): Promise<void> {
    return this.post(`/reviews/${reviewId}/vote`, { isHelpful });
  }
  
  async removeReviewVote(reviewId: string): Promise<void> {
    return this.delete(`/reviews/${reviewId}/vote`);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // COMMENTS APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getChapterComments(chapterId: string, filters?: CommentFilters): Promise<PaginatedResponse<Comment>> {
    return this.getWithMeta<Comment[]>(`/chapters/${chapterId}/comments`, { params: filters });
  }
  
  async getComment(commentId: string): Promise<Comment> {
    return this.get<Comment>(`/comments/${commentId}`);
  }
  
  async createComment(data: CreateCommentData): Promise<Comment> {
    return this.post<Comment>('/comments', data);
  }
  
  async updateComment(commentId: string, content: string): Promise<Comment> {
    return this.put<Comment>(`/comments/${commentId}`, { content });
  }
  
  async deleteComment(commentId: string): Promise<void> {
    return this.delete(`/comments/${commentId}`);
  }
  
  async replyToComment(commentId: string, content: string): Promise<Comment> {
    return this.post<Comment>(`/comments/${commentId}/reply`, { content });
  }
  
  async getCommentReplies(commentId: string): Promise<Comment[]> {
    return this.get<Comment[]>(`/comments/${commentId}/replies`);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // REPORT APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async reportStory(storyId: string, data: ReportData): Promise<void> {
    return this.post(`/stories/${storyId}/report`, data);
  }
  
  async reportChapter(chapterId: string, data: ReportData): Promise<void> {
    return this.post(`/chapters/${chapterId}/report`, data);
  }
  
  async reportComment(commentId: string, data: ReportData): Promise<void> {
    return this.post(`/comments/${commentId}/report`, data);
  }
}

export const interactionService = new InteractionService();
```

### 10.3 Reading List Service

```typescript
// features/library/api/readingListService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  ReadingList, 
  ReadingListDetail,
  CreateReadingListData,
  UpdateReadingListData 
} from '../types';

class ReadingListService extends BaseService {
  constructor() {
    super('');
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // READING LIST CRUD (Backend: 04_INTERACTIONS_COMPONENT - ReadingList entity)
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getMyReadingLists(): Promise<ReadingList[]> {
    return this.get<ReadingList[]>('/me/reading-lists');
  }
  
  async getReadingList(listId: string): Promise<ReadingListDetail> {
    return this.get<ReadingListDetail>(`/reading-lists/${listId}`);
  }
  
  async createReadingList(data: CreateReadingListData): Promise<ReadingList> {
    return this.post<ReadingList>('/me/reading-lists', data);
  }
  
  async updateReadingList(listId: string, data: UpdateReadingListData): Promise<ReadingList> {
    return this.put<ReadingList>(`/reading-lists/${listId}`, data);
  }
  
  async deleteReadingList(listId: string): Promise<void> {
    return this.delete(`/reading-lists/${listId}`);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // READING LIST ITEMS
  // ═══════════════════════════════════════════════════════════════════════════
  
  async addStoryToList(listId: string, storyId: string): Promise<void> {
    return this.post(`/reading-lists/${listId}/stories`, { storyId });
  }
  
  async removeStoryFromList(listId: string, storyId: string): Promise<void> {
    return this.delete(`/reading-lists/${listId}/stories/${storyId}`);
  }
  
  async reorderListItems(listId: string, storyIds: string[]): Promise<void> {
    return this.put(`/reading-lists/${listId}/reorder`, { storyIds });
  }
}

export const readingListService = new ReadingListService();
```

### 10.4 Reading Progress Service

```typescript
// features/readings/api/readingProgressService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  ReadingProgress, 
  ReadingHistory,
  ReadingStreak,
  ReadingGoal,
  ChapterBookmark 
} from '../types';

class ReadingProgressService extends BaseService {
  constructor() {
    super('');
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // PROGRESS APIs (Backend: 05_READINGS_COMPONENT)
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getProgress(storyId: string): Promise<ReadingProgress> {
    return this.get<ReadingProgress>(`/me/reading-progress/${storyId}`);
  }
  
  async updateProgress(storyId: string, data: UpdateProgressData): Promise<ReadingProgress> {
    return this.put<ReadingProgress>(`/me/reading-progress/${storyId}`, data);
  }
  
  async markChapterRead(chapterId: string): Promise<void> {
    return this.post(`/chapters/${chapterId}/read`);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // HISTORY APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getReadingHistory(page?: number): Promise<PaginatedResponse<ReadingHistory>> {
    return this.getWithMeta<ReadingHistory[]>('/me/reading-history', { params: { page } });
  }
  
  async clearHistory(): Promise<void> {
    return this.delete('/me/reading-history');
  }
  
  async removeFromHistory(storyId: string): Promise<void> {
    return this.delete(`/me/reading-history/${storyId}`);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // STREAK & GOALS APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getStreak(): Promise<ReadingStreak> {
    return this.get<ReadingStreak>('/me/reading-streak');
  }
  
  async getGoals(): Promise<ReadingGoal> {
    return this.get<ReadingGoal>('/me/reading-goals');
  }
  
  async updateGoals(data: UpdateGoalData): Promise<ReadingGoal> {
    return this.put<ReadingGoal>('/me/reading-goals', data);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // BOOKMARKS APIs (In-chapter bookmarks)
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getChapterBookmarks(chapterId: string): Promise<ChapterBookmark[]> {
    return this.get<ChapterBookmark[]>(`/chapters/${chapterId}/bookmarks`);
  }
  
  async createChapterBookmark(chapterId: string, position: number, note?: string): Promise<ChapterBookmark> {
    return this.post<ChapterBookmark>(`/chapters/${chapterId}/bookmarks`, { position, note });
  }
  
  async deleteChapterBookmark(bookmarkId: string): Promise<void> {
    return this.delete(`/bookmarks/${bookmarkId}`);
  }
}

export const readingProgressService = new ReadingProgressService();
```

### 10.5 Notification Service

```typescript
// features/notifications/api/notificationService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  Notification, 
  NotificationPreferences,
  PaginatedResponse 
} from '../types';

class NotificationService extends BaseService {
  constructor() {
    super('');
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // NOTIFICATIONS APIs (Backend: 07_NOTIFICATIONS_COMPONENT)
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getNotifications(page?: number): Promise<PaginatedResponse<Notification>> {
    return this.getWithMeta<Notification[]>('/me/notifications', { params: { page } });
  }
  
  async getUnreadCount(): Promise<{ count: number }> {
    return this.get<{ count: number }>('/me/notifications/unread-count');
  }
  
  async markAsRead(notificationId: string): Promise<void> {
    return this.put(`/me/notifications/${notificationId}/read`);
  }
  
  async markAllAsRead(): Promise<void> {
    return this.put('/me/notifications/read-all');
  }
  
  async deleteNotification(notificationId: string): Promise<void> {
    return this.delete(`/me/notifications/${notificationId}`);
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // NOTIFICATION SETTINGS APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getSettings(): Promise<NotificationPreferences> {
    return this.get<NotificationPreferences>('/me/notification-settings');
  }
  
  async updateSettings(data: Partial<NotificationPreferences>): Promise<NotificationPreferences> {
    return this.put<NotificationPreferences>('/me/notification-settings', data);
  }
}

export const notificationService = new NotificationService();
```

### 10.6 Search Service

```typescript
// features/search/api/searchService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  SearchResult, 
  SearchSuggestion,
  TrendingSearch,
  SearchFilters 
} from '../types';

class SearchService extends BaseService {
  constructor() {
    super('/search');
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // SEARCH APIs (Backend: 13_API_CATALOG - Search section)
  // ═══════════════════════════════════════════════════════════════════════════
  
  async globalSearch(query: string, filters?: SearchFilters): Promise<SearchResult> {
    return this.get<SearchResult>('', { params: { q: query, ...filters } });
  }
  
  async searchStories(query: string, filters?: SearchFilters): Promise<PaginatedResponse<Story>> {
    return this.getWithMeta<Story[]>('/stories', { params: { q: query, ...filters } });
  }
  
  async searchAuthors(query: string, page?: number): Promise<PaginatedResponse<User>> {
    return this.getWithMeta<User[]>('/authors', { params: { q: query, page } });
  }
  
  async searchTags(query: string): Promise<Tag[]> {
    return this.get<Tag[]>('/tags', { params: { q: query } });
  }
  
  async getSuggestions(query: string): Promise<SearchSuggestion[]> {
    return this.get<SearchSuggestion[]>('/suggestions', { params: { q: query } });
  }
  
  async getTrending(): Promise<TrendingSearch[]> {
    return this.get<TrendingSearch[]>('/trending');
  }
}

export const searchService = new SearchService();
```

### 10.7 Author Service

```typescript
// features/author/api/authorService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  AuthorDashboard,
  AuthorStory,
  EarningsSummary,
  PayoutRequest,
  PayoutHistory,
  StoryAnalytics 
} from '../types';

class AuthorService extends BaseService {
  constructor() {
    super('/author');
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // DASHBOARD APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getDashboard(): Promise<AuthorDashboard> {
    return this.get<AuthorDashboard>('/dashboard');
  }
  
  async getMyStories(filters?: AuthorStoryFilters): Promise<PaginatedResponse<AuthorStory>> {
    return this.getWithMeta<AuthorStory[]>('/stories', { params: filters });
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // ANALYTICS APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getAnalytics(period?: string): Promise<AuthorAnalytics> {
    return this.get<AuthorAnalytics>('/analytics', { params: { period } });
  }
  
  async getStoryAnalytics(storyId: string, period?: string): Promise<StoryAnalytics> {
    return this.get<StoryAnalytics>(`/stories/${storyId}/analytics`, { params: { period } });
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // EARNINGS & PAYOUTS APIs (Backend: 03_PAYMENTS_COMPONENT)
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getEarnings(): Promise<EarningsSummary> {
    return this.get<EarningsSummary>('/earnings');
  }
  
  async requestPayout(amount: number, payoutMethod: string): Promise<PayoutRequest> {
    return this.post<PayoutRequest>('/payout', { amount, payoutMethod });
  }
  
  async getPayoutHistory(): Promise<PayoutHistory[]> {
    return this.get<PayoutHistory[]>('/payouts');
  }
}

export const authorService = new AuthorService();
```

### 10.8 Admin Service

```typescript
// features/admin/api/adminService.ts
import { BaseService } from '@/lib/api/baseService';
import type { 
  AdminDashboard,
  Report,
  AdminUser,
  AuditLog,
  PendingPayout,
  ModeratorAction 
} from '../types';

class AdminService extends BaseService {
  constructor() {
    super('/admin');
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // DASHBOARD APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getDashboard(): Promise<AdminDashboard> {
    return this.get<AdminDashboard>('/dashboard');
  }
  
  async getAnalytics(period?: string): Promise<AdminAnalytics> {
    return this.get<AdminAnalytics>('/analytics', { params: { period } });
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // MODERATION APIs (Backend: 13_API_CATALOG - Admin section)
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getReports(filters?: ReportFilters): Promise<PaginatedResponse<Report>> {
    return this.getWithMeta<Report[]>('/reports', { params: filters });
  }
  
  async getReport(reportId: string): Promise<Report> {
    return this.get<Report>(`/reports/${reportId}`);
  }
  
  async resolveReport(reportId: string, resolution: ReportResolution): Promise<void> {
    return this.post(`/reports/${reportId}/resolve`, resolution);
  }
  
  async hideContent(contentType: string, contentId: string, reason: string): Promise<void> {
    return this.post(`/content/${contentType}/${contentId}/hide`, { reason });
  }
  
  async deleteContent(contentType: string, contentId: string, reason: string): Promise<void> {
    return this.delete(`/content/${contentType}/${contentId}`, { data: { reason } });
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // USER MANAGEMENT APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getUsers(filters?: UserFilters): Promise<PaginatedResponse<AdminUser>> {
    return this.getWithMeta<AdminUser[]>('/users', { params: filters });
  }
  
  async warnUser(userId: string, reason: string): Promise<void> {
    return this.post(`/users/${userId}/warn`, { reason });
  }
  
  async suspendUser(userId: string, duration: string, reason: string): Promise<void> {
    return this.post(`/users/${userId}/suspend`, { duration, reason });
  }
  
  async banUser(userId: string, reason: string): Promise<void> {
    return this.post(`/users/${userId}/ban`, { reason });
  }
  
  async updateUserRoles(userId: string, roles: string[]): Promise<void> {
    return this.put(`/users/${userId}/roles`, { roles });
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // FINANCIAL APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getPendingPayouts(): Promise<PendingPayout[]> {
    return this.get<PendingPayout[]>('/payouts/pending');
  }
  
  async approvePayout(payoutId: string): Promise<void> {
    return this.post(`/payouts/${payoutId}/approve`);
  }
  
  async rejectPayout(payoutId: string, reason: string): Promise<void> {
    return this.post(`/payouts/${payoutId}/reject`, { reason });
  }
  
  async processRefund(transactionId: string, reason: string): Promise<void> {
    return this.post('/refunds', { transactionId, reason });
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // SYSTEM MANAGEMENT APIs
  // ═══════════════════════════════════════════════════════════════════════════
  
  async getCategories(): Promise<Category[]> {
    return this.get<Category[]>('/categories');
  }
  
  async createCategory(data: CreateCategoryData): Promise<Category> {
    return this.post<Category>('/categories', data);
  }
  
  async updateCategory(categoryId: string, data: UpdateCategoryData): Promise<Category> {
    return this.put<Category>(`/categories/${categoryId}`, data);
  }
  
  async getAuditLogs(filters?: AuditLogFilters): Promise<PaginatedResponse<AuditLog>> {
    return this.getWithMeta<AuditLog[]>('/audit-logs', { params: filters });
  }
}

export const adminService = new AdminService();
```

---

## 11. Extended Type Definitions

### 11.1 User Types

```typescript
// features/users/types/index.ts

export interface UserPreferences {
  theme: 'light' | 'dark' | 'system';
  language: string;
  fontSize: 'small' | 'medium' | 'large';
  lineHeight: 'compact' | 'normal' | 'relaxed';
  fontFamily: string;
  readingMode: 'scroll' | 'paginated';
  autoBookmark: boolean;
  showMatureContent: boolean;
}

export interface AuthorApplication {
  penName: string;
  bio: string;
  genres: string[];
  experience: string;
  sampleWork?: string;
}
```

### 11.2 Interaction Types

```typescript
// features/interactions/types/index.ts

export interface Review {
  id: string;
  storyId: string;
  userId: string;
  user: UserSummary;
  rating: number;
  title?: string;
  content: string;
  hasSpoilers: boolean;
  helpfulCount: number;
  notHelpfulCount: number;
  userVote?: 'helpful' | 'not_helpful';
  status: 'VISIBLE' | 'HIDDEN' | 'DELETED';
  createdAt: string;
  updatedAt: string;
}

export interface CreateReviewData {
  rating: number;
  title?: string;
  content: string;
  hasSpoilers?: boolean;
}

export interface UpdateReviewData {
  rating?: number;
  title?: string;
  content?: string;
  hasSpoilers?: boolean;
}

export interface ReviewFilters {
  page?: number;
  size?: number;
  sort?: 'newest' | 'oldest' | 'helpful' | 'rating_high' | 'rating_low';
  rating?: number;
}

export interface Comment {
  id: string;
  chapterId: string;
  userId: string;
  user: UserSummary;
  parentId?: string;
  content: string;
  likeCount: number;
  isLiked: boolean;
  replyCount: number;
  depth: number; // max 3 levels (Backend: 04_INTERACTIONS_COMPONENT)
  status: 'VISIBLE' | 'HIDDEN' | 'DELETED';
  createdAt: string;
  updatedAt: string;
}

export interface CreateCommentData {
  chapterId: string;
  content: string;
  parentId?: string;
}

export interface CommentFilters {
  page?: number;
  size?: number;
  sort?: 'newest' | 'oldest' | 'popular';
}

export interface ReportData {
  reason: 'SPAM' | 'HARASSMENT' | 'HATE_SPEECH' | 'VIOLENCE' | 'COPYRIGHT' | 'OTHER';
  description?: string;
}
```

### 11.3 Reading List Types

```typescript
// features/library/types/index.ts

export interface ReadingList {
  id: string;
  name: string;
  description?: string;
  isPublic: boolean;
  isDefault: boolean; // Backend: 04_INTERACTIONS_COMPONENT - is_default
  storyCount: number;
  coverUrl?: string;
  createdAt: string;
  updatedAt: string;
}

export interface ReadingListDetail extends ReadingList {
  stories: Story[];
}

export interface CreateReadingListData {
  name: string;
  description?: string;
  isPublic?: boolean;
}

export interface UpdateReadingListData {
  name?: string;
  description?: string;
  isPublic?: boolean;
}
```

### 11.4 Reading Progress Types

```typescript
// features/readings/types/index.ts

export interface ReadingProgress {
  storyId: string;
  currentChapterId: string;
  chapterIndex: number;
  scrollPosition: number;
  completedChapters: number;
  totalChapters: number;
  completionPercent: number;
  lastReadAt: string;
}

export interface UpdateProgressData {
  currentChapterId: string;
  scrollPosition?: number;
}

export interface ReadingHistory {
  storyId: string;
  story: StorySummary;
  currentChapterIndex: number;
  completionPercent: number;
  lastReadAt: string;
}

export interface ReadingStreak {
  currentStreak: number;
  longestStreak: number;
  lastReadDate: string;
  streakStartedAt: string;
}

export interface ReadingGoal {
  dailyMinutesGoal: number;
  dailyChaptersGoal: number;
  weeklyChaptersGoal: number;
  isActive: boolean;
}

export interface ChapterBookmark {
  id: string;
  chapterId: string;
  position: number;
  note?: string;
  createdAt: string;
}
```

### 11.5 Notification Types

```typescript
// features/notifications/types/index.ts

export interface Notification {
  id: string;
  type: NotificationType;
  category: NotificationCategory;
  title: string;
  body: string;
  data: Record<string, any>;
  isRead: boolean;
  readAt?: string;
  actorId?: string;
  actor?: UserSummary;
  targetType?: string;
  targetId?: string;
  createdAt: string;
}

export type NotificationType = 
  | 'NEW_CHAPTER'
  | 'NEW_FOLLOWER'
  | 'STORY_LIKE'
  | 'STORY_REVIEW'
  | 'COMMENT_REPLY'
  | 'COMMENT_LIKE'
  | 'DONATION_RECEIVED'
  | 'PAYOUT_PROCESSED'
  | 'STREAK_MILESTONE'
  | 'SYSTEM_ANNOUNCEMENT';

export type NotificationCategory = 
  | 'SOCIAL'
  | 'STORY_UPDATES'
  | 'PAYMENTS'
  | 'SYSTEM';

export interface NotificationPreferences {
  emailEnabled: boolean;
  pushEnabled: boolean;
  inAppEnabled: boolean;
  categorySettings: Record<NotificationCategory, boolean>;
  quietHoursStart?: string; // HH:mm format
  quietHoursEnd?: string;
  quietHoursTimezone?: string;
  digestFrequency: 'NONE' | 'DAILY' | 'WEEKLY';
}
```

### 11.6 Payment Types (Extended)

```typescript
// features/payments/types/index.ts

export interface Wallet {
  id: string;
  coinBalance: number;
  earningsBalance: number; // For authors (Backend: 03_PAYMENTS_COMPONENT)
  heldAmount: number;
  currency: string;
  updatedAt: string;
}

export interface Transaction {
  id: string;
  type: TransactionType;
  amount: number;
  balance: number;
  description: string;
  referenceType?: string;
  referenceId?: string;
  status: 'PENDING' | 'COMPLETED' | 'FAILED' | 'REFUNDED';
  createdAt: string;
}

export type TransactionType = 
  | 'TOPUP'
  | 'CHAPTER_UNLOCK'
  | 'DONATION_SENT'
  | 'DONATION_RECEIVED'
  | 'SUBSCRIPTION'
  | 'PAYOUT'
  | 'REFUND';

export interface TransactionFilters {
  page?: number;
  size?: number;
  type?: TransactionType;
  startDate?: string;
  endDate?: string;
}

export interface CoinPackage {
  id: string;
  name: string;
  coins: number;
  price: number;
  currency: string;
  bonusCoins: number;
  discountPercent: number;
  isPopular: boolean;
}

export interface TopUpRequest {
  packageId: string;
  paymentMethod: 'STRIPE' | 'PAYPAL' | 'MOMO';
  discountCode?: string;
}

export interface Donation {
  id: string;
  donorId: string;
  donor?: UserSummary;
  recipientId: string;
  recipient?: UserSummary;
  storyId?: string;
  story?: StorySummary;
  amount: number;
  message?: string;
  isAnonymous: boolean;
  createdAt: string;
}

export interface EarningsSummary {
  totalEarnings: number;
  availableBalance: number;
  pendingPayouts: number;
  thisMonthEarnings: number;
  lastMonthEarnings: number;
  earningsBySource: {
    chapterUnlocks: number;
    donations: number;
    subscriptions: number;
  };
}

export interface PayoutRequest {
  id: string;
  amount: number;
  payoutMethod: string;
  status: 'PENDING' | 'APPROVED' | 'PROCESSING' | 'COMPLETED' | 'REJECTED';
  requestedAt: string;
  processedAt?: string;
  rejectionReason?: string;
}
```

---

## 12. WebSocket Event Types (Backend: 06_REALTIME_AND_EVENTS)

```typescript
// lib/websocket/types.ts

export type WebSocketEvent = 
  | NotificationEvent
  | CommentEvent
  | ChapterEvent
  | WalletEvent
  | StreakEvent;

export interface NotificationEvent {
  type: 'NOTIFICATION';
  payload: Notification;
}

export interface CommentEvent {
  type: 'NEW_COMMENT' | 'COMMENT_DELETED';
  payload: {
    chapterId: string;
    comment?: Comment;
    commentId?: string;
  };
}

export interface ChapterEvent {
  type: 'CHAPTER_PUBLISHED' | 'CHAPTER_UPDATED';
  payload: {
    storyId: string;
    chapterId: string;
    chapterIndex: number;
    title: string;
  };
}

export interface WalletEvent {
  type: 'WALLET_CREDITED' | 'WALLET_DEBITED';
  payload: {
    amount: number;
    newBalance: number;
    transactionType: TransactionType;
  };
}

export interface StreakEvent {
  type: 'STREAK_MILESTONE' | 'STREAK_AT_RISK';
  payload: {
    currentStreak: number;
    milestone?: number;
  };
}

// WebSocket channel subscription
export interface ChannelSubscription {
  channel: 'user' | 'story' | 'chapter';
  id: string;
}
```

---

## 13. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Frontend Architecture Team | Initial release |
| 2.0 | 2026-01-05 | Frontend Architecture Team | Added complete service layer aligned with Backend API Catalog |

---

## 14. Related Documents

- [04_STATE_MANAGEMENT.md](./04_STATE_MANAGEMENT.md) - State management with React Query
- [05_ROUTING_NAVIGATION.md](./05_ROUTING_NAVIGATION.md) - Route definitions
- [10_SECURITY_COMPLIANCE.md](./10_SECURITY_COMPLIANCE.md) - Security patterns
- [Backend 13_API_CATALOG.md](../../Backend/N9/Documentation/Specification/13_API_CATALOG.md) - Backend API Reference
- [Backend 03_PAYMENTS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/03_PAYMENTS_COMPONENT.md) - Payment System Design
- [Backend 04_INTERACTIONS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/04_INTERACTIONS_COMPONENT.md) - Interactions Design
- [Backend 05_READINGS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/05_READINGS_COMPONENT.md) - Reading Progress Design
- [Backend 07_NOTIFICATIONS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/07_NOTIFICATIONS_COMPONENT.md) - Notifications Design
- [Backend 06_REALTIME_AND_EVENTS.md](../../Backend/N9/Documentation/Specification/06_REALTIME_AND_EVENTS.md) - Event System
