# Design System Guidelines

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Design & Frontend Team |
| Review Cycle | Quarterly |
| Implementation | Tailwind CSS + shadcn/ui |

---

## 2. Overview

### 2.1 Purpose
This document defines the **design system** for the N9 platform, establishing visual language, component patterns, animation tokens, interaction patterns, and UX guidelines to ensure a consistent, accessible, and delightful user experience across all platforms.

### 2.2 Design Principles

| Principle | Description |
|-----------|-------------|
| **Readability First** | Optimize for comfortable reading experience |
| **Clean & Focused** | Minimize distractions, prioritize content |
| **Consistent** | Predictable patterns across all interfaces |
| **Accessible** | WCAG 2.1 AA compliance minimum |
| **Responsive** | Seamless experience across devices |
| **Performant** | Fast, smooth interactions |

---

## 3. Color System

### 3.1 Brand Colors

```css
:root {
  /* Primary - Deep Blue */
  --primary-50: #eff6ff;
  --primary-100: #dbeafe;
  --primary-200: #bfdbfe;
  --primary-300: #93c5fd;
  --primary-400: #60a5fa;
  --primary-500: #3b82f6;  /* Main */
  --primary-600: #2563eb;
  --primary-700: #1d4ed8;
  --primary-800: #1e40af;
  --primary-900: #1e3a8a;
  --primary-950: #172554;
  
  /* Secondary - Warm Orange */
  --secondary-50: #fff7ed;
  --secondary-100: #ffedd5;
  --secondary-200: #fed7aa;
  --secondary-300: #fdba74;
  --secondary-400: #fb923c;
  --secondary-500: #f97316;  /* Main */
  --secondary-600: #ea580c;
  --secondary-700: #c2410c;
  --secondary-800: #9a3412;
  --secondary-900: #7c2d12;
  
  /* Accent - Purple */
  --accent-500: #8b5cf6;
  --accent-600: #7c3aed;
}
```

### 3.2 Semantic Colors

```css
:root {
  /* Success */
  --success-50: #f0fdf4;
  --success-500: #22c55e;
  --success-600: #16a34a;
  
  /* Warning */
  --warning-50: #fffbeb;
  --warning-500: #f59e0b;
  --warning-600: #d97706;
  
  /* Error */
  --error-50: #fef2f2;
  --error-500: #ef4444;
  --error-600: #dc2626;
  
  /* Info */
  --info-50: #eff6ff;
  --info-500: #3b82f6;
  --info-600: #2563eb;
}
```

### 3.3 Neutral Colors

```css
:root {
  /* Gray Scale */
  --gray-50: #f9fafb;
  --gray-100: #f3f4f6;
  --gray-200: #e5e7eb;
  --gray-300: #d1d5db;
  --gray-400: #9ca3af;
  --gray-500: #6b7280;
  --gray-600: #4b5563;
  --gray-700: #374151;
  --gray-800: #1f2937;
  --gray-900: #111827;
  --gray-950: #030712;
}
```

### 3.4 Theme Configurations

#### Light Theme
```css
[data-theme="light"] {
  --background: #ffffff;
  --foreground: #111827;
  --card: #ffffff;
  --card-foreground: #111827;
  --muted: #f3f4f6;
  --muted-foreground: #6b7280;
  --border: #e5e7eb;
  --input: #e5e7eb;
  --ring: #3b82f6;
}
```

#### Dark Theme
```css
[data-theme="dark"] {
  --background: #0f172a;
  --foreground: #f8fafc;
  --card: #1e293b;
  --card-foreground: #f8fafc;
  --muted: #334155;
  --muted-foreground: #94a3b8;
  --border: #334155;
  --input: #334155;
  --ring: #60a5fa;
}
```

#### Sepia Theme (Reading)
```css
[data-theme="sepia"] {
  --background: #f5f0e6;
  --foreground: #433422;
  --card: #faf7f2;
  --card-foreground: #433422;
  --muted: #e8e0d0;
  --muted-foreground: #7a6b5a;
  --border: #d4c9b8;
}
```

---

## 4. Typography

### 4.1 Font Families

```css
:root {
  /* Primary - UI Text */
  --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  
  /* Reading - Story Content */
  --font-serif: 'Merriweather', 'Georgia', serif;
  
  /* Code - Technical Content */
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;
}
```

### 4.2 Type Scale

