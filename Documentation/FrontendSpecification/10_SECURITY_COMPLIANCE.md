# Security & Compliance Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Security Team |
| Review Cycle | Quarterly |
| Related Documents | 02_FRONTEND_ARCHITECTURE.md, 06_API_INTEGRATION.md |

---

## 2. Security Architecture

### 2.1 Security Layers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SECURITY ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ LAYER 1: TRANSPORT SECURITY                                             ││
│  │ • HTTPS/TLS 1.3 enforced                                                ││
│  │ • HSTS headers                                                          ││
│  │ • Certificate pinning (mobile)                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ LAYER 2: CONTENT SECURITY                                               ││
│  │ • Content Security Policy (CSP)                                         ││
│  │ • XSS protection                                                        ││
│  │ • Input sanitization                                                    ││
│  │ • Output encoding                                                       ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ LAYER 3: AUTHENTICATION                                                 ││
│  │ • JWT access tokens (short-lived)                                       ││
│  │ • Refresh token rotation                                                ││
│  │ • OAuth 2.0 / OIDC                                                      ││
│  │ • Multi-factor authentication                                           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ LAYER 4: AUTHORIZATION                                                  ││
│  │ • Role-based access control                                             ││
│  │ • Route guards                                                          ││
│  │ • Component-level permissions                                           ││
│  │ • Feature flags                                                         ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ LAYER 5: DATA PROTECTION                                                ││
│  │ • Sensitive data encryption                                             ││
│  │ • Secure storage                                                        ││
│  │ • Data masking                                                          ││
│  │ • Privacy controls                                                      ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Authentication Security

### 3.1 Token Management

```typescript
// lib/security/tokenManager.ts
import { secureStorage } from './secureStorage';

const ACCESS_TOKEN_KEY = 'n9_access_token';
const REFRESH_TOKEN_KEY = 'n9_refresh_token';
const TOKEN_EXPIRY_KEY = 'n9_token_expiry';

// Token expiry buffer (refresh 5 minutes before expiry)
const EXPIRY_BUFFER_MS = 5 * 60 * 1000;

interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

export const tokenManager = {
  // Store tokens securely
  setTokens({ accessToken, refreshToken, expiresIn }: TokenPair) {
    const expiryTime = Date.now() + (expiresIn * 1000);
    
    // Use secure storage (HttpOnly cookies preferred)
    secureStorage.setItem(ACCESS_TOKEN_KEY, accessToken);
    secureStorage.setItem(REFRESH_TOKEN_KEY, refreshToken);
    secureStorage.setItem(TOKEN_EXPIRY_KEY, String(expiryTime));
  },
  
  // Get access token
  getAccessToken(): string | null {
    return secureStorage.getItem(ACCESS_TOKEN_KEY);
  },
  
  // Check if token needs refresh
  isTokenExpiring(): boolean {
    const expiry = secureStorage.getItem(TOKEN_EXPIRY_KEY);
    if (!expiry) return true;
    
    return Date.now() + EXPIRY_BUFFER_MS >= parseInt(expiry);
  },
  
  // Clear all tokens
  clearTokens() {
    secureStorage.removeItem(ACCESS_TOKEN_KEY);
    secureStorage.removeItem(REFRESH_TOKEN_KEY);
    secureStorage.removeItem(TOKEN_EXPIRY_KEY);
  },
  
  // Get refresh token for rotation
  getRefreshToken(): string | null {
    return secureStorage.getItem(REFRESH_TOKEN_KEY);
  },
};
```

### 3.2 Secure Storage

