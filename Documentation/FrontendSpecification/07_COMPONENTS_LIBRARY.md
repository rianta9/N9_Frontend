# Components Library Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Frontend Architecture Team |
| Review Cycle | Quarterly |
| Related Documents | 03_DESIGN_SYSTEM.md, 08_PAGES_SPECIFICATION.md |

---

## 2. Component Architecture

### 2.1 Component Classification

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        COMPONENT HIERARCHY                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ PRIMITIVE COMPONENTS (atoms)                                            ││
│  │ • Button, Input, Checkbox, Radio, Select                                ││
│  │ • Badge, Avatar, Icon, Spinner                                          ││
│  │ • Typography (Heading, Text, Label)                                     ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                            │                                                 │
│                            ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ COMPOSITE COMPONENTS (molecules)                                        ││
│  │ • Card, Modal, Dropdown, Tabs, Accordion                                ││
│  │ • Form controls (FormField, SearchInput)                                ││
│  │ • Navigation (Breadcrumb, Pagination)                                   ││
│  │ • Feedback (Toast, Alert, Progress)                                     ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                            │                                                 │
│                            ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ FEATURE COMPONENTS (organisms)                                          ││
│  │ • StoryCard, ChapterList, CommentThread                                 ││
│  │ • UserCard, AuthorBadge, WalletBalance                                  ││
│  │ • RatingDisplay, BookmarkButton                                         ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                            │                                                 │
│                            ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ LAYOUT COMPONENTS (templates)                                           ││
│  │ • PageLayout, SidebarLayout, DashboardLayout                            ││
│  │ • Header, Footer, Sidebar, Navigation                                   ││
│  │ • Container, Grid, Stack                                                ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Component File Structure

```
components/
├── ui/                          # Primitive/shadcn components
│   ├── button.tsx
│   ├── input.tsx
│   ├── card.tsx
│   └── ...
├── common/                      # Reusable composite components
│   ├── FormField/
│   │   ├── index.tsx
│   │   ├── FormField.tsx
│   │   └── FormField.test.tsx
│   ├── Modal/
│   ├── DataTable/
│   └── ...
├── layout/                      # Layout components
│   ├── Header/
│   ├── Footer/
│   ├── Sidebar/
│   └── PageContainer/
├── skeletons/                   # Loading skeletons
│   ├── StoryCardSkeleton.tsx
│   ├── PageSkeleton.tsx
│   └── ...
└── icons/                       # Custom icons
    └── index.tsx
```

---

## 3. Primitive Components (UI)

### 3.1 Button

```typescript
// components/ui/button.tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { Slot } from '@radix-ui/react-slot';
import { forwardRef } from 'react';
import { cn } from '@/lib/utils';
import { Loader2 } from 'lucide-react';

const buttonVariants = cva(
  'inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
);

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
  loading?: boolean;
}

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, loading, children, disabled, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button';
    
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        disabled={disabled || loading}
        {...props}
      >
        {loading && <Loader2 className="h-4 w-4 animate-spin" />}
        {children}
      </Comp>
    );
  }
);
Button.displayName = 'Button';

export { Button, buttonVariants };
export type { ButtonProps };
```

### 3.2 Input

```typescript
// components/ui/input.tsx
import { forwardRef } from 'react';
import { cn } from '@/lib/utils';

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  error?: boolean;
}

const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ className, type, error, ...props }, ref) => {
    return (
      <input
        type={type}
        className={cn(
          'flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background',
          'file:border-0 file:bg-transparent file:text-sm file:font-medium',
          'placeholder:text-muted-foreground',
          'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2',
          'disabled:cursor-not-allowed disabled:opacity-50',
          error && 'border-destructive focus-visible:ring-destructive',
          className
        )}
        ref={ref}
        {...props}
      />
    );
  }
);
Input.displayName = 'Input';

export { Input };
export type { InputProps };
```

### 3.3 Badge

```typescript
// components/ui/badge.tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const badgeVariants = cva(
  'inline-flex items-center rounded-full border px-2.5 py-0.5 text-xs font-semibold transition-colors focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2',
  {
    variants: {
      variant: {
        default: 'border-transparent bg-primary text-primary-foreground',
        secondary: 'border-transparent bg-secondary text-secondary-foreground',
        destructive: 'border-transparent bg-destructive text-destructive-foreground',
        success: 'border-transparent bg-success text-success-foreground',
        warning: 'border-transparent bg-warning text-warning-foreground',
        outline: 'text-foreground',
      },
    },
    defaultVariants: {
      variant: 'default',
    },
  }
);

interface BadgeProps
  extends React.HTMLAttributes<HTMLDivElement>,
    VariantProps<typeof badgeVariants> {}

function Badge({ className, variant, ...props }: BadgeProps) {
  return (
    <div className={cn(badgeVariants({ variant }), className)} {...props} />
  );
}

export { Badge, badgeVariants };
export type { BadgeProps };
```

