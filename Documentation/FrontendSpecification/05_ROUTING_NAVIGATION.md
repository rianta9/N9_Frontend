# Routing and Navigation Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Frontend Architecture Team |
| Review Cycle | Quarterly |
| Related Documents | 02_FRONTEND_ARCHITECTURE.md, 08_PAGES_SPECIFICATION.md |

---

## 2. Route Architecture

### 2.1 Route Hierarchy Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              N9 ROUTE MAP                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ PUBLIC ROUTES (No Auth Required)                                        ││
│  │ /                          Home / Discovery                             ││
│  │ /browse                    Browse Stories                               ││
│  │ /browse/:genre             Browse by Genre                              ││
│  │ /search                    Search Results                               ││
│  │ /story/:slug               Story Detail                                 ││
│  │ /story/:slug/chapter/:num  Chapter Reader                               ││
│  │ /author/:username          Author Profile                               ││
│  │ /rankings                  Story Rankings                               ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ AUTHENTICATION ROUTES (Guest Only)                                      ││
│  │ /login                     Login Page                                   ││
│  │ /register                  Registration                                 ││
│  │ /forgot-password           Password Recovery                            ││
│  │ /reset-password/:token     Password Reset                               ││
│  │ /verify-email/:token       Email Verification                           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ PROTECTED ROUTES (Auth Required)                                        ││
│  │ /library                   User Library                                 ││
│  │ /library/history           Reading History                              ││
│  │ /library/bookmarks         Bookmarked Stories                           ││
│  │ /library/collections       Story Collections                            ││
│  │ /notifications             Notifications                                ││
│  │ /wallet                    Digital Wallet                               ││
│  │ /wallet/transactions       Transaction History                          ││
│  │ /wallet/buy-coins          Purchase Coins                               ││
│  │ /settings                  User Settings                                ││
│  │ /settings/profile          Profile Settings                             ││
│  │ /settings/preferences      Reading Preferences                          ││
│  │ /settings/security         Security Settings                            ││
│  │ /settings/notifications    Notification Settings                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ AUTHOR ROUTES (Author Role Required)                                    ││
│  │ /author-dashboard          Dashboard Overview                           ││
│  │ /author-dashboard/stories  Manage Stories                               ││
│  │ /author-dashboard/create   Create New Story                             ││
│  │ /author-dashboard/story/:id/edit       Edit Story                       ││
│  │ /author-dashboard/story/:id/chapters   Manage Chapters                  ││
│  │ /author-dashboard/story/:id/chapter/new        New Chapter              ││
│  │ /author-dashboard/story/:id/chapter/:cid/edit  Edit Chapter             ││
│  │ /author-dashboard/analytics            Analytics Overview               ││
│  │ /author-dashboard/earnings             Earnings & Payouts               ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ ADMIN ROUTES (Admin/Moderator Role Required)                            ││
│  │ /admin                     Admin Dashboard                              ││
│  │ /admin/users               User Management                              ││
│  │ /admin/stories             Story Moderation                             ││
│  │ /admin/reports             Reports Queue                                ││
│  │ /admin/analytics           Platform Analytics                           ││
│  │ /admin/settings            System Settings                              ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ STATIC/UTILITY ROUTES                                                   ││
│  │ /about                     About N9                                     ││
│  │ /help                      Help Center                                  ││
│  │ /terms                     Terms of Service                             ││
│  │ /privacy                   Privacy Policy                               ││
│  │ /contact                   Contact Us                                   ││
│  │ /404                       Not Found                                    ││
│  │ /500                       Server Error                                 ││
│  │ /maintenance               Maintenance Mode                             ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Router Configuration

### 3.1 Main Router Setup