```typescript
// lib/security/secureStorage.ts

// Detect storage availability and security context
const isSecureContext = window.isSecureContext;
const hasLocalStorage = typeof localStorage !== 'undefined';

// Encryption for localStorage fallback (when cookies not available)
async function encrypt(data: string): Promise<string> {
  const encoder = new TextEncoder();
  const dataBuffer = encoder.encode(data);
  
  // Derive key from a static salt (stored in env)
  const key = await deriveKey();
  
  // Generate random IV
  const iv = crypto.getRandomValues(new Uint8Array(12));
  
  // Encrypt
  const encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    dataBuffer
  );
  
  // Combine IV + encrypted data
  const combined = new Uint8Array(iv.length + encrypted.byteLength);
  combined.set(iv);
  combined.set(new Uint8Array(encrypted), iv.length);
  
  return btoa(String.fromCharCode(...combined));
}

async function decrypt(encryptedData: string): Promise<string> {
  const combined = Uint8Array.from(atob(encryptedData), c => c.charCodeAt(0));
  
  const iv = combined.slice(0, 12);
  const data = combined.slice(12);
  
  const key = await deriveKey();
  
  const decrypted = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv },
    key,
    data
  );
  
  return new TextDecoder().decode(decrypted);
}

export const secureStorage = {
  async setItem(key: string, value: string) {
    if (!isSecureContext) {
      console.warn('Not in secure context - storage may be compromised');
    }
    
    const encrypted = await encrypt(value);
    localStorage.setItem(key, encrypted);
  },
  
  async getItem(key: string): Promise<string | null> {
    const encrypted = localStorage.getItem(key);
    if (!encrypted) return null;
    
    try {
      return await decrypt(encrypted);
    } catch {
      // Corrupted data - clear it
      localStorage.removeItem(key);
      return null;
    }
  },
  
  removeItem(key: string) {
    localStorage.removeItem(key);
  },
  
  clear() {
    localStorage.clear();
  },
};
```

### 3.3 Session Security

```typescript
// hooks/useSessionSecurity.ts
import { useEffect, useCallback } from 'react';
import { useAuthStore } from '@/stores/authStore';
import { tokenManager } from '@/lib/security/tokenManager';

const IDLE_TIMEOUT_MS = 30 * 60 * 1000; // 30 minutes
const WARNING_BEFORE_MS = 5 * 60 * 1000; // 5 minutes before

export function useSessionSecurity() {
  const { logout, refreshSession, isAuthenticated } = useAuthStore();
  
  // Activity tracking
  const resetIdleTimer = useCallback(() => {
    if (!isAuthenticated) return;
    
    // Update last activity timestamp
    sessionStorage.setItem('lastActivity', String(Date.now()));
  }, [isAuthenticated]);
  
  // Check session validity
  useEffect(() => {
    if (!isAuthenticated) return;
    
    const checkSession = () => {
      const lastActivity = sessionStorage.getItem('lastActivity');
      if (!lastActivity) return;
      
      const idleTime = Date.now() - parseInt(lastActivity);
      
      if (idleTime >= IDLE_TIMEOUT_MS) {
        // Session expired - logout
        logout('Session expired due to inactivity');
      } else if (idleTime >= IDLE_TIMEOUT_MS - WARNING_BEFORE_MS) {
        // Warn user
        // Show session expiry warning modal
      }
    };
    
    const interval = setInterval(checkSession, 60000); // Check every minute
    return () => clearInterval(interval);
  }, [isAuthenticated, logout]);
  
  // Track user activity
  useEffect(() => {
    const events = ['mousedown', 'keydown', 'scroll', 'touchstart'];
    
    events.forEach(event => {
      window.addEventListener(event, resetIdleTimer, { passive: true });
    });
    
    return () => {
      events.forEach(event => {
        window.removeEventListener(event, resetIdleTimer);
      });
    };
  }, [resetIdleTimer]);
  
  // Handle tab visibility
  useEffect(() => {
    const handleVisibilityChange = () => {
      if (document.visibilityState === 'visible') {
        // Check token validity when returning to tab
        if (tokenManager.isTokenExpiring()) {
          refreshSession();
        }
      }
    };
    
    document.addEventListener('visibilitychange', handleVisibilityChange);
    return () => document.removeEventListener('visibilitychange', handleVisibilityChange);
  }, [refreshSession]);
}
```

---

## 4. XSS Protection

### 4.1 Input Sanitization