### 3.4 Avatar

```typescript
// components/ui/avatar.tsx
import * as AvatarPrimitive from '@radix-ui/react-avatar';
import { forwardRef } from 'react';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const avatarVariants = cva(
  'relative flex shrink-0 overflow-hidden rounded-full',
  {
    variants: {
      size: {
        xs: 'h-6 w-6',
        sm: 'h-8 w-8',
        md: 'h-10 w-10',
        lg: 'h-12 w-12',
        xl: 'h-16 w-16',
        '2xl': 'h-24 w-24',
      },
    },
    defaultVariants: {
      size: 'md',
    },
  }
);

interface AvatarProps
  extends React.ComponentPropsWithoutRef<typeof AvatarPrimitive.Root>,
    VariantProps<typeof avatarVariants> {}

const Avatar = forwardRef<
  React.ElementRef<typeof AvatarPrimitive.Root>,
  AvatarProps
>(({ className, size, ...props }, ref) => (
  <AvatarPrimitive.Root
    ref={ref}
    className={cn(avatarVariants({ size, className }))}
    {...props}
  />
));
Avatar.displayName = AvatarPrimitive.Root.displayName;

const AvatarImage = forwardRef<
  React.ElementRef<typeof AvatarPrimitive.Image>,
  React.ComponentPropsWithoutRef<typeof AvatarPrimitive.Image>
>(({ className, ...props }, ref) => (
  <AvatarPrimitive.Image
    ref={ref}
    className={cn('aspect-square h-full w-full', className)}
    {...props}
  />
));
AvatarImage.displayName = AvatarPrimitive.Image.displayName;

const AvatarFallback = forwardRef<
  React.ElementRef<typeof AvatarPrimitive.Fallback>,
  React.ComponentPropsWithoutRef<typeof AvatarPrimitive.Fallback>
>(({ className, ...props }, ref) => (
  <AvatarPrimitive.Fallback
    ref={ref}
    className={cn(
      'flex h-full w-full items-center justify-center rounded-full bg-muted text-muted-foreground font-medium',
      className
    )}
    {...props}
  />
));
AvatarFallback.displayName = AvatarPrimitive.Fallback.displayName;

export { Avatar, AvatarImage, AvatarFallback, avatarVariants };
export type { AvatarProps };
```

---

## 4. Composite Components

### 4.1 Card

```typescript
// components/ui/card.tsx
import { forwardRef } from 'react';
import { cn } from '@/lib/utils';

const Card = forwardRef<HTMLDivElement, React.HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div
      ref={ref}
      className={cn(
        'rounded-lg border bg-card text-card-foreground shadow-sm',
        className
      )}
      {...props}
    />
  )
);
Card.displayName = 'Card';

const CardHeader = forwardRef<HTMLDivElement, React.HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div
      ref={ref}
      className={cn('flex flex-col space-y-1.5 p-6', className)}
      {...props}
    />
  )
);
CardHeader.displayName = 'CardHeader';

const CardTitle = forwardRef<HTMLHeadingElement, React.HTMLAttributes<HTMLHeadingElement>>(
  ({ className, ...props }, ref) => (
    <h3
      ref={ref}
      className={cn('text-2xl font-semibold leading-none tracking-tight', className)}
      {...props}
    />
  )
);
CardTitle.displayName = 'CardTitle';

const CardDescription = forwardRef<HTMLParagraphElement, React.HTMLAttributes<HTMLParagraphElement>>(
  ({ className, ...props }, ref) => (
    <p
      ref={ref}
      className={cn('text-sm text-muted-foreground', className)}
      {...props}
    />
  )
);
CardDescription.displayName = 'CardDescription';

const CardContent = forwardRef<HTMLDivElement, React.HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div ref={ref} className={cn('p-6 pt-0', className)} {...props} />
  )
);
CardContent.displayName = 'CardContent';

const CardFooter = forwardRef<HTMLDivElement, React.HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div
      ref={ref}
      className={cn('flex items-center p-6 pt-0', className)}
      {...props}
    />
  )
);
CardFooter.displayName = 'CardFooter';

export { Card, CardHeader, CardFooter, CardTitle, CardDescription, CardContent };
```

### 4.2 Modal/Dialog