```typescript
// router/index.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { Suspense } from 'react';

// Layouts
import { RootLayout } from '@/layouts/RootLayout';
import { MainLayout } from '@/layouts/MainLayout';
import { ReaderLayout } from '@/layouts/ReaderLayout';
import { AuthLayout } from '@/layouts/AuthLayout';
import { DashboardLayout } from '@/layouts/DashboardLayout';
import { AdminLayout } from '@/layouts/AdminLayout';

// Guards
import { AuthGuard } from '@/guards/AuthGuard';
import { GuestGuard } from '@/guards/GuestGuard';
import { RoleGuard } from '@/guards/RoleGuard';

// Loading
import { PageSkeleton } from '@/components/skeletons/PageSkeleton';

// Route Modules
import { publicRoutes } from './routes/publicRoutes';
import { authRoutes } from './routes/authRoutes';
import { protectedRoutes } from './routes/protectedRoutes';
import { authorRoutes } from './routes/authorRoutes';
import { adminRoutes } from './routes/adminRoutes';
import { staticRoutes } from './routes/staticRoutes';
import { errorRoutes } from './routes/errorRoutes';

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorBoundaryPage />,
    children: [
      // Public routes with main layout
      {
        element: <MainLayout />,
        children: publicRoutes,
      },
      
      // Auth routes with auth layout
      {
        element: (
          <GuestGuard>
            <AuthLayout />
          </GuestGuard>
        ),
        children: authRoutes,
      },
      
      // Protected routes with main layout
      {
        element: (
          <AuthGuard>
            <MainLayout />
          </AuthGuard>
        ),
        children: protectedRoutes,
      },
      
      // Author dashboard routes
      {
        element: (
          <AuthGuard>
            <RoleGuard roles={['AUTHOR']}>
              <DashboardLayout />
            </RoleGuard>
          </AuthGuard>
        ),
        children: authorRoutes,
      },
      
      // Admin routes
      {
        element: (
          <AuthGuard>
            <RoleGuard roles={['ADMIN', 'MODERATOR']}>
              <AdminLayout />
            </RoleGuard>
          </AuthGuard>
        ),
        children: adminRoutes,
      },
      
      // Reader layout (immersive)
      {
        element: <ReaderLayout />,
        children: [
          {
            path: 'story/:slug/chapter/:chapterNumber',
            lazy: () => import('@/pages/reader/ChapterReaderPage'),
          },
        ],
      },
      
      // Static pages
      {
        element: <MainLayout variant="minimal" />,
        children: staticRoutes,
      },
      
      // Error routes
      ...errorRoutes,
    ],
  },
]);

export function AppRouter() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <RouterProvider router={router} />
    </Suspense>
  );
}
```

### 3.2 Route Definitions

```typescript
// router/routes/publicRoutes.tsx
import { RouteObject } from 'react-router-dom';
import { lazy } from 'react';

// Lazy load pages
const HomePage = lazy(() => import('@/pages/home/HomePage'));
const BrowsePage = lazy(() => import('@/pages/browse/BrowsePage'));
const SearchPage = lazy(() => import('@/pages/search/SearchPage'));
const StoryDetailPage = lazy(() => import('@/pages/story/StoryDetailPage'));
const AuthorProfilePage = lazy(() => import('@/pages/author/AuthorProfilePage'));
const RankingsPage = lazy(() => import('@/pages/rankings/RankingsPage'));

export const publicRoutes: RouteObject[] = [
  {
    index: true,
    element: <HomePage />,
    handle: {
      title: 'Discover Stories',
      breadcrumb: 'Home',
    },
  },
  {
    path: 'browse',
    element: <BrowsePage />,
    handle: {
      title: 'Browse Stories',
      breadcrumb: 'Browse',
    },
  },
  {
    path: 'browse/:genre',
    element: <BrowsePage />,
    handle: {
      title: (params) => `${params.genre} Stories`,
      breadcrumb: (params) => params.genre,
    },
  },
  {
    path: 'search',
    element: <SearchPage />,
    handle: {
      title: 'Search',
      breadcrumb: 'Search',
    },
  },
  {
    path: 'story/:slug',
    element: <StoryDetailPage />,
    handle: {
      title: (params, data) => data?.story?.title ?? 'Story',
      breadcrumb: (params, data) => data?.story?.title ?? 'Story',
    },
  },
  {
    path: 'author/:username',
    element: <AuthorProfilePage />,
    handle: {
      title: (params) => `@${params.username}`,
      breadcrumb: (params) => `@${params.username}`,
    },
  },
  {
    path: 'rankings',
    element: <RankingsPage />,
    handle: {
      title: 'Rankings',
      breadcrumb: 'Rankings',
    },
  },
];
```

