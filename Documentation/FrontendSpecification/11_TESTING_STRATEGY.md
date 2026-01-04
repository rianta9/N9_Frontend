# Testing Strategy Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | QA Engineering Team |
| Review Cycle | Quarterly |
| Related Documents | 02_FRONTEND_ARCHITECTURE.md, 07_COMPONENTS_LIBRARY.md |

---

## 2. Testing Philosophy

### 2.1 Testing Pyramid

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          TESTING PYRAMID                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              /\                                              │
│                             /  \        E2E Tests                            │
│                            / 10%\       (Playwright)                         │
│                           /──────\      Critical user flows                  │
│                          /        \                                          │
│                         /          \    Integration Tests                    │
│                        /    20%     \   (React Testing Library)              │
│                       /──────────────\  Component + hooks integration        │
│                      /                \                                      │
│                     /                  \  Unit Tests                         │
│                    /        70%         \ (Vitest)                           │
│                   /──────────────────────\ Utils, hooks, pure functions      │
│                  /                        \                                  │
│                 /══════════════════════════\                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Coverage Targets

| Test Type | Coverage Target | Description |
|-----------|-----------------|-------------|
| Unit Tests | ≥ 80% | Pure functions, utilities, hooks |
| Integration Tests | ≥ 70% | Components, feature interactions |
| E2E Tests | Critical paths | Login, purchase, reading flow |
| **Overall** | **≥ 75%** | Combined coverage |

---

## 3. Testing Stack

### 3.1 Tools Overview

| Tool | Purpose | Configuration |
|------|---------|---------------|
| **Vitest** | Unit & integration tests | Fast, Vite-native |
| **React Testing Library** | Component testing | User-centric approach |
| **Playwright** | E2E testing | Cross-browser support |
| **MSW** | API mocking | Service worker based |
| **Faker.js** | Test data generation | Realistic data |

### 3.2 Configuration Files

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    include: ['**/*.{test,spec}.{ts,tsx}'],
    exclude: ['node_modules', 'dist', 'e2e'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html', 'lcov'],
      exclude: [
        'node_modules/',
        'src/test/',
        '**/*.d.ts',
        '**/*.config.{ts,js}',
        '**/types/**',
      ],
      thresholds: {
        global: {
          branches: 75,
          functions: 80,
          lines: 80,
          statements: 80,
        },
      },
    },
  },
  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
    },
  },
});
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['junit', { outputFile: 'test-results/junit.xml' }],
  ],
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'mobile-safari',
      use: { ...devices['iPhone 12'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## 4. Unit Testing

### 4.1 Test Setup

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom';
import { cleanup } from '@testing-library/react';
import { afterEach, vi } from 'vitest';
import { server } from './mocks/server';

// MSW server setup
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterAll(() => server.close());
afterEach(() => {
  server.resetHandlers();
  cleanup();
});

// Mock window.matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
});

// Mock IntersectionObserver
class MockIntersectionObserver {
  observe = vi.fn();
  disconnect = vi.fn();
  unobserve = vi.fn();
}
Object.defineProperty(window, 'IntersectionObserver', {
  writable: true,
  value: MockIntersectionObserver,
});

// Mock ResizeObserver
class MockResizeObserver {
  observe = vi.fn();
  disconnect = vi.fn();
  unobserve = vi.fn();
}
Object.defineProperty(window, 'ResizeObserver', {
  writable: true,
  value: MockResizeObserver,
});
```

### 4.2 Utility Testing

```typescript
// lib/utils/__tests__/formatters.test.ts
import { describe, it, expect } from 'vitest';
import { 
  formatNumber, 
  formatDate, 
  formatCurrency,
  truncateText 
} from '../formatters';