```typescript
// components/common/Modal/Modal.tsx
import * as Dialog from '@radix-ui/react-dialog';
import { X } from 'lucide-react';
import { cn } from '@/lib/utils';

interface ModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  title?: string;
  description?: string;
  children: React.ReactNode;
  className?: string;
  size?: 'sm' | 'md' | 'lg' | 'xl' | 'full';
}

const sizeClasses = {
  sm: 'max-w-sm',
  md: 'max-w-md',
  lg: 'max-w-lg',
  xl: 'max-w-xl',
  full: 'max-w-[90vw]',
};

export function Modal({
  open,
  onOpenChange,
  title,
  description,
  children,
  className,
  size = 'md',
}: ModalProps) {
  return (
    <Dialog.Root open={open} onOpenChange={onOpenChange}>
      <Dialog.Portal>
        <Dialog.Overlay className="fixed inset-0 z-modal-backdrop bg-black/50 data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0" />
        <Dialog.Content
          className={cn(
            'fixed left-[50%] top-[50%] z-modal w-full translate-x-[-50%] translate-y-[-50%]',
            'rounded-lg bg-background p-6 shadow-lg',
            'data-[state=open]:animate-in data-[state=closed]:animate-out',
            'data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0',
            'data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95',
            'data-[state=closed]:slide-out-to-left-1/2 data-[state=closed]:slide-out-to-top-[48%]',
            'data-[state=open]:slide-in-from-left-1/2 data-[state=open]:slide-in-from-top-[48%]',
            sizeClasses[size],
            className
          )}
        >
          {title && (
            <Dialog.Title className="text-lg font-semibold">
              {title}
            </Dialog.Title>
          )}
          {description && (
            <Dialog.Description className="mt-1 text-sm text-muted-foreground">
              {description}
            </Dialog.Description>
          )}
          <div className="mt-4">{children}</div>
          <Dialog.Close asChild>
            <button
              className="absolute right-4 top-4 rounded-sm opacity-70 ring-offset-background transition-opacity hover:opacity-100 focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2"
              aria-label="Close"
            >
              <X className="h-4 w-4" />
            </button>
          </Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

### 4.3 Toast Notification

```typescript
// components/common/Toast/Toast.tsx
import { useEffect } from 'react';
import { motion, AnimatePresence } from 'framer-motion';
import { X, CheckCircle, AlertCircle, AlertTriangle, Info } from 'lucide-react';
import { useUIStore } from '@/stores/uiStore';
import { cn } from '@/lib/utils';

const icons = {
  success: CheckCircle,
  error: AlertCircle,
  warning: AlertTriangle,
  info: Info,
};

const colors = {
  success: 'bg-success text-success-foreground border-success',
  error: 'bg-destructive text-destructive-foreground border-destructive',
  warning: 'bg-warning text-warning-foreground border-warning',
  info: 'bg-primary text-primary-foreground border-primary',
};

export function ToastContainer() {
  const { toasts, removeToast } = useUIStore();
  
  return (
    <div className="fixed bottom-4 right-4 z-toast flex flex-col gap-2">
      <AnimatePresence mode="popLayout">
        {toasts.map((toast) => {
          const Icon = icons[toast.type];
          
          return (
            <motion.div
              key={toast.id}
              initial={{ opacity: 0, y: 50, scale: 0.3 }}
              animate={{ opacity: 1, y: 0, scale: 1 }}
              exit={{ opacity: 0, scale: 0.5, transition: { duration: 0.2 } }}
              className={cn(
                'flex items-start gap-3 rounded-lg border p-4 shadow-lg',
                'min-w-[300px] max-w-[400px]',
                colors[toast.type]
              )}
            >
              <Icon className="h-5 w-5 shrink-0" />
              <div className="flex-1">
                <p className="font-medium">{toast.title}</p>
                {toast.message && (
                  <p className="mt-1 text-sm opacity-90">{toast.message}</p>
                )}
              </div>
              <button
                onClick={() => removeToast(toast.id)}
                className="shrink-0 rounded-sm opacity-70 hover:opacity-100"
              >
                <X className="h-4 w-4" />
              </button>
            </motion.div>
          );
        })}
      </AnimatePresence>
    </div>
  );
}
```

---

## 5. Feature Components

### 5.1 Story Card

```typescript
// features/stories/components/StoryCard/StoryCard.tsx
import { Link } from 'react-router-dom';
import { Eye, Heart, Star, BookOpen } from 'lucide-react';
import { Card, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Avatar, AvatarImage, AvatarFallback } from '@/components/ui/avatar';
import { cn, formatNumber, getInitials } from '@/lib/utils';
import type { Story } from '../../types';

interface StoryCardProps {
  story: Story;
  variant?: 'default' | 'compact' | 'horizontal';
  className?: string;
  showAuthor?: boolean;
  showStats?: boolean;
}