| Name | Size | Line Height | Weight | Usage |
|------|------|-------------|--------|-------|
| `display-xl` | 4.5rem (72px) | 1.1 | 700 | Hero headlines |
| `display-lg` | 3.75rem (60px) | 1.1 | 700 | Page titles |
| `display-md` | 3rem (48px) | 1.2 | 600 | Section titles |
| `heading-xl` | 2.25rem (36px) | 1.25 | 600 | Major headings |
| `heading-lg` | 1.875rem (30px) | 1.3 | 600 | Sub-headings |
| `heading-md` | 1.5rem (24px) | 1.35 | 600 | Card titles |
| `heading-sm` | 1.25rem (20px) | 1.4 | 600 | Minor headings |
| `body-lg` | 1.125rem (18px) | 1.75 | 400 | Large body text |
| `body-md` | 1rem (16px) | 1.6 | 400 | Default body |
| `body-sm` | 0.875rem (14px) | 1.5 | 400 | Small text |
| `caption` | 0.75rem (12px) | 1.4 | 400 | Captions, labels |

### 4.3 Reading Typography

```css
.reading-content {
  font-family: var(--font-serif);
  font-size: var(--reading-font-size, 1.125rem);
  line-height: var(--reading-line-height, 1.8);
  letter-spacing: 0.01em;
  
  /* Customizable by user */
  --reading-font-size: 1.125rem;
  --reading-line-height: 1.8;
}

/* Font size options */
.reading-sm { --reading-font-size: 1rem; }
.reading-md { --reading-font-size: 1.125rem; }
.reading-lg { --reading-font-size: 1.25rem; }
.reading-xl { --reading-font-size: 1.5rem; }
```

---

## 5. Spacing System

### 5.1 Base Scale

```css
:root {
  --space-0: 0;
  --space-1: 0.25rem;   /* 4px */
  --space-2: 0.5rem;    /* 8px */
  --space-3: 0.75rem;   /* 12px */
  --space-4: 1rem;      /* 16px */
  --space-5: 1.25rem;   /* 20px */
  --space-6: 1.5rem;    /* 24px */
  --space-8: 2rem;      /* 32px */
  --space-10: 2.5rem;   /* 40px */
  --space-12: 3rem;     /* 48px */
  --space-16: 4rem;     /* 64px */
  --space-20: 5rem;     /* 80px */
  --space-24: 6rem;     /* 96px */
}
```

### 5.2 Component Spacing

| Component | Padding | Gap |
|-----------|---------|-----|
| Button (sm) | `0.5rem 0.75rem` | - |
| Button (md) | `0.625rem 1rem` | - |
| Button (lg) | `0.75rem 1.5rem` | - |
| Card | `1.5rem` | - |
| Form Field | `0.75rem 1rem` | `0.5rem` |
| Section | `4rem 0` | `2rem` |
| Page | `2rem` | `1.5rem` |

---

## 6. Layout System

### 6.1 Breakpoints

```css
:root {
  --screen-sm: 640px;   /* Mobile landscape */
  --screen-md: 768px;   /* Tablet */
  --screen-lg: 1024px;  /* Desktop */
  --screen-xl: 1280px;  /* Large desktop */
  --screen-2xl: 1536px; /* Extra large */
}
```

### 6.2 Container Widths

```css
.container {
  width: 100%;
  margin-inline: auto;
  padding-inline: 1rem;
}

@media (min-width: 640px) {
  .container { max-width: 640px; }
}
@media (min-width: 768px) {
  .container { max-width: 768px; }
}
@media (min-width: 1024px) {
  .container { max-width: 1024px; }
}
@media (min-width: 1280px) {
  .container { max-width: 1280px; }
}
```

### 6.3 Grid System

```css
/* 12-column grid */
.grid-12 {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  gap: var(--space-6);
}

/* Common layouts */
.layout-sidebar {
  display: grid;
  grid-template-columns: 280px 1fr;
  gap: var(--space-8);
}

.layout-content {
  display: grid;
  grid-template-columns: 1fr min(720px, 100%) 1fr;
}

.layout-3-col {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: var(--space-6);
}
```

### 6.4 Page Layouts

#### Main Layout
```
┌─────────────────────────────────────────────────────────────┐
│                         Header                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Logo        Navigation              User Menu           ││
│  └─────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                     Main Content                             │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                                                          ││
│  │              Page-specific content                       ││
│  │                                                          ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                         Footer                               │
│  Links | Social | Copyright                                  │
└─────────────────────────────────────────────────────────────┘
```

#### Reading Layout
```
┌─────────────────────────────────────────────────────────────┐
│  ← Back    Chapter Title              Settings  ☰           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                                                              │
│                                                              │
│                   Chapter Content                            │
│                  (max-width: 720px)                          │
│                  (centered, serif)                           │
│                                                              │
│                                                              │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│           ← Previous    Progress Bar    Next →               │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Component Library

### 7.1 Buttons

#### Variants
```tsx
// Primary - Main actions
<Button variant="primary">Publish Story</Button>

