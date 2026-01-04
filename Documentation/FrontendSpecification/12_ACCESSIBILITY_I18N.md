# Accessibility & Internationalization Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | UX/Frontend Team |
| Review Cycle | Quarterly |
| Related Documents | 03_DESIGN_SYSTEM.md, 07_COMPONENTS_LIBRARY.md |

---

## 2. Accessibility Standards

### 2.1 WCAG Compliance

| Level | Target | Description |
|-------|--------|-------------|
| **WCAG 2.1 Level AA** | Primary | All public pages |
| **WCAG 2.1 Level AAA** | Stretch | Reading experience |

### 2.2 Accessibility Principles

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ACCESSIBILITY PRINCIPLES (POUR)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ P - PERCEIVABLE                                                         ││
│  │ • Text alternatives for non-text content                                ││
│  │ • Captions and audio descriptions                                       ││
│  │ • Content distinguishable without color alone                           ││
│  │ • Sufficient color contrast                                             ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ O - OPERABLE                                                            ││
│  │ • Full keyboard accessibility                                           ││
│  │ • Sufficient time to read content                                       ││
│  │ • No content that causes seizures                                       ││
│  │ • Clear navigation and wayfinding                                       ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ U - UNDERSTANDABLE                                                      ││
│  │ • Readable text content                                                 ││
│  │ • Predictable behavior                                                  ││
│  │ • Input assistance and error prevention                                 ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ R - ROBUST                                                              ││
│  │ • Compatible with assistive technologies                                ││
│  │ • Valid, semantic HTML                                                  ││
│  │ • ARIA used correctly                                                   ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Color Accessibility

### 3.1 Contrast Requirements

| Use Case | Ratio | WCAG Level |
|----------|-------|------------|
| Normal text | 4.5:1 | AA |
| Large text (18px+ bold, 24px+) | 3:1 | AA |
| UI components | 3:1 | AA |
| Enhanced text | 7:1 | AAA |

### 3.2 Color Palette Accessibility

```typescript
// lib/accessibility/colorContrast.ts

// Verified accessible color combinations
export const accessibleColors = {
  light: {
    text: {
      primary: '#1a1a1a',     // 16:1 on white
      secondary: '#525252',   // 7:1 on white
      tertiary: '#737373',    // 4.6:1 on white
    },
    background: {
      primary: '#ffffff',
      secondary: '#f5f5f5',
      accent: '#f0f9ff',
    },
    border: '#d4d4d4',        // 3:1 for UI components
  },
  
  dark: {
    text: {
      primary: '#fafafa',     // 17:1 on dark
      secondary: '#a1a1aa',   // 7:1 on dark
      tertiary: '#71717a',    // 4.5:1 on dark
    },
    background: {
      primary: '#09090b',
      secondary: '#18181b',
      accent: '#1e3a5f',
    },
    border: '#3f3f46',        // 3:1 for UI components
  },
  
  sepia: {
    text: {
      primary: '#3d2914',     // 12:1 on sepia
      secondary: '#5c4033',   // 7:1 on sepia
      tertiary: '#7a5c42',    // 4.5:1 on sepia
    },
    background: {
      primary: '#f5f0e6',
      secondary: '#ebe5d8',
      accent: '#e6dfd0',
    },
    border: '#c4b89a',        // 3:1 for UI components
  },
};

// Utility to check contrast ratio
export function getContrastRatio(fg: string, bg: string): number {
  const fgLum = getLuminance(fg);
  const bgLum = getLuminance(bg);
  const lighter = Math.max(fgLum, bgLum);
  const darker = Math.min(fgLum, bgLum);
  return (lighter + 0.05) / (darker + 0.05);
}

function getLuminance(hex: string): number {
  const rgb = hexToRgb(hex);
  const [r, g, b] = rgb.map(c => {
    c = c / 255;
    return c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4);
  });
  return 0.2126 * r + 0.7152 * g + 0.0722 * b;
}
```

### 3.3 Non-Color Indicators

