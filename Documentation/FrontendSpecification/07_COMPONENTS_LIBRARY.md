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

---

## 8. Extended Feature Components (Aligned with Backend Design)

### 8.1 Reading List Components (Backend: 04_INTERACTIONS_COMPONENT)

#### 8.1.1 Reading List Card

```typescript
// features/library/components/ReadingListCard/ReadingListCard.tsx
import { Link } from 'react-router-dom';
import { MoreHorizontal, Globe, Lock, BookOpen, Trash, Pencil } from 'lucide-react';
import { Card, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';
import { cn } from '@/lib/utils';
import type { ReadingList } from '../../types';

interface ReadingListCardProps {
  list: ReadingList;
  onEdit: (id: string) => void;
  onDelete: (id: string) => void;
  className?: string;
}

export function ReadingListCard({
  list,
  onEdit,
  onDelete,
  className,
}: ReadingListCardProps) {
  return (
    <Card className={cn('group relative overflow-hidden', className)}>
      <Link to={`/library/lists/${list.id}`}>
        {/* Cover collage */}
        <div className="aspect-[4/3] bg-muted relative overflow-hidden">
          {list.covers && list.covers.length > 0 ? (
            <div className="grid grid-cols-2 h-full">
              {list.covers.slice(0, 4).map((cover, idx) => (
                <img
                  key={idx}
                  src={cover}
                  alt=""
                  className="h-full w-full object-cover"
                />
              ))}
            </div>
          ) : (
            <div className="flex h-full items-center justify-center">
              <BookOpen className="h-12 w-12 text-muted-foreground/50" />
            </div>
          )}
          
          {/* Visibility badge */}
          <div className="absolute top-2 left-2">
            {list.isPublic ? (
              <Badge variant="secondary" className="gap-1">
                <Globe className="h-3 w-3" /> Public
              </Badge>
            ) : (
              <Badge variant="outline" className="gap-1 bg-background/80">
                <Lock className="h-3 w-3" /> Private
              </Badge>
            )}
          </div>
          
          {list.isDefault && (
            <Badge className="absolute top-2 right-2">Default</Badge>
          )}
        </div>
      </Link>
      
      <CardContent className="p-4">
        <div className="flex items-start justify-between">
          <div className="min-w-0 flex-1">
            <Link to={`/library/lists/${list.id}`}>
              <h3 className="font-semibold line-clamp-1 group-hover:text-primary">
                {list.name}
              </h3>
            </Link>
            {list.description && (
              <p className="mt-1 text-sm text-muted-foreground line-clamp-2">
                {list.description}
              </p>
            )}
            <p className="mt-2 text-xs text-muted-foreground">
              {list.storyCount} {list.storyCount === 1 ? 'story' : 'stories'}
            </p>
          </div>
          
          {!list.isDefault && (
            <DropdownMenu>
              <DropdownMenuTrigger asChild>
                <Button
                  variant="ghost"
                  size="icon"
                  className="shrink-0 opacity-0 group-hover:opacity-100"
                >
                  <MoreHorizontal className="h-4 w-4" />
                </Button>
              </DropdownMenuTrigger>
              <DropdownMenuContent align="end">
                <DropdownMenuItem onClick={() => onEdit(list.id)}>
                  <Pencil className="mr-2 h-4 w-4" />
                  Edit
                </DropdownMenuItem>
                <DropdownMenuSeparator />
                <DropdownMenuItem
                  className="text-destructive"
                  onClick={() => onDelete(list.id)}
                >
                  <Trash className="mr-2 h-4 w-4" />
                  Delete
                </DropdownMenuItem>
              </DropdownMenuContent>
            </DropdownMenu>
          )}
        </div>
      </CardContent>
    </Card>
  );
}
```

#### 8.1.2 Add to Reading List Modal