```typescript
// router/routes/protectedRoutes.tsx
import { RouteObject } from 'react-router-dom';
import { lazy } from 'react';

const LibraryPage = lazy(() => import('@/pages/library/LibraryPage'));
const HistoryPage = lazy(() => import('@/pages/library/HistoryPage'));
const BookmarksPage = lazy(() => import('@/pages/library/BookmarksPage'));
const CollectionsPage = lazy(() => import('@/pages/library/CollectionsPage'));
const NotificationsPage = lazy(() => import('@/pages/notifications/NotificationsPage'));
const WalletPage = lazy(() => import('@/pages/wallet/WalletPage'));
const TransactionsPage = lazy(() => import('@/pages/wallet/TransactionsPage'));
const BuyCoinsPage = lazy(() => import('@/pages/wallet/BuyCoinsPage'));
const SettingsLayout = lazy(() => import('@/pages/settings/SettingsLayout'));
const ProfileSettingsPage = lazy(() => import('@/pages/settings/ProfileSettingsPage'));
const PreferencesPage = lazy(() => import('@/pages/settings/PreferencesPage'));
const SecurityPage = lazy(() => import('@/pages/settings/SecurityPage'));

export const protectedRoutes: RouteObject[] = [
  {
    path: 'library',
    element: <LibraryPage />,
    handle: {
      title: 'My Library',
      breadcrumb: 'Library',
    },
    children: [
      {
        index: true,
        element: <LibraryPage />,
      },
      {
        path: 'history',
        element: <HistoryPage />,
        handle: {
          title: 'Reading History',
          breadcrumb: 'History',
        },
      },
      {
        path: 'bookmarks',
        element: <BookmarksPage />,
        handle: {
          title: 'Bookmarks',
          breadcrumb: 'Bookmarks',
        },
      },
      {
        path: 'collections',
        element: <CollectionsPage />,
        handle: {
          title: 'Collections',
          breadcrumb: 'Collections',
        },
      },
    ],
  },
  {
    path: 'notifications',
    element: <NotificationsPage />,
    handle: {
      title: 'Notifications',
      breadcrumb: 'Notifications',
    },
  },
  {
    path: 'wallet',
    children: [
      {
        index: true,
        element: <WalletPage />,
        handle: {
          title: 'My Wallet',
          breadcrumb: 'Wallet',
        },
      },
      {
        path: 'transactions',
        element: <TransactionsPage />,
        handle: {
          title: 'Transaction History',
          breadcrumb: 'Transactions',
        },
      },
      {
        path: 'buy-coins',
        element: <BuyCoinsPage />,
        handle: {
          title: 'Buy Coins',
          breadcrumb: 'Buy Coins',
        },
      },
    ],
  },
  {
    path: 'settings',
    element: <SettingsLayout />,
    handle: {
      title: 'Settings',
      breadcrumb: 'Settings',
    },
    children: [
      {
        index: true,
        element: <ProfileSettingsPage />,
      },
      {
        path: 'profile',
        element: <ProfileSettingsPage />,
        handle: {
          title: 'Profile Settings',
          breadcrumb: 'Profile',
        },
      },
      {
        path: 'preferences',
        element: <PreferencesPage />,
        handle: {
          title: 'Preferences',
          breadcrumb: 'Preferences',
        },
      },
      {
        path: 'security',
        element: <SecurityPage />,
        handle: {
          title: 'Security',
          breadcrumb: 'Security',
        },
      },
    ],
  },
];
```

---

## 4. Route Guards

### 4.1 Authentication Guard

```typescript
// guards/AuthGuard.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuthStore } from '@/stores/authStore';
import { PageSkeleton } from '@/components/skeletons/PageSkeleton';

interface AuthGuardProps {
  children: React.ReactNode;
  fallbackPath?: string;
}

export function AuthGuard({ children, fallbackPath = '/login' }: AuthGuardProps) {
  const { isAuthenticated, isLoading } = useAuthStore();
  const location = useLocation();
  
  // Show loading while checking auth status
  if (isLoading) {
    return <PageSkeleton />;
  }
  
  // Redirect to login if not authenticated
  if (!isAuthenticated) {
    return (
      <Navigate 
        to={fallbackPath} 
        state={{ from: location }} 
        replace 
      />
    );
  }
  
  return <>{children}</>;
}
```

### 4.2 Guest Guard

```typescript
// guards/GuestGuard.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuthStore } from '@/stores/authStore';

interface GuestGuardProps {
  children: React.ReactNode;
  fallbackPath?: string;
}

export function GuestGuard({ children, fallbackPath = '/' }: GuestGuardProps) {
  const { isAuthenticated, isLoading } = useAuthStore();
  const location = useLocation();
  
  // Get the redirect path from state or default
  const from = location.state?.from?.pathname || fallbackPath;
  
  // If authenticated, redirect away from auth pages
  if (!isLoading && isAuthenticated) {
    return <Navigate to={from} replace />;
  }
  
  return <>{children}</>;
}
```