```tsx
// Always provide non-color indicators

// ❌ Bad - Color only
<span className={error ? 'text-red-500' : 'text-green-500'}>
  {status}
</span>

// ✅ Good - Color + icon + text
<span className={cn(
  error ? 'text-red-500' : 'text-green-500',
  'flex items-center gap-1'
)}>
  {error ? <AlertCircle size={16} /> : <CheckCircle size={16} />}
  <span>{error ? 'Error' : 'Success'}: {status}</span>
</span>
```

---

## 4. Keyboard Navigation

### 4.1 Focus Management

```typescript
// hooks/useFocusManagement.ts
import { useRef, useEffect } from 'react';

// Focus trap for modals and dialogs
export function useFocusTrap(isActive: boolean) {
  const containerRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (!isActive || !containerRef.current) return;
    
    const container = containerRef.current;
    const focusableElements = container.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    
    const firstElement = focusableElements[0] as HTMLElement;
    const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement;
    
    // Focus first element
    firstElement?.focus();
    
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      
      if (e.shiftKey) {
        if (document.activeElement === firstElement) {
          e.preventDefault();
          lastElement?.focus();
        }
      } else {
        if (document.activeElement === lastElement) {
          e.preventDefault();
          firstElement?.focus();
        }
      }
    };
    
    container.addEventListener('keydown', handleKeyDown);
    return () => container.removeEventListener('keydown', handleKeyDown);
  }, [isActive]);
  
  return containerRef;
}

// Restore focus after modal closes
export function useRestoreFocus() {
  const previousFocusRef = useRef<HTMLElement | null>(null);
  
  useEffect(() => {
    previousFocusRef.current = document.activeElement as HTMLElement;
    
    return () => {
      previousFocusRef.current?.focus();
    };
  }, []);
}
```

### 4.2 Skip Links

```tsx
// components/layout/SkipLinks/SkipLinks.tsx
export function SkipLinks() {
  return (
    <nav 
      aria-label="Skip links"
      className="sr-only focus-within:not-sr-only"
    >
      <ul className="fixed top-0 left-0 z-50 flex gap-2 p-2 bg-white shadow-lg">
        <li>
          <a 
            href="#main-content"
            className="px-4 py-2 bg-primary text-white rounded focus:outline-none focus:ring-2"
          >
            Skip to main content
          </a>
        </li>
        <li>
          <a 
            href="#main-navigation"
            className="px-4 py-2 bg-primary text-white rounded focus:outline-none focus:ring-2"
          >
            Skip to navigation
          </a>
        </li>
        <li>
          <a 
            href="#search"
            className="px-4 py-2 bg-primary text-white rounded focus:outline-none focus:ring-2"
          >
            Skip to search
          </a>
        </li>
      </ul>
    </nav>
  );
}
```

### 4.3 Keyboard Shortcuts

```typescript
// hooks/useKeyboardShortcuts.ts
import { useEffect, useCallback } from 'react';
import { useNavigate } from 'react-router-dom';

interface Shortcut {
  key: string;
  ctrl?: boolean;
  shift?: boolean;
  alt?: boolean;
  action: () => void;
  description: string;
}

export function useKeyboardShortcuts() {
  const navigate = useNavigate();
  
  const shortcuts: Shortcut[] = [
    // Navigation
    { key: 'h', alt: true, action: () => navigate('/'), description: 'Go to Home' },
    { key: 'b', alt: true, action: () => navigate('/browse'), description: 'Go to Browse' },
    { key: 'l', alt: true, action: () => navigate('/library'), description: 'Go to Library' },
    
    // Reader controls
    { key: 'ArrowLeft', action: () => navigateChapter('prev'), description: 'Previous chapter' },
    { key: 'ArrowRight', action: () => navigateChapter('next'), description: 'Next chapter' },
    { key: 's', alt: true, action: () => toggleReaderSettings(), description: 'Reader settings' },
    
    // Actions
    { key: '/', ctrl: true, action: () => focusSearch(), description: 'Focus search' },
    { key: 'Escape', action: () => closeModals(), description: 'Close modals' },
    { key: '?', shift: true, action: () => showShortcutHelp(), description: 'Show shortcuts' },
  ];
  
  const handleKeyDown = useCallback((e: KeyboardEvent) => {
    // Don't trigger in inputs
    if (['INPUT', 'TEXTAREA', 'SELECT'].includes((e.target as HTMLElement).tagName)) {
      return;
    }
    
    for (const shortcut of shortcuts) {
      if (
        e.key === shortcut.key &&
        !!e.ctrlKey === !!shortcut.ctrl &&
        !!e.shiftKey === !!shortcut.shift &&
        !!e.altKey === !!shortcut.alt
      ) {
        e.preventDefault();
        shortcut.action();
        return;
      }
    }
  }, [shortcuts]);
  
  useEffect(() => {
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [handleKeyDown]);
  
  return shortcuts;
}
```