```typescript
// features/library/components/AddToListModal/AddToListModal.tsx
import { useState } from 'react';
import { Plus, Check, BookOpen } from 'lucide-react';
import { Modal } from '@/components/common/Modal';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Checkbox } from '@/components/ui/checkbox';
import { ScrollArea } from '@/components/ui/scroll-area';
import { cn } from '@/lib/utils';
import { useReadingLists, useAddToList, useCreateList } from '../../hooks';
import type { ReadingList } from '../../types';

interface AddToListModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  storyId: string;
  storyTitle: string;
}

export function AddToListModal({
  open,
  onOpenChange,
  storyId,
  storyTitle,
}: AddToListModalProps) {
  const [showCreate, setShowCreate] = useState(false);
  const [newListName, setNewListName] = useState('');
  
  const { data: lists, isLoading } = useReadingLists();
  const addToList = useAddToList();
  const createList = useCreateList();
  
  const handleAddToList = async (listId: string) => {
    await addToList.mutateAsync({ listId, storyId });
  };
  
  const handleCreateList = async () => {
    if (!newListName.trim()) return;
    
    const newList = await createList.mutateAsync({ name: newListName });
    await addToList.mutateAsync({ listId: newList.id, storyId });
    setNewListName('');
    setShowCreate(false);
    onOpenChange(false);
  };
  
  return (
    <Modal
      open={open}
      onOpenChange={onOpenChange}
      title="Add to Reading List"
      description={`Add "${storyTitle}" to a reading list`}
    >
      <div className="space-y-4">
        <ScrollArea className="h-[300px] pr-4">
          {isLoading ? (
            <div className="space-y-2">
              {[...Array(3)].map((_, i) => (
                <div key={i} className="h-12 animate-pulse rounded-md bg-muted" />
              ))}
            </div>
          ) : lists && lists.length > 0 ? (
            <div className="space-y-2">
              {lists.map((list) => (
                <ReadingListItem
                  key={list.id}
                  list={list}
                  storyId={storyId}
                  onAdd={() => handleAddToList(list.id)}
                />
              ))}
            </div>
          ) : (
            <div className="flex flex-col items-center justify-center py-8 text-center">
              <BookOpen className="h-12 w-12 text-muted-foreground/50" />
              <p className="mt-2 text-sm text-muted-foreground">
                No reading lists yet
              </p>
            </div>
          )}
        </ScrollArea>
        
        {showCreate ? (
          <div className="flex gap-2">
            <Input
              placeholder="New list name"
              value={newListName}
              onChange={(e) => setNewListName(e.target.value)}
              autoFocus
            />
            <Button
              onClick={handleCreateList}
              disabled={!newListName.trim() || createList.isPending}
            >
              Create
            </Button>
            <Button variant="ghost" onClick={() => setShowCreate(false)}>
              Cancel
            </Button>
          </div>
        ) : (
          <Button
            variant="outline"
            className="w-full"
            onClick={() => setShowCreate(true)}
          >
            <Plus className="mr-2 h-4 w-4" />
            Create New List
          </Button>
        )}
      </div>
    </Modal>
  );
}

function ReadingListItem({ 
  list, 
  storyId, 
  onAdd 
}: { 
  list: ReadingList; 
  storyId: string; 
  onAdd: () => void;
}) {
  const isInList = list.storyIds?.includes(storyId);
  
  return (
    <button
      className={cn(
        'flex w-full items-center gap-3 rounded-md p-3 text-left transition-colors',
        'hover:bg-accent',
        isInList && 'bg-accent/50'
      )}
      onClick={onAdd}
      disabled={isInList}
    >
      <div className="flex h-10 w-10 items-center justify-center rounded bg-muted">
        {isInList ? (
          <Check className="h-5 w-5 text-success" />
        ) : (
          <BookOpen className="h-5 w-5 text-muted-foreground" />
        )}
      </div>
      <div className="flex-1 min-w-0">
        <p className="font-medium line-clamp-1">{list.name}</p>
        <p className="text-xs text-muted-foreground">
          {list.storyCount} stories
        </p>
      </div>
    </button>
  );
}
```

### 8.2 Review Components (Backend: 04_INTERACTIONS_COMPONENT)

#### 8.2.1 Review Card

