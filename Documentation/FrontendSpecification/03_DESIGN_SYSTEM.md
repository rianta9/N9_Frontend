# Design System Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Design & Frontend Team |
| Review Cycle | Quarterly |
| Related Documents | 02_FRONTEND_ARCHITECTURE.md, Frontend Design Guidelines |

---

## 2. Design Philosophy

### 2.1 Core Principles

| Principle | Description | Implementation |
|-----------|-------------|----------------|
| **Readability First** | Optimize for comfortable reading | Generous spacing, optimal line length |
| **Clean & Focused** | Minimize distractions | Content-centric layouts |
| **Consistent** | Predictable patterns | Design tokens, component library |
| **Accessible** | Inclusive by default | WCAG 2.1 AA minimum |
| **Responsive** | Seamless across devices | Mobile-first approach |
| **Performant** | Fast, smooth interactions | Optimized animations |

### 2.2 Design Language Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           N9 DESIGN LANGUAGE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  VISUAL IDENTITY                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                                                                          ││
│  │  • Modern & Clean aesthetic                                              ││
│  │  • Book/Reading-inspired elements                                        ││
│  │  • Warm, inviting color palette                                          ││
│  │  • Typography-focused hierarchy                                          ││
│  │  • Subtle, purposeful animations                                         ││
│  │                                                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  EMOTIONAL TARGETS                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                                                                          ││
│  │  Readers:     Immersive • Comfortable • Trustworthy                      ││
│  │  Authors:     Professional • Empowering • Rewarding                      ││
│  │  Discovery:   Exciting • Curated • Personalized                          ││
│  │                                                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Color System

### 3.1 Brand Colors

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* PRIMARY - Deep Blue (Trust, Reliability, Depth)                          */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --primary-50: #eff6ff;
  --primary-100: #dbeafe;
  --primary-200: #bfdbfe;
  --primary-300: #93c5fd;
  --primary-400: #60a5fa;
  --primary-500: #3b82f6;    /* Main brand color */
  --primary-600: #2563eb;    /* Interactive states */
  --primary-700: #1d4ed8;    /* Hover states */
  --primary-800: #1e40af;
  --primary-900: #1e3a8a;
  --primary-950: #172554;
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* SECONDARY - Warm Orange (Creativity, Energy, Action)                     */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --secondary-50: #fff7ed;
  --secondary-100: #ffedd5;
  --secondary-200: #fed7aa;
  --secondary-300: #fdba74;
  --secondary-400: #fb923c;
  --secondary-500: #f97316;   /* Accent color */
  --secondary-600: #ea580c;
  --secondary-700: #c2410c;
  --secondary-800: #9a3412;
  --secondary-900: #7c2d12;
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* ACCENT - Purple (Premium, Creative, Magical)                             */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --accent-50: #faf5ff;
  --accent-100: #f3e8ff;
  --accent-200: #e9d5ff;
  --accent-300: #d8b4fe;
  --accent-400: #c084fc;
  --accent-500: #a855f7;      /* Premium features */
  --accent-600: #9333ea;
  --accent-700: #7c3aed;
  --accent-800: #6b21a8;
  --accent-900: #581c87;
}
```

### 3.2 Semantic Colors

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* SUCCESS - Green (Positive outcomes, Confirmations)                       */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --success-50: #f0fdf4;
  --success-100: #dcfce7;
  --success-200: #bbf7d0;
  --success-500: #22c55e;     /* Main success */
  --success-600: #16a34a;
  --success-700: #15803d;
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* WARNING - Amber (Caution, Attention needed)                              */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --warning-50: #fffbeb;
  --warning-100: #fef3c7;
  --warning-200: #fde68a;
  --warning-500: #f59e0b;     /* Main warning */
  --warning-600: #d97706;
  --warning-700: #b45309;
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* ERROR - Red (Errors, Destructive actions)                                */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --error-50: #fef2f2;
  --error-100: #fee2e2;
  --error-200: #fecaca;
  --error-500: #ef4444;       /* Main error */
  --error-600: #dc2626;
  --error-700: #b91c1c;
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* INFO - Blue (Information, Help)                                          */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --info-50: #eff6ff;
  --info-100: #dbeafe;
  --info-500: #3b82f6;        /* Main info */
  --info-600: #2563eb;
  --info-700: #1d4ed8;
}
```