---

## 5. ARIA Implementation

### 5.1 Landmark Regions

```tsx
// Layout structure with landmarks
function AppLayout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <SkipLinks />
      
      <header role="banner">
        <nav id="main-navigation" aria-label="Main navigation">
          {/* Navigation content */}
        </nav>
      </header>
      
      <main id="main-content" role="main" tabIndex={-1}>
        {children}
      </main>
      
      <aside role="complementary" aria-label="Related content">
        {/* Sidebar content */}
      </aside>
      
      <footer role="contentinfo">
        {/* Footer content */}
      </footer>
    </>
  );
}
```

### 5.2 Accessible Components

```tsx
// components/common/Accordion/Accordion.tsx
import { useState } from 'react';

interface AccordionItemProps {
  id: string;
  title: string;
  children: React.ReactNode;
}

export function AccordionItem({ id, title, children }: AccordionItemProps) {
  const [isExpanded, setIsExpanded] = useState(false);
  const headerId = `accordion-header-${id}`;
  const panelId = `accordion-panel-${id}`;
  
  return (
    <div className="border-b">
      <h3>
        <button
          id={headerId}
          aria-expanded={isExpanded}
          aria-controls={panelId}
          onClick={() => setIsExpanded(!isExpanded)}
          className="w-full flex justify-between items-center p-4 text-left"
        >
          <span>{title}</span>
          <ChevronDown 
            className={cn(
              'transition-transform',
              isExpanded && 'rotate-180'
            )}
            aria-hidden="true"
          />
        </button>
      </h3>
      <div
        id={panelId}
        role="region"
        aria-labelledby={headerId}
        hidden={!isExpanded}
        className="p-4"
      >
        {children}
      </div>
    </div>
  );
}
```

### 5.3 Live Regions

```tsx
// components/common/LiveAnnouncer/LiveAnnouncer.tsx
import { createContext, useContext, useState, useCallback, useRef } from 'react';

interface Announcement {
  message: string;
  politeness: 'polite' | 'assertive';
}

const AnnouncerContext = createContext<{
  announce: (message: string, politeness?: 'polite' | 'assertive') => void;
}>({ announce: () => {} });

export function LiveAnnouncerProvider({ children }: { children: React.ReactNode }) {
  const [politeMessage, setPoliteMessage] = useState('');
  const [assertiveMessage, setAssertiveMessage] = useState('');
  
  const announce = useCallback((
    message: string, 
    politeness: 'polite' | 'assertive' = 'polite'
  ) => {
    if (politeness === 'assertive') {
      setAssertiveMessage(message);
      setTimeout(() => setAssertiveMessage(''), 1000);
    } else {
      setPoliteMessage(message);
      setTimeout(() => setPoliteMessage(''), 1000);
    }
  }, []);
  
  return (
    <AnnouncerContext.Provider value={{ announce }}>
      {children}
      
      {/* Polite announcements */}
      <div
        role="status"
        aria-live="polite"
        aria-atomic="true"
        className="sr-only"
      >
        {politeMessage}
      </div>
      
      {/* Assertive announcements */}
      <div
        role="alert"
        aria-live="assertive"
        aria-atomic="true"
        className="sr-only"
      >
        {assertiveMessage}
      </div>
    </AnnouncerContext.Provider>
  );
}

export function useAnnounce() {
  return useContext(AnnouncerContext).announce;
}

// Usage
function LoadMoreButton() {
  const announce = useAnnounce();
  
  const handleLoadMore = async () => {
    await loadMore();
    announce('10 more stories loaded');
  };
  
  return (
    <button onClick={handleLoadMore}>
      Load More
    </button>
  );
}
```