```typescript
// features/interactions/components/ReviewCard/ReviewCard.tsx
import { useState } from 'react';
import { Link } from 'react-router-dom';
import { ThumbsUp, ThumbsDown, Flag, MoreHorizontal, AlertTriangle, Pencil, Trash } from 'lucide-react';
import { Avatar, AvatarImage, AvatarFallback } from '@/components/ui/avatar';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';
import { RatingDisplay } from '../RatingDisplay';
import { cn, formatRelativeTime, getInitials } from '@/lib/utils';
import { useAuthStore } from '@/stores/authStore';
import type { Review } from '../../types';

interface ReviewCardProps {
  review: Review;
  onVoteHelpful: (reviewId: string, isHelpful: boolean) => void;
  onEdit?: (reviewId: string) => void;
  onDelete?: (reviewId: string) => void;
  onReport: (reviewId: string) => void;
  className?: string;
}

export function ReviewCard({
  review,
  onVoteHelpful,
  onEdit,
  onDelete,
  onReport,
  className,
}: ReviewCardProps) {
  const [showSpoiler, setShowSpoiler] = useState(!review.hasSpoilers);
  const { user } = useAuthStore();
  const isOwner = user?.id === review.user.id;
  
  return (
    <div className={cn('group rounded-lg border p-4', className)}>
      {/* Header */}
      <div className="flex items-start justify-between gap-4">
        <div className="flex items-center gap-3">
          <Link to={`/author/${review.user.username}`}>
            <Avatar>
              <AvatarImage src={review.user.avatarUrl} />
              <AvatarFallback>{getInitials(review.user.displayName)}</AvatarFallback>
            </Avatar>
          </Link>
          <div>
            <div className="flex items-center gap-2">
              <Link
                to={`/author/${review.user.username}`}
                className="font-medium hover:text-primary"
              >
                {review.user.displayName}
              </Link>
              {review.user.isVerified && (
                <Badge variant="secondary" className="text-xs">Verified</Badge>
              )}
            </div>
            <div className="flex items-center gap-2">
              <RatingDisplay rating={review.rating} showValue={false} size="sm" />
              <span className="text-xs text-muted-foreground">
                {formatRelativeTime(review.createdAt)}
              </span>
            </div>
          </div>
        </div>
        
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button
              variant="ghost"
              size="icon"
              className="shrink-0 opacity-0 group-hover:opacity-100"
            >
              <MoreHorizontal className="h-4 w-4" />
            </Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent align="end">
            {isOwner ? (
              <>
                {onEdit && (
                  <DropdownMenuItem onClick={() => onEdit(review.id)}>
                    <Pencil className="mr-2 h-4 w-4" />
                    Edit
                  </DropdownMenuItem>
                )}
                {onDelete && (
                  <DropdownMenuItem
                    className="text-destructive"
                    onClick={() => onDelete(review.id)}
                  >
                    <Trash className="mr-2 h-4 w-4" />
                    Delete
                  </DropdownMenuItem>
                )}
              </>
            ) : (
              <DropdownMenuItem onClick={() => onReport(review.id)}>
                <Flag className="mr-2 h-4 w-4" />
                Report
              </DropdownMenuItem>
            )}
          </DropdownMenuContent>
        </DropdownMenu>
      </div>
      
      {/* Title */}
      {review.title && (
        <h4 className="mt-3 font-semibold">{review.title}</h4>
      )}
      
      {/* Content with spoiler protection */}
      <div className="mt-2">
        {review.hasSpoilers && !showSpoiler ? (
          <div className="rounded-md bg-muted p-4 text-center">
            <AlertTriangle className="mx-auto h-6 w-6 text-warning" />
            <p className="mt-2 text-sm font-medium">This review contains spoilers</p>
            <Button
              variant="outline"
              size="sm"
              className="mt-2"
              onClick={() => setShowSpoiler(true)}
            >
              Show Review
            </Button>
          </div>
        ) : (
          <>
            {review.hasSpoilers && (
              <Badge variant="warning" className="mb-2">Contains Spoilers</Badge>
            )}
            <p className="text-sm whitespace-pre-wrap">{review.content}</p>
          </>
        )}
      </div>
      
      {/* Helpful votes */}
      <div className="mt-4 flex items-center gap-4 border-t pt-3">
        <span className="text-sm text-muted-foreground">Was this review helpful?</span>
        <div className="flex items-center gap-2">
          <Button
            variant={review.userVote === 'helpful' ? 'default' : 'ghost'}
            size="sm"
            className="gap-1"
            onClick={() => onVoteHelpful(review.id, true)}
          >
            <ThumbsUp className="h-4 w-4" />
            <span>{review.helpfulCount}</span>
          </Button>
          <Button
            variant={review.userVote === 'not_helpful' ? 'default' : 'ghost'}
            size="sm"
            className="gap-1"
            onClick={() => onVoteHelpful(review.id, false)}
          >
            <ThumbsDown className="h-4 w-4" />
          </Button>
        </div>
      </div>
    </div>
  );
}
```

#### 8.2.2 Review Form