// Secondary - Alternative actions
<Button variant="secondary">Save Draft</Button>

// Outline - Tertiary actions
<Button variant="outline">Cancel</Button>

// Ghost - Minimal actions
<Button variant="ghost">Learn More</Button>

// Destructive - Dangerous actions
<Button variant="destructive">Delete</Button>
```

#### Sizes
```tsx
<Button size="sm">Small</Button>    // 32px height
<Button size="md">Medium</Button>   // 40px height
<Button size="lg">Large</Button>    // 48px height
```

#### States
| State | Visual Treatment |
|-------|------------------|
| Default | Base colors |
| Hover | Darken 10% |
| Active | Darken 15% |
| Disabled | Opacity 50%, no cursor |
| Loading | Spinner + text |

### 7.2 Form Components

#### Text Input
```tsx
<Input
  label="Story Title"
  placeholder="Enter your story title"
  error="Title is required"
  hint="Maximum 200 characters"
  required
/>
```

#### Text Area
```tsx
<Textarea
  label="Description"
  placeholder="Write a compelling description..."
  rows={4}
  maxLength={2000}
  showCount
/>
```

#### Select
```tsx
<Select
  label="Category"
  options={categories}
  placeholder="Select a category"
  searchable
/>
```

#### Checkbox & Radio
```tsx
<Checkbox label="I accept the terms" />
<RadioGroup
  options={[
    { value: 'public', label: 'Public' },
    { value: 'private', label: 'Private' },
  ]}
/>
```

### 7.3 Cards

#### Story Card
```
┌─────────────────────────────────────┐
│  ┌─────────────┐                    │
│  │             │  Title             │
│  │   Cover     │  Author            │
│  │   Image     │  ★ 4.5 • 1.2K     │
│  │             │  Genre Tags        │
│  └─────────────┘                    │
│  Description text preview...        │
│  [Read Now]                         │
└─────────────────────────────────────┘
```

#### Chapter Card
```
┌─────────────────────────────────────┐
│  Ch.1  │  Chapter Title             │
│        │  3,500 words • 15 min      │
│  🔒    │  Published: Dec 30, 2025   │
└─────────────────────────────────────┘
```

### 7.4 Navigation

#### Header Navigation
```tsx
<Header>
  <Logo />
  <Nav>
    <NavLink to="/">Home</NavLink>
    <NavLink to="/browse">Browse</NavLink>
    <NavLink to="/library">Library</NavLink>
  </Nav>
  <SearchInput />
  <UserMenu />
</Header>
```

#### Tab Navigation
```tsx
<Tabs defaultValue="overview">
  <TabsList>
    <TabsTrigger value="overview">Overview</TabsTrigger>
    <TabsTrigger value="chapters">Chapters</TabsTrigger>
    <TabsTrigger value="reviews">Reviews</TabsTrigger>
  </TabsList>
  <TabsContent value="overview">...</TabsContent>
</Tabs>
```

### 7.5 Feedback Components

#### Toast Notifications
```tsx
// Success
toast.success('Chapter published successfully!');

// Error
toast.error('Failed to save changes');

// Info
toast.info('New chapter available');

// Loading
toast.loading('Publishing...');
```

#### Modals
```tsx
<Modal>
  <ModalTrigger asChild>
    <Button>Open Modal</Button>
  </ModalTrigger>
  <ModalContent>
    <ModalHeader>
      <ModalTitle>Confirm Action</ModalTitle>
      <ModalDescription>
        Are you sure you want to proceed?
      </ModalDescription>
    </ModalHeader>
    <ModalFooter>
      <Button variant="outline">Cancel</Button>
      <Button>Confirm</Button>
    </ModalFooter>
  </ModalContent>
</Modal>
```

### 7.6 Loading States

#### Skeleton
```tsx
// Card skeleton
<SkeletonCard />

// Text skeleton
<Skeleton className="h-4 w-[200px]" />

// Image skeleton
<Skeleton className="h-[200px] w-full rounded-lg" />
```

#### Spinner
```tsx
<Spinner size="sm" />  // 16px
<Spinner size="md" />  // 24px
<Spinner size="lg" />  // 32px
```

---

## 8. Icons & Imagery

### 8.1 Icon System

```tsx
// Using Lucide React
import { 
  Home, Search, Book, Heart, Bookmark,
  User, Settings, Bell, Menu, X 
} from 'lucide-react';