export function StoryCard({
  story,
  variant = 'default',
  className,
  showAuthor = true,
  showStats = true,
}: StoryCardProps) {
  const statusBadge = {
    COMPLETED: { label: 'Completed', variant: 'success' as const },
    ONGOING: { label: 'Ongoing', variant: 'default' as const },
    HIATUS: { label: 'Hiatus', variant: 'warning' as const },
  };
  
  if (variant === 'horizontal') {
    return (
      <Card className={cn('flex overflow-hidden', className)}>
        <Link to={`/story/${story.slug}`} className="shrink-0">
          <img
            src={story.coverUrl}
            alt={story.title}
            className="h-32 w-24 object-cover"
          />
        </Link>
        <CardContent className="flex flex-1 flex-col justify-between p-4">
          <div>
            <Link to={`/story/${story.slug}`}>
              <h3 className="font-semibold line-clamp-1 hover:text-primary">
                {story.title}
              </h3>
            </Link>
            <p className="mt-1 text-sm text-muted-foreground line-clamp-2">
              {story.description}
            </p>
          </div>
          {showStats && (
            <div className="mt-2 flex items-center gap-4 text-xs text-muted-foreground">
              <span className="flex items-center gap-1">
                <Eye className="h-3 w-3" />
                {formatNumber(story.stats.views)}
              </span>
              <span className="flex items-center gap-1">
                <Heart className="h-3 w-3" />
                {formatNumber(story.stats.likes)}
              </span>
              <span className="flex items-center gap-1">
                <Star className="h-3 w-3" />
                {story.stats.rating.toFixed(1)}
              </span>
            </div>
          )}
        </CardContent>
      </Card>
    );
  }
  
  if (variant === 'compact') {
    return (
      <Link
        to={`/story/${story.slug}`}
        className={cn('group flex items-center gap-3', className)}
      >
        <img
          src={story.coverUrl}
          alt={story.title}
          className="h-16 w-12 rounded object-cover"
        />
        <div className="flex-1 min-w-0">
          <h4 className="font-medium line-clamp-1 group-hover:text-primary">
            {story.title}
          </h4>
          <p className="text-sm text-muted-foreground">
            {story.chapterCount} chapters
          </p>
        </div>
      </Link>
    );
  }
  
  // Default variant
  return (
    <Card className={cn('overflow-hidden group', className)}>
      <Link to={`/story/${story.slug}`} className="block relative">
        <div className="aspect-[2/3] overflow-hidden">
          <img
            src={story.coverUrl}
            alt={story.title}
            className="h-full w-full object-cover transition-transform group-hover:scale-105"
          />
        </div>
        {story.status && statusBadge[story.status] && (
          <Badge
            variant={statusBadge[story.status].variant}
            className="absolute top-2 right-2"
          >
            {statusBadge[story.status].label}
          </Badge>
        )}
      </Link>
      
      <CardContent className="p-4">
        <Link to={`/story/${story.slug}`}>
          <h3 className="font-semibold line-clamp-1 hover:text-primary">
            {story.title}
          </h3>
        </Link>
        
        {showAuthor && story.author && (
          <Link
            to={`/author/${story.author.username}`}
            className="mt-2 flex items-center gap-2 text-sm text-muted-foreground hover:text-foreground"
          >
            <Avatar size="xs">
              <AvatarImage src={story.author.avatarUrl} />
              <AvatarFallback>{getInitials(story.author.displayName)}</AvatarFallback>
            </Avatar>
            <span className="line-clamp-1">{story.author.displayName}</span>
          </Link>
        )}
        
        <div className="mt-2 flex flex-wrap gap-1">
          {story.categories?.slice(0, 2).map((category) => (
            <Badge key={category.id} variant="secondary" className="text-xs">
              {category.name}
            </Badge>
          ))}
        </div>
        
        {showStats && (
          <div className="mt-3 flex items-center justify-between text-xs text-muted-foreground">
            <div className="flex items-center gap-3">
              <span className="flex items-center gap-1">
                <Eye className="h-3 w-3" />
                {formatNumber(story.stats.views)}
              </span>
              <span className="flex items-center gap-1">
                <Heart className="h-3 w-3" />
                {formatNumber(story.stats.likes)}
              </span>
            </div>
            <span className="flex items-center gap-1">
              <Star className="h-3 w-3 fill-warning text-warning" />
              {story.stats.rating.toFixed(1)}
            </span>
          </div>
        )}
        
        <div className="mt-2 flex items-center gap-1 text-xs text-muted-foreground">
          <BookOpen className="h-3 w-3" />
          <span>{story.chapterCount} chapters</span>
        </div>
      </CardContent>
    </Card>
  );
}
```

### 5.2 Story Card Skeleton

```typescript
// components/skeletons/StoryCardSkeleton.tsx
import { Card, CardContent } from '@/components/ui/card';
import { Skeleton } from '@/components/ui/skeleton';
import { cn } from '@/lib/utils';