```typescript
// features/interactions/components/ReviewForm/ReviewForm.tsx
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Star, AlertTriangle } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Checkbox } from '@/components/ui/checkbox';
import { Label } from '@/components/ui/label';
import { cn } from '@/lib/utils';

const reviewSchema = z.object({
  rating: z.number().min(1, 'Please select a rating').max(5),
  title: z.string().max(200).optional(),
  content: z.string()
    .min(50, 'Review must be at least 50 characters')
    .max(5000, 'Review cannot exceed 5000 characters'),
  hasSpoilers: z.boolean(),
});

type ReviewFormData = z.infer<typeof reviewSchema>;

interface ReviewFormProps {
  defaultValues?: Partial<ReviewFormData>;
  onSubmit: (data: ReviewFormData) => Promise<void>;
  onCancel?: () => void;
  isSubmitting?: boolean;
}

export function ReviewForm({
  defaultValues,
  onSubmit,
  onCancel,
  isSubmitting,
}: ReviewFormProps) {
  const form = useForm<ReviewFormData>({
    resolver: zodResolver(reviewSchema),
    defaultValues: {
      rating: 0,
      title: '',
      content: '',
      hasSpoilers: false,
      ...defaultValues,
    },
  });
  
  const { watch, setValue } = form;
  const rating = watch('rating');
  const content = watch('content');
  
  return (
    <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
      {/* Rating Stars */}
      <div>
        <Label className="text-base font-medium">Your Rating</Label>
        <Controller
          control={form.control}
          name="rating"
          render={({ field, fieldState }) => (
            <>
              <div className="mt-2 flex gap-1">
                {[1, 2, 3, 4, 5].map((star) => (
                  <button
                    key={star}
                    type="button"
                    onClick={() => setValue('rating', star)}
                    className="p-1 focus:outline-none"
                  >
                    <Star
                      className={cn(
                        'h-8 w-8 transition-colors',
                        star <= field.value
                          ? 'fill-warning text-warning'
                          : 'text-muted-foreground/30 hover:text-warning/50'
                      )}
                    />
                  </button>
                ))}
              </div>
              {fieldState.error && (
                <p className="mt-1 text-sm text-destructive">
                  {fieldState.error.message}
                </p>
              )}
            </>
          )}
        />
      </div>
      
      {/* Title (optional) */}
      <div>
        <Label htmlFor="title">Review Title (optional)</Label>
        <Input
          id="title"
          {...form.register('title')}
          placeholder="Summarize your review"
          className="mt-1"
        />
      </div>
      
      {/* Content */}
      <div>
        <Label htmlFor="content">Your Review</Label>
        <Textarea
          id="content"
          {...form.register('content')}
          placeholder="What did you think about this story?"
          rows={6}
          className="mt-1"
        />
        <div className="mt-1 flex justify-between text-xs text-muted-foreground">
          <span>Minimum 50 characters</span>
          <span className={cn(
            content.length > 5000 && 'text-destructive'
          )}>
            {content.length} / 5000
          </span>
        </div>
        {form.formState.errors.content && (
          <p className="mt-1 text-sm text-destructive">
            {form.formState.errors.content.message}
          </p>
        )}
      </div>
      
      {/* Spoiler checkbox */}
      <div className="flex items-center gap-2 rounded-md border border-warning/50 bg-warning/10 p-3">
        <Controller
          control={form.control}
          name="hasSpoilers"
          render={({ field }) => (
            <Checkbox
              id="spoilers"
              checked={field.value}
              onCheckedChange={field.onChange}
            />
          )}
        />
        <div className="flex items-center gap-2">
          <AlertTriangle className="h-4 w-4 text-warning" />
          <Label htmlFor="spoilers" className="cursor-pointer">
            This review contains spoilers
          </Label>
        </div>
      </div>
      
      {/* Actions */}
      <div className="flex justify-end gap-2">
        {onCancel && (
          <Button type="button" variant="outline" onClick={onCancel}>
            Cancel
          </Button>
        )}
        <Button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Submitting...' : defaultValues ? 'Update Review' : 'Submit Review'}
        </Button>
      </div>
    </form>
  );
}
```

### 8.3 Payment Components (Backend: 03_PAYMENTS_COMPONENT)

#### 8.3.1 Coin Package Card

```typescript
// features/payments/components/CoinPackageCard/CoinPackageCard.tsx
import { Check, Sparkles, Coins } from 'lucide-react';
import { Card, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { cn, formatCurrency } from '@/lib/utils';
import type { CoinPackage } from '../../types';

interface CoinPackageCardProps {
  package_: CoinPackage;
  isSelected?: boolean;
  onSelect: (id: string) => void;
  className?: string;
}

export function CoinPackageCard({
  package_,
  isSelected,
  onSelect,
  className,
}: CoinPackageCardProps) {
  const totalCoins = package_.coins + package_.bonusCoins;
  
  return (
    <Card
      className={cn(
        'relative cursor-pointer transition-all hover:border-primary',
        isSelected && 'border-primary ring-2 ring-primary/20',
        package_.isPopular && 'border-warning',
        className
      )}
      onClick={() => onSelect(package_.id)}
    >
      {package_.isPopular && (
        <Badge className="absolute -top-2 left-1/2 -translate-x-1/2 bg-warning">
          <Sparkles className="mr-1 h-3 w-3" />
          Most Popular
        </Badge>
      )}
      
      <CardContent className="p-6">
        <div className="text-center">
          <div className="flex items-center justify-center gap-2">
            <Coins className="h-8 w-8 text-warning" />
            <span className="text-3xl font-bold">{package_.coins}</span>
          </div>
          
          {package_.bonusCoins > 0 && (
            <Badge variant="success" className="mt-2">
              +{package_.bonusCoins} bonus coins
            </Badge>
          )}
          
          <div className="mt-4">
            {package_.discountPercent > 0 && (
              <span className="text-sm text-muted-foreground line-through">
                {formatCurrency(package_.originalPrice || package_.price * 1.2)}
              </span>
            )}
            <p className="text-2xl font-bold">
              {formatCurrency(package_.price, package_.currency)}
            </p>
          </div>
          
          <p className="mt-1 text-xs text-muted-foreground">
            {(package_.price / totalCoins).toFixed(3)} per coin
          </p>
        </div>
        
        {isSelected && (
          <div className="mt-4 flex items-center justify-center gap-2 text-sm text-primary">
            <Check className="h-4 w-4" />
            Selected
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

#### 8.3.2 Donation Modal

```typescript
// features/payments/components/DonationModal/DonationModal.tsx
import { useState } from 'react';
import { Heart, Coins, Gift, MessageSquare } from 'lucide-react';
import { Modal } from '@/components/common/Modal';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Label } from '@/components/ui/label';
import { Avatar, AvatarImage, AvatarFallback } from '@/components/ui/avatar';
import { cn, formatNumber, getInitials } from '@/lib/utils';
import { useWallet, useSendDonation } from '../../hooks';
import type { UserSummary } from '@/features/users/types';