```typescript
// lib/security/sanitize.ts
import DOMPurify from 'dompurify';

// Configure DOMPurify
DOMPurify.setConfig({
  ALLOWED_TAGS: [
    'p', 'br', 'strong', 'em', 'u', 's', 'h1', 'h2', 'h3', 'h4',
    'ul', 'ol', 'li', 'blockquote', 'a', 'img', 'pre', 'code'
  ],
  ALLOWED_ATTR: ['href', 'src', 'alt', 'title', 'class'],
  ALLOW_DATA_ATTR: false,
  FORBID_TAGS: ['script', 'style', 'iframe', 'object', 'embed', 'form'],
  FORBID_ATTR: ['onerror', 'onclick', 'onload', 'onmouseover'],
});

// Sanitize HTML content (for rich text)
export function sanitizeHtml(dirty: string): string {
  return DOMPurify.sanitize(dirty, {
    RETURN_TRUSTED_TYPE: false,
  });
}

// Sanitize plain text (escape HTML entities)
export function sanitizeText(text: string): string {
  const div = document.createElement('div');
  div.textContent = text;
  return div.innerHTML;
}

// Sanitize URL (prevent javascript: URLs)
export function sanitizeUrl(url: string): string {
  try {
    const parsed = new URL(url);
    
    // Only allow http, https, and mailto
    if (!['http:', 'https:', 'mailto:'].includes(parsed.protocol)) {
      return '#';
    }
    
    return url;
  } catch {
    // Invalid URL
    return '#';
  }
}

// Sanitize for JSON injection
export function sanitizeForJson(obj: unknown): unknown {
  if (typeof obj === 'string') {
    return obj
      .replace(/</g, '\\u003C')
      .replace(/>/g, '\\u003E')
      .replace(/&/g, '\\u0026');
  }
  
  if (Array.isArray(obj)) {
    return obj.map(sanitizeForJson);
  }
  
  if (typeof obj === 'object' && obj !== null) {
    return Object.fromEntries(
      Object.entries(obj).map(([k, v]) => [k, sanitizeForJson(v)])
    );
  }
  
  return obj;
}
```

### 4.2 Safe Content Rendering

```typescript
// components/common/SafeHtml/SafeHtml.tsx
import { sanitizeHtml } from '@/lib/security/sanitize';
import { useMemo } from 'react';

interface SafeHtmlProps {
  html: string;
  className?: string;
  allowedTags?: string[];
}

export function SafeHtml({ html, className, allowedTags }: SafeHtmlProps) {
  const sanitizedHtml = useMemo(() => {
    return sanitizeHtml(html);
  }, [html]);
  
  return (
    <div
      className={className}
      dangerouslySetInnerHTML={{ __html: sanitizedHtml }}
    />
  );
}

// Safe text component (auto-escape)
export function SafeText({ children }: { children: string }) {
  // React automatically escapes text content
  return <>{children}</>;
}

// Link component with URL validation
export function SafeLink({ 
  href, 
  children,
  external = false,
  ...props 
}: React.AnchorHTMLAttributes<HTMLAnchorElement> & { external?: boolean }) {
  const safeHref = useMemo(() => sanitizeUrl(href || '#'), [href]);
  
  return (
    <a
      href={safeHref}
      {...(external && {
        target: '_blank',
        rel: 'noopener noreferrer',
      })}
      {...props}
    >
      {children}
    </a>
  );
}
```

---

## 5. CSRF Protection

### 5.1 CSRF Token Management

```typescript
// lib/security/csrf.ts

const CSRF_COOKIE_NAME = 'XSRF-TOKEN';
const CSRF_HEADER_NAME = 'X-XSRF-TOKEN';

// Get CSRF token from cookie
export function getCsrfToken(): string | null {
  const cookies = document.cookie.split(';');
  
  for (const cookie of cookies) {
    const [name, value] = cookie.trim().split('=');
    if (name === CSRF_COOKIE_NAME) {
      return decodeURIComponent(value);
    }
  }
  
  return null;
}

// Add CSRF token to requests
export function addCsrfHeader(headers: Headers): Headers {
  const token = getCsrfToken();
  
  if (token) {
    headers.set(CSRF_HEADER_NAME, token);
  }
  
  return headers;
}
```