### 3.3 Neutral Colors

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* GRAY SCALE - For text, backgrounds, borders                              */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --gray-50: #f9fafb;         /* Lightest backgrounds */
  --gray-100: #f3f4f6;        /* Card backgrounds */
  --gray-200: #e5e7eb;        /* Borders, dividers */
  --gray-300: #d1d5db;        /* Disabled states */
  --gray-400: #9ca3af;        /* Placeholder text */
  --gray-500: #6b7280;        /* Secondary text */
  --gray-600: #4b5563;        /* Body text */
  --gray-700: #374151;        /* Headings */
  --gray-800: #1f2937;        /* Primary text */
  --gray-900: #111827;        /* Darkest text */
  --gray-950: #030712;        /* Near black */
}
```

### 3.4 Theme Configuration

#### 3.4.1 Light Theme (Default)

```css
[data-theme="light"] {
  /* Backgrounds */
  --background: #ffffff;
  --background-secondary: #f9fafb;
  --background-tertiary: #f3f4f6;
  
  /* Foregrounds */
  --foreground: #111827;
  --foreground-secondary: #4b5563;
  --foreground-muted: #6b7280;
  
  /* Cards & Surfaces */
  --card: #ffffff;
  --card-foreground: #111827;
  --card-border: #e5e7eb;
  
  /* Interactive */
  --muted: #f3f4f6;
  --muted-foreground: #6b7280;
  --border: #e5e7eb;
  --input: #e5e7eb;
  --ring: #3b82f6;
  
  /* Overlays */
  --overlay: rgba(0, 0, 0, 0.5);
  --backdrop: rgba(255, 255, 255, 0.8);
}
```

#### 3.4.2 Dark Theme

```css
[data-theme="dark"] {
  /* Backgrounds */
  --background: #0f172a;
  --background-secondary: #1e293b;
  --background-tertiary: #334155;
  
  /* Foregrounds */
  --foreground: #f8fafc;
  --foreground-secondary: #cbd5e1;
  --foreground-muted: #94a3b8;
  
  /* Cards & Surfaces */
  --card: #1e293b;
  --card-foreground: #f8fafc;
  --card-border: #334155;
  
  /* Interactive */
  --muted: #334155;
  --muted-foreground: #94a3b8;
  --border: #334155;
  --input: #334155;
  --ring: #60a5fa;
  
  /* Overlays */
  --overlay: rgba(0, 0, 0, 0.7);
  --backdrop: rgba(15, 23, 42, 0.8);
}
```

#### 3.4.3 Sepia Theme (Reading Mode)

```css
[data-theme="sepia"] {
  /* Backgrounds */
  --background: #f5f0e6;
  --background-secondary: #ebe4d6;
  --background-tertiary: #e0d7c6;
  
  /* Foregrounds */
  --foreground: #433422;
  --foreground-secondary: #5c4a36;
  --foreground-muted: #7a6b5a;
  
  /* Cards & Surfaces */
  --card: #faf7f2;
  --card-foreground: #433422;
  --card-border: #d4c9b8;
  
  /* Interactive */
  --muted: #e8e0d0;
  --muted-foreground: #7a6b5a;
  --border: #d4c9b8;
  --input: #d4c9b8;
  --ring: #8b7355;
}
```

### 3.5 Color Usage Guidelines

| Use Case | Light Theme | Dark Theme | Sepia Theme |
|----------|-------------|------------|-------------|
| **Page Background** | `--background` | `--background` | `--background` |
| **Primary Text** | `--foreground` | `--foreground` | `--foreground` |
| **Secondary Text** | `--foreground-secondary` | `--foreground-secondary` | `--foreground-secondary` |
| **Card Background** | `--card` | `--card` | `--card` |
| **Primary Button** | `--primary-500` | `--primary-400` | `--primary-600` |
| **Borders** | `--border` | `--border` | `--border` |
| **Links** | `--primary-600` | `--primary-400` | `--primary-700` |
| **Focus Ring** | `--ring` | `--ring` | `--ring` |

---

## 4. Typography

### 4.1 Font Families

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* PRIMARY - UI & Interface Text                                            */
  /* Clean, highly readable sans-serif for UI elements                        */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', 
               Roboto, 'Helvetica Neue', Arial, sans-serif;
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* READING - Story Content                                                  */
  /* Elegant serif for comfortable long-form reading                          */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --font-serif: 'Merriweather', 'Georgia', 'Cambria', 
                'Times New Roman', serif;
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* DISPLAY - Headlines & Marketing                                          */
  /* Bold, impactful font for hero sections                                   */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --font-display: 'Plus Jakarta Sans', var(--font-sans);
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* MONO - Code & Technical Content                                          */
  /* Monospace for code blocks and technical data                             */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --font-mono: 'JetBrains Mono', 'Fira Code', 'Consolas', 
               'Monaco', monospace;
}
```