interface DonationModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  recipient: UserSummary;
  storyId?: string;
  storyTitle?: string;
}

const PRESET_AMOUNTS = [10, 50, 100, 500];

export function DonationModal({
  open,
  onOpenChange,
  recipient,
  storyId,
  storyTitle,
}: DonationModalProps) {
  const [amount, setAmount] = useState<number>(50);
  const [customAmount, setCustomAmount] = useState('');
  const [message, setMessage] = useState('');
  
  const { data: wallet } = useWallet();
  const sendDonation = useSendDonation();
  
  const selectedAmount = customAmount ? parseInt(customAmount, 10) : amount;
  const hasEnough = (wallet?.coinBalance ?? 0) >= selectedAmount;
  
  const handleSend = async () => {
    await sendDonation.mutateAsync({
      recipientId: recipient.id,
      amount: selectedAmount,
      message: message || undefined,
      storyId,
    });
    onOpenChange(false);
  };
  
  return (
    <Modal
      open={open}
      onOpenChange={onOpenChange}
      title="Send a Gift"
    >
      <div className="space-y-6">
        {/* Recipient info */}
        <div className="flex items-center gap-3 rounded-lg bg-muted p-4">
          <Avatar size="lg">
            <AvatarImage src={recipient.avatarUrl} />
            <AvatarFallback>{getInitials(recipient.displayName)}</AvatarFallback>
          </Avatar>
          <div>
            <p className="font-medium">{recipient.displayName}</p>
            {storyTitle && (
              <p className="text-sm text-muted-foreground">
                For: {storyTitle}
              </p>
            )}
          </div>
        </div>
        
        {/* Amount selection */}
        <div>
          <Label>Select Amount</Label>
          <div className="mt-2 grid grid-cols-4 gap-2">
            {PRESET_AMOUNTS.map((preset) => (
              <Button
                key={preset}
                type="button"
                variant={amount === preset && !customAmount ? 'default' : 'outline'}
                onClick={() => {
                  setAmount(preset);
                  setCustomAmount('');
                }}
                className="gap-1"
              >
                <Coins className="h-4 w-4" />
                {preset}
              </Button>
            ))}
          </div>
          
          <div className="mt-3">
            <Label htmlFor="custom-amount">Or enter custom amount</Label>
            <div className="mt-1 relative">
              <Coins className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
              <Input
                id="custom-amount"
                type="number"
                min={1}
                placeholder="Enter amount"
                value={customAmount}
                onChange={(e) => setCustomAmount(e.target.value)}
                className="pl-9"
              />
            </div>
          </div>
        </div>
        
        {/* Message */}
        <div>
          <Label htmlFor="message">Message (optional)</Label>
          <div className="mt-1 relative">
            <Textarea
              id="message"
              placeholder="Write a message to the author..."
              value={message}
              onChange={(e) => setMessage(e.target.value)}
              maxLength={500}
              rows={3}
            />
            <span className="absolute bottom-2 right-2 text-xs text-muted-foreground">
              {message.length}/500
            </span>
          </div>
        </div>
        
        {/* Balance & action */}
        <div className="space-y-3 rounded-lg border p-4">
          <div className="flex justify-between text-sm">
            <span className="text-muted-foreground">Your balance</span>
            <span className="flex items-center gap-1">
              <Coins className="h-4 w-4" />
              {formatNumber(wallet?.coinBalance ?? 0)}
            </span>
          </div>
          <div className="flex justify-between text-sm font-medium">
            <span>Total to send</span>
            <span className={cn(
              'flex items-center gap-1',
              !hasEnough && 'text-destructive'
            )}>
              <Coins className="h-4 w-4" />
              {formatNumber(selectedAmount || 0)}
            </span>
          </div>
          
          {!hasEnough && (
            <p className="text-xs text-destructive">
              Insufficient balance. Please top up your wallet.
            </p>
          )}
        </div>
        
        <div className="flex gap-2">
          <Button
            variant="outline"
            className="flex-1"
            onClick={() => onOpenChange(false)}
          >
            Cancel
          </Button>
          <Button
            className="flex-1 gap-2"
            onClick={handleSend}
            disabled={!hasEnough || selectedAmount < 1 || sendDonation.isPending}
          >
            <Gift className="h-4 w-4" />
            {sendDonation.isPending ? 'Sending...' : 'Send Gift'}
          </Button>
        </div>
      </div>
    </Modal>
  );
}
```

#### 8.3.3 Transaction History Item

```typescript
// features/payments/components/TransactionItem/TransactionItem.tsx
import { Coins, ArrowUpRight, ArrowDownLeft, Gift, Unlock, CreditCard, RefreshCcw } from 'lucide-react';
import { Badge } from '@/components/ui/badge';
import { cn, formatNumber, formatRelativeTime } from '@/lib/utils';
import type { Transaction, TransactionType } from '../../types';