### 4.3 Role Guard

```typescript
// guards/RoleGuard.tsx
import { Navigate } from 'react-router-dom';
import { useAuthStore } from '@/stores/authStore';

interface RoleGuardProps {
  children: React.ReactNode;
  roles: string[];
  fallbackPath?: string;
}

export function RoleGuard({ 
  children, 
  roles, 
  fallbackPath = '/' 
}: RoleGuardProps) {
  const user = useAuthStore((state) => state.user);
  
  // Check if user has any of the required roles
  const hasRequiredRole = roles.some(role => 
    user?.roles?.includes(role)
  );
  
  if (!hasRequiredRole) {
    return <Navigate to={fallbackPath} replace />;
  }
  
  return <>{children}</>;
}
```

---

## 5. Navigation Components

### 5.1 Navigation Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NAVIGATION COMPONENTS                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ HEADER NAVIGATION                                                       │ │
│  │ ┌────────────────────────────────────────────────────────────────────┐ │ │
│  │ │ [Logo]  Browse  Rankings  Search               [Wallet] [User]    │ │ │
│  │ └────────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ SIDEBAR NAVIGATION (Mobile)                                            │ │
│  │ ┌──────────────────────────┐                                          │ │
│  │ │ [User Avatar]            │                                          │ │
│  │ │ Username                 │                                          │ │
│  │ │ ─────────────────────    │                                          │ │
│  │ │ 🏠 Home                  │                                          │ │
│  │ │ 📚 Browse                │                                          │ │
│  │ │ 🔍 Search                │                                          │ │
│  │ │ 📖 My Library            │                                          │ │
│  │ │ 💰 Wallet                │                                          │ │
│  │ │ ⚙️ Settings              │                                          │ │
│  │ │ ─────────────────────    │                                          │ │
│  │ │ 🚪 Logout                │                                          │ │
│  │ └──────────────────────────┘                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ MOBILE BOTTOM TAB NAVIGATION                                           │ │
│  │ ┌────────────────────────────────────────────────────────────────────┐ │ │
│  │ │  [Home]    [Browse]   [Search]   [Library]   [Profile]            │ │ │
│  │ └────────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ DASHBOARD SIDEBAR NAVIGATION                                           │ │
│  │ ┌──────────────────────────┐                                          │ │
│  │ │ 📊 Dashboard             │                                          │ │
│  │ │ 📝 My Stories            │                                          │ │
│  │ │ ➕ Create Story          │                                          │ │
│  │ │ 📈 Analytics             │                                          │ │
│  │ │ 💵 Earnings              │                                          │ │
│  │ │ ─────────────────────    │                                          │ │
│  │ │ ← Back to Reader         │                                          │ │
│  │ └──────────────────────────┘                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Header Navigation Component

```typescript
// components/navigation/HeaderNavigation.tsx
import { Link, NavLink, useNavigate } from 'react-router-dom';
import { useAuthStore } from '@/stores/authStore';
import { cn } from '@/lib/utils';

const mainNavItems = [
  { label: 'Browse', href: '/browse' },
  { label: 'Rankings', href: '/rankings' },
];

const userNavItems = [
  { label: 'Library', href: '/library', auth: true },
  { label: 'Dashboard', href: '/author-dashboard', role: 'AUTHOR' },
];

export function HeaderNavigation() {
  const { isAuthenticated, user } = useAuthStore();
  const navigate = useNavigate();
  
  return (
    <header className="sticky top-0 z-header bg-background/80 backdrop-blur-md border-b">
      <nav className="container mx-auto px-4 h-16 flex items-center justify-between">
        {/* Logo */}
        <Link to="/" className="flex items-center gap-2">
          <img src="/logo.svg" alt="N9" className="h-8 w-auto" />
          <span className="font-bold text-xl text-primary">N9</span>
        </Link>
        
        {/* Main Navigation */}
        <div className="hidden md:flex items-center gap-6">
          {mainNavItems.map((item) => (
            <NavLink
              key={item.href}
              to={item.href}
              className={({ isActive }) => cn(
                'text-sm font-medium transition-colors hover:text-primary',
                isActive ? 'text-primary' : 'text-muted-foreground'
              )}
            >
              {item.label}
            </NavLink>
          ))}
          
          {userNavItems.map((item) => {
            // Check auth requirement
            if (item.auth && !isAuthenticated) return null;
            // Check role requirement
            if (item.role && !user?.roles?.includes(item.role)) return null;
            
            return (
              <NavLink
                key={item.href}
                to={item.href}
                className={({ isActive }) => cn(
                  'text-sm font-medium transition-colors hover:text-primary',
                  isActive ? 'text-primary' : 'text-muted-foreground'
                )}
              >
                {item.label}
              </NavLink>
            );
          })}
        </div>
        
        {/* Search */}
        <SearchTrigger />
        
        {/* User Actions */}
        <div className="flex items-center gap-4">
          {isAuthenticated ? (
            <>
              <WalletIndicator />
              <NotificationBell />
              <UserMenu />
            </>
          ) : (
            <>
              <Button variant="ghost" onClick={() => navigate('/login')}>
                Log In
              </Button>
              <Button onClick={() => navigate('/register')}>
                Sign Up
              </Button>
            </>
          )}
        </div>
      </nav>
    </header>
  );
}
```