describe('formatters', () => {
  describe('formatNumber', () => {
    it('formats numbers with K suffix for thousands', () => {
      expect(formatNumber(1000)).toBe('1K');
      expect(formatNumber(1500)).toBe('1.5K');
      expect(formatNumber(999999)).toBe('999.9K');
    });
    
    it('formats numbers with M suffix for millions', () => {
      expect(formatNumber(1000000)).toBe('1M');
      expect(formatNumber(2500000)).toBe('2.5M');
    });
    
    it('returns number as-is for values under 1000', () => {
      expect(formatNumber(0)).toBe('0');
      expect(formatNumber(999)).toBe('999');
    });
  });
  
  describe('formatDate', () => {
    it('formats relative dates for recent items', () => {
      const now = new Date();
      const hourAgo = new Date(now.getTime() - 60 * 60 * 1000);
      
      expect(formatDate(hourAgo)).toBe('1 hour ago');
    });
    
    it('formats absolute dates for older items', () => {
      const oldDate = new Date('2024-01-15');
      expect(formatDate(oldDate)).toMatch(/Jan 15, 2024/);
    });
  });
  
  describe('formatCurrency', () => {
    it('formats currency with correct symbol', () => {
      expect(formatCurrency(1000, 'USD')).toBe('$1,000.00');
      expect(formatCurrency(1000, 'EUR')).toBe('€1,000.00');
    });
    
    it('handles coin currency', () => {
      expect(formatCurrency(500, 'COIN')).toBe('500 coins');
    });
  });
  
  describe('truncateText', () => {
    it('truncates text exceeding max length', () => {
      const text = 'This is a very long text that needs truncating';
      expect(truncateText(text, 20)).toBe('This is a very long...');
    });
    
    it('preserves short text', () => {
      expect(truncateText('Short', 20)).toBe('Short');
    });
    
    it('handles empty string', () => {
      expect(truncateText('', 20)).toBe('');
    });
  });
});
```

### 4.3 Hook Testing

```typescript
// hooks/__tests__/useDebounce.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { renderHook, act } from '@testing-library/react';
import { useDebounce } from '../useDebounce';

describe('useDebounce', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });
  
  afterEach(() => {
    vi.useRealTimers();
  });
  
  it('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('initial', 500));
    
    expect(result.current).toBe('initial');
  });
  
  it('debounces value updates', () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: 'initial' } }
    );
    
    // Update value
    rerender({ value: 'updated' });
    
    // Value should not change immediately
    expect(result.current).toBe('initial');
    
    // Fast-forward time
    act(() => {
      vi.advanceTimersByTime(500);
    });
    
    // Value should now be updated
    expect(result.current).toBe('updated');
  });
  
  it('cancels pending updates on unmount', () => {
    const { result, rerender, unmount } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: 'initial' } }
    );
    
    rerender({ value: 'updated' });
    unmount();
    
    // Should not throw
    act(() => {
      vi.advanceTimersByTime(500);
    });
  });
});
```

---

## 5. Component Testing

### 5.1 Test Utilities

```typescript
// src/test/utils.tsx
import { ReactElement, ReactNode } from 'react';
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';
import { ThemeProvider } from '@/components/theme/ThemeProvider';

// Create test query client
function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: 0,
      },
    },
  });
}

// All providers wrapper
interface AllProvidersProps {
  children: ReactNode;
}

function AllProviders({ children }: AllProvidersProps) {
  const queryClient = createTestQueryClient();
  
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <ThemeProvider defaultTheme="light">
          {children}
        </ThemeProvider>
      </BrowserRouter>
    </QueryClientProvider>
  );
}

// Custom render
function customRender(
  ui: ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) {
  return render(ui, { wrapper: AllProviders, ...options });
}

// Re-export everything
export * from '@testing-library/react';
export { customRender as render };

// User event re-export
export { default as userEvent } from '@testing-library/user-event';
```

### 5.2 Component Test Examples

```typescript
// components/story/StoryCard/__tests__/StoryCard.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, userEvent } from '@/test/utils';
import { StoryCard } from '../StoryCard';
import { createMockStory } from '@/test/factories/story';