const transactionConfig: Record<TransactionType, {
  icon: React.ComponentType<{ className?: string }>;
  label: string;
  color: string;
}> = {
  TOPUP: { icon: CreditCard, label: 'Top Up', color: 'text-success' },
  CHAPTER_UNLOCK: { icon: Unlock, label: 'Chapter Unlock', color: 'text-primary' },
  DONATION_SENT: { icon: Gift, label: 'Gift Sent', color: 'text-warning' },
  DONATION_RECEIVED: { icon: Gift, label: 'Gift Received', color: 'text-success' },
  SUBSCRIPTION: { icon: CreditCard, label: 'Subscription', color: 'text-primary' },
  PAYOUT: { icon: ArrowUpRight, label: 'Payout', color: 'text-muted-foreground' },
  REFUND: { icon: RefreshCcw, label: 'Refund', color: 'text-warning' },
};

interface TransactionItemProps {
  transaction: Transaction;
  className?: string;
}

export function TransactionItem({ transaction, className }: TransactionItemProps) {
  const config = transactionConfig[transaction.type];
  const Icon = config?.icon || Coins;
  const isCredit = transaction.amount > 0;
  
  return (
    <div className={cn('flex items-center justify-between py-3', className)}>
      <div className="flex items-center gap-3">
        <div className={cn(
          'flex h-10 w-10 items-center justify-center rounded-full',
          isCredit ? 'bg-success/10' : 'bg-muted'
        )}>
          <Icon className={cn('h-5 w-5', config?.color)} />
        </div>
        
        <div>
          <p className="font-medium">{config?.label || transaction.type}</p>
          <p className="text-sm text-muted-foreground">
            {transaction.description}
          </p>
          <p className="text-xs text-muted-foreground">
            {formatRelativeTime(transaction.createdAt)}
          </p>
        </div>
      </div>
      
      <div className="text-right">
        <p className={cn(
          'font-semibold',
          isCredit ? 'text-success' : 'text-foreground'
        )}>
          {isCredit ? '+' : ''}{formatNumber(transaction.amount)}
          <Coins className="ml-1 inline h-4 w-4" />
        </p>
        <Badge
          variant={transaction.status === 'COMPLETED' ? 'success' : 
                   transaction.status === 'PENDING' ? 'warning' : 'destructive'}
          className="text-xs"
        >
          {transaction.status}
        </Badge>
      </div>
    </div>
  );
}
```

### 8.4 Notification Components (Backend: 07_NOTIFICATIONS_COMPONENT)

#### 8.4.1 Notification Item

```typescript
// features/notifications/components/NotificationItem/NotificationItem.tsx
import { Link } from 'react-router-dom';
import { 
  Bell, Heart, MessageCircle, BookOpen, UserPlus, 
  Gift, DollarSign, Award, AlertCircle 
} from 'lucide-react';
import { Avatar, AvatarImage, AvatarFallback } from '@/components/ui/avatar';
import { cn, formatRelativeTime, getInitials } from '@/lib/utils';
import type { Notification, NotificationType } from '../../types';

const notificationConfig: Record<NotificationType, {
  icon: React.ComponentType<{ className?: string }>;
  color: string;
}> = {
  NEW_CHAPTER: { icon: BookOpen, color: 'text-primary' },
  NEW_FOLLOWER: { icon: UserPlus, color: 'text-success' },
  STORY_LIKE: { icon: Heart, color: 'text-destructive' },
  STORY_REVIEW: { icon: Award, color: 'text-warning' },
  COMMENT_REPLY: { icon: MessageCircle, color: 'text-primary' },
  COMMENT_LIKE: { icon: Heart, color: 'text-destructive' },
  DONATION_RECEIVED: { icon: Gift, color: 'text-warning' },
  PAYOUT_PROCESSED: { icon: DollarSign, color: 'text-success' },
  STREAK_MILESTONE: { icon: Award, color: 'text-warning' },
  SYSTEM_ANNOUNCEMENT: { icon: Bell, color: 'text-primary' },
};

interface NotificationItemProps {
  notification: Notification;
  onClick?: () => void;
  className?: string;
}