interface StoryCardSkeletonProps {
  variant?: 'default' | 'compact' | 'horizontal';
  className?: string;
}

export function StoryCardSkeleton({ variant = 'default', className }: StoryCardSkeletonProps) {
  if (variant === 'horizontal') {
    return (
      <Card className={cn('flex overflow-hidden', className)}>
        <Skeleton className="h-32 w-24 shrink-0" />
        <CardContent className="flex flex-1 flex-col justify-between p-4">
          <div>
            <Skeleton className="h-5 w-3/4" />
            <Skeleton className="mt-2 h-4 w-full" />
            <Skeleton className="mt-1 h-4 w-2/3" />
          </div>
          <div className="mt-2 flex gap-4">
            <Skeleton className="h-4 w-12" />
            <Skeleton className="h-4 w-12" />
            <Skeleton className="h-4 w-12" />
          </div>
        </CardContent>
      </Card>
    );
  }
  
  if (variant === 'compact') {
    return (
      <div className={cn('flex items-center gap-3', className)}>
        <Skeleton className="h-16 w-12 rounded" />
        <div className="flex-1">
          <Skeleton className="h-4 w-3/4" />
          <Skeleton className="mt-2 h-3 w-1/2" />
        </div>
      </div>
    );
  }
  
  return (
    <Card className={cn('overflow-hidden', className)}>
      <Skeleton className="aspect-[2/3]" />
      <CardContent className="p-4">
        <Skeleton className="h-5 w-3/4" />
        <div className="mt-2 flex items-center gap-2">
          <Skeleton className="h-6 w-6 rounded-full" />
          <Skeleton className="h-4 w-24" />
        </div>
        <div className="mt-2 flex gap-1">
          <Skeleton className="h-5 w-16" />
          <Skeleton className="h-5 w-14" />
        </div>
        <div className="mt-3 flex justify-between">
          <Skeleton className="h-4 w-20" />
          <Skeleton className="h-4 w-12" />
        </div>
      </CardContent>
    </Card>
  );
}
```

### 5.3 Chapter List

```typescript
// features/stories/components/ChapterList/ChapterList.tsx
import { Link } from 'react-router-dom';
import { Lock, Eye, Clock, CheckCircle } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { cn, formatNumber, formatRelativeTime } from '@/lib/utils';
import type { ChapterSummary } from '../../types';

interface ChapterListProps {
  storySlug: string;
  chapters: ChapterSummary[];
  readChapters?: Set<string>;
  className?: string;
}

export function ChapterList({
  storySlug,
  chapters,
  readChapters = new Set(),
  className,
}: ChapterListProps) {
  return (
    <div className={cn('divide-y', className)}>
      {chapters.map((chapter) => {
        const isRead = readChapters.has(chapter.id);
        const isLocked = chapter.isPremium && !chapter.isUnlocked;
        
        return (
          <div
            key={chapter.id}
            className={cn(
              'flex items-center justify-between py-3',
              isRead && 'opacity-60'
            )}
          >
            <div className="flex items-center gap-3 min-w-0 flex-1">
              {isRead && (
                <CheckCircle className="h-4 w-4 text-success shrink-0" />
              )}
              
              <Link
                to={isLocked ? '#' : `/story/${storySlug}/chapter/${chapter.number}`}
                className={cn(
                  'flex-1 min-w-0',
                  isLocked && 'pointer-events-none'
                )}
              >
                <div className="flex items-center gap-2">
                  <span className="text-sm text-muted-foreground shrink-0">
                    Ch. {chapter.number}
                  </span>
                  <h4 className={cn(
                    'font-medium line-clamp-1',
                    !isLocked && 'hover:text-primary'
                  )}>
                    {chapter.title}
                  </h4>
                </div>
                
                <div className="mt-1 flex items-center gap-3 text-xs text-muted-foreground">
                  <span className="flex items-center gap-1">
                    <Clock className="h-3 w-3" />
                    {formatRelativeTime(chapter.publishedAt)}
                  </span>
                  <span className="flex items-center gap-1">
                    <Eye className="h-3 w-3" />
                    {formatNumber(chapter.views)}
                  </span>
                </div>
              </Link>
            </div>
            
            {isLocked ? (
              <Button variant="outline" size="sm" className="shrink-0 gap-1">
                <Lock className="h-3 w-3" />
                <span>{chapter.coinPrice}</span>
              </Button>
            ) : chapter.isPremium ? (
              <Badge variant="success" className="shrink-0">Unlocked</Badge>
            ) : null}
          </div>
        );
      })}
    </div>
  );
}
```

### 5.4 Comment Thread

```typescript
// features/interactions/components/CommentThread/CommentThread.tsx
import { useState } from 'react';
import { Link } from 'react-router-dom';
import { Heart, MessageCircle, MoreHorizontal, Flag, Trash } from 'lucide-react';
import { Avatar, AvatarImage, AvatarFallback } from '@/components/ui/avatar';
import { Button } from '@/components/ui/button';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';
import { cn, formatRelativeTime, getInitials } from '@/lib/utils';
import { useAuthStore } from '@/stores/authStore';
import type { Comment } from '../../types';

