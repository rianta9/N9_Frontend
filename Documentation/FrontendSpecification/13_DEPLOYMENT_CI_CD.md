# Deployment & CI/CD Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | DevOps Team |
| Review Cycle | Quarterly |
| Related Documents | 09_PERFORMANCE_OPTIMIZATION.md |

---

## 2. Deployment Architecture

### 2.1 Environment Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DEPLOYMENT ENVIRONMENTS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ DEVELOPMENT                                                             ││
│  │ • Branch: feature/*                                                     ││
│  │ • URL: localhost:5173                                                   ││
│  │ • Purpose: Local development                                            ││
│  │ • API: Mock server / Dev backend                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                            ↓                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ STAGING                                                                 ││
│  │ • Branch: develop                                                       ││
│  │ • URL: staging.n9.com                                                   ││
│  │ • Purpose: Integration testing, QA                                      ││
│  │ • API: Staging backend                                                  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                            ↓                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ PRODUCTION                                                              ││
│  │ • Branch: main                                                          ││
│  │ • URL: www.n9.com                                                       ││
│  │ • Purpose: Live user traffic                                            ││
│  │ • API: Production backend                                               ││
│  │ • CDN: Cloudflare / Azure CDN                                           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Infrastructure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PRODUCTION INFRASTRUCTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│     [Users]                                                                  │
│        │                                                                     │
│        ▼                                                                     │
│  ┌──────────┐                                                                │
│  │   CDN    │  Cloudflare / Azure CDN                                        │
│  │ (Global) │  • Static asset caching                                        │
│  └────┬─────┘  • DDoS protection                                             │
│       │        • SSL termination                                             │
│       ▼                                                                      │
│  ┌──────────┐                                                                │
│  │   Load   │  Azure Traffic Manager                                         │
│  │ Balancer │  • Health checks                                               │
│  └────┬─────┘  • Geo-routing                                                 │
│       │                                                                      │
│       ├─────────────────┬─────────────────┐                                  │
│       ▼                 ▼                 ▼                                  │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐                               │
│  │ Region  │      │ Region  │      │ Region  │  Azure Static Web Apps        │
│  │  East   │      │  West   │      │  Asia   │  • Auto-scaling               │
│  └─────────┘      └─────────┘      └─────────┘  • Preview deployments        │
│                                                                              │
│       │                 │                 │                                  │
│       └─────────────────┼─────────────────┘                                  │
│                         ▼                                                    │
│                  ┌──────────────┐                                            │
│                  │   Backend    │  API Gateway                               │
│                  │   API        │  • Rate limiting                           │
│                  └──────────────┘  • Authentication                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. CI/CD Pipeline

### 3.1 GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '20'
  PNPM_VERSION: '8'

jobs:
  # ============================================
  # LINT & TYPE CHECK
  # ============================================
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      
      - run: pnpm install --frozen-lockfile
      
      - name: Type Check
        run: pnpm type-check
      
      - name: Lint
        run: pnpm lint
      
      - name: Format Check
        run: pnpm format:check

  # ============================================
  # UNIT & INTEGRATION TESTS
  # ============================================
  test:
    name: Unit & Integration Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      
      - run: pnpm install --frozen-lockfile
      
      - name: Run Tests
        run: pnpm test:coverage
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: true

  # ============================================
  # E2E TESTS
  # ============================================
  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      
      - run: pnpm install --frozen-lockfile
      
      - name: Install Playwright
        run: pnpm exec playwright install --with-deps chromium
      
      - name: Run E2E Tests
        run: pnpm test:e2e
      
      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7

  # ============================================
  # BUILD
  # ============================================
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      
      - run: pnpm install --frozen-lockfile
      
      - name: Build
        run: pnpm build
        env:
          VITE_API_URL: ${{ vars.VITE_API_URL }}
          VITE_WS_URL: ${{ vars.VITE_WS_URL }}
      
      - name: Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7

  # ============================================
  # DEPLOY STAGING
  # ============================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build, e2e]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.n9.com
    steps:
      - uses: actions/checkout@v4
      
      - name: Download Build Artifact
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      
      - name: Deploy to Azure Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_SWA_TOKEN_STAGING }}
          action: 'upload'
          app_location: 'dist'
          skip_app_build: true

  # ============================================
  # DEPLOY PRODUCTION
  # ============================================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, e2e]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://www.n9.com
    steps:
      - uses: actions/checkout@v4
      
      - name: Download Build Artifact
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      
      - name: Deploy to Azure Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_SWA_TOKEN_PRODUCTION }}
          action: 'upload'
          app_location: 'dist'
          skip_app_build: true
      
      - name: Purge CDN Cache
        uses: azure/CLI@v1
        with:
          inlineScript: |
            az cdn endpoint purge \
              --resource-group n9-production \
              --profile-name n9-cdn \
              --name n9-frontend \
              --content-paths '/*'
```

### 3.2 Preview Deployments

```yaml
# .github/workflows/preview.yml
name: Preview Deployment

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-preview:
    name: Deploy Preview
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
        env:
          VITE_API_URL: ${{ vars.VITE_API_URL_STAGING }}
      
      - name: Deploy Preview
        id: deploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_SWA_TOKEN_STAGING }}
          action: 'upload'
          app_location: 'dist'
          skip_app_build: true
      
      - name: Comment Preview URL
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Preview deployed to: ${{ steps.deploy.outputs.static_web_app_url }}'
            })
```

---

## 4. Build Configuration

### 4.1 Environment Variables

```typescript
// env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  // API
  readonly VITE_API_URL: string;
  readonly VITE_WS_URL: string;
  readonly VITE_CDN_URL: string;
  
  // Auth
  readonly VITE_GOOGLE_CLIENT_ID: string;
  readonly VITE_FACEBOOK_APP_ID: string;
  
  // Analytics
  readonly VITE_GA_MEASUREMENT_ID: string;
  readonly VITE_SENTRY_DSN: string;
  
  // Features
  readonly VITE_ENABLE_MOCK_API: string;
  readonly VITE_ENABLE_DEVTOOLS: string;
  
  // Build info
  readonly VITE_BUILD_VERSION: string;
  readonly VITE_BUILD_DATE: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### 4.2 Environment Files

```bash
# .env.example
# API Configuration
VITE_API_URL=http://localhost:8080/api
VITE_WS_URL=ws://localhost:8080/ws
VITE_CDN_URL=https://cdn.n9.com

# OAuth Providers
VITE_GOOGLE_CLIENT_ID=
VITE_FACEBOOK_APP_ID=

# Analytics
VITE_GA_MEASUREMENT_ID=
VITE_SENTRY_DSN=

# Feature Flags
VITE_ENABLE_MOCK_API=false
VITE_ENABLE_DEVTOOLS=true
```

```bash
# .env.staging
VITE_API_URL=https://api.staging.n9.com
VITE_WS_URL=wss://ws.staging.n9.com
VITE_CDN_URL=https://cdn.staging.n9.com
VITE_ENABLE_MOCK_API=false
VITE_ENABLE_DEVTOOLS=true
```

```bash
# .env.production
VITE_API_URL=https://api.n9.com
VITE_WS_URL=wss://ws.n9.com
VITE_CDN_URL=https://cdn.n9.com
VITE_ENABLE_MOCK_API=false
VITE_ENABLE_DEVTOOLS=false
```

---

## 5. Build Optimization

### 5.1 Production Build

```typescript
// vite.config.ts (production optimizations)
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { compression } from 'vite-plugin-compression2';
import { ViteImageOptimizer } from 'vite-plugin-image-optimizer';

export default defineConfig({
  plugins: [
    react(),
    
    // Gzip compression
    compression({
      algorithm: 'gzip',
      exclude: [/\.(br)$/, /\.(gz)$/],
    }),
    
    // Brotli compression
    compression({
      algorithm: 'brotliCompress',
      exclude: [/\.(br)$/, /\.(gz)$/],
    }),
    
    // Image optimization
    ViteImageOptimizer({
      png: { quality: 80 },
      jpeg: { quality: 80 },
      webp: { quality: 80 },
    }),
  ],
  
  build: {
    target: 'es2020',
    minify: 'terser',
    sourcemap: true,
    
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
    
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'router': ['react-router-dom'],
          'query': ['@tanstack/react-query'],
          'ui': [
            '@radix-ui/react-dialog',
            '@radix-ui/react-dropdown-menu',
            '@radix-ui/react-toast',
          ],
        },
      },
    },
    
    // Report compressed size
    reportCompressedSize: true,
  },
});
```

### 5.2 Bundle Analysis

```json
// package.json scripts
{
  "scripts": {
    "build": "vite build",
    "build:analyze": "ANALYZE=true vite build",
    "build:preview": "vite build && vite preview"
  }
}
```

---

## 6. Hosting Configuration

### 6.1 Azure Static Web Apps

```json
// staticwebapp.config.json
{
  "routes": [
    {
      "route": "/api/*",
      "allowedRoles": ["anonymous"]
    },
    {
      "route": "/*",
      "serve": "/index.html",
      "statusCode": 200
    }
  ],
  
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/images/*.{png,jpg,gif,svg}", "/css/*", "/js/*", "/api/*"]
  },
  
  "globalHeaders": {
    "X-Frame-Options": "DENY",
    "X-Content-Type-Options": "nosniff",
    "X-XSS-Protection": "1; mode=block",
    "Referrer-Policy": "strict-origin-when-cross-origin",
    "Permissions-Policy": "camera=(), microphone=(), geolocation=()",
    "Content-Security-Policy": "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.google-analytics.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https://cdn.n9.com; connect-src 'self' https://api.n9.com wss://ws.n9.com"
  },
  
  "mimeTypes": {
    ".json": "application/json",
    ".woff2": "font/woff2"
  },
  
  "responseOverrides": {
    "404": {
      "rewrite": "/index.html",
      "statusCode": 200
    }
  }
}
```

### 6.2 Cache Headers

```json
// Cache configuration
{
  "routes": [
    {
      "route": "/assets/*",
      "headers": {
        "Cache-Control": "public, max-age=31536000, immutable"
      }
    },
    {
      "route": "/index.html",
      "headers": {
        "Cache-Control": "no-cache, no-store, must-revalidate"
      }
    },
    {
      "route": "/locales/*",
      "headers": {
        "Cache-Control": "public, max-age=86400"
      }
    }
  ]
}
```

---

## 7. Monitoring & Observability

### 7.1 Error Tracking

```typescript
// lib/monitoring/sentry.ts
import * as Sentry from '@sentry/react';
import { BrowserTracing } from '@sentry/tracing';

export function initSentry() {
  if (import.meta.env.PROD && import.meta.env.VITE_SENTRY_DSN) {
    Sentry.init({
      dsn: import.meta.env.VITE_SENTRY_DSN,
      
      integrations: [
        new BrowserTracing({
          tracingOrigins: ['localhost', 'n9.com', /^\//],
          routingInstrumentation: Sentry.reactRouterV6Instrumentation(
            React.useEffect,
            useLocation,
            useNavigationType,
            createRoutesFromChildren,
            matchRoutes
          ),
        }),
      ],
      
      tracesSampleRate: 0.2,
      
      environment: import.meta.env.MODE,
      release: import.meta.env.VITE_BUILD_VERSION,
      
      beforeSend(event) {
        // Filter sensitive data
        if (event.request?.headers) {
          delete event.request.headers['Authorization'];
        }
        return event;
      },
    });
  }
}
```

### 7.2 Analytics

```typescript
// lib/monitoring/analytics.ts
import ReactGA from 'react-ga4';

export function initAnalytics() {
  if (import.meta.env.PROD && import.meta.env.VITE_GA_MEASUREMENT_ID) {
    ReactGA.initialize(import.meta.env.VITE_GA_MEASUREMENT_ID);
  }
}

export function trackPageView(path: string) {
  ReactGA.send({ hitType: 'pageview', page: path });
}

export function trackEvent(category: string, action: string, label?: string, value?: number) {
  ReactGA.event({ category, action, label, value });
}

// Analytics hook
export function useAnalytics() {
  const location = useLocation();
  
  useEffect(() => {
    trackPageView(location.pathname);
  }, [location]);
  
  return { trackEvent };
}
```

---

## 8. Health Checks

### 8.1 Application Health

```typescript
// components/monitoring/HealthCheck.tsx
import { useQuery } from '@tanstack/react-query';

export function useHealthCheck() {
  return useQuery({
    queryKey: ['health'],
    queryFn: async () => {
      const [api, ws] = await Promise.allSettled([
        fetch(`${import.meta.env.VITE_API_URL}/health`),
        testWebSocket(),
      ]);
      
      return {
        api: api.status === 'fulfilled' && api.value.ok,
        ws: ws.status === 'fulfilled',
        timestamp: new Date().toISOString(),
      };
    },
    refetchInterval: 60000, // Check every minute
  });
}

async function testWebSocket(): Promise<boolean> {
  return new Promise((resolve) => {
    const ws = new WebSocket(import.meta.env.VITE_WS_URL);
    ws.onopen = () => { ws.close(); resolve(true); };
    ws.onerror = () => resolve(false);
    setTimeout(() => resolve(false), 5000);
  });
}
```

---

## 9. Rollback Strategy

### 9.1 Deployment Rollback

```yaml
# .github/workflows/rollback.yml
name: Rollback Deployment

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to rollback'
        required: true
        type: choice
        options:
          - staging
          - production
      version:
        description: 'Version to rollback to (commit SHA)'
        required: true

jobs:
  rollback:
    name: Rollback to ${{ inputs.version }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.version }}
      
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      
      - name: Deploy Rollback
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets[format('AZURE_SWA_TOKEN_{0}', inputs.environment)] }}
          action: 'upload'
          app_location: 'dist'
          skip_app_build: true
      
      - name: Notify Team
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🔄 Rollback completed for ${{ inputs.environment }} to version ${{ inputs.version }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 10. Scripts Reference

```json
// package.json
{
  "scripts": {
    // Development
    "dev": "vite",
    "dev:mock": "VITE_ENABLE_MOCK_API=true vite",
    
    // Build
    "build": "tsc && vite build",
    "build:staging": "tsc && vite build --mode staging",
    "build:production": "tsc && vite build --mode production",
    "build:analyze": "ANALYZE=true vite build",
    "preview": "vite preview",
    
    // Quality
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,css}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,css}\"",
    "type-check": "tsc --noEmit",
    
    // Testing
    "test": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    
    // Utilities
    "clean": "rimraf dist coverage .turbo",
    "prepare": "husky install"
  }
}
```

---

## 11. Deployment Checklist

### 11.1 Pre-Deployment

- [ ] All tests passing
- [ ] No lint errors
- [ ] Type checking passes
- [ ] Bundle size within budget
- [ ] Environment variables configured
- [ ] Feature flags reviewed
- [ ] Database migrations complete (backend)
- [ ] API compatibility verified

### 11.2 Post-Deployment

- [ ] Smoke tests passing
- [ ] Error rates normal
- [ ] Performance metrics normal
- [ ] CDN cache purged
- [ ] Monitoring alerts configured
- [ ] Rollback plan ready
- [ ] Stakeholders notified

---

## 12. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | DevOps Team | Initial release |

---

## 13. Related Documents

- [09_PERFORMANCE_OPTIMIZATION.md](./09_PERFORMANCE_OPTIMIZATION.md) - Build optimization
- [10_SECURITY_COMPLIANCE.md](./10_SECURITY_COMPLIANCE.md) - Security headers
- [11_TESTING_STRATEGY.md](./11_TESTING_STRATEGY.md) - CI testing