### 5.2 Axios CSRF Interceptor

```typescript
// lib/api/interceptors/csrfInterceptor.ts
import { AxiosInstance } from 'axios';
import { getCsrfToken } from '@/lib/security/csrf';

export function setupCsrfInterceptor(api: AxiosInstance) {
  api.interceptors.request.use((config) => {
    // Add CSRF token to mutating requests
    if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(config.method?.toUpperCase() || '')) {
      const token = getCsrfToken();
      
      if (token) {
        config.headers['X-XSRF-TOKEN'] = token;
      }
    }
    
    return config;
  });
}
```

---

## 6. Content Security Policy

### 6.1 CSP Configuration

```typescript
// CSP directives for N9 platform
export const cspDirectives = {
  'default-src': ["'self'"],
  'script-src': [
    "'self'",
    "'strict-dynamic'",
    `'nonce-${generateNonce()}'`,
    // Required for analytics
    'https://www.google-analytics.com',
    'https://www.googletagmanager.com',
  ],
  'style-src': [
    "'self'",
    "'unsafe-inline'", // Required for Tailwind CSS
  ],
  'img-src': [
    "'self'",
    'data:',
    'blob:',
    'https://cdn.n9.com',      // CDN
    'https://*.cloudinary.com', // Image service
  ],
  'font-src': [
    "'self'",
    'https://fonts.gstatic.com',
  ],
  'connect-src': [
    "'self'",
    'https://api.n9.com',
    'wss://ws.n9.com',         // WebSocket
    'https://www.google-analytics.com',
  ],
  'media-src': [
    "'self'",
    'https://cdn.n9.com',
  ],
  'object-src': ["'none'"],
  'frame-src': [
    "'self'",
    'https://www.google.com', // reCAPTCHA
  ],
  'base-uri': ["'self'"],
  'form-action': ["'self'"],
  'frame-ancestors': ["'none'"],
  'upgrade-insecure-requests': [],
};
```

---

## 7. Authorization

### 7.1 Role-Based Access Control

```typescript
// lib/security/rbac.ts

export enum UserRole {
  GUEST = 'GUEST',
  READER = 'READER',
  AUTHOR = 'AUTHOR',
  MODERATOR = 'MODERATOR',
  ADMIN = 'ADMIN',
}

export enum Permission {
  // Stories
  READ_STORIES = 'READ_STORIES',
  CREATE_STORY = 'CREATE_STORY',
  EDIT_OWN_STORY = 'EDIT_OWN_STORY',
  DELETE_OWN_STORY = 'DELETE_OWN_STORY',
  
  // Chapters
  READ_FREE_CHAPTERS = 'READ_FREE_CHAPTERS',
  READ_PREMIUM_CHAPTERS = 'READ_PREMIUM_CHAPTERS',
  CREATE_CHAPTER = 'CREATE_CHAPTER',
  
  // Comments
  CREATE_COMMENT = 'CREATE_COMMENT',
  MODERATE_COMMENTS = 'MODERATE_COMMENTS',
  
  // Users
  MANAGE_USERS = 'MANAGE_USERS',
  MANAGE_REPORTS = 'MANAGE_REPORTS',
  
  // System
  ACCESS_ADMIN = 'ACCESS_ADMIN',
  MANAGE_SYSTEM = 'MANAGE_SYSTEM',
}

// Permission matrix
const rolePermissions: Record<UserRole, Permission[]> = {
  [UserRole.GUEST]: [
    Permission.READ_STORIES,
    Permission.READ_FREE_CHAPTERS,
  ],
  
  [UserRole.READER]: [
    Permission.READ_STORIES,
    Permission.READ_FREE_CHAPTERS,
    Permission.READ_PREMIUM_CHAPTERS,
    Permission.CREATE_COMMENT,
  ],
  
  [UserRole.AUTHOR]: [
    Permission.READ_STORIES,
    Permission.READ_FREE_CHAPTERS,
    Permission.READ_PREMIUM_CHAPTERS,
    Permission.CREATE_COMMENT,
    Permission.CREATE_STORY,
    Permission.EDIT_OWN_STORY,
    Permission.DELETE_OWN_STORY,
    Permission.CREATE_CHAPTER,
  ],
  
  [UserRole.MODERATOR]: [
    Permission.READ_STORIES,
    Permission.READ_FREE_CHAPTERS,
    Permission.READ_PREMIUM_CHAPTERS,
    Permission.CREATE_COMMENT,
    Permission.MODERATE_COMMENTS,
    Permission.MANAGE_REPORTS,
  ],
  
  [UserRole.ADMIN]: Object.values(Permission),
};

// Check permission
export function hasPermission(role: UserRole, permission: Permission): boolean {
  return rolePermissions[role]?.includes(permission) ?? false;
}

// Check multiple permissions
export function hasAllPermissions(role: UserRole, permissions: Permission[]): boolean {
  return permissions.every(p => hasPermission(role, p));
}

export function hasAnyPermission(role: UserRole, permissions: Permission[]): boolean {
  return permissions.some(p => hasPermission(role, p));
}
```