### 5.3 Breadcrumb Component

```typescript
// components/navigation/Breadcrumbs.tsx
import { Link, useMatches } from 'react-router-dom';
import { ChevronRight, Home } from 'lucide-react';
import { Fragment } from 'react';

interface BreadcrumbHandle {
  breadcrumb?: string | ((params: any, data?: any) => string);
}

export function Breadcrumbs() {
  const matches = useMatches();
  
  const breadcrumbs = matches
    .filter((match) => Boolean((match.handle as BreadcrumbHandle)?.breadcrumb))
    .map((match) => {
      const handle = match.handle as BreadcrumbHandle;
      const breadcrumb = typeof handle.breadcrumb === 'function'
        ? handle.breadcrumb(match.params, match.data)
        : handle.breadcrumb;
      
      return {
        path: match.pathname,
        label: breadcrumb,
      };
    });
  
  if (breadcrumbs.length <= 1) return null;
  
  return (
    <nav aria-label="Breadcrumb" className="mb-4">
      <ol className="flex items-center gap-2 text-sm text-muted-foreground">
        <li>
          <Link to="/" className="hover:text-foreground transition-colors">
            <Home className="h-4 w-4" />
            <span className="sr-only">Home</span>
          </Link>
        </li>
        
        {breadcrumbs.map((crumb, index) => (
          <Fragment key={crumb.path}>
            <ChevronRight className="h-4 w-4" aria-hidden />
            <li>
              {index === breadcrumbs.length - 1 ? (
                <span className="text-foreground font-medium" aria-current="page">
                  {crumb.label}
                </span>
              ) : (
                <Link 
                  to={crumb.path} 
                  className="hover:text-foreground transition-colors"
                >
                  {crumb.label}
                </Link>
              )}
            </li>
          </Fragment>
        ))}
      </ol>
    </nav>
  );
}
```

---

## 6. Navigation Hooks

### 6.1 Navigation Utilities

```typescript
// hooks/useNavigationUtils.ts
import { 
  useNavigate, 
  useLocation, 
  useSearchParams,
  useParams 
} from 'react-router-dom';
import { useCallback } from 'react';

export function useNavigationUtils() {
  const navigate = useNavigate();
  const location = useLocation();
  
  const goBack = useCallback(() => {
    if (window.history.length > 2) {
      navigate(-1);
    } else {
      navigate('/');
    }
  }, [navigate]);
  
  const goToStory = useCallback((slug: string) => {
    navigate(`/story/${slug}`);
  }, [navigate]);
  
  const goToChapter = useCallback((storySlug: string, chapterNumber: number) => {
    navigate(`/story/${storySlug}/chapter/${chapterNumber}`);
  }, [navigate]);
  
  const goToAuthor = useCallback((username: string) => {
    navigate(`/author/${username}`);
  }, [navigate]);
  
  const goToSearch = useCallback((query?: string) => {
    if (query) {
      navigate(`/search?q=${encodeURIComponent(query)}`);
    } else {
      navigate('/search');
    }
  }, [navigate]);
  
  const goToLogin = useCallback((returnPath?: string) => {
    navigate('/login', { 
      state: { from: { pathname: returnPath || location.pathname } } 
    });
  }, [navigate, location]);
  
  return {
    navigate,
    goBack,
    goToStory,
    goToChapter,
    goToAuthor,
    goToSearch,
    goToLogin,
  };
}
```