### 4.2 Type Scale

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* DISPLAY SIZES - Hero headlines, marketing                                */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --text-display-2xl: 4.5rem;    /* 72px - Hero headlines */
  --text-display-xl: 3.75rem;    /* 60px - Page titles */
  --text-display-lg: 3rem;       /* 48px - Section titles */
  --text-display-md: 2.25rem;    /* 36px - Major headings */
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* HEADING SIZES - Section and component headings                           */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --text-heading-xl: 1.875rem;   /* 30px - H1 equivalent */
  --text-heading-lg: 1.5rem;     /* 24px - H2 equivalent */
  --text-heading-md: 1.25rem;    /* 20px - H3 equivalent */
  --text-heading-sm: 1.125rem;   /* 18px - H4 equivalent */
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* BODY SIZES - Content and interface text                                  */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --text-body-lg: 1.125rem;      /* 18px - Large body text */
  --text-body-md: 1rem;          /* 16px - Default body */
  --text-body-sm: 0.875rem;      /* 14px - Small text */
  --text-body-xs: 0.75rem;       /* 12px - Captions */
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* LINE HEIGHTS                                                             */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --leading-none: 1;
  --leading-tight: 1.25;
  --leading-snug: 1.375;
  --leading-normal: 1.5;
  --leading-relaxed: 1.625;
  --leading-loose: 2;
  --leading-reading: 1.8;        /* Optimized for reading */
  
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* FONT WEIGHTS                                                             */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --font-thin: 100;
  --font-light: 300;
  --font-normal: 400;
  --font-medium: 500;
  --font-semibold: 600;
  --font-bold: 700;
  --font-extrabold: 800;
}
```

### 4.3 Typography Presets

| Preset | Size | Weight | Line Height | Font | Use Case |
|--------|------|--------|-------------|------|----------|
| `display-hero` | 72px | 700 | 1.1 | Display | Hero sections |
| `display-title` | 48px | 700 | 1.2 | Display | Page titles |
| `heading-1` | 30px | 600 | 1.3 | Sans | Major headings |
| `heading-2` | 24px | 600 | 1.35 | Sans | Sub-headings |
| `heading-3` | 20px | 600 | 1.4 | Sans | Card titles |
| `heading-4` | 18px | 600 | 1.4 | Sans | Minor headings |
| `body-large` | 18px | 400 | 1.6 | Sans | Intro text |
| `body` | 16px | 400 | 1.6 | Sans | Default body |
| `body-small` | 14px | 400 | 1.5 | Sans | Secondary text |
| `caption` | 12px | 400 | 1.4 | Sans | Labels, captions |
| `reading` | 18px | 400 | 1.8 | Serif | Story content |
| `reading-large` | 20px | 400 | 1.8 | Serif | Comfortable reading |

### 4.4 Reading Typography (Special)

```css
/* ═══════════════════════════════════════════════════════════════════════════ */
/* READER-SPECIFIC TYPOGRAPHY                                                  */
/* Customizable settings for the immersive reading experience                  */
/* ═══════════════════════════════════════════════════════════════════════════ */