### 7.2 Permission Components

```typescript
// components/auth/Permission/Permission.tsx
import { useAuth } from '@/hooks/useAuth';
import { Permission, hasPermission, hasAllPermissions, hasAnyPermission } from '@/lib/security/rbac';

interface PermissionGateProps {
  permission?: Permission;
  permissions?: Permission[];
  require?: 'all' | 'any';
  fallback?: React.ReactNode;
  children: React.ReactNode;
}

export function PermissionGate({
  permission,
  permissions,
  require = 'all',
  fallback = null,
  children,
}: PermissionGateProps) {
  const { user } = useAuth();
  
  if (!user) return <>{fallback}</>;
  
  const hasAccess = (() => {
    if (permission) {
      return hasPermission(user.role, permission);
    }
    
    if (permissions) {
      return require === 'all'
        ? hasAllPermissions(user.role, permissions)
        : hasAnyPermission(user.role, permissions);
    }
    
    return false;
  })();
  
  return hasAccess ? <>{children}</> : <>{fallback}</>;
}

// Hook for permission checks
export function usePermission(permission: Permission): boolean {
  const { user } = useAuth();
  
  if (!user) return false;
  return hasPermission(user.role, permission);
}

// Usage
function StoryActions({ story }: { story: Story }) {
  const canEdit = usePermission(Permission.EDIT_OWN_STORY);
  
  return (
    <div>
      <PermissionGate permission={Permission.EDIT_OWN_STORY}>
        <Button onClick={() => editStory(story)}>Edit</Button>
      </PermissionGate>
      
      <PermissionGate permission={Permission.MODERATE_COMMENTS}>
        <Button onClick={() => moderateStory(story)}>Moderate</Button>
      </PermissionGate>
    </div>
  );
}
```

---

## 8. Data Protection

### 8.1 Sensitive Data Handling

```typescript
// lib/security/dataProtection.ts

// Mask sensitive data for display
export function maskEmail(email: string): string {
  const [local, domain] = email.split('@');
  if (local.length <= 2) {
    return `${local[0]}***@${domain}`;
  }
  return `${local.slice(0, 2)}***@${domain}`;
}

export function maskPhone(phone: string): string {
  const digits = phone.replace(/\D/g, '');
  return `***-***-${digits.slice(-4)}`;
}

export function maskCardNumber(cardNumber: string): string {
  const digits = cardNumber.replace(/\D/g, '');
  return `****-****-****-${digits.slice(-4)}`;
}

// Redact sensitive fields from objects (for logging)
const SENSITIVE_FIELDS = ['password', 'token', 'secret', 'creditCard', 'ssn'];

export function redactSensitiveData(obj: Record<string, unknown>): Record<string, unknown> {
  const redacted: Record<string, unknown> = {};
  
  for (const [key, value] of Object.entries(obj)) {
    if (SENSITIVE_FIELDS.some(field => key.toLowerCase().includes(field))) {
      redacted[key] = '[REDACTED]';
    } else if (typeof value === 'object' && value !== null) {
      redacted[key] = redactSensitiveData(value as Record<string, unknown>);
    } else {
      redacted[key] = value;
    }
  }
  
  return redacted;
}
```