describe('StoryCard', () => {
  const mockStory = createMockStory();
  
  it('renders story information correctly', () => {
    render(<StoryCard story={mockStory} />);
    
    expect(screen.getByText(mockStory.title)).toBeInTheDocument();
    expect(screen.getByText(mockStory.author.displayName)).toBeInTheDocument();
    expect(screen.getByRole('img', { name: /cover/i })).toHaveAttribute(
      'src',
      expect.stringContaining(mockStory.coverImage)
    );
  });
  
  it('shows genre badges', () => {
    render(<StoryCard story={mockStory} />);
    
    mockStory.genres.forEach(genre => {
      expect(screen.getByText(genre.name)).toBeInTheDocument();
    });
  });
  
  it('displays stats correctly', () => {
    render(<StoryCard story={mockStory} />);
    
    // Views, likes should be formatted
    expect(screen.getByLabelText(/views/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/likes/i)).toBeInTheDocument();
  });
  
  it('handles click navigation', async () => {
    const user = userEvent.setup();
    render(<StoryCard story={mockStory} />);
    
    const card = screen.getByRole('article');
    await user.click(card);
    
    // Should navigate to story detail
    expect(window.location.pathname).toBe(`/story/${mockStory.slug}`);
  });
  
  it('shows bookmark button for authenticated users', () => {
    // Mock authenticated state
    render(<StoryCard story={mockStory} />, {
      initialAuthState: { isAuthenticated: true },
    });
    
    expect(screen.getByRole('button', { name: /bookmark/i })).toBeInTheDocument();
  });
  
  it('handles bookmark toggle', async () => {
    const user = userEvent.setup();
    const onBookmark = vi.fn();
    
    render(
      <StoryCard story={mockStory} onBookmark={onBookmark} />,
      { initialAuthState: { isAuthenticated: true } }
    );
    
    const bookmarkButton = screen.getByRole('button', { name: /bookmark/i });
    await user.click(bookmarkButton);
    
    expect(onBookmark).toHaveBeenCalledWith(mockStory.id);
  });
});
```

### 5.3 Form Testing

```typescript
// components/auth/LoginForm/__tests__/LoginForm.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, userEvent, waitFor } from '@/test/utils';
import { LoginForm } from '../LoginForm';

describe('LoginForm', () => {
  const mockOnSubmit = vi.fn();
  
  beforeEach(() => {
    mockOnSubmit.mockClear();
  });
  
  it('renders form fields', () => {
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/password/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /sign in/i })).toBeInTheDocument();
  });
  
  it('validates required fields', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    // Submit without filling fields
    await user.click(screen.getByRole('button', { name: /sign in/i }));
    
    await waitFor(() => {
      expect(screen.getByText(/email is required/i)).toBeInTheDocument();
      expect(screen.getByText(/password is required/i)).toBeInTheDocument();
    });
    
    expect(mockOnSubmit).not.toHaveBeenCalled();
  });
  
  it('validates email format', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    await user.type(screen.getByLabelText(/email/i), 'invalid-email');
    await user.click(screen.getByRole('button', { name: /sign in/i }));
    
    await waitFor(() => {
      expect(screen.getByText(/invalid email/i)).toBeInTheDocument();
    });
  });
  
  it('submits form with valid data', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));
    
    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'password123',
      });
    });
  });
  
  it('shows loading state during submission', async () => {
    const user = userEvent.setup();
    mockOnSubmit.mockImplementation(() => new Promise(() => {})); // Never resolves
    
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));
    
    await waitFor(() => {
      expect(screen.getByRole('button', { name: /signing in/i })).toBeDisabled();
    });
  });
  
  it('displays error message on failure', async () => {
    const user = userEvent.setup();
    mockOnSubmit.mockRejectedValue(new Error('Invalid credentials'));
    
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'wrong-password');
    await user.click(screen.getByRole('button', { name: /sign in/i }));
    
    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent(/invalid credentials/i);
    });
  });
});
```

---

## 6. Integration Testing

### 6.1 API Mocking with MSW

```typescript
// src/test/mocks/handlers.ts
import { http, HttpResponse, delay } from 'msw';
import { createMockStory, createMockUser } from '@/test/factories';

const API_BASE = import.meta.env.VITE_API_URL;

export const handlers = [
  // Auth handlers
  http.post(`${API_BASE}/auth/login`, async ({ request }) => {
    const body = await request.json();
    
    if (body.email === 'test@example.com' && body.password === 'password123') {
      return HttpResponse.json({
        user: createMockUser(),
        accessToken: 'mock-access-token',
        refreshToken: 'mock-refresh-token',
        expiresIn: 3600,
      });
    }
    
    return HttpResponse.json(
      { message: 'Invalid credentials' },
      { status: 401 }
    );
  }),
  
  http.post(`${API_BASE}/auth/refresh`, async () => {
    return HttpResponse.json({
      accessToken: 'new-mock-access-token',
      expiresIn: 3600,
    });
  }),
  
  // Story handlers
  http.get(`${API_BASE}/stories`, async ({ request }) => {
    const url = new URL(request.url);
    const page = parseInt(url.searchParams.get('page') || '1');
    const size = parseInt(url.searchParams.get('size') || '10');
    
    const stories = Array.from({ length: size }, () => createMockStory());
    
    await delay(100); // Simulate network latency
    
    return HttpResponse.json({
      content: stories,
      totalElements: 100,
      totalPages: Math.ceil(100 / size),
      number: page - 1,
      size,
    });
  }),
  
  http.get(`${API_BASE}/stories/:slug`, async ({ params }) => {
    return HttpResponse.json(
      createMockStory({ slug: params.slug as string })
    );
  }),
  
  // User handlers
  http.get(`${API_BASE}/users/me`, async () => {
    return HttpResponse.json(createMockUser());
  }),
  
  http.patch(`${API_BASE}/users/me`, async ({ request }) => {
    const updates = await request.json();
    return HttpResponse.json(createMockUser(updates));
  }),
];
```

### 6.2 Test Factories

```typescript
// src/test/factories/story.ts
import { faker } from '@faker-js/faker';
import { Story, Genre, Author } from '@/types';