.reader-content {
  /* User-adjustable via CSS variables */
  font-family: var(--reader-font-family, var(--font-serif));
  font-size: var(--reader-font-size, 1.125rem);
  line-height: var(--reader-line-height, 1.8);
  letter-spacing: var(--reader-letter-spacing, 0.01em);
  
  /* Optimal reading width */
  max-width: var(--reader-max-width, 65ch);
  margin: 0 auto;
  
  /* Paragraph spacing */
  p {
    margin-bottom: var(--reader-paragraph-spacing, 1.5em);
    text-indent: var(--reader-text-indent, 0);
  }
  
  /* First paragraph special treatment */
  p:first-of-type {
    text-indent: 0;
  }
  
  /* Chapter title styling */
  h1, h2 {
    font-family: var(--font-display);
    text-align: center;
    margin-bottom: 2em;
  }
}

/* Reader font size presets */
:root {
  --reader-size-xs: 0.875rem;    /* 14px */
  --reader-size-sm: 1rem;        /* 16px */
  --reader-size-md: 1.125rem;    /* 18px - Default */
  --reader-size-lg: 1.25rem;     /* 20px */
  --reader-size-xl: 1.375rem;    /* 22px */
  --reader-size-2xl: 1.5rem;     /* 24px */
}
```

---

## 5. Spacing System

### 5.1 Spacing Scale

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* BASE SPACING SCALE (8px grid system)                                     */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --space-0: 0;
  --space-0.5: 0.125rem;    /* 2px */
  --space-1: 0.25rem;       /* 4px */
  --space-1.5: 0.375rem;    /* 6px */
  --space-2: 0.5rem;        /* 8px */
  --space-2.5: 0.625rem;    /* 10px */
  --space-3: 0.75rem;       /* 12px */
  --space-3.5: 0.875rem;    /* 14px */
  --space-4: 1rem;          /* 16px */
  --space-5: 1.25rem;       /* 20px */
  --space-6: 1.5rem;        /* 24px */
  --space-7: 1.75rem;       /* 28px */
  --space-8: 2rem;          /* 32px */
  --space-9: 2.25rem;       /* 36px */
  --space-10: 2.5rem;       /* 40px */
  --space-11: 2.75rem;      /* 44px */
  --space-12: 3rem;         /* 48px */
  --space-14: 3.5rem;       /* 56px */
  --space-16: 4rem;         /* 64px */
  --space-20: 5rem;         /* 80px */
  --space-24: 6rem;         /* 96px */
  --space-28: 7rem;         /* 112px */
  --space-32: 8rem;         /* 128px */
}
```

### 5.2 Semantic Spacing

| Token | Value | Use Case |
|-------|-------|----------|
| `--space-xs` | 4px | Tight spacing, icon gaps |
| `--space-sm` | 8px | Small element spacing |
| `--space-md` | 16px | Default component spacing |
| `--space-lg` | 24px | Section spacing |
| `--space-xl` | 32px | Large section gaps |
| `--space-2xl` | 48px | Major section divisions |
| `--space-3xl` | 64px | Page section breaks |

### 5.3 Component Spacing Patterns

```css
/* Button internal spacing */
.btn {
  padding: var(--space-2) var(--space-4);
}

.btn-sm {
  padding: var(--space-1.5) var(--space-3);
}

.btn-lg {
  padding: var(--space-3) var(--space-6);
}

/* Card spacing */
.card {
  padding: var(--space-4);
}

.card-header {
  padding: var(--space-4);
  margin: calc(var(--space-4) * -1);
  margin-bottom: var(--space-4);
}

/* Form spacing */
.form-group {
  margin-bottom: var(--space-4);
}

.form-label {
  margin-bottom: var(--space-1.5);
}

/* Section spacing */
.section {
  padding: var(--space-16) 0;
}

.section-header {
  margin-bottom: var(--space-8);
}
```