### 8.2 Privacy Controls

```typescript
// components/privacy/PrivacySettings/PrivacySettings.tsx
import { useState } from 'react';
import { usePrivacySettings } from '@/hooks/usePrivacySettings';

export function PrivacySettings() {
  const { settings, updateSettings, isLoading } = usePrivacySettings();
  
  return (
    <div className="space-y-6">
      <h2 className="text-xl font-semibold">Privacy Settings</h2>
      
      <div className="space-y-4">
        {/* Profile visibility */}
        <div className="flex items-center justify-between">
          <div>
            <p className="font-medium">Profile Visibility</p>
            <p className="text-sm text-gray-500">
              Control who can see your profile
            </p>
          </div>
          <Select
            value={settings.profileVisibility}
            onChange={(v) => updateSettings({ profileVisibility: v })}
            options={[
              { value: 'PUBLIC', label: 'Everyone' },
              { value: 'FOLLOWERS', label: 'Followers Only' },
              { value: 'PRIVATE', label: 'Only Me' },
            ]}
          />
        </div>
        
        {/* Reading history */}
        <div className="flex items-center justify-between">
          <div>
            <p className="font-medium">Reading History</p>
            <p className="text-sm text-gray-500">
              Show your reading activity
            </p>
          </div>
          <Switch
            checked={settings.showReadingHistory}
            onChange={(v) => updateSettings({ showReadingHistory: v })}
          />
        </div>
        
        {/* Library visibility */}
        <div className="flex items-center justify-between">
          <div>
            <p className="font-medium">Library Visibility</p>
            <p className="text-sm text-gray-500">
              Allow others to see your library
            </p>
          </div>
          <Switch
            checked={settings.showLibrary}
            onChange={(v) => updateSettings({ showLibrary: v })}
          />
        </div>
        
        {/* Data export */}
        <div className="border-t pt-4">
          <Button variant="outline" onClick={() => requestDataExport()}>
            Request Data Export
          </Button>
          <p className="mt-2 text-sm text-gray-500">
            Download all your data in a portable format
          </p>
        </div>
        
        {/* Account deletion */}
        <div className="border-t pt-4">
          <Button variant="destructive" onClick={() => requestAccountDeletion()}>
            Delete Account
          </Button>
          <p className="mt-2 text-sm text-gray-500">
            Permanently delete your account and all associated data
          </p>
        </div>
      </div>
    </div>
  );
}
```

---

## 9. Security Headers

### 9.1 Recommended Headers

```typescript
// Security headers configuration
export const securityHeaders = {
  // Prevent clickjacking
  'X-Frame-Options': 'DENY',
  
  // Prevent MIME type sniffing
  'X-Content-Type-Options': 'nosniff',
  
  // XSS protection (legacy browsers)
  'X-XSS-Protection': '1; mode=block',
  
  // Referrer policy
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  
  // Permissions policy
  'Permissions-Policy': [
    'camera=()',
    'microphone=()',
    'geolocation=()',
    'payment=(self)',
  ].join(', '),
  
  // HSTS (HTTP Strict Transport Security)
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains; preload',
};
```

---

## 10. Compliance Requirements

### 10.1 GDPR Compliance