interface CommentThreadProps {
  comment: Comment;
  onLike: (commentId: string) => void;
  onReply: (commentId: string) => void;
  onDelete: (commentId: string) => void;
  onReport: (commentId: string) => void;
  depth?: number;
  maxDepth?: number;
}

export function CommentThread({
  comment,
  onLike,
  onReply,
  onDelete,
  onReport,
  depth = 0,
  maxDepth = 3,
}: CommentThreadProps) {
  const [showReplies, setShowReplies] = useState(depth < 2);
  const { user } = useAuthStore();
  const isOwner = user?.id === comment.author.id;
  
  return (
    <div className={cn('group', depth > 0 && 'ml-8 mt-4')}>
      <div className="flex gap-3">
        <Link to={`/author/${comment.author.username}`}>
          <Avatar size="sm">
            <AvatarImage src={comment.author.avatarUrl} />
            <AvatarFallback>{getInitials(comment.author.displayName)}</AvatarFallback>
          </Avatar>
        </Link>
        
        <div className="flex-1 min-w-0">
          <div className="flex items-center gap-2">
            <Link
              to={`/author/${comment.author.username}`}
              className="font-medium hover:text-primary"
            >
              {comment.author.displayName}
            </Link>
            {comment.author.isAuthor && (
              <span className="rounded bg-primary/10 px-1.5 py-0.5 text-xs text-primary">
                Author
              </span>
            )}
            <span className="text-xs text-muted-foreground">
              {formatRelativeTime(comment.createdAt)}
            </span>
          </div>
          
          <p className="mt-1 text-sm whitespace-pre-wrap">{comment.content}</p>
          
          <div className="mt-2 flex items-center gap-4">
            <Button
              variant="ghost"
              size="sm"
              className={cn(
                'h-auto p-0 text-muted-foreground hover:text-foreground',
                comment.isLiked && 'text-destructive'
              )}
              onClick={() => onLike(comment.id)}
            >
              <Heart className={cn('h-4 w-4', comment.isLiked && 'fill-current')} />
              <span className="ml-1 text-xs">{comment.likeCount}</span>
            </Button>
            
            {depth < maxDepth && (
              <Button
                variant="ghost"
                size="sm"
                className="h-auto p-0 text-muted-foreground hover:text-foreground"
                onClick={() => onReply(comment.id)}
              >
                <MessageCircle className="h-4 w-4" />
                <span className="ml-1 text-xs">Reply</span>
              </Button>
            )}
            
            <DropdownMenu>
              <DropdownMenuTrigger asChild>
                <Button
                  variant="ghost"
                  size="sm"
                  className="h-auto p-0 text-muted-foreground opacity-0 group-hover:opacity-100 hover:text-foreground"
                >
                  <MoreHorizontal className="h-4 w-4" />
                </Button>
              </DropdownMenuTrigger>
              <DropdownMenuContent align="end">
                {isOwner ? (
                  <DropdownMenuItem
                    className="text-destructive"
                    onClick={() => onDelete(comment.id)}
                  >
                    <Trash className="mr-2 h-4 w-4" />
                    Delete
                  </DropdownMenuItem>
                ) : (
                  <DropdownMenuItem onClick={() => onReport(comment.id)}>
                    <Flag className="mr-2 h-4 w-4" />
                    Report
                  </DropdownMenuItem>
                )}
              </DropdownMenuContent>
            </DropdownMenu>
          </div>
        </div>
      </div>
      
      {comment.replies && comment.replies.length > 0 && (
        <>
          {!showReplies ? (
            <Button
              variant="link"
              size="sm"
              className="ml-11 mt-2"
              onClick={() => setShowReplies(true)}
            >
              Show {comment.replies.length} replies
            </Button>
          ) : (
            <div className="mt-2">
              {comment.replies.map((reply) => (
                <CommentThread
                  key={reply.id}
                  comment={reply}
                  onLike={onLike}
                  onReply={onReply}
                  onDelete={onDelete}
                  onReport={onReport}
                  depth={depth + 1}
                  maxDepth={maxDepth}
                />
              ))}
            </div>
          )}
        </>
      )}
    </div>
  );
}
```

### 5.5 Rating Display

```typescript
// features/interactions/components/RatingDisplay/RatingDisplay.tsx
import { Star, StarHalf } from 'lucide-react';
import { cn } from '@/lib/utils';