export function NotificationItem({ 
  notification, 
  onClick, 
  className 
}: NotificationItemProps) {
  const config = notificationConfig[notification.type];
  const Icon = config?.icon || Bell;
  
  const getLink = (): string => {
    switch (notification.type) {
      case 'NEW_CHAPTER':
        return `/story/${notification.data.storySlug}/chapter/${notification.data.chapterNumber}`;
      case 'NEW_FOLLOWER':
        return `/author/${notification.data.username}`;
      case 'STORY_LIKE':
      case 'STORY_REVIEW':
        return `/story/${notification.data.storySlug}`;
      case 'COMMENT_REPLY':
      case 'COMMENT_LIKE':
        return `/story/${notification.data.storySlug}/chapter/${notification.data.chapterNumber}#comment-${notification.data.commentId}`;
      case 'DONATION_RECEIVED':
      case 'PAYOUT_PROCESSED':
        return '/author/earnings';
      default:
        return '#';
    }
  };
  
  const content = (
    <div className={cn(
      'flex gap-3 p-4 transition-colors',
      !notification.isRead && 'bg-primary/5',
      'hover:bg-accent',
      className
    )}>
      {/* Avatar or Icon */}
      {notification.actor ? (
        <Avatar size="sm">
          <AvatarImage src={notification.actor.avatarUrl} />
          <AvatarFallback>{getInitials(notification.actor.displayName)}</AvatarFallback>
        </Avatar>
      ) : (
        <div className={cn(
          'flex h-8 w-8 items-center justify-center rounded-full',
          'bg-muted'
        )}>
          <Icon className={cn('h-4 w-4', config?.color)} />
        </div>
      )}
      
      {/* Content */}
      <div className="flex-1 min-w-0">
        <p className={cn(
          'text-sm',
          !notification.isRead && 'font-medium'
        )}>
          {notification.title}
        </p>
        {notification.body && (
          <p className="mt-0.5 text-xs text-muted-foreground line-clamp-2">
            {notification.body}
          </p>
        )}
        <p className="mt-1 text-xs text-muted-foreground">
          {formatRelativeTime(notification.createdAt)}
        </p>
      </div>
      
      {/* Unread indicator */}
      {!notification.isRead && (
        <div className="shrink-0">
          <div className="h-2 w-2 rounded-full bg-primary" />
        </div>
      )}
    </div>
  );
  
  return (
    <Link to={getLink()} onClick={onClick}>
      {content}
    </Link>
  );
}
```

#### 8.4.2 Notification Badge

```typescript
// features/notifications/components/NotificationBadge/NotificationBadge.tsx
import { Bell } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils';
import { useNotificationStore } from '@/stores/notificationStore';

interface NotificationBadgeProps {
  onClick?: () => void;
  className?: string;
}

export function NotificationBadge({ onClick, className }: NotificationBadgeProps) {
  const unreadCount = useNotificationStore((s) => s.unreadCount);
  
  return (
    <Button
      variant="ghost"
      size="icon"
      onClick={onClick}
      className={cn('relative', className)}
    >
      <Bell className="h-5 w-5" />
      {unreadCount > 0 && (
        <span className={cn(
          'absolute -right-1 -top-1 flex items-center justify-center',
          'h-5 min-w-5 rounded-full bg-destructive px-1',
          'text-[10px] font-medium text-destructive-foreground'
        )}>
          {unreadCount > 99 ? '99+' : unreadCount}
        </span>
      )}
    </Button>
  );
}
```

### 8.5 Reading Progress Components (Backend: 05_READINGS_COMPONENT)

#### 8.5.1 Reading Streak Card

```typescript
// features/readings/components/ReadingStreakCard/ReadingStreakCard.tsx
import { Flame, Target, TrendingUp } from 'lucide-react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Progress } from '@/components/ui/progress';
import { Badge } from '@/components/ui/badge';
import { cn } from '@/lib/utils';
import { useStreak, useGoals } from '../../hooks';

interface ReadingStreakCardProps {
  className?: string;
}