export function createMockGenre(overrides: Partial<Genre> = {}): Genre {
  return {
    id: faker.string.uuid(),
    name: faker.helpers.arrayElement([
      'Fantasy', 'Romance', 'Sci-Fi', 'Mystery', 'Horror'
    ]),
    slug: faker.helpers.slugify(faker.word.noun()),
    ...overrides,
  };
}

export function createMockAuthor(overrides: Partial<Author> = {}): Author {
  return {
    id: faker.string.uuid(),
    username: faker.internet.userName(),
    displayName: faker.person.fullName(),
    avatarUrl: faker.image.avatar(),
    bio: faker.lorem.paragraph(),
    followerCount: faker.number.int({ min: 0, max: 100000 }),
    storyCount: faker.number.int({ min: 1, max: 50 }),
    ...overrides,
  };
}

export function createMockStory(overrides: Partial<Story> = {}): Story {
  const title = overrides.title || faker.lorem.words(3);
  
  return {
    id: faker.string.uuid(),
    title,
    slug: faker.helpers.slugify(title.toLowerCase()),
    synopsis: faker.lorem.paragraphs(2),
    coverImage: faker.image.url(),
    status: faker.helpers.arrayElement(['ONGOING', 'COMPLETED', 'HIATUS']),
    contentRating: faker.helpers.arrayElement(['EVERYONE', 'TEEN', 'MATURE']),
    author: createMockAuthor(),
    genres: Array.from({ length: 2 }, () => createMockGenre()),
    stats: {
      views: faker.number.int({ min: 0, max: 1000000 }),
      likes: faker.number.int({ min: 0, max: 100000 }),
      comments: faker.number.int({ min: 0, max: 10000 }),
      bookmarks: faker.number.int({ min: 0, max: 50000 }),
      rating: faker.number.float({ min: 3, max: 5, fractionDigits: 1 }),
      ratingCount: faker.number.int({ min: 0, max: 10000 }),
    },
    chapterCount: faker.number.int({ min: 1, max: 200 }),
    createdAt: faker.date.past().toISOString(),
    updatedAt: faker.date.recent().toISOString(),
    ...overrides,
  };
}
```

### 6.3 Feature Integration Test

```typescript
// features/stories/__tests__/StoryBrowse.integration.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, userEvent, waitFor, within } from '@/test/utils';
import { server } from '@/test/mocks/server';
import { http, HttpResponse } from 'msw';
import { BrowsePage } from '@/pages/browse/BrowsePage';