---

## 6. Screen Reader Support

### 6.1 Descriptive Elements

```tsx
// Proper labeling for interactive elements

// Image
<img
  src={story.coverImage}
  alt={`Cover image for ${story.title} by ${story.author.name}`}
/>

// Decorative image
<img src="/decorative-line.svg" alt="" role="presentation" />

// Icon button
<button aria-label="Add to bookmarks">
  <BookmarkIcon aria-hidden="true" />
</button>

// Link with context
<a href={`/story/${story.slug}`}>
  <span className="sr-only">Read story: </span>
  {story.title}
</a>

// Complex stat
<div>
  <span aria-hidden="true">👁️</span>
  <span className="sr-only">Views: </span>
  <span>{formatNumber(story.views)}</span>
</div>
```

### 6.2 Form Accessibility

```tsx
// components/form/FormField/FormField.tsx
interface FormFieldProps {
  id: string;
  label: string;
  error?: string;
  hint?: string;
  required?: boolean;
  children: React.ReactElement;
}

export function FormField({
  id,
  label,
  error,
  hint,
  required,
  children,
}: FormFieldProps) {
  const hintId = hint ? `${id}-hint` : undefined;
  const errorId = error ? `${id}-error` : undefined;
  const describedBy = [hintId, errorId].filter(Boolean).join(' ') || undefined;
  
  return (
    <div className="space-y-1">
      <label htmlFor={id} className="block text-sm font-medium">
        {label}
        {required && (
          <span aria-hidden="true" className="text-red-500 ml-1">*</span>
        )}
        {required && <span className="sr-only">(required)</span>}
      </label>
      
      {hint && (
        <p id={hintId} className="text-sm text-gray-500">
          {hint}
        </p>
      )}
      
      {React.cloneElement(children, {
        id,
        'aria-invalid': !!error,
        'aria-describedby': describedBy,
        'aria-required': required,
      })}
      
      {error && (
        <p id={errorId} role="alert" className="text-sm text-red-600">
          {error}
        </p>
      )}
    </div>
  );
}
```

---

## 7. Internationalization (i18n)

### 7.1 i18n Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        I18N ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ SUPPORTED LANGUAGES                                                     ││
│  │                                                                          ││
│  │ • English (en) - Default                                                ││
│  │ • Vietnamese (vi)                                                       ││
│  │ • Chinese Simplified (zh-CN)                                            ││
│  │ • Japanese (ja)                                                         ││
│  │ • Korean (ko)                                                           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ LOCALE FEATURES                                                         ││
│  │                                                                          ││
│  │ • UI translations                                                       ││
│  │ • Date/time formatting                                                  ││
│  │ • Number formatting                                                     ││
│  │ • Currency formatting                                                   ││
│  │ • Pluralization rules                                                   ││
│  │ • RTL support ready                                                     ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 i18n Configuration

```typescript
// lib/i18n/config.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';
import Backend from 'i18next-http-backend';

export const supportedLanguages = {
  en: { name: 'English', nativeName: 'English', dir: 'ltr' },
  vi: { name: 'Vietnamese', nativeName: 'Tiếng Việt', dir: 'ltr' },
  'zh-CN': { name: 'Chinese (Simplified)', nativeName: '简体中文', dir: 'ltr' },
  ja: { name: 'Japanese', nativeName: '日本語', dir: 'ltr' },
  ko: { name: 'Korean', nativeName: '한국어', dir: 'ltr' },
};

i18n
  .use(Backend)
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    fallbackLng: 'en',
    supportedLngs: Object.keys(supportedLanguages),
    
    ns: ['common', 'auth', 'stories', 'reader', 'author', 'wallet'],
    defaultNS: 'common',
    
    backend: {
      loadPath: '/locales/{{lng}}/{{ns}}.json',
    },
    
    detection: {
      order: ['localStorage', 'navigator'],
      caches: ['localStorage'],
    },
    
    interpolation: {
      escapeValue: false, // React escapes by default
    },
    
    react: {
      useSuspense: true,
    },
  });

export default i18n;
```

