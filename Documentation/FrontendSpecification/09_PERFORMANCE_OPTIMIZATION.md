# Performance Optimization Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Frontend Architecture Team |
| Review Cycle | Quarterly |
| Related Documents | 02_FRONTEND_ARCHITECTURE.md, 04_STATE_MANAGEMENT.md |

---

## 2. Performance Targets

### 2.1 Core Web Vitals

| Metric | Target | Maximum | Description |
|--------|--------|---------|-------------|
| **LCP** | < 2.0s | < 2.5s | Largest Contentful Paint |
| **FID** | < 50ms | < 100ms | First Input Delay |
| **INP** | < 150ms | < 200ms | Interaction to Next Paint |
| **CLS** | < 0.05 | < 0.1 | Cumulative Layout Shift |
| **TTFB** | < 400ms | < 800ms | Time to First Byte |
| **FCP** | < 1.0s | < 1.8s | First Contentful Paint |

### 2.2 Application Metrics

| Metric | Target | Description |
|--------|--------|-------------|
| Initial Bundle Size | < 150KB (gzipped) | Main JS bundle |
| Time to Interactive | < 3.0s | On 3G connection |
| Route Change Time | < 200ms | Client-side navigation |
| API Response Display | < 100ms | After data received |
| Image Load Time | < 1.0s | Above-fold images |

---

## 3. Bundle Optimization

### 3.1 Code Splitting Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          BUNDLE ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ MAIN BUNDLE (Critical - loads immediately)                              ││
│  │ • React core, React DOM                                                 ││
│  │ • Router (base)                                                         ││
│  │ • Core UI components                                                    ││
│  │ • Auth store                                                            ││
│  │ Target: < 80KB gzipped                                                  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ VENDOR BUNDLE (Cached separately)                                       ││
│  │ • TanStack Query                                                        ││
│  │ • Zustand                                                               ││
│  │ • Framer Motion                                                         ││
│  │ • Form libraries                                                        ││
│  │ Target: < 100KB gzipped                                                 ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ ROUTE CHUNKS (Lazy loaded per route)                                    ││
│  │ • /                   → home.chunk.js                                   ││
│  │ • /browse             → browse.chunk.js                                 ││
│  │ • /story/:slug        → story.chunk.js                                  ││
│  │ • /story/.../chapter  → reader.chunk.js                                 ││
│  │ • /author-dashboard/* → dashboard.chunk.js                              ││
│  │ • /admin/*            → admin.chunk.js                                  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ FEATURE CHUNKS (On-demand)                                              ││
│  │ • Rich text editor    → editor.chunk.js                                 ││
│  │ • Charts/Analytics    → charts.chunk.js                                 ││
│  │ • Image cropper       → cropper.chunk.js                                ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Vite Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      filename: './dist/stats.html',
      gzipSize: true,
    }),
  ],
  
  build: {
    target: 'es2020',
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
    rollupOptions: {
      output: {
        manualChunks: {
          // Vendor chunks
          'react-vendor': ['react', 'react-dom'],
          'router': ['react-router-dom'],
          'query': ['@tanstack/react-query'],
          'ui': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          'animation': ['framer-motion'],
          
          // Feature chunks
          'editor': ['@tiptap/react', '@tiptap/starter-kit'],
          'charts': ['recharts'],
        },
        
        // Dynamic chunk naming
        chunkFileNames: (chunkInfo) => {
          const facadeModuleId = chunkInfo.facadeModuleId ?? '';
          if (facadeModuleId.includes('pages/')) {
            return 'pages/[name]-[hash].js';
          }
          return 'chunks/[name]-[hash].js';
        },
      },
    },
    
    // Chunk size warnings
    chunkSizeWarningLimit: 500,
  },
  
  // Optimize deps
  optimizeDeps: {
    include: ['react', 'react-dom', 'react-router-dom'],
  },
});
```

### 3.3 Lazy Loading Implementation

```typescript
// router/lazyRoutes.ts
import { lazy, Suspense } from 'react';
import { PageSkeleton } from '@/components/skeletons/PageSkeleton';

// Lazy page components with prefetch hints
export const HomePage = lazy(() => 
  import(/* webpackChunkName: "home" */ '@/pages/home/HomePage')
);