interface RatingDisplayProps {
  rating: number;
  ratingCount?: number;
  size?: 'sm' | 'md' | 'lg';
  showValue?: boolean;
  className?: string;
}

const sizeClasses = {
  sm: 'h-3 w-3',
  md: 'h-4 w-4',
  lg: 'h-5 w-5',
};

export function RatingDisplay({
  rating,
  ratingCount,
  size = 'md',
  showValue = true,
  className,
}: RatingDisplayProps) {
  const fullStars = Math.floor(rating);
  const hasHalfStar = rating % 1 >= 0.5;
  const emptyStars = 5 - fullStars - (hasHalfStar ? 1 : 0);
  
  return (
    <div className={cn('flex items-center gap-1', className)}>
      <div className="flex">
        {[...Array(fullStars)].map((_, i) => (
          <Star
            key={`full-${i}`}
            className={cn(sizeClasses[size], 'fill-warning text-warning')}
          />
        ))}
        {hasHalfStar && (
          <div className="relative">
            <Star className={cn(sizeClasses[size], 'text-muted-foreground/30')} />
            <div className="absolute inset-0 overflow-hidden" style={{ width: '50%' }}>
              <Star className={cn(sizeClasses[size], 'fill-warning text-warning')} />
            </div>
          </div>
        )}
        {[...Array(emptyStars)].map((_, i) => (
          <Star
            key={`empty-${i}`}
            className={cn(sizeClasses[size], 'text-muted-foreground/30')}
          />
        ))}
      </div>
      
      {showValue && (
        <span className="text-sm font-medium">{rating.toFixed(1)}</span>
      )}
      
      {ratingCount !== undefined && (
        <span className="text-sm text-muted-foreground">
          ({ratingCount.toLocaleString()})
        </span>
      )}
    </div>
  );
}
```

### 5.6 Wallet Balance

```typescript
// features/payments/components/WalletBalance/WalletBalance.tsx
import { Link } from 'react-router-dom';
import { Coins, Plus } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { cn, formatNumber } from '@/lib/utils';
import { useWallet } from '../../hooks/useWallet';

interface WalletBalanceProps {
  variant?: 'header' | 'full';
  className?: string;
}

export function WalletBalance({ variant = 'header', className }: WalletBalanceProps) {
  const { data: wallet, isLoading } = useWallet();
  
  if (variant === 'header') {
    return (
      <Link
        to="/wallet"
        className={cn(
          'flex items-center gap-1.5 rounded-full bg-primary/10 px-3 py-1.5 text-sm font-medium text-primary hover:bg-primary/20 transition-colors',
          className
        )}
      >
        <Coins className="h-4 w-4" />
        {isLoading ? (
          <span className="w-8 h-4 animate-pulse bg-primary/20 rounded" />
        ) : (
          <span>{formatNumber(wallet?.balance ?? 0)}</span>
        )}
      </Link>
    );
  }
  
  // Full variant
  return (
    <div className={cn('rounded-lg border bg-card p-6', className)}>
      <div className="flex items-center justify-between">
        <div>
          <p className="text-sm text-muted-foreground">Available Balance</p>
          <div className="mt-1 flex items-center gap-2">
            <Coins className="h-6 w-6 text-warning" />
            {isLoading ? (
              <span className="w-20 h-8 animate-pulse bg-muted rounded" />
            ) : (
              <span className="text-3xl font-bold">
                {formatNumber(wallet?.balance ?? 0)}
              </span>
            )}
            <span className="text-lg text-muted-foreground">coins</span>
          </div>
        </div>
        
        <Button asChild>
          <Link to="/wallet/buy-coins">
            <Plus className="mr-2 h-4 w-4" />
            Buy Coins
          </Link>
        </Button>
      </div>
      
      {wallet?.pendingEarnings && wallet.pendingEarnings > 0 && (
        <div className="mt-4 rounded-md bg-muted p-3">
          <p className="text-sm text-muted-foreground">
            Pending Earnings:{' '}
            <span className="font-medium text-foreground">
              {formatNumber(wallet.pendingEarnings)} coins
            </span>
          </p>
        </div>
      )}
    </div>
  );
}
```

---

## 6. Layout Components

### 6.1 Page Container

```typescript
// components/layout/PageContainer.tsx
import { cn } from '@/lib/utils';