### 7.3 Translation Files

```json
// public/locales/en/common.json
{
  "nav": {
    "home": "Home",
    "browse": "Browse",
    "library": "Library",
    "search": "Search",
    "login": "Sign In",
    "signup": "Sign Up"
  },
  "actions": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "edit": "Edit",
    "submit": "Submit",
    "loading": "Loading...",
    "retry": "Retry"
  },
  "errors": {
    "generic": "Something went wrong. Please try again.",
    "network": "Network error. Please check your connection.",
    "notFound": "Page not found",
    "unauthorized": "Please sign in to continue"
  },
  "time": {
    "justNow": "Just now",
    "minutesAgo": "{{count}} minute ago",
    "minutesAgo_other": "{{count}} minutes ago",
    "hoursAgo": "{{count}} hour ago",
    "hoursAgo_other": "{{count}} hours ago",
    "daysAgo": "{{count}} day ago",
    "daysAgo_other": "{{count}} days ago"
  },
  "numbers": {
    "views": "{{count}} view",
    "views_other": "{{count}} views",
    "likes": "{{count}} like",
    "likes_other": "{{count}} likes",
    "chapters": "{{count}} chapter",
    "chapters_other": "{{count}} chapters"
  }
}
```

```json
// public/locales/vi/common.json
{
  "nav": {
    "home": "Trang chủ",
    "browse": "Khám phá",
    "library": "Thư viện",
    "search": "Tìm kiếm",
    "login": "Đăng nhập",
    "signup": "Đăng ký"
  },
  "actions": {
    "save": "Lưu",
    "cancel": "Hủy",
    "delete": "Xóa",
    "edit": "Chỉnh sửa",
    "submit": "Gửi",
    "loading": "Đang tải...",
    "retry": "Thử lại"
  },
  "errors": {
    "generic": "Đã có lỗi xảy ra. Vui lòng thử lại.",
    "network": "Lỗi kết nối. Vui lòng kiểm tra kết nối mạng.",
    "notFound": "Không tìm thấy trang",
    "unauthorized": "Vui lòng đăng nhập để tiếp tục"
  }
}
```

### 7.4 Translation Components

```tsx
// Usage examples

// Basic translation
import { useTranslation } from 'react-i18next';

function NavBar() {
  const { t } = useTranslation();
  
  return (
    <nav>
      <Link to="/">{t('nav.home')}</Link>
      <Link to="/browse">{t('nav.browse')}</Link>
      <Link to="/library">{t('nav.library')}</Link>
    </nav>
  );
}

// With interpolation
function StoryStats({ views, likes }: { views: number; likes: number }) {
  const { t } = useTranslation();
  
  return (
    <div>
      <span>{t('numbers.views', { count: views })}</span>
      <span>{t('numbers.likes', { count: likes })}</span>
    </div>
  );
}

// Trans component for complex translations
import { Trans } from 'react-i18next';

function WelcomeMessage({ name }: { name: string }) {
  return (
    <Trans i18nKey="welcome.message" values={{ name }}>
      Welcome back, <strong>{{ name }}</strong>!
    </Trans>
  );
}
```

### 7.5 Language Switcher

```tsx
// components/common/LanguageSwitcher/LanguageSwitcher.tsx
import { useTranslation } from 'react-i18next';
import { supportedLanguages } from '@/lib/i18n/config';

export function LanguageSwitcher() {
  const { i18n } = useTranslation();
  
  const handleChange = (lang: string) => {
    i18n.changeLanguage(lang);
    document.documentElement.lang = lang;
    document.documentElement.dir = supportedLanguages[lang].dir;
  };
  
  return (
    <select
      value={i18n.language}
      onChange={(e) => handleChange(e.target.value)}
      aria-label="Select language"
      className="border rounded px-2 py-1"
    >
      {Object.entries(supportedLanguages).map(([code, { nativeName }]) => (
        <option key={code} value={code}>
          {nativeName}
        </option>
      ))}
    </select>
  );
}
```