export const BrowsePage = lazy(() => 
  import(/* webpackChunkName: "browse" */ '@/pages/browse/BrowsePage')
);

export const StoryDetailPage = lazy(() => 
  import(/* webpackChunkName: "story" */ '@/pages/story/StoryDetailPage')
);

export const ChapterReaderPage = lazy(() => 
  import(/* webpackChunkName: "reader" */ '@/pages/reader/ChapterReaderPage')
);

// Author dashboard - separate chunk
export const AuthorDashboardPage = lazy(() => 
  import(/* webpackChunkName: "dashboard" */ '@/pages/author-dashboard/DashboardPage')
);

// Admin - separate chunk (only for admins)
export const AdminDashboardPage = lazy(() => 
  import(/* webpackChunkName: "admin" */ '@/pages/admin/AdminDashboardPage')
);

// Wrapper for lazy components
export function LazyPage({ 
  children, 
  fallback = <PageSkeleton /> 
}: { 
  children: React.ReactNode; 
  fallback?: React.ReactNode 
}) {
  return <Suspense fallback={fallback}>{children}</Suspense>;
}
```

---

## 4. Data Fetching Optimization

### 4.1 Query Prefetching

```typescript
// lib/prefetch.ts
import { queryClient } from '@/config/queryClient';
import { queryKeys } from '@/lib/queryKeys';
import { storyService } from '@/features/stories/api/storyService';

// Prefetch on route hover
export function prefetchStory(slug: string) {
  queryClient.prefetchQuery({
    queryKey: queryKeys.stories.detail(slug),
    queryFn: () => storyService.getBySlug(slug),
    staleTime: 5 * 60 * 1000,
  });
}

// Prefetch on app init
export async function prefetchInitialData() {
  await Promise.all([
    queryClient.prefetchQuery({
      queryKey: queryKeys.stories.trending(),
      queryFn: () => storyService.getTrending(),
    }),
    queryClient.prefetchQuery({
      queryKey: queryKeys.stories.featured(),
      queryFn: () => storyService.getFeatured(),
    }),
  ]);
}

// Link component with prefetch
export function PrefetchLink({ 
  to, 
  children, 
  prefetch 
}: { 
  to: string; 
  children: React.ReactNode;
  prefetch?: () => void;
}) {
  const handleMouseEnter = () => {
    prefetch?.();
  };
  
  return (
    <Link to={to} onMouseEnter={handleMouseEnter}>
      {children}
    </Link>
  );
}
```

### 4.2 Data Caching Strategy

```typescript
// config/queryClient.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Global defaults
      staleTime: 5 * 60 * 1000,      // 5 minutes
      gcTime: 30 * 60 * 1000,         // 30 minutes
      refetchOnWindowFocus: false,
      retry: 2,
      
      // Structural sharing for performance
      structuralSharing: true,
    },
  },
});

// Query-specific cache times
export const cacheConfig = {
  // Static data - cache longer
  genres: { staleTime: 24 * 60 * 60 * 1000 },  // 24 hours
  coinPackages: { staleTime: 60 * 60 * 1000 }, // 1 hour
  
  // Semi-static
  trending: { staleTime: 5 * 60 * 1000 },      // 5 minutes
  featured: { staleTime: 15 * 60 * 1000 },     // 15 minutes
  
  // Dynamic data
  storyDetail: { staleTime: 10 * 60 * 1000 },  // 10 minutes
  chapters: { staleTime: 5 * 60 * 1000 },      // 5 minutes
  comments: { staleTime: 1 * 60 * 1000 },      // 1 minute
  
  // Real-time data
  notifications: { staleTime: 30 * 1000 },     // 30 seconds
  wallet: { staleTime: 60 * 1000 },            // 1 minute
};
```

### 4.3 Pagination & Infinite Scroll

```typescript
// hooks/useInfiniteScroll.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { useInView } from 'react-intersection-observer';
import { useEffect } from 'react';