### 6.2 Active Link Detection

```typescript
// hooks/useActiveLink.ts
import { useLocation, matchPath } from 'react-router-dom';
import { useMemo } from 'react';

export function useActiveLink(path: string, options: { exact?: boolean } = {}) {
  const { pathname } = useLocation();
  const { exact = false } = options;
  
  return useMemo(() => {
    if (exact) {
      return pathname === path;
    }
    return matchPath({ path, end: false }, pathname) !== null;
  }, [pathname, path, exact]);
}

export function useActiveLinks(paths: string[]) {
  const { pathname } = useLocation();
  
  return useMemo(() => {
    return paths.reduce((acc, path) => {
      acc[path] = matchPath({ path, end: false }, pathname) !== null;
      return acc;
    }, {} as Record<string, boolean>);
  }, [pathname, paths]);
}
```

### 6.3 Route Change Listener

```typescript
// hooks/useRouteChange.ts
import { useLocation } from 'react-router-dom';
import { useEffect, useRef } from 'react';

interface UseRouteChangeOptions {
  onRouteChange?: (location: Location) => void;
  scrollToTop?: boolean;
}

export function useRouteChange(options: UseRouteChangeOptions = {}) {
  const { onRouteChange, scrollToTop = true } = options;
  const location = useLocation();
  const prevLocationRef = useRef(location);
  
  useEffect(() => {
    if (prevLocationRef.current.pathname !== location.pathname) {
      // Scroll to top on route change
      if (scrollToTop) {
        window.scrollTo({ top: 0, behavior: 'smooth' });
      }
      
      // Callback
      onRouteChange?.(location);
      
      prevLocationRef.current = location;
    }
  }, [location, scrollToTop, onRouteChange]);
}
```

---

## 7. Deep Linking Specification

### 7.1 URL Patterns

| Feature | URL Pattern | Example |
|---------|-------------|---------|
| Story Detail | `/story/:slug` | `/story/the-great-adventure` |
| Chapter | `/story/:slug/chapter/:num` | `/story/the-great-adventure/chapter/15` |
| Author | `/author/:username` | `/author/john_doe` |
| Genre Browse | `/browse/:genre` | `/browse/fantasy` |
| Search | `/search?q=:query` | `/search?q=dragon` |
| Filtered Browse | `/browse?genre=:g&status=:s` | `/browse?genre=romance&status=completed` |
| Rankings | `/rankings?period=:p&type=:t` | `/rankings?period=weekly&type=trending` |

### 7.2 Query Parameter Specifications

```typescript
// types/urlParams.ts

// Browse page filters
interface BrowseParams {
  genre?: string;           // Genre slug
  status?: 'ongoing' | 'completed' | 'hiatus';
  rating?: 'general' | 'teen' | 'mature';
  sort?: 'trending' | 'popular' | 'new' | 'updated' | 'top_rated';
  page?: number;
  limit?: number;
}

// Search page filters
interface SearchParams {
  q: string;                // Search query
  type?: 'all' | 'stories' | 'authors' | 'tags';
  genre?: string;
  status?: string;
  page?: number;
}

// Rankings page filters
interface RankingsParams {
  period?: 'daily' | 'weekly' | 'monthly' | 'all_time';
  type?: 'trending' | 'popular' | 'top_rated' | 'most_read';
  genre?: string;
}

// Transaction history filters
interface TransactionParams {
  type?: 'all' | 'purchase' | 'earn' | 'spend' | 'withdraw';
  dateFrom?: string;        // ISO date
  dateTo?: string;          // ISO date
  page?: number;
}
```

---

## 8. Scroll Restoration

### 8.1 Scroll Position Management

```typescript
// hooks/useScrollRestoration.ts
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';

const scrollPositions = new Map<string, number>();

export function useScrollRestoration() {
  const { pathname, key } = useLocation();
  
  useEffect(() => {
    // Restore scroll position
    const savedPosition = scrollPositions.get(key);
    if (savedPosition !== undefined) {
      window.scrollTo(0, savedPosition);
    } else {
      window.scrollTo(0, 0);
    }
    
    // Save scroll position on unmount
    return () => {
      scrollPositions.set(key, window.scrollY);
    };
  }, [key, pathname]);
}
```

### 8.2 Chapter Reading Position