---

## 6. Layout & Grid

### 6.1 Breakpoint System

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* BREAKPOINTS                                                              */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --breakpoint-xs: 320px;     /* Small phones */
  --breakpoint-sm: 480px;     /* Large phones */
  --breakpoint-md: 768px;     /* Tablets */
  --breakpoint-lg: 1024px;    /* Small desktops */
  --breakpoint-xl: 1280px;    /* Large desktops */
  --breakpoint-2xl: 1440px;   /* Extra large screens */
}
```

### 6.2 Container Widths

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* CONTAINER MAX-WIDTHS                                                     */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --container-xs: 320px;
  --container-sm: 640px;
  --container-md: 768px;
  --container-lg: 1024px;
  --container-xl: 1280px;
  --container-2xl: 1440px;
  
  /* Specific use-case containers */
  --container-prose: 65ch;     /* Optimal reading width */
  --container-narrow: 640px;   /* Centered content */
  --container-default: 1280px; /* Standard content */
  --container-wide: 1440px;    /* Full-width content */
}
```

### 6.3 Grid System

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              12-COLUMN GRID                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Desktop (≥1024px): 12 columns, 24px gap                                    │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                         │
│  │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12 │                         │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                         │
│                                                                              │
│  Tablet (768px-1023px): 8 columns, 20px gap                                 │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐                         │
│  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │  8  │                         │
│  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘                         │
│                                                                              │
│  Mobile (< 768px): 4 columns, 16px gap                                      │
│  ┌───────────┬───────────┬───────────┬───────────┐                         │
│  │     1     │     2     │     3     │     4     │                         │
│  └───────────┴───────────┴───────────┴───────────┘                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.4 Common Layout Patterns

```css
/* Story grid - responsive card layout */
.story-grid {
  display: grid;
  gap: var(--space-4);
  grid-template-columns: repeat(2, 1fr);
}

@media (min-width: 768px) {
  .story-grid {
    grid-template-columns: repeat(3, 1fr);
    gap: var(--space-6);
  }
}

@media (min-width: 1024px) {
  .story-grid {
    grid-template-columns: repeat(4, 1fr);
  }
}

@media (min-width: 1280px) {
  .story-grid {
    grid-template-columns: repeat(5, 1fr);
  }
}

/* Two-column layout (sidebar + content) */
.layout-sidebar {
  display: grid;
  gap: var(--space-6);
  grid-template-columns: 1fr;
}

@media (min-width: 1024px) {
  .layout-sidebar {
    grid-template-columns: 280px 1fr;
  }
}

/* Centered content layout */
.layout-centered {
  max-width: var(--container-narrow);
  margin: 0 auto;
  padding: 0 var(--space-4);
}
```

---

## 7. Elevation & Shadows

### 7.1 Shadow Scale

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* SHADOW SYSTEM                                                            */
  /* ═══════════════════════════════════════════════════════════════════════ */
  
  --shadow-xs: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  
  --shadow-sm: 0 1px 3px 0 rgb(0 0 0 / 0.1),
               0 1px 2px -1px rgb(0 0 0 / 0.1);
  
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1),
               0 2px 4px -2px rgb(0 0 0 / 0.1);
  
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1),
               0 4px 6px -4px rgb(0 0 0 / 0.1);
  
  --shadow-xl: 0 20px 25px -5px rgb(0 0 0 / 0.1),
               0 8px 10px -6px rgb(0 0 0 / 0.1);
  
  --shadow-2xl: 0 25px 50px -12px rgb(0 0 0 / 0.25);
  
  --shadow-inner: inset 0 2px 4px 0 rgb(0 0 0 / 0.05);
  
  /* Focus ring shadow */
  --shadow-focus: 0 0 0 3px var(--primary-500 / 0.3);
}
```

### 7.2 Shadow Usage

| Level | Shadow | Use Case |
|-------|--------|----------|
| **Level 0** | None | Flat elements, backgrounds |
| **Level 1** | `shadow-xs` | Subtle elevation, inputs |
| **Level 2** | `shadow-sm` | Cards, buttons |
| **Level 3** | `shadow-md` | Dropdowns, tooltips |
| **Level 4** | `shadow-lg` | Popovers, modals |
| **Level 5** | `shadow-xl` | Large modals, overlays |
| **Level 6** | `shadow-2xl` | Hero cards, featured content |

---

## 8. Border Radius

### 8.1 Radius Scale

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* BORDER RADIUS SCALE                                                      */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --radius-none: 0;
  --radius-sm: 0.125rem;      /* 2px */
  --radius-md: 0.375rem;      /* 6px */
  --radius-lg: 0.5rem;        /* 8px */
  --radius-xl: 0.75rem;       /* 12px */
  --radius-2xl: 1rem;         /* 16px */
  --radius-3xl: 1.5rem;       /* 24px */
  --radius-full: 9999px;      /* Pill/Circle */
}
```