export function useInfiniteScroll<T>({
  queryKey,
  queryFn,
  getNextPageParam,
}: {
  queryKey: QueryKey;
  queryFn: (page: number) => Promise<PaginatedResponse<T>>;
  getNextPageParam: (lastPage: PaginatedResponse<T>) => number | undefined;
}) {
  const { ref, inView } = useInView({
    threshold: 0,
    rootMargin: '200px', // Load 200px before reaching bottom
  });
  
  const query = useInfiniteQuery({
    queryKey,
    queryFn: ({ pageParam = 1 }) => queryFn(pageParam),
    getNextPageParam,
    initialPageParam: 1,
  });
  
  // Auto-fetch next page when sentinel is visible
  useEffect(() => {
    if (inView && query.hasNextPage && !query.isFetchingNextPage) {
      query.fetchNextPage();
    }
  }, [inView, query.hasNextPage, query.isFetchingNextPage]);
  
  return {
    ...query,
    sentinelRef: ref,
  };
}
```

---

## 5. Image Optimization

### 5.1 Image Loading Strategy

```typescript
// components/common/OptimizedImage/OptimizedImage.tsx
import { useState, useRef, useEffect } from 'react';
import { cn } from '@/lib/utils';

interface OptimizedImageProps {
  src: string;
  alt: string;
  width: number;
  height: number;
  className?: string;
  priority?: boolean;
  placeholder?: 'blur' | 'empty';
  blurDataUrl?: string;
}

export function OptimizedImage({
  src,
  alt,
  width,
  height,
  className,
  priority = false,
  placeholder = 'blur',
  blurDataUrl,
}: OptimizedImageProps) {
  const [isLoaded, setIsLoaded] = useState(false);
  const [isInView, setIsInView] = useState(priority);
  const imgRef = useRef<HTMLImageElement>(null);
  
  // Intersection observer for lazy loading
  useEffect(() => {
    if (priority) return;
    
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsInView(true);
          observer.disconnect();
        }
      },
      { rootMargin: '200px' }
    );
    
    if (imgRef.current) {
      observer.observe(imgRef.current);
    }
    
    return () => observer.disconnect();
  }, [priority]);
  
  // Generate srcset for responsive images
  const srcset = generateSrcSet(src, [width, width * 1.5, width * 2]);
  
  return (
    <div
      ref={imgRef}
      className={cn('relative overflow-hidden', className)}
      style={{ aspectRatio: `${width}/${height}` }}
    >
      {/* Blur placeholder */}
      {placeholder === 'blur' && !isLoaded && (
        <div
          className="absolute inset-0 bg-cover bg-center blur-xl scale-110"
          style={{
            backgroundImage: blurDataUrl ? `url(${blurDataUrl})` : undefined,
            backgroundColor: !blurDataUrl ? '#e5e7eb' : undefined,
          }}
        />
      )}
      
      {/* Main image */}
      {isInView && (
        <img
          src={src}
          srcSet={srcset}
          sizes={`${width}px`}
          alt={alt}
          width={width}
          height={height}
          loading={priority ? 'eager' : 'lazy'}
          decoding={priority ? 'sync' : 'async'}
          onLoad={() => setIsLoaded(true)}
          className={cn(
            'absolute inset-0 w-full h-full object-cover transition-opacity duration-300',
            isLoaded ? 'opacity-100' : 'opacity-0'
          )}
        />
      )}
    </div>
  );
}

function generateSrcSet(src: string, widths: number[]): string {
  const cdnBase = import.meta.env.VITE_CDN_URL;
  
  return widths
    .map((w) => {
      const optimizedUrl = `${cdnBase}/optimize?url=${encodeURIComponent(src)}&w=${Math.round(w)}&q=80`;
      return `${optimizedUrl} ${Math.round(w)}w`;
    })
    .join(', ');
}
```

### 5.2 Image CDN Configuration

```typescript
// lib/imageUtils.ts