interface PageContainerProps {
  children: React.ReactNode;
  className?: string;
  size?: 'default' | 'narrow' | 'wide' | 'full';
}

const sizeClasses = {
  default: 'max-w-7xl',
  narrow: 'max-w-4xl',
  wide: 'max-w-[1400px]',
  full: 'max-w-full',
};

export function PageContainer({ 
  children, 
  className, 
  size = 'default' 
}: PageContainerProps) {
  return (
    <div className={cn('container mx-auto px-4', sizeClasses[size], className)}>
      {children}
    </div>
  );
}
```

### 6.2 Section

```typescript
// components/layout/Section.tsx
import { cn } from '@/lib/utils';

interface SectionProps {
  children: React.ReactNode;
  className?: string;
  title?: string;
  description?: string;
  action?: React.ReactNode;
}

export function Section({
  children,
  className,
  title,
  description,
  action,
}: SectionProps) {
  return (
    <section className={cn('py-8', className)}>
      {(title || action) && (
        <div className="mb-6 flex items-center justify-between">
          <div>
            {title && (
              <h2 className="text-2xl font-bold tracking-tight">{title}</h2>
            )}
            {description && (
              <p className="mt-1 text-muted-foreground">{description}</p>
            )}
          </div>
          {action}
        </div>
      )}
      {children}
    </section>
  );
}
```

---

## 7. Component Index

### 7.1 Primitive Components

| Component | Location | Status |
|-----------|----------|--------|
| Button | `components/ui/button.tsx` | ✅ Complete |
| Input | `components/ui/input.tsx` | ✅ Complete |
| Textarea | `components/ui/textarea.tsx` | ✅ Complete |
| Select | `components/ui/select.tsx` | ✅ Complete |
| Checkbox | `components/ui/checkbox.tsx` | ✅ Complete |
| Radio | `components/ui/radio-group.tsx` | ✅ Complete |
| Switch | `components/ui/switch.tsx` | ✅ Complete |
| Badge | `components/ui/badge.tsx` | ✅ Complete |
| Avatar | `components/ui/avatar.tsx` | ✅ Complete |
| Skeleton | `components/ui/skeleton.tsx` | ✅ Complete |
| Separator | `components/ui/separator.tsx` | ✅ Complete |

### 7.2 Composite Components

| Component | Location | Status |
|-----------|----------|--------|
| Card | `components/ui/card.tsx` | ✅ Complete |
| Modal/Dialog | `components/common/Modal/` | ✅ Complete |
| Dropdown | `components/ui/dropdown-menu.tsx` | ✅ Complete |
| Tabs | `components/ui/tabs.tsx` | ✅ Complete |
| Accordion | `components/ui/accordion.tsx` | ✅ Complete |
| Toast | `components/common/Toast/` | ✅ Complete |
| Alert | `components/ui/alert.tsx` | ✅ Complete |
| Tooltip | `components/ui/tooltip.tsx` | ✅ Complete |
| Popover | `components/ui/popover.tsx` | ✅ Complete |
| Progress | `components/ui/progress.tsx` | ✅ Complete |
| Pagination | `components/common/Pagination/` | ✅ Complete |
| DataTable | `components/common/DataTable/` | ✅ Complete |

### 7.3 Feature Components

| Component | Location | Status |
|-----------|----------|--------|
| StoryCard | `features/stories/components/StoryCard/` | ✅ Complete |
| ChapterList | `features/stories/components/ChapterList/` | ✅ Complete |
| CommentThread | `features/interactions/components/CommentThread/` | ✅ Complete |
| RatingDisplay | `features/interactions/components/RatingDisplay/` | ✅ Complete |
| RatingInput | `features/interactions/components/RatingInput/` | ✅ Complete |
| UserCard | `features/users/components/UserCard/` | ✅ Complete |
| WalletBalance | `features/payments/components/WalletBalance/` | ✅ Complete |
| SearchBar | `features/search/components/SearchBar/` | ✅ Complete |
| GenreSelector | `features/browse/components/GenreSelector/` | ✅ Complete |

---

## 8. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Frontend Architecture Team | Initial release |

---

## 9. Related Documents

- [03_DESIGN_SYSTEM.md](./03_DESIGN_SYSTEM.md) - Design tokens and styling
- [08_PAGES_SPECIFICATION.md](./08_PAGES_SPECIFICATION.md) - Page compositions
- [12_ACCESSIBILITY_I18N.md](./12_ACCESSIBILITY_I18N.md) - Accessibility guidelines