### 8.2 Radius Usage

| Token | Value | Use Case |
|-------|-------|----------|
| `radius-sm` | 2px | Tags, badges, small elements |
| `radius-md` | 6px | Buttons, inputs |
| `radius-lg` | 8px | Cards, modals |
| `radius-xl` | 12px | Large cards, containers |
| `radius-2xl` | 16px | Hero cards, featured content |
| `radius-full` | 9999px | Avatars, pill buttons |

---

## 9. Animation & Motion

### 9.1 Duration Scale

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* ANIMATION DURATIONS                                                      */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --duration-fastest: 50ms;    /* Micro-interactions */
  --duration-fast: 100ms;      /* Button states */
  --duration-normal: 200ms;    /* Most transitions */
  --duration-slow: 300ms;      /* Complex transitions */
  --duration-slower: 400ms;    /* Page transitions */
  --duration-slowest: 500ms;   /* Large animations */
}
```

### 9.2 Easing Functions

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* EASING FUNCTIONS                                                         */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --ease-linear: linear;
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
  
  /* Special easings */
  --ease-bounce: cubic-bezier(0.68, -0.55, 0.265, 1.55);
  --ease-spring: cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
```

### 9.3 Animation Presets

```css
/* ═══════════════════════════════════════════════════════════════════════════ */
/* ANIMATION PRESETS                                                           */
/* ═══════════════════════════════════════════════════════════════════════════ */

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

/* Pulse */
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

/* Spin */
@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

/* Utility classes */
.animate-fadeIn { animation: fadeIn var(--duration-normal) var(--ease-out); }
.animate-slideUp { animation: slideUp var(--duration-normal) var(--ease-out); }
.animate-scaleIn { animation: scaleIn var(--duration-normal) var(--ease-out); }
.animate-pulse { animation: pulse 2s var(--ease-in-out) infinite; }
.animate-spin { animation: spin 1s linear infinite; }
```

### 9.4 Motion Principles

| Principle | Description | Implementation |
|-----------|-------------|----------------|
| **Purpose** | Every animation has a reason | Guide attention, provide feedback |
| **Duration** | Fast enough to not delay | 100-300ms for most transitions |
| **Easing** | Natural, not robotic | Use ease-out for entries |
| **Consistency** | Same motion language | Reuse animation presets |
| **Performance** | Smooth 60fps | Transform/opacity only |
| **Accessibility** | Respect preferences | `prefers-reduced-motion` |

### 9.5 Reduced Motion Support