const CDN_BASE = import.meta.env.VITE_CDN_URL;

interface ImageTransformOptions {
  width?: number;
  height?: number;
  quality?: number;
  format?: 'webp' | 'avif' | 'auto';
  fit?: 'cover' | 'contain' | 'fill';
}

export function getOptimizedImageUrl(
  src: string,
  options: ImageTransformOptions = {}
): string {
  const {
    width,
    height,
    quality = 80,
    format = 'auto',
    fit = 'cover',
  } = options;
  
  const params = new URLSearchParams();
  params.set('url', src);
  if (width) params.set('w', String(width));
  if (height) params.set('h', String(height));
  params.set('q', String(quality));
  params.set('f', format);
  params.set('fit', fit);
  
  return `${CDN_BASE}/image?${params.toString()}`;
}

// Presets for common sizes
export const imagePresets = {
  storyCover: {
    thumbnail: { width: 100, height: 150 },
    card: { width: 200, height: 300 },
    detail: { width: 400, height: 600 },
  },
  avatar: {
    small: { width: 32, height: 32 },
    medium: { width: 64, height: 64 },
    large: { width: 128, height: 128 },
  },
  banner: {
    mobile: { width: 640, height: 360 },
    desktop: { width: 1280, height: 400 },
  },
};
```

---

## 6. Rendering Optimization

### 6.1 Component Memoization

```typescript
// Memoization patterns

// 1. Memo for expensive list items
const StoryCard = memo(function StoryCard({ story }: { story: Story }) {
  return (
    <Card>
      {/* ... */}
    </Card>
  );
}, (prevProps, nextProps) => {
  // Custom comparison for complex objects
  return (
    prevProps.story.id === nextProps.story.id &&
    prevProps.story.stats.likes === nextProps.story.stats.likes
  );
});

// 2. useMemo for derived data
function StoryList({ stories, filter }: Props) {
  const filteredStories = useMemo(() => {
    return stories.filter(s => matchesFilter(s, filter));
  }, [stories, filter]);
  
  const sortedStories = useMemo(() => {
    return [...filteredStories].sort((a, b) => b.stats.views - a.stats.views);
  }, [filteredStories]);
  
  return (
    <div>
      {sortedStories.map(story => (
        <StoryCard key={story.id} story={story} />
      ))}
    </div>
  );
}

// 3. useCallback for event handlers
function CommentInput({ onSubmit }: { onSubmit: (text: string) => void }) {
  const [text, setText] = useState('');
  
  const handleSubmit = useCallback(() => {
    if (text.trim()) {
      onSubmit(text);
      setText('');
    }
  }, [text, onSubmit]);
  
  return (
    <div>
      <textarea value={text} onChange={(e) => setText(e.target.value)} />
      <Button onClick={handleSubmit}>Submit</Button>
    </div>
  );
}
```

### 6.2 Virtual Lists

```typescript
// components/common/VirtualList/VirtualList.tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

interface VirtualListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  estimateSize: number;
  overscan?: number;
  className?: string;
}

export function VirtualList<T>({
  items,
  renderItem,
  estimateSize,
  overscan = 5,
  className,
}: VirtualListProps<T>) {
  const parentRef = useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => estimateSize,
    overscan,
  });
  
  return (
    <div
      ref={parentRef}
      className={cn('overflow-auto', className)}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualItem.size}px`,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            {renderItem(items[virtualItem.index], virtualItem.index)}
          </div>
        ))}
      </div>
    </div>
  );
}

// Usage for chapter list
function ChapterListVirtual({ chapters }: { chapters: Chapter[] }) {
  return (
    <VirtualList
      items={chapters}
      estimateSize={72}
      overscan={10}
      className="h-[600px]"
      renderItem={(chapter) => <ChapterRow chapter={chapter} />}
    />
  );
}
```

---

## 7. Animation Performance

### 7.1 Animation Best Practices