| Requirement | Implementation |
|-------------|----------------|
| **Consent** | Cookie consent banner with granular controls |
| **Right to Access** | Data export feature in account settings |
| **Right to Erasure** | Account deletion with 30-day grace period |
| **Data Portability** | JSON/CSV export of user data |
| **Privacy by Design** | Minimal data collection, encryption at rest |
| **Breach Notification** | Automated alerting system |

### 10.2 Cookie Consent

```typescript
// components/privacy/CookieConsent/CookieConsent.tsx
import { useState, useEffect } from 'react';
import { getCookieConsent, setCookieConsent, CookiePreferences } from '@/lib/cookies';

export function CookieConsent() {
  const [showBanner, setShowBanner] = useState(false);
  const [preferences, setPreferences] = useState<CookiePreferences>({
    necessary: true, // Always required
    analytics: false,
    marketing: false,
    personalization: false,
  });
  
  useEffect(() => {
    const consent = getCookieConsent();
    if (!consent) {
      setShowBanner(true);
    }
  }, []);
  
  const handleAcceptAll = () => {
    const allAccepted: CookiePreferences = {
      necessary: true,
      analytics: true,
      marketing: true,
      personalization: true,
    };
    setCookieConsent(allAccepted);
    setShowBanner(false);
  };
  
  const handleAcceptSelected = () => {
    setCookieConsent(preferences);
    setShowBanner(false);
  };
  
  if (!showBanner) return null;
  
  return (
    <div className="fixed bottom-0 inset-x-0 bg-white border-t shadow-lg p-4 z-50">
      <div className="max-w-7xl mx-auto">
        <h3 className="font-semibold">Cookie Preferences</h3>
        <p className="text-sm text-gray-600 mt-1">
          We use cookies to enhance your experience. Please select your preferences.
        </p>
        
        <div className="mt-4 grid grid-cols-2 md:grid-cols-4 gap-4">
          <label className="flex items-center">
            <input type="checkbox" checked disabled />
            <span className="ml-2 text-sm">Necessary</span>
          </label>
          <label className="flex items-center">
            <input 
              type="checkbox" 
              checked={preferences.analytics}
              onChange={(e) => setPreferences(p => ({ ...p, analytics: e.target.checked }))}
            />
            <span className="ml-2 text-sm">Analytics</span>
          </label>
          <label className="flex items-center">
            <input 
              type="checkbox" 
              checked={preferences.marketing}
              onChange={(e) => setPreferences(p => ({ ...p, marketing: e.target.checked }))}
            />
            <span className="ml-2 text-sm">Marketing</span>
          </label>
          <label className="flex items-center">
            <input 
              type="checkbox" 
              checked={preferences.personalization}
              onChange={(e) => setPreferences(p => ({ ...p, personalization: e.target.checked }))}
            />
            <span className="ml-2 text-sm">Personalization</span>
          </label>
        </div>
        
        <div className="mt-4 flex gap-2">
          <Button onClick={handleAcceptSelected}>Accept Selected</Button>
          <Button variant="primary" onClick={handleAcceptAll}>Accept All</Button>
        </div>
      </div>
    </div>
  );
}
```

---

## 11. Security Checklist

### 11.1 Pre-Launch Security Checklist

- [ ] HTTPS enforced on all pages
- [ ] CSP headers configured
- [ ] XSS protection implemented
- [ ] CSRF protection enabled
- [ ] Input validation on all forms
- [ ] Output encoding for dynamic content
- [ ] Secure cookie flags set
- [ ] JWT tokens short-lived
- [ ] Refresh token rotation enabled
- [ ] Rate limiting configured
- [ ] Error messages sanitized
- [ ] Sensitive data encrypted
- [ ] GDPR compliance verified
- [ ] Security audit completed

---

## 12. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Security Team | Initial release |

---

## 13. Related Documents

- [02_FRONTEND_ARCHITECTURE.md](./02_FRONTEND_ARCHITECTURE.md) - Architecture patterns
- [06_API_INTEGRATION.md](./06_API_INTEGRATION.md) - API security
- [05_ROUTING_NAVIGATION.md](./05_ROUTING_NAVIGATION.md) - Route guards