```typescript
// hooks/useChapterProgress.ts
import { useEffect, useRef, useCallback } from 'react';
import { useParams } from 'react-router-dom';

const STORAGE_KEY = 'n9-reading-progress';

interface ReadingProgress {
  chapterId: string;
  scrollPosition: number;
  percentage: number;
  timestamp: number;
}

export function useChapterProgress() {
  const { slug, chapterNumber } = useParams();
  const progressKey = `${slug}-${chapterNumber}`;
  const containerRef = useRef<HTMLDivElement>(null);
  
  // Load saved progress
  const loadProgress = useCallback((): ReadingProgress | null => {
    try {
      const stored = localStorage.getItem(STORAGE_KEY);
      if (!stored) return null;
      
      const progress = JSON.parse(stored);
      return progress[progressKey] || null;
    } catch {
      return null;
    }
  }, [progressKey]);
  
  // Save progress
  const saveProgress = useCallback((scrollPosition: number) => {
    if (!containerRef.current) return;
    
    const { scrollHeight, clientHeight } = containerRef.current;
    const percentage = Math.round((scrollPosition / (scrollHeight - clientHeight)) * 100);
    
    const progress: ReadingProgress = {
      chapterId: progressKey,
      scrollPosition,
      percentage,
      timestamp: Date.now(),
    };
    
    try {
      const stored = localStorage.getItem(STORAGE_KEY);
      const allProgress = stored ? JSON.parse(stored) : {};
      allProgress[progressKey] = progress;
      localStorage.setItem(STORAGE_KEY, JSON.stringify(allProgress));
    } catch (error) {
      console.warn('Failed to save reading progress:', error);
    }
  }, [progressKey]);
  
  // Restore position on mount
  useEffect(() => {
    const progress = loadProgress();
    if (progress && containerRef.current) {
      setTimeout(() => {
        containerRef.current?.scrollTo(0, progress.scrollPosition);
      }, 100);
    }
  }, [loadProgress]);
  
  // Save position on scroll (debounced)
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;
    
    let timeoutId: NodeJS.Timeout;
    const handleScroll = () => {
      clearTimeout(timeoutId);
      timeoutId = setTimeout(() => {
        saveProgress(container.scrollTop);
      }, 500);
    };
    
    container.addEventListener('scroll', handleScroll, { passive: true });
    return () => {
      container.removeEventListener('scroll', handleScroll);
      clearTimeout(timeoutId);
    };
  }, [saveProgress]);
  
  return {
    containerRef,
    loadProgress,
    saveProgress,
  };
}
```

---

## 9. Page Title Management

### 9.1 Document Title Hook

```typescript
// hooks/useDocumentTitle.ts
import { useEffect } from 'react';
import { useMatches } from 'react-router-dom';

const BASE_TITLE = 'N9 - Story Platform';

interface TitleHandle {
  title?: string | ((params: any, data?: any) => string);
}

export function useDocumentTitle(customTitle?: string) {
  const matches = useMatches();
  
  useEffect(() => {
    let title = customTitle;
    
    if (!title) {
      // Find the deepest match with a title handle
      const matchWithTitle = [...matches].reverse().find(
        (match) => Boolean((match.handle as TitleHandle)?.title)
      );
      
      if (matchWithTitle) {
        const handle = matchWithTitle.handle as TitleHandle;
        title = typeof handle.title === 'function'
          ? handle.title(matchWithTitle.params, matchWithTitle.data)
          : handle.title;
      }
    }
    
    document.title = title ? `${title} | ${BASE_TITLE}` : BASE_TITLE;
    
    return () => {
      document.title = BASE_TITLE;
    };
  }, [matches, customTitle]);
}
```

---

## 10. Navigation Analytics

### 10.1 Page View Tracking

```typescript
// hooks/usePageTracking.ts
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';
import { analytics } from '@/lib/analytics';

export function usePageTracking() {
  const location = useLocation();
  
  useEffect(() => {
    analytics.pageView({
      path: location.pathname,
      search: location.search,
      title: document.title,
    });
  }, [location.pathname, location.search]);
}
```

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Frontend Architecture Team | Initial release |

---

## 12. Related Documents

- [02_FRONTEND_ARCHITECTURE.md](./02_FRONTEND_ARCHITECTURE.md) - Architecture overview
- [04_STATE_MANAGEMENT.md](./04_STATE_MANAGEMENT.md) - State management
- [08_PAGES_SPECIFICATION.md](./08_PAGES_SPECIFICATION.md) - Page specifications