```typescript
// Use CSS transforms and opacity (GPU accelerated)
const animationVariants = {
  initial: { opacity: 0, transform: 'translateY(20px)' },
  animate: { opacity: 1, transform: 'translateY(0)' },
  exit: { opacity: 0, transform: 'translateY(-20px)' },
};

// Avoid animating layout properties
// ❌ Bad
const badAnimation = { width: '0%', width: '100%' };

// ✅ Good
const goodAnimation = { 
  transform: 'scaleX(0)', 
  transform: 'scaleX(1)',
  transformOrigin: 'left',
};

// Use will-change sparingly
const optimizedStyles = {
  willChange: 'transform, opacity', // Only when animating
};
```

### 7.2 Reduced Motion Support

```typescript
// hooks/useReducedMotion.ts
import { useEffect, useState } from 'react';

export function useReducedMotion(): boolean {
  const [reducedMotion, setReducedMotion] = useState(
    () => window.matchMedia('(prefers-reduced-motion: reduce)').matches
  );
  
  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
    const handler = (e: MediaQueryListEvent) => setReducedMotion(e.matches);
    
    mediaQuery.addEventListener('change', handler);
    return () => mediaQuery.removeEventListener('change', handler);
  }, []);
  
  return reducedMotion;
}

// Usage
function AnimatedComponent() {
  const reducedMotion = useReducedMotion();
  
  return (
    <motion.div
      initial={{ opacity: 0, y: reducedMotion ? 0 : 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ 
        duration: reducedMotion ? 0 : 0.3 
      }}
    >
      Content
    </motion.div>
  );
}
```

---

## 8. Performance Monitoring

### 8.1 Core Web Vitals Tracking

```typescript
// lib/performance/vitals.ts
import { onCLS, onFID, onFCP, onLCP, onTTFB, onINP } from 'web-vitals';

interface VitalMetric {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  delta: number;
  id: string;
}

function sendToAnalytics(metric: VitalMetric) {
  // Send to analytics service
  if (import.meta.env.PROD) {
    fetch('/api/analytics/vitals', {
      method: 'POST',
      body: JSON.stringify(metric),
      headers: { 'Content-Type': 'application/json' },
      keepalive: true,
    });
  }
  
  // Log in development
  if (import.meta.env.DEV) {
    console.log(`[Web Vital] ${metric.name}:`, metric.value, metric.rating);
  }
}

export function initWebVitals() {
  onCLS(sendToAnalytics);
  onFID(sendToAnalytics);
  onFCP(sendToAnalytics);
  onLCP(sendToAnalytics);
  onTTFB(sendToAnalytics);
  onINP(sendToAnalytics);
}
```

### 8.2 Performance Budget

```typescript
// Performance budget configuration
export const performanceBudget = {
  // Bundle sizes (gzipped)
  bundles: {
    main: 80 * 1024,      // 80KB
    vendor: 100 * 1024,   // 100KB
    total: 200 * 1024,    // 200KB
  },
  
  // Timing thresholds (ms)
  timing: {
    fcp: 1800,
    lcp: 2500,
    tti: 3800,
    tbt: 300,
  },
  
  // Resource counts
  resources: {
    scripts: 10,
    styles: 5,
    images: 20,
    fonts: 4,
  },
};
```

---

## 9. Checklist

### 9.1 Pre-Launch Performance Checklist

- [ ] Bundle size under 200KB gzipped
- [ ] All routes lazy loaded
- [ ] Images optimized and lazy loaded
- [ ] Core Web Vitals passing
- [ ] Service worker configured
- [ ] CDN configured
- [ ] Compression enabled
- [ ] Caching headers set
- [ ] No render-blocking resources
- [ ] Critical CSS inlined

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Frontend Architecture Team | Initial release |

---

## 11. Related Documents

- [02_FRONTEND_ARCHITECTURE.md](./02_FRONTEND_ARCHITECTURE.md) - Architecture patterns
- [04_STATE_MANAGEMENT.md](./04_STATE_MANAGEMENT.md) - Data caching
- [13_DEPLOYMENT_CI_CD.md](./13_DEPLOYMENT_CI_CD.md) - Build optimization