describe('Story Browse Feature', () => {
  it('loads and displays stories', async () => {
    render(<BrowsePage />);
    
    // Should show loading state
    expect(screen.getByTestId('story-grid-skeleton')).toBeInTheDocument();
    
    // Wait for stories to load
    await waitFor(() => {
      expect(screen.queryByTestId('story-grid-skeleton')).not.toBeInTheDocument();
    });
    
    // Should display story cards
    const storyCards = screen.getAllByRole('article');
    expect(storyCards.length).toBeGreaterThan(0);
  });
  
  it('filters by genre', async () => {
    const user = userEvent.setup();
    render(<BrowsePage />);
    
    // Wait for initial load
    await waitFor(() => {
      expect(screen.getAllByRole('article').length).toBeGreaterThan(0);
    });
    
    // Open genre filter
    await user.click(screen.getByRole('button', { name: /genre/i }));
    
    // Select a genre
    await user.click(screen.getByRole('option', { name: /fantasy/i }));
    
    // Should update URL
    await waitFor(() => {
      expect(window.location.search).toContain('genre=fantasy');
    });
  });
  
  it('handles search', async () => {
    const user = userEvent.setup();
    render(<BrowsePage />);
    
    // Wait for initial load
    await waitFor(() => {
      expect(screen.getAllByRole('article').length).toBeGreaterThan(0);
    });
    
    // Type in search
    const searchInput = screen.getByPlaceholderText(/search stories/i);
    await user.type(searchInput, 'dragon');
    
    // Wait for debounced search
    await waitFor(() => {
      expect(window.location.search).toContain('q=dragon');
    }, { timeout: 1000 });
  });
  
  it('handles pagination', async () => {
    const user = userEvent.setup();
    render(<BrowsePage />);
    
    // Wait for initial load
    await waitFor(() => {
      expect(screen.getAllByRole('article').length).toBeGreaterThan(0);
    });
    
    // Click next page
    const nextButton = screen.getByRole('button', { name: /next/i });
    await user.click(nextButton);
    
    // Should update page
    await waitFor(() => {
      expect(window.location.search).toContain('page=2');
    });
  });
  
  it('handles API errors gracefully', async () => {
    // Override handler to return error
    server.use(
      http.get('*/stories', () => {
        return HttpResponse.json(
          { message: 'Server error' },
          { status: 500 }
        );
      })
    );
    
    render(<BrowsePage />);
    
    // Should show error state
    await waitFor(() => {
      expect(screen.getByText(/something went wrong/i)).toBeInTheDocument();
    });
    
    // Should have retry button
    expect(screen.getByRole('button', { name: /retry/i })).toBeInTheDocument();
  });
  
  it('handles empty results', async () => {
    server.use(
      http.get('*/stories', () => {
        return HttpResponse.json({
          content: [],
          totalElements: 0,
          totalPages: 0,
          number: 0,
          size: 10,
        });
      })
    );
    
    render(<BrowsePage />);
    
    await waitFor(() => {
      expect(screen.getByText(/no stories found/i)).toBeInTheDocument();
    });
  });
});
```

---

## 7. E2E Testing

### 7.1 Page Objects

```typescript
// e2e/pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;
  readonly googleLoginButton: Locator;
  
  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel(/email/i);
    this.passwordInput = page.getByLabel(/password/i);
    this.submitButton = page.getByRole('button', { name: /sign in/i });
    this.errorMessage = page.getByRole('alert');
    this.googleLoginButton = page.getByRole('button', { name: /google/i });
  }
  
  async goto() {
    await this.page.goto('/login');
  }
  
  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
  
  async expectError(message: string | RegExp) {
    await this.errorMessage.waitFor();
    await expect(this.errorMessage).toContainText(message);
  }
}
```

### 7.2 E2E Test Examples

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';

test.describe('Authentication', () => {
  test('successful login flow', async ({ page }) => {
    const loginPage = new LoginPage(page);
    
    await loginPage.goto();
    await loginPage.login('test@example.com', 'password123');
    
    // Should redirect to home
    await expect(page).toHaveURL('/');
    
    // Should show user menu
    await expect(page.getByTestId('user-menu')).toBeVisible();
  });
  
  test('failed login shows error', async ({ page }) => {
    const loginPage = new LoginPage(page);
    
    await loginPage.goto();
    await loginPage.login('test@example.com', 'wrong-password');
    
    await loginPage.expectError(/invalid credentials/i);
    
    // Should stay on login page
    await expect(page).toHaveURL('/login');
  });
  
  test('logout clears session', async ({ page }) => {
    // Start logged in
    await page.goto('/');
    
    // Open user menu
    await page.getByTestId('user-menu').click();
    
    // Click logout
    await page.getByRole('menuitem', { name: /logout/i }).click();
    
    // Should redirect to home
    await expect(page).toHaveURL('/');
    
    // Should show login button
    await expect(page.getByRole('link', { name: /sign in/i })).toBeVisible();
  });
});
```

### 7.3 Critical User Flows