// Icon sizes
<Icon size={16} /> // sm
<Icon size={20} /> // md (default)
<Icon size={24} /> // lg
<Icon size={32} /> // xl
```

### 8.2 Icon Usage

| Context | Icon Size | Stroke Width |
|---------|-----------|--------------|
| Inline text | 16px | 2 |
| Buttons | 18-20px | 2 |
| Navigation | 20-24px | 1.5 |
| Feature icons | 24-32px | 1.5 |
| Hero icons | 48-64px | 1 |

### 8.3 Image Guidelines

| Image Type | Aspect Ratio | Sizes |
|------------|--------------|-------|
| Story Cover | 2:3 | 200×300, 400×600 |
| User Avatar | 1:1 | 40×40, 80×80, 160×160 |
| Banner | 16:9 | 1200×675 |
| Thumbnail | 16:9 | 320×180 |

---

## 9. Motion & Animation

### 9.1 Timing Functions

```css
:root {
  --ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --ease-spring: cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
```

### 9.2 Duration Scale

| Name | Duration | Usage |
|------|----------|-------|
| `instant` | 75ms | Micro-interactions |
| `fast` | 150ms | Hover, focus states |
| `normal` | 200ms | Standard transitions |
| `slow` | 300ms | Complex transitions |
| `slower` | 500ms | Page transitions |

### 9.3 Animation Patterns

```css
/* Fade in */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* Slide up */
@keyframes slideUp {
  from { 
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Scale in */
@keyframes scaleIn {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}
```

### 9.4 Framer Motion Variants

```tsx
// Stagger children
const container = {
  hidden: { opacity: 0 },
  show: {
    opacity: 1,
    transition: {
      staggerChildren: 0.1,
    },
  },
};

const item = {
  hidden: { opacity: 0, y: 20 },
  show: { opacity: 1, y: 0 },
};

// Page transition
const pageTransition = {
  initial: { opacity: 0, x: -20 },
  animate: { opacity: 1, x: 0 },
  exit: { opacity: 0, x: 20 },
};
```

---

## 10. Accessibility

### 10.1 Color Contrast

| Text Type | Minimum Ratio | Target Ratio |
|-----------|---------------|--------------|
| Body text | 4.5:1 | 7:1 |
| Large text (18px+) | 3:1 | 4.5:1 |
| UI components | 3:1 | 4.5:1 |
| Focus indicators | 3:1 | 4.5:1 |

### 10.2 Focus States

```css
/* Visible focus ring */
:focus-visible {
  outline: 2px solid var(--ring);
  outline-offset: 2px;
}

/* Remove default for mouse users */
:focus:not(:focus-visible) {
  outline: none;
}
```

### 10.3 Keyboard Navigation

| Key | Action |
|-----|--------|
| `Tab` | Move to next focusable element |
| `Shift+Tab` | Move to previous focusable element |
| `Enter` | Activate button/link |
| `Space` | Activate button, toggle checkbox |
| `Escape` | Close modal/dropdown |
| `Arrow keys` | Navigate within component |

### 10.4 Screen Reader Guidelines

```tsx
// Provide context for icons
<Button aria-label="Close dialog">
  <X size={20} aria-hidden="true" />
</Button>

// Live regions for dynamic content
<div aria-live="polite" aria-atomic="true">
  {notification}
</div>

// Accessible loading states
<Button disabled aria-busy="true">
  <Spinner aria-hidden="true" />
  <span>Loading...</span>
</Button>
```

---

## 11. Dark Mode

### 11.1 Implementation

```tsx
// ThemeProvider
const ThemeContext = createContext<{
  theme: 'light' | 'dark' | 'sepia' | 'system';
  setTheme: (theme: string) => void;
}>({ theme: 'system', setTheme: () => {} });

// Usage
function ThemeToggle() {
  const { theme, setTheme } = useTheme();
  
  return (
    <Select value={theme} onValueChange={setTheme}>
      <SelectItem value="light">Light</SelectItem>
      <SelectItem value="dark">Dark</SelectItem>
      <SelectItem value="sepia">Sepia</SelectItem>
      <SelectItem value="system">System</SelectItem>
    </Select>
  );
}
```

### 11.2 Color Mapping

| Semantic | Light | Dark |
|----------|-------|------|
| Background | `#ffffff` | `#0f172a` |
| Foreground | `#111827` | `#f8fafc` |
| Card | `#ffffff` | `#1e293b` |
| Border | `#e5e7eb` | `#334155` |
| Muted | `#f3f4f6` | `#334155` |

---

## 12. Responsive Patterns

### 12.1 Mobile-First Approach

```css
/* Base styles (mobile) */
.component {
  padding: var(--space-4);
  font-size: 1rem;
}

/* Tablet and up */
@media (min-width: 768px) {
  .component {
    padding: var(--space-6);
  }
}

/* Desktop and up */
@media (min-width: 1024px) {
  .component {
    padding: var(--space-8);
    font-size: 1.125rem;
  }
}
```

### 12.2 Responsive Components

| Component | Mobile | Tablet | Desktop |
|-----------|--------|--------|---------|
| Navigation | Hamburger menu | Horizontal nav | Full nav + search |
| Story Grid | 1 column | 2 columns | 3-4 columns |
| Sidebar | Bottom sheet | Side drawer | Fixed sidebar |
| Modal | Full screen | Centered | Centered (max 600px) |

---

## 13. Animation & Motion

### 13.1 Animation Tokens

```css
:root {
  /* Duration */
  --duration-instant: 50ms;
  --duration-fast: 150ms;
  --duration-normal: 250ms;
  --duration-slow: 400ms;
  --duration-slower: 600ms;
  
  /* Easing */
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-bounce: cubic-bezier(0.34, 1.56, 0.64, 1);
}
```

### 13.2 Animation Patterns

| Pattern | Duration | Easing | Usage |
|---------|----------|--------|-------|
| Fade | 150ms | ease-out | Tooltips, toasts |
| Slide | 250ms | ease-out | Drawers, modals |
| Scale | 200ms | ease-bounce | Buttons, cards |
| Collapse | 300ms | ease-in-out | Accordions |
| Page transition | 300ms | ease-out | Route changes |

### 13.3 Framer Motion Presets

```typescript
export const fadeIn = {
  initial: { opacity: 0 },
  animate: { opacity: 1 },
  exit: { opacity: 0 },
  transition: { duration: 0.15 },
};

export const slideUp = {
  initial: { opacity: 0, y: 20 },
  animate: { opacity: 1, y: 0 },
  exit: { opacity: 0, y: 20 },
  transition: { duration: 0.25, ease: 'easeOut' },
};

export const scaleIn = {
  initial: { opacity: 0, scale: 0.95 },
  animate: { opacity: 1, scale: 1 },
  exit: { opacity: 0, scale: 0.95 },
  transition: { duration: 0.2, ease: [0.34, 1.56, 0.64, 1] },
};
```

### 13.4 Reduced Motion Support

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## 14. Interaction Patterns

### 14.1 Loading States

| State | Component | Behavior |
|-------|-----------|----------|
| Initial load | Skeleton | Content-shaped placeholders |
| Button loading | Spinner + disabled | Replace text with spinner |
| Infinite scroll | Bottom loader | Show at 80% scroll |
| Refresh | Pull-to-refresh | Mobile gesture |
| Optimistic | Immediate UI | Revert on error |

### 14.2 Empty States

```tsx
<EmptyState
  icon={<BookOpen />}
  title="No stories yet"
  description="Start browsing to find your next great read"
  action={<Button onClick={browse}>Browse Stories</Button>}
/>
```

### 14.3 Error States

| Error Type | Display | Recovery |
|------------|---------|----------|
| Network | Toast + retry | Retry button |
| Validation | Inline field error | Auto-clear on fix |
| Server (500) | Error page | Reload button |
| Not Found | 404 page | Navigation links |
| Unauthorized | Redirect | Login page |

### 14.4 Success Feedback

| Action | Feedback | Duration |
|--------|----------|----------|
| Save | Toast "Saved" | 3s auto-dismiss |
| Delete | Toast + Undo | 5s with undo |
| Create | Toast + Navigate | 3s |
| Submit | Success modal | Until dismissed |

---

## 15. References

### 15.1 Related Documents

| Document | Purpose |
|----------|----------|
| [01_FRONTEND_ARCHITECTURE.md](01_FRONTEND_ARCHITECTURE.md) | Technical architecture |
| [04_SHARED_COMPONENTS.md](04_SHARED_COMPONENTS.md) | Component specifications |
| [05_MOBILE_RESPONSIVE_DESIGN.md](05_MOBILE_RESPONSIVE_DESIGN.md) | Mobile patterns |

### 15.2 External Resources

- [Tailwind CSS Documentation](https://tailwindcss.com/docs)
- [shadcn/ui Components](https://ui.shadcn.com)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [Framer Motion](https://www.framer.com/motion/)

---

## 16. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|----------|
| 1.0 | 2025-12-31 | Design Team | Initial design system |
| 2.0 | 2026-01-04 | Design Team | Added animation tokens, motion presets, interaction patterns, loading/empty/error states, reduced motion support |