export function ReadingStreakCard({ className }: ReadingStreakCardProps) {
  const { data: streak, isLoading: streakLoading } = useStreak();
  const { data: goals, isLoading: goalsLoading } = useGoals();
  
  if (streakLoading || goalsLoading) {
    return (
      <Card className={className}>
        <CardContent className="p-6">
          <div className="h-32 animate-pulse rounded bg-muted" />
        </CardContent>
      </Card>
    );
  }
  
  return (
    <Card className={className}>
      <CardHeader className="pb-2">
        <CardTitle className="text-lg flex items-center gap-2">
          <Flame className="h-5 w-5 text-warning" />
          Reading Streak
        </CardTitle>
      </CardHeader>
      <CardContent>
        <div className="flex items-center justify-between">
          <div>
            <p className="text-4xl font-bold">{streak?.currentStreak ?? 0}</p>
            <p className="text-sm text-muted-foreground">days</p>
          </div>
          
          <div className="text-right">
            <Badge variant="outline" className="gap-1">
              <TrendingUp className="h-3 w-3" />
              Best: {streak?.longestStreak ?? 0} days
            </Badge>
          </div>
        </div>
        
        {goals?.isActive && (
          <div className="mt-4 space-y-3">
            {goals.dailyChaptersGoal > 0 && (
              <div>
                <div className="flex justify-between text-xs mb-1">
                  <span className="text-muted-foreground">Daily chapters</span>
                  <span>{goals.todayChapters ?? 0}/{goals.dailyChaptersGoal}</span>
                </div>
                <Progress 
                  value={((goals.todayChapters ?? 0) / goals.dailyChaptersGoal) * 100} 
                />
              </div>
            )}
            
            {goals.dailyMinutesGoal > 0 && (
              <div>
                <div className="flex justify-between text-xs mb-1">
                  <span className="text-muted-foreground">Reading time</span>
                  <span>{goals.todayMinutes ?? 0}/{goals.dailyMinutesGoal} min</span>
                </div>
                <Progress 
                  value={((goals.todayMinutes ?? 0) / goals.dailyMinutesGoal) * 100} 
                />
              </div>
            )}
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

#### 8.5.2 Continue Reading Card

```typescript
// features/readings/components/ContinueReadingCard/ContinueReadingCard.tsx
import { Link } from 'react-router-dom';
import { Play, BookOpen } from 'lucide-react';
import { Card, CardContent } from '@/components/ui/card';
import { Progress } from '@/components/ui/progress';
import { Button } from '@/components/ui/button';
import { cn, formatRelativeTime } from '@/lib/utils';
import type { ReadingHistory } from '../../types';

interface ContinueReadingCardProps {
  reading: ReadingHistory;
  className?: string;
}

export function ContinueReadingCard({ reading, className }: ContinueReadingCardProps) {
  const { story, currentChapterIndex, completionPercent, lastReadAt } = reading;
  
  return (
    <Card className={cn('overflow-hidden', className)}>
      <div className="flex">
        <Link 
          to={`/story/${story.slug}`}
          className="shrink-0"
        >
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
            <p className="mt-1 text-sm text-muted-foreground">
              Chapter {currentChapterIndex} of {story.chapterCount}
            </p>
            <p className="text-xs text-muted-foreground">
              {formatRelativeTime(lastReadAt)}
            </p>
          </div>
          
          <div className="mt-3 space-y-2">
            <Progress value={completionPercent} className="h-1.5" />
            <div className="flex items-center justify-between">
              <span className="text-xs text-muted-foreground">
                {completionPercent}% complete
              </span>
              <Button size="sm" asChild>
                <Link to={`/story/${story.slug}/chapter/${currentChapterIndex}`}>
                  <Play className="mr-1 h-3 w-3" />
                  Continue
                </Link>
              </Button>
            </div>
          </div>
        </CardContent>
      </div>
    </Card>
  );
}
```

---

## 9. Extended Component Index

### 9.1 Feature Components (Extended)

| Component | Location | Status | Backend Reference |
|-----------|----------|--------|-------------------|
| ReadingListCard | `features/library/components/` | ✅ Complete | 04_INTERACTIONS_COMPONENT |
| AddToListModal | `features/library/components/` | ✅ Complete | 04_INTERACTIONS_COMPONENT |
| ReviewCard | `features/interactions/components/` | ✅ Complete | 04_INTERACTIONS_COMPONENT |
| ReviewForm | `features/interactions/components/` | ✅ Complete | 04_INTERACTIONS_COMPONENT |
| CoinPackageCard | `features/payments/components/` | ✅ Complete | 03_PAYMENTS_COMPONENT |
| DonationModal | `features/payments/components/` | ✅ Complete | 03_PAYMENTS_COMPONENT |
| TransactionItem | `features/payments/components/` | ✅ Complete | 03_PAYMENTS_COMPONENT |
| NotificationItem | `features/notifications/components/` | ✅ Complete | 07_NOTIFICATIONS_COMPONENT |
| NotificationBadge | `features/notifications/components/` | ✅ Complete | 07_NOTIFICATIONS_COMPONENT |
| ReadingStreakCard | `features/readings/components/` | ✅ Complete | 05_READINGS_COMPONENT |
| ContinueReadingCard | `features/readings/components/` | ✅ Complete | 05_READINGS_COMPONENT |

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-04 | Frontend Architecture Team | Initial release |
| 2.0 | 2026-01-05 | Frontend Architecture Team | Added Reading List, Review, Payment, Notification, and Reading Progress components aligned with Backend Design |

---

## 11. Related Documents

- [03_DESIGN_SYSTEM.md](./03_DESIGN_SYSTEM.md) - Design tokens and styling
- [08_PAGES_SPECIFICATION.md](./08_PAGES_SPECIFICATION.md) - Page compositions
- [12_ACCESSIBILITY_I18N.md](./12_ACCESSIBILITY_I18N.md) - Accessibility guidelines
- [Backend 03_PAYMENTS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/03_PAYMENTS_COMPONENT.md) - Payment System Design
- [Backend 04_INTERACTIONS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/04_INTERACTIONS_COMPONENT.md) - Interactions Design
- [Backend 05_READINGS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/05_READINGS_COMPONENT.md) - Reading Progress Design
- [Backend 07_NOTIFICATIONS_COMPONENT.md](../../Backend/N9/Documentation/BackendDesign/Components/07_NOTIFICATIONS_COMPONENT.md) - Notifications Design