```css
/* Respect user preferences */
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

## 10. Icons & Imagery

### 10.1 Icon System

| Property | Value |
|----------|-------|
| **Library** | Lucide Icons |
| **Stroke Width** | 2px default |
| **Size Scale** | 16, 20, 24, 32, 48px |
| **Style** | Outlined, consistent stroke |

### 10.2 Icon Sizes

| Size | Pixels | Use Case |
|------|--------|----------|
| `icon-xs` | 12px | Inline indicators |
| `icon-sm` | 16px | Button icons, inline |
| `icon-md` | 20px | Default size |
| `icon-lg` | 24px | Standalone icons |
| `icon-xl` | 32px | Feature icons |
| `icon-2xl` | 48px | Hero icons |

### 10.3 Image Ratios

| Ratio | Use Case |
|-------|----------|
| **1:1** | Avatars, thumbnails |
| **2:3** | Book covers, story cards |
| **3:4** | Portrait images |
| **16:9** | Hero images, banners |
| **4:3** | Feature images |

### 10.4 Placeholder & Loading States

```css
/* Image placeholder */
.image-placeholder {
  background: var(--muted);
  background-image: linear-gradient(
    90deg,
    var(--muted) 0%,
    var(--background-secondary) 50%,
    var(--muted) 100%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
```

---

## 11. Z-Index Scale

```css
:root {
  /* ═══════════════════════════════════════════════════════════════════════ */
  /* Z-INDEX SCALE                                                            */
  /* ═══════════════════════════════════════════════════════════════════════ */
  --z-base: 0;
  --z-dropdown: 10;
  --z-sticky: 20;
  --z-fixed: 30;
  --z-modal-backdrop: 40;
  --z-modal: 50;
  --z-popover: 60;
  --z-tooltip: 70;
  --z-toast: 80;
  --z-maximum: 9999;
}
```

### 11.1 Z-Index Usage

| Layer | Z-Index | Components |
|-------|---------|------------|
| **Base** | 0 | Regular content |
| **Dropdown** | 10 | Menus, selects |
| **Sticky** | 20 | Sticky headers, sidebars |
| **Fixed** | 30 | Fixed navigation |
| **Modal Backdrop** | 40 | Overlay backgrounds |
| **Modal** | 50 | Dialogs, modals |
| **Popover** | 60 | Popovers, menus |
| **Tooltip** | 70 | Tooltips |
| **Toast** | 80 | Notifications |

---

## 12. Design Tokens Summary

### 12.1 Complete Token Reference

```css
/* ═══════════════════════════════════════════════════════════════════════════ */
/* N9 DESIGN TOKENS - COMPLETE REFERENCE                                       */
/* ═══════════════════════════════════════════════════════════════════════════ */

:root {
  /* Colors */
  --color-primary: var(--primary-500);
  --color-secondary: var(--secondary-500);
  --color-accent: var(--accent-500);
  --color-success: var(--success-500);
  --color-warning: var(--warning-500);
  --color-error: var(--error-500);
  --color-info: var(--info-500);
  
  /* Typography */
  --font-family-sans: var(--font-sans);
  --font-family-serif: var(--font-serif);
  --font-family-mono: var(--font-mono);
  --font-size-base: var(--text-body-md);
  --line-height-base: var(--leading-normal);
  
  /* Spacing */
  --spacing-base: var(--space-4);
  --spacing-section: var(--space-16);
  
  /* Layout */
  --container-max: var(--container-xl);
  --sidebar-width: 280px;
  --header-height: 64px;
  --footer-height: 200px;
  
  /* Borders */
  --border-width: 1px;
  --border-color: var(--border);
  --border-radius: var(--radius-lg);
  
  /* Shadows */
  --shadow-default: var(--shadow-md);
  --shadow-card: var(--shadow-sm);
  --shadow-modal: var(--shadow-xl);
  
  /* Transitions */
  --transition-fast: var(--duration-fast) var(--ease-out);
  --transition-normal: var(--duration-normal) var(--ease-out);
  --transition-slow: var(--duration-slow) var(--ease-out);
}
```

---

## 13. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Design & Frontend Team | Initial release |

---

## 14. Related Documents

- [02_FRONTEND_ARCHITECTURE.md](./02_FRONTEND_ARCHITECTURE.md) - Architecture specification
- [07_COMPONENTS_LIBRARY.md](./07_COMPONENTS_LIBRARY.md) - Component library
- [Frontend Design - Design System](../FrontendDesign/02_DESIGN_SYSTEM_GUIDELINES.md)