```typescript
// e2e/reading-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Reading Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    await page.goto('/login');
    await page.getByLabel(/email/i).fill('reader@example.com');
    await page.getByLabel(/password/i).fill('password123');
    await page.getByRole('button', { name: /sign in/i }).click();
    await expect(page).toHaveURL('/');
  });
  
  test('browse, select, and read a story', async ({ page }) => {
    // Navigate to browse
    await page.getByRole('link', { name: /browse/i }).click();
    await expect(page).toHaveURL('/browse');
    
    // Wait for stories to load
    await page.waitForSelector('[data-testid="story-card"]');
    
    // Click first story
    await page.getByTestId('story-card').first().click();
    
    // Should be on story detail page
    await expect(page.getByTestId('story-detail')).toBeVisible();
    
    // Check story info is displayed
    await expect(page.getByRole('heading', { level: 1 })).toBeVisible();
    await expect(page.getByTestId('chapter-list')).toBeVisible();
    
    // Start reading first chapter
    await page.getByTestId('chapter-item').first().click();
    
    // Should be in reader mode
    await expect(page.getByTestId('chapter-reader')).toBeVisible();
    
    // Chapter content should be visible
    await expect(page.getByTestId('chapter-content')).toBeVisible();
  });
  
  test('reader settings persist', async ({ page }) => {
    // Go to a chapter
    await page.goto('/story/test-story/chapter/1');
    
    // Open settings
    await page.getByTestId('reader-settings-toggle').click();
    
    // Change font size
    await page.getByLabel(/font size/i).selectOption('large');
    
    // Change theme
    await page.getByRole('button', { name: /sepia/i }).click();
    
    // Verify settings applied
    const content = page.getByTestId('chapter-content');
    await expect(content).toHaveClass(/text-lg/);
    await expect(content).toHaveClass(/sepia-theme/);
    
    // Navigate to another chapter
    await page.getByRole('button', { name: /next chapter/i }).click();
    
    // Settings should persist
    await expect(content).toHaveClass(/text-lg/);
    await expect(content).toHaveClass(/sepia-theme/);
  });
});
```

### 7.4 Purchase Flow

```typescript
// e2e/purchase-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Purchase Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login with coins
    await page.goto('/login');
    // ... login steps
  });
  
  test('purchase premium chapter with coins', async ({ page }) => {
    // Navigate to premium chapter
    await page.goto('/story/test-story/chapter/5');
    
    // Should show paywall
    await expect(page.getByTestId('chapter-paywall')).toBeVisible();
    
    // Get initial balance
    const initialBalance = await page.getByTestId('coin-balance').textContent();
    
    // Click purchase button
    await page.getByRole('button', { name: /unlock chapter/i }).click();
    
    // Confirm purchase
    await page.getByRole('button', { name: /confirm/i }).click();
    
    // Should unlock chapter
    await expect(page.getByTestId('chapter-content')).toBeVisible();
    await expect(page.getByTestId('chapter-paywall')).not.toBeVisible();
    
    // Balance should be updated
    const newBalance = await page.getByTestId('coin-balance').textContent();
    expect(parseInt(newBalance!)).toBeLessThan(parseInt(initialBalance!));
  });
  
  test('redirect to top-up when insufficient coins', async ({ page }) => {
    // Set low balance for this test
    // Navigate to expensive chapter
    await page.goto('/story/test-story/chapter/5');
    
    // Click purchase
    await page.getByRole('button', { name: /unlock chapter/i }).click();
    
    // Should show insufficient coins message
    await expect(page.getByText(/insufficient coins/i)).toBeVisible();
    
    // Click top-up
    await page.getByRole('button', { name: /top up/i }).click();
    
    // Should navigate to wallet
    await expect(page).toHaveURL('/wallet/top-up');
  });
});
```

---

## 8. Test Reporting

### 8.1 CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      
      - run: pnpm install
      - run: pnpm test:unit --coverage
      
      - uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: unittests
  
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      
      - run: pnpm install
      - run: pnpm exec playwright install --with-deps
      - run: pnpm test:e2e
      
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
```

---

## 9. Test Commands

```json
{
  "scripts": {
    "test": "vitest",
    "test:unit": "vitest run",
    "test:watch": "vitest watch",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:headed": "playwright test --headed"
  }
}
```

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | QA Engineering Team | Initial release |

---

## 11. Related Documents

- [02_FRONTEND_ARCHITECTURE.md](./02_FRONTEND_ARCHITECTURE.md) - Architecture patterns
- [07_COMPONENTS_LIBRARY.md](./07_COMPONENTS_LIBRARY.md) - Component library
- [06_API_INTEGRATION.md](./06_API_INTEGRATION.md) - API integration