---

## 8. Date & Number Formatting

### 8.1 Intl Formatters

```typescript
// lib/i18n/formatters.ts
import { useTranslation } from 'react-i18next';

// Date formatter hook
export function useDateFormatter() {
  const { i18n } = useTranslation();
  
  const formatDate = (date: Date | string, options?: Intl.DateTimeFormatOptions) => {
    const d = typeof date === 'string' ? new Date(date) : date;
    return new Intl.DateTimeFormat(i18n.language, {
      dateStyle: 'medium',
      ...options,
    }).format(d);
  };
  
  const formatRelative = (date: Date | string) => {
    const d = typeof date === 'string' ? new Date(date) : date;
    const now = new Date();
    const diffMs = now.getTime() - d.getTime();
    const diffSec = Math.floor(diffMs / 1000);
    const diffMin = Math.floor(diffSec / 60);
    const diffHour = Math.floor(diffMin / 60);
    const diffDay = Math.floor(diffHour / 24);
    
    const rtf = new Intl.RelativeTimeFormat(i18n.language, { numeric: 'auto' });
    
    if (diffSec < 60) return rtf.format(-diffSec, 'second');
    if (diffMin < 60) return rtf.format(-diffMin, 'minute');
    if (diffHour < 24) return rtf.format(-diffHour, 'hour');
    if (diffDay < 7) return rtf.format(-diffDay, 'day');
    
    return formatDate(d);
  };
  
  return { formatDate, formatRelative };
}

// Number formatter hook
export function useNumberFormatter() {
  const { i18n } = useTranslation();
  
  const formatNumber = (num: number, options?: Intl.NumberFormatOptions) => {
    return new Intl.NumberFormat(i18n.language, options).format(num);
  };
  
  const formatCompact = (num: number) => {
    return new Intl.NumberFormat(i18n.language, {
      notation: 'compact',
      maximumFractionDigits: 1,
    }).format(num);
  };
  
  const formatCurrency = (amount: number, currency = 'USD') => {
    return new Intl.NumberFormat(i18n.language, {
      style: 'currency',
      currency,
    }).format(amount);
  };
  
  return { formatNumber, formatCompact, formatCurrency };
}
```

---

## 9. Testing Accessibility

### 9.1 Automated Testing

```typescript
// tests/accessibility/a11y.test.ts
import { describe, it, expect } from 'vitest';
import { render } from '@/test/utils';
import { axe, toHaveNoViolations } from 'jest-axe';
import { HomePage } from '@/pages/home/HomePage';
import { LoginPage } from '@/pages/auth/LoginPage';
import { StoryDetailPage } from '@/pages/story/StoryDetailPage';

expect.extend(toHaveNoViolations);

describe('Accessibility', () => {
  it('HomePage has no accessibility violations', async () => {
    const { container } = render(<HomePage />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
  
  it('LoginPage has no accessibility violations', async () => {
    const { container } = render(<LoginPage />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
  
  it('StoryDetailPage has no accessibility violations', async () => {
    const { container } = render(<StoryDetailPage />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

### 9.2 Manual Testing Checklist

- [ ] Keyboard navigation works throughout
- [ ] Focus visible on all interactive elements
- [ ] Screen reader announces content correctly
- [ ] Skip links work
- [ ] Forms have proper labels and error messages
- [ ] Color contrast meets WCAG AA
- [ ] Content readable at 200% zoom
- [ ] Media has alternatives (captions, descriptions)
- [ ] No content flashes more than 3 times/second
- [ ] Touch targets are at least 44x44px

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | UX/Frontend Team | Initial release |

---

## 11. Related Documents

- [03_DESIGN_SYSTEM.md](./03_DESIGN_SYSTEM.md) - Design tokens and colors
- [07_COMPONENTS_LIBRARY.md](./07_COMPONENTS_LIBRARY.md) - Accessible components
- [08_PAGES_SPECIFICATION.md](./08_PAGES_SPECIFICATION.md) - Page layouts
