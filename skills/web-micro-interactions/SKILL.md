---
name: web-micro-interactions
description: Micro-interaction patterns that make websites feel alive — hover states, loading skeletons, optimistic UI, toast notifications, button feedback, empty states, and transitions that delight without distracting.
origin: community
tags: [micro-interactions, loading, skeleton, toast, hover, optimistic-ui, ux]
---

# Web Micro-Interactions

## 1. When to Use (and the line between delight vs distraction)

Micro-interactions earn their place when they:

- **Confirm an action** — the user did something; the UI acknowledges it
- **Communicate state** — loading, success, error, empty, disabled
- **Orient the user** — what changed, where they are, what happens next
- **Reduce anxiety** — upload is still running, server did receive the request

They cross into distraction when they:

- Animate things the user didn't touch
- Play on every render cycle
- Last longer than 400ms for routine UI responses
- Compete with primary content for attention
- Cannot be disabled (violates `prefers-reduced-motion`)

**The test:** Cover the element with your hand. Does the page still communicate what it needs to? If yes, the animation is decoration. If no, it is information — keep it.

**Always wrap motion in a reduced-motion guard:**

```tsx
// hooks/useReducedMotion.ts
import { useEffect, useState } from 'react'

export function useReducedMotion() {
  const [reduced, setReduced] = useState(false)

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)')
    setReduced(mq.matches)
    const handler = (e: MediaQueryListEvent) => setReduced(e.matches)
    mq.addEventListener('change', handler)
    return () => mq.removeEventListener('change', handler)
  }, [])

  return reduced
}
```

---

## 2. Button States — Full State Machine

A button has five meaningful states. Design all five before shipping any.

```
idle → hover → loading → success → idle
                       → error   → idle
```

### State Machine Hook

```tsx
// hooks/useButtonState.ts
import { useState, useCallback } from 'react'

type ButtonState = 'idle' | 'loading' | 'success' | 'error'

interface UseButtonStateOptions {
  successDuration?: number
  errorDuration?: number
}

export function useButtonState({
  successDuration = 2000,
  errorDuration = 2000,
}: UseButtonStateOptions = {}) {
  const [state, setState] = useState<ButtonState>('idle')

  const execute = useCallback(
    async (fn: () => Promise<void>) => {
      if (state === 'loading') return
      setState('loading')
      try {
        await fn()
        setState('success')
        setTimeout(() => setState('idle'), successDuration)
      } catch {
        setState('error')
        setTimeout(() => setState('idle'), errorDuration)
      }
    },
    [state, successDuration, errorDuration]
  )

  return { state, execute }
}
```

### Animated Button Component

```tsx
// components/ui/ActionButton.tsx
'use client'

import { motion, AnimatePresence } from 'framer-motion'
import { CheckIcon, XMarkIcon, ArrowPathIcon } from '@heroicons/react/24/outline'
import { cn } from '@/lib/utils'
import { useButtonState } from '@/hooks/useButtonState'

interface ActionButtonProps {
  children: React.ReactNode
  onClick: () => Promise<void>
  className?: string
  variant?: 'primary' | 'secondary' | 'destructive'
}

const variants = {
  primary: {
    idle: 'bg-blue-600 hover:bg-blue-700 text-white',
    loading: 'bg-blue-500 text-white cursor-not-allowed',
    success: 'bg-green-600 text-white',
    error: 'bg-red-600 text-white',
  },
  secondary: {
    idle: 'bg-zinc-100 hover:bg-zinc-200 text-zinc-900',
    loading: 'bg-zinc-100 text-zinc-500 cursor-not-allowed',
    success: 'bg-green-100 text-green-700',
    error: 'bg-red-100 text-red-700',
  },
  destructive: {
    idle: 'bg-red-600 hover:bg-red-700 text-white',
    loading: 'bg-red-500 text-white cursor-not-allowed',
    success: 'bg-green-600 text-white',
    error: 'bg-red-800 text-white',
  },
}

export function ActionButton({
  children,
  onClick,
  className,
  variant = 'primary',
}: ActionButtonProps) {
  const { state, execute } = useButtonState()

  const iconMap = {
    idle: null,
    loading: (
      <ArrowPathIcon className="h-4 w-4 animate-spin" />
    ),
    success: <CheckIcon className="h-4 w-4" />,
    error: <XMarkIcon className="h-4 w-4" />,
  }

  const labelMap = {
    idle: children,
    loading: 'Processing…',
    success: 'Done',
    error: 'Failed — try again',
  }

  return (
    <motion.button
      whileHover={state === 'idle' ? { scale: 1.02 } : {}}
      whileTap={state === 'idle' ? { scale: 0.97 } : {}}
      transition={{ type: 'spring', stiffness: 400, damping: 25 }}
      onClick={() => execute(onClick)}
      disabled={state === 'loading'}
      className={cn(
        'relative flex items-center gap-2 rounded-lg px-4 py-2 text-sm font-medium',
        'transition-colors duration-150 focus-visible:outline-none focus-visible:ring-2',
        'focus-visible:ring-blue-500 focus-visible:ring-offset-2',
        variants[variant][state],
        className
      )}
    >
      <AnimatePresence mode="wait">
        <motion.span
          key={state}
          initial={{ opacity: 0, y: 4 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: -4 }}
          transition={{ duration: 0.15 }}
          className="flex items-center gap-2"
        >
          {iconMap[state]}
          {labelMap[state]}
        </motion.span>
      </AnimatePresence>
    </motion.button>
  )
}
```

### Usage

```tsx
<ActionButton
  onClick={async () => {
    await saveDocument(doc)
  }}
>
  Save changes
</ActionButton>
```

---

## 3. Skeleton Screens

Skeletons communicate structure before data. Spinners communicate waiting. Use skeletons when you know the shape of the content. Use spinners for indeterminate operations (export, upload).

### Shimmer Animation (global CSS)

```css
/* styles/global.css */
@keyframes shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}

.skeleton-shimmer {
  background: linear-gradient(
    90deg,
    oklch(93% 0 0) 25%,
    oklch(96% 0 0) 50%,
    oklch(93% 0 0) 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

.dark .skeleton-shimmer {
  background: linear-gradient(
    90deg,
    oklch(22% 0 0) 25%,
    oklch(26% 0 0) 50%,
    oklch(22% 0 0) 75%
  );
  background-size: 200% 100%;
}
```

### Base Skeleton Primitive

```tsx
// components/ui/Skeleton.tsx
import { cn } from '@/lib/utils'

interface SkeletonProps {
  className?: string
  rounded?: 'sm' | 'md' | 'lg' | 'full'
}

export function Skeleton({ className, rounded = 'md' }: SkeletonProps) {
  const roundedMap = {
    sm: 'rounded',
    md: 'rounded-md',
    lg: 'rounded-xl',
    full: 'rounded-full',
  }

  return (
    <div
      aria-hidden="true"
      className={cn('skeleton-shimmer', roundedMap[rounded], className)}
    />
  )
}
```

### Content-Aware Skeleton: User Card

```tsx
// components/skeletons/UserCardSkeleton.tsx
import { Skeleton } from '@/components/ui/Skeleton'

export function UserCardSkeleton() {
  return (
    <div
      role="status"
      aria-label="Loading user profile"
      className="flex items-start gap-4 rounded-xl border border-zinc-200 p-4"
    >
      {/* Avatar */}
      <Skeleton className="h-12 w-12 shrink-0" rounded="full" />

      <div className="flex-1 space-y-2">
        {/* Name */}
        <Skeleton className="h-4 w-2/5" />
        {/* Role */}
        <Skeleton className="h-3 w-1/4" />
        {/* Bio — two lines */}
        <div className="space-y-1 pt-1">
          <Skeleton className="h-3 w-full" />
          <Skeleton className="h-3 w-4/5" />
        </div>
      </div>
    </div>
  )
}
```

### Content-Aware Skeleton: Data Table

```tsx
// components/skeletons/TableSkeleton.tsx
import { Skeleton } from '@/components/ui/Skeleton'

interface TableSkeletonProps {
  rows?: number
  columns?: number
}

export function TableSkeleton({ rows = 5, columns = 4 }: TableSkeletonProps) {
  return (
    <div role="status" aria-label="Loading table data" className="w-full">
      {/* Header */}
      <div className="flex gap-4 border-b border-zinc-200 px-4 py-3">
        {Array.from({ length: columns }).map((_, i) => (
          <Skeleton key={i} className="h-3 flex-1" />
        ))}
      </div>

      {/* Rows — vary widths so it reads as real data */}
      {Array.from({ length: rows }).map((_, rowIdx) => (
        <div
          key={rowIdx}
          className="flex gap-4 border-b border-zinc-100 px-4 py-3"
        >
          {Array.from({ length: columns }).map((_, colIdx) => (
            <Skeleton
              key={colIdx}
              className="h-3 flex-1"
              style={{
                maxWidth: `${70 + ((rowIdx * colIdx) % 30)}%`,
              }}
            />
          ))}
        </div>
      ))}
    </div>
  )
}
```

### Skeleton vs Spinner Decision Rule

| Situation | Use |
|-----------|-----|
| Page / route loading with known layout | Skeleton |
| Card or list with known item shape | Skeleton |
| Inline data fetch (replacing one element) | Skeleton |
| File upload, export, email send | Spinner |
| Auth check before redirect | Spinner (full-page) |
| Unknown result count | Spinner |

---

## 4. Toast / Notification System

Use **Sonner** — it handles stacking, positioning, accessibility, and promise toasts out of the box.

### Setup

```bash
npm install sonner
```

```tsx
// app/layout.tsx
import { Toaster } from 'sonner'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Toaster
          position="bottom-right"
          toastOptions={{
            duration: 4000,
            classNames: {
              toast: 'font-sans text-sm',
              title: 'font-medium',
              description: 'text-zinc-500',
            },
          }}
          richColors
          closeButton
        />
      </body>
    </html>
  )
}
```

### Toast Patterns

```tsx
// lib/toast.ts — typed wrappers around sonner
import { toast } from 'sonner'

export const notify = {
  success: (message: string, description?: string) =>
    toast.success(message, { description }),

  error: (message: string, description?: string) =>
    toast.error(message, { description }),

  info: (message: string, description?: string) =>
    toast.info(message, { description }),

  warning: (message: string, description?: string) =>
    toast.warning(message, { description }),

  promise: <T>(
    promise: Promise<T>,
    messages: {
      loading: string
      success: string | ((data: T) => string)
      error: string | ((err: unknown) => string)
    }
  ) => toast.promise(promise, messages),

  action: (
    message: string,
    actionLabel: string,
    onAction: () => void
  ) =>
    toast(message, {
      action: {
        label: actionLabel,
        onClick: onAction,
      },
    }),
}
```

### Usage Examples

```tsx
// Success
notify.success('Changes saved', 'Your document has been updated.')

// Error
notify.error('Upload failed', 'File exceeds the 10 MB limit.')

// Promise toast — tracks async state automatically
notify.promise(uploadFile(file), {
  loading: 'Uploading…',
  success: (data) => `Uploaded ${data.filename}`,
  error: (err) => err instanceof Error ? err.message : 'Upload failed',
})

// Undo action toast
notify.action('Contact archived', 'Undo', () => restoreContact(id))
```

### Positioning Guide

| Position | Best for |
|----------|----------|
| `bottom-right` | Apps — out of primary content path |
| `top-center` | Marketing sites — high visibility |
| `bottom-center` | Mobile — thumb reach |
| `top-right` | Dashboards — conventional |

---

## 5. Optimistic UI

Update the UI immediately. Rollback if the server rejects it. The user should never wait for a simple toggle, like, or delete.

### Pattern: Optimistic List Delete

```tsx
// hooks/useOptimisticList.ts
import { useState, useCallback } from 'react'

interface UseOptimisticListOptions<T> {
  items: T[]
  onDelete: (id: string) => Promise<void>
  getId: (item: T) => string
}

export function useOptimisticList<T>({
  items: initialItems,
  onDelete,
  getId,
}: UseOptimisticListOptions<T>) {
  const [items, setItems] = useState(initialItems)
  const [errors, setErrors] = useState<Record<string, string>>({})

  const deleteItem = useCallback(
    async (id: string) => {
      const snapshot = [...items]

      setItems((prev) => prev.filter((item) => getId(item) !== id))
      setErrors((prev) => {
        const next = { ...prev }
        delete next[id]
        return next
      })

      try {
        await onDelete(id)
      } catch (err) {
        setItems(snapshot)
        setErrors((prev) => ({
          ...prev,
          [id]: err instanceof Error ? err.message : 'Delete failed',
        }))
      }
    },
    [items, onDelete, getId]
  )

  return { items, deleteItem, errors }
}
```

### Pattern: Optimistic Toggle (Like Button)

```tsx
// components/LikeButton.tsx
'use client'

import { useState } from 'react'
import { motion, AnimatePresence } from 'framer-motion'
import { HeartIcon } from '@heroicons/react/24/outline'
import { HeartIcon as HeartSolid } from '@heroicons/react/24/solid'
import { notify } from '@/lib/toast'
import { cn } from '@/lib/utils'

interface LikeButtonProps {
  postId: string
  initialLiked: boolean
  initialCount: number
  onToggle: (postId: string, liked: boolean) => Promise<void>
}

export function LikeButton({
  postId,
  initialLiked,
  initialCount,
  onToggle,
}: LikeButtonProps) {
  const [liked, setLiked] = useState(initialLiked)
  const [count, setCount] = useState(initialCount)

  const handleToggle = async () => {
    const prevLiked = liked
    const prevCount = count

    setLiked(!prevLiked)
    setCount(prevLiked ? prevCount - 1 : prevCount + 1)

    try {
      await onToggle(postId, !prevLiked)
    } catch {
      setLiked(prevLiked)
      setCount(prevCount)
      notify.error('Could not update like', 'Please try again.')
    }
  }

  return (
    <button
      onClick={handleToggle}
      aria-label={liked ? 'Unlike post' : 'Like post'}
      aria-pressed={liked}
      className={cn(
        'flex items-center gap-1.5 rounded-full px-3 py-1.5 text-sm font-medium',
        'transition-colors duration-150 focus-visible:outline-none',
        'focus-visible:ring-2 focus-visible:ring-rose-500 focus-visible:ring-offset-2',
        liked
          ? 'bg-rose-50 text-rose-600 hover:bg-rose-100'
          : 'text-zinc-500 hover:bg-zinc-100 hover:text-zinc-900'
      )}
    >
      <motion.span
        animate={liked ? { scale: [1, 1.3, 1] } : {}}
        transition={{ duration: 0.25, times: [0, 0.5, 1] }}
      >
        {liked ? (
          <HeartSolid className="h-4 w-4 text-rose-500" />
        ) : (
          <HeartIcon className="h-4 w-4" />
        )}
      </motion.span>

      <AnimatePresence mode="wait">
        <motion.span
          key={count}
          initial={{ opacity: 0, y: liked ? -8 : 8 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: liked ? 8 : -8 }}
          transition={{ duration: 0.15 }}
        >
          {count}
        </motion.span>
      </AnimatePresence>
    </button>
  )
}
```

### React 19 / Next.js — `useOptimistic`

```tsx
// Using the built-in hook (React 19+)
'use client'

import { useOptimistic, useTransition } from 'react'
import { toggleLike } from '@/app/actions'

export function LikeButtonNative({
  postId,
  liked,
  count,
}: {
  postId: string
  liked: boolean
  count: number
}) {
  const [isPending, startTransition] = useTransition()
  const [optimisticState, setOptimistic] = useOptimistic(
    { liked, count },
    (current, newLiked: boolean) => ({
      liked: newLiked,
      count: newLiked ? current.count + 1 : current.count - 1,
    })
  )

  return (
    <button
      disabled={isPending}
      onClick={() => {
        startTransition(async () => {
          setOptimistic(!optimisticState.liked)
          await toggleLike(postId)
        })
      }}
    >
      {optimisticState.liked ? '♥' : '♡'} {optimisticState.count}
    </button>
  )
}
```

---

## 6. Hover Card Previews

Show a content preview on hover with a deliberate delay (prevents flicker on cursor pass-through). Must be keyboard-accessible.

### Install

```bash
npm install @radix-ui/react-hover-card
```

### Hover Card Component

```tsx
// components/ui/HoverCard.tsx
'use client'

import * as HoverCardPrimitive from '@radix-ui/react-hover-card'
import { motion, AnimatePresence } from 'framer-motion'
import { useState } from 'react'
import { cn } from '@/lib/utils'

interface HoverCardProps {
  trigger: React.ReactNode
  children: React.ReactNode
  side?: 'top' | 'bottom' | 'left' | 'right'
  openDelay?: number
  closeDelay?: number
  className?: string
}

export function HoverCard({
  trigger,
  children,
  side = 'bottom',
  openDelay = 300,
  closeDelay = 150,
  className,
}: HoverCardProps) {
  const [open, setOpen] = useState(false)

  return (
    <HoverCardPrimitive.Root
      open={open}
      onOpenChange={setOpen}
      openDelay={openDelay}
      closeDelay={closeDelay}
    >
      <HoverCardPrimitive.Trigger asChild>
        {trigger}
      </HoverCardPrimitive.Trigger>

      <HoverCardPrimitive.Portal>
        <HoverCardPrimitive.Content
          side={side}
          sideOffset={8}
          align="start"
          className={cn(
            'z-50 w-72 rounded-xl border border-zinc-200 bg-white p-4 shadow-lg',
            'outline-none dark:border-zinc-800 dark:bg-zinc-900',
            className
          )}
          asChild
        >
          <AnimatePresence>
            {open && (
              <motion.div
                initial={{ opacity: 0, scale: 0.95, y: side === 'bottom' ? -4 : 4 }}
                animate={{ opacity: 1, scale: 1, y: 0 }}
                exit={{ opacity: 0, scale: 0.95 }}
                transition={{ duration: 0.15, ease: [0.16, 1, 0.3, 1] }}
              >
                {children}
              </motion.div>
            )}
          </AnimatePresence>
        </HoverCardPrimitive.Content>
      </HoverCardPrimitive.Portal>
    </HoverCardPrimitive.Root>
  )
}
```

### User Profile Hover Card

```tsx
// components/UserHoverCard.tsx
import { HoverCard } from '@/components/ui/HoverCard'
import { Avatar } from '@/components/ui/Avatar'

interface User {
  name: string
  handle: string
  avatarUrl: string
  bio: string
  followers: number
  following: number
}

export function UserHoverCard({
  user,
  children,
}: {
  user: User
  children: React.ReactNode
}) {
  return (
    <HoverCard
      trigger={
        <span className="cursor-pointer underline decoration-dotted underline-offset-2">
          {children}
        </span>
      }
    >
      <div className="space-y-3">
        <div className="flex items-start justify-between">
          <Avatar src={user.avatarUrl} alt={user.name} size="lg" />
          <button className="rounded-full border border-zinc-200 px-3 py-1 text-xs font-medium hover:bg-zinc-50">
            Follow
          </button>
        </div>

        <div>
          <p className="font-semibold text-zinc-900">{user.name}</p>
          <p className="text-sm text-zinc-500">@{user.handle}</p>
        </div>

        <p className="text-sm text-zinc-700 leading-relaxed">{user.bio}</p>

        <div className="flex gap-4 text-sm">
          <span>
            <strong className="text-zinc-900">{user.following.toLocaleString()}</strong>{' '}
            <span className="text-zinc-500">Following</span>
          </span>
          <span>
            <strong className="text-zinc-900">{user.followers.toLocaleString()}</strong>{' '}
            <span className="text-zinc-500">Followers</span>
          </span>
        </div>
      </div>
    </HoverCard>
  )
}
```

---

## 7. Empty States

An empty state is an opportunity. It should answer: why is this empty, and what can I do about it?

Three distinct scenarios require distinct copy and CTAs:

| Scenario | Headline pattern | CTA |
|----------|-----------------|-----|
| First-time (no data yet) | "Start by adding your first X" | Primary action button |
| No results (filtered/searched) | "No X match your search" | Clear filters link |
| Error (fetch failed) | "Couldn't load X" | Retry button |

### Empty State Component

```tsx
// components/ui/EmptyState.tsx
import { motion } from 'framer-motion'
import { cn } from '@/lib/utils'

interface EmptyStateProps {
  icon?: React.ReactNode
  illustration?: React.ReactNode
  headline: string
  description?: string
  action?: {
    label: string
    onClick: () => void
    variant?: 'primary' | 'ghost'
  }
  secondaryAction?: {
    label: string
    onClick: () => void
  }
  className?: string
}

export function EmptyState({
  icon,
  illustration,
  headline,
  description,
  action,
  secondaryAction,
  className,
}: EmptyStateProps) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 8 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.3, ease: [0.16, 1, 0.3, 1] }}
      className={cn(
        'flex flex-col items-center justify-center px-6 py-16 text-center',
        className
      )}
    >
      {illustration ?? (
        icon && (
          <div className="mb-4 flex h-16 w-16 items-center justify-center rounded-2xl bg-zinc-100 text-zinc-400 dark:bg-zinc-800">
            {icon}
          </div>
        )
      )}

      <h3 className="mt-2 text-base font-semibold text-zinc-900 dark:text-zinc-100">
        {headline}
      </h3>

      {description && (
        <p className="mt-1 max-w-sm text-sm text-zinc-500 leading-relaxed">
          {description}
        </p>
      )}

      {(action || secondaryAction) && (
        <div className="mt-6 flex items-center gap-3">
          {action && (
            <button
              onClick={action.onClick}
              className={cn(
                'rounded-lg px-4 py-2 text-sm font-medium transition-colors',
                action.variant === 'ghost'
                  ? 'text-zinc-600 hover:bg-zinc-100 hover:text-zinc-900'
                  : 'bg-blue-600 text-white hover:bg-blue-700'
              )}
            >
              {action.label}
            </button>
          )}

          {secondaryAction && (
            <button
              onClick={secondaryAction.onClick}
              className="text-sm text-zinc-500 underline underline-offset-2 hover:text-zinc-700"
            >
              {secondaryAction.label}
            </button>
          )}
        </div>
      )}
    </motion.div>
  )
}
```

### Usage: All Three Scenarios

```tsx
// First-time empty
<EmptyState
  icon={<DocumentPlusIcon className="h-7 w-7" />}
  headline="No projects yet"
  description="Create your first project to start organizing your work."
  action={{ label: 'Create project', onClick: openCreateModal }}
/>

// No results
<EmptyState
  icon={<MagnifyingGlassIcon className="h-7 w-7" />}
  headline={`No results for "${query}"`}
  description="Try different keywords or clear your filters."
  action={{ label: 'Clear filters', onClick: clearFilters, variant: 'ghost' }}
/>

// Error state
<EmptyState
  icon={<ExclamationTriangleIcon className="h-7 w-7 text-red-400" />}
  headline="Couldn't load projects"
  description="Something went wrong on our end. Your data is safe."
  action={{ label: 'Try again', onClick: refetch }}
  secondaryAction={{ label: 'Contact support', onClick: openSupport }}
/>
```

---

## 8. Progress Indicators

### Linear Progress Bar

```tsx
// components/ui/ProgressBar.tsx
'use client'

import { motion } from 'framer-motion'
import { cn } from '@/lib/utils'

interface ProgressBarProps {
  value: number // 0-100
  className?: string
  color?: 'blue' | 'green' | 'amber' | 'red'
  showLabel?: boolean
  size?: 'sm' | 'md' | 'lg'
  animated?: boolean
}

const colorMap = {
  blue: 'bg-blue-500',
  green: 'bg-green-500',
  amber: 'bg-amber-500',
  red: 'bg-red-500',
}

const sizeMap = {
  sm: 'h-1',
  md: 'h-2',
  lg: 'h-3',
}

export function ProgressBar({
  value,
  className,
  color = 'blue',
  showLabel = false,
  size = 'md',
  animated = true,
}: ProgressBarProps) {
  const clamped = Math.max(0, Math.min(100, value))

  return (
    <div className={cn('w-full', className)}>
      {showLabel && (
        <div className="mb-1 flex justify-between text-xs text-zinc-500">
          <span>Progress</span>
          <span>{clamped}%</span>
        </div>
      )}
      <div
        role="progressbar"
        aria-valuenow={clamped}
        aria-valuemin={0}
        aria-valuemax={100}
        className={cn(
          'w-full overflow-hidden rounded-full bg-zinc-100 dark:bg-zinc-800',
          sizeMap[size]
        )}
      >
        <motion.div
          className={cn('h-full rounded-full', colorMap[color])}
          initial={{ width: 0 }}
          animate={{ width: `${clamped}%` }}
          transition={
            animated
              ? { duration: 0.4, ease: [0.16, 1, 0.3, 1] }
              : { duration: 0 }
          }
        />
      </div>
    </div>
  )
}
```

### Upload Progress Component

```tsx
// components/UploadProgress.tsx
'use client'

import { ProgressBar } from '@/components/ui/ProgressBar'
import { motion, AnimatePresence } from 'framer-motion'
import { CheckCircleIcon, XCircleIcon } from '@heroicons/react/24/solid'
import { formatBytes } from '@/lib/format'

interface UploadFile {
  id: string
  name: string
  size: number
  progress: number
  status: 'uploading' | 'done' | 'error'
  error?: string
}

export function UploadProgress({ files }: { files: UploadFile[] }) {
  return (
    <div className="space-y-3">
      <AnimatePresence>
        {files.map((file) => (
          <motion.div
            key={file.id}
            initial={{ opacity: 0, height: 0 }}
            animate={{ opacity: 1, height: 'auto' }}
            exit={{ opacity: 0, height: 0 }}
            transition={{ duration: 0.2 }}
            className="overflow-hidden rounded-lg border border-zinc-200 p-3"
          >
            <div className="mb-2 flex items-center justify-between">
              <div className="min-w-0 flex-1">
                <p className="truncate text-sm font-medium text-zinc-900">
                  {file.name}
                </p>
                <p className="text-xs text-zinc-500">{formatBytes(file.size)}</p>
              </div>

              <div className="ml-3 shrink-0">
                {file.status === 'done' && (
                  <CheckCircleIcon className="h-5 w-5 text-green-500" />
                )}
                {file.status === 'error' && (
                  <XCircleIcon className="h-5 w-5 text-red-500" />
                )}
                {file.status === 'uploading' && (
                  <span className="text-xs text-zinc-500">
                    {file.progress}%
                  </span>
                )}
              </div>
            </div>

            {file.status !== 'error' && (
              <ProgressBar
                value={file.progress}
                color={file.status === 'done' ? 'green' : 'blue'}
                size="sm"
              />
            )}

            {file.error && (
              <p className="mt-1 text-xs text-red-500">{file.error}</p>
            )}
          </motion.div>
        ))}
      </AnimatePresence>
    </div>
  )
}
```

### Step-Based Progress

```tsx
// components/ui/StepProgress.tsx
import { CheckIcon } from '@heroicons/react/24/solid'
import { cn } from '@/lib/utils'

interface Step {
  label: string
  description?: string
}

interface StepProgressProps {
  steps: Step[]
  currentStep: number // 0-indexed
}

export function StepProgress({ steps, currentStep }: StepProgressProps) {
  return (
    <nav aria-label="Progress">
      <ol className="flex items-center gap-0">
        {steps.map((step, idx) => {
          const status =
            idx < currentStep
              ? 'complete'
              : idx === currentStep
              ? 'current'
              : 'upcoming'

          return (
            <li key={step.label} className="flex flex-1 items-center">
              <div className="flex flex-col items-center">
                <div
                  aria-current={status === 'current' ? 'step' : undefined}
                  className={cn(
                    'flex h-9 w-9 items-center justify-center rounded-full',
                    'border-2 transition-colors duration-200',
                    status === 'complete' &&
                      'border-blue-600 bg-blue-600 text-white',
                    status === 'current' &&
                      'border-blue-600 bg-white text-blue-600',
                    status === 'upcoming' &&
                      'border-zinc-300 bg-white text-zinc-400'
                  )}
                >
                  {status === 'complete' ? (
                    <CheckIcon className="h-4 w-4" />
                  ) : (
                    <span className="text-sm font-medium">{idx + 1}</span>
                  )}
                </div>
                <span
                  className={cn(
                    'mt-1 text-xs font-medium',
                    status === 'current' ? 'text-blue-600' : 'text-zinc-500'
                  )}
                >
                  {step.label}
                </span>
              </div>

              {idx < steps.length - 1 && (
                <div
                  className={cn(
                    'mx-2 h-0.5 flex-1 transition-colors duration-300',
                    idx < currentStep ? 'bg-blue-600' : 'bg-zinc-200'
                  )}
                />
              )}
            </li>
          )
        })}
      </ol>
    </nav>
  )
}
```

---

## 9. Smooth Page Transitions — Next.js App Router

### Layout-Level Fade

```tsx
// app/template.tsx  (template.tsx re-mounts on every route change — layout.tsx does not)
'use client'

import { motion } from 'framer-motion'

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      exit={{ opacity: 0 }}
      transition={{ duration: 0.2, ease: 'easeOut' }}
    >
      {children}
    </motion.div>
  )
}
```

### Slide Transition (per page)

```tsx
// app/dashboard/page.tsx
'use client'

import { motion } from 'framer-motion'

const slideIn = {
  initial: { x: 24, opacity: 0 },
  animate: { x: 0, opacity: 1 },
  transition: { duration: 0.25, ease: [0.16, 1, 0.3, 1] },
}

export default function DashboardPage() {
  return (
    <motion.main {...slideIn}>
      {/* page content */}
    </motion.main>
  )
}
```

### Staggered List Entry

```tsx
// components/StaggeredList.tsx
'use client'

import { motion } from 'framer-motion'

const container = {
  hidden: {},
  show: {
    transition: {
      staggerChildren: 0.06,
    },
  },
}

const item = {
  hidden: { opacity: 0, y: 12 },
  show: { opacity: 1, y: 0, transition: { duration: 0.25, ease: [0.16, 1, 0.3, 1] } },
}

export function StaggeredList({ children }: { children: React.ReactNode[] }) {
  return (
    <motion.ul variants={container} initial="hidden" animate="show">
      {children.map((child, i) => (
        <motion.li key={i} variants={item}>
          {child}
        </motion.li>
      ))}
    </motion.ul>
  )
}
```

---

## 10. Number Animations

### Count-Up Hook

```tsx
// hooks/useCountUp.ts
import { useEffect, useRef, useState } from 'react'

interface UseCountUpOptions {
  from?: number
  to: number
  duration?: number
  decimals?: number
  easing?: (t: number) => number
}

function easeOutExpo(t: number): number {
  return t === 1 ? 1 : 1 - Math.pow(2, -10 * t)
}

export function useCountUp({
  from = 0,
  to,
  duration = 1200,
  decimals = 0,
  easing = easeOutExpo,
}: UseCountUpOptions) {
  const [value, setValue] = useState(from)
  const startTime = useRef<number | null>(null)
  const rafRef = useRef<number>()

  useEffect(() => {
    startTime.current = null

    const animate = (timestamp: number) => {
      if (!startTime.current) startTime.current = timestamp
      const elapsed = timestamp - startTime.current
      const progress = Math.min(elapsed / duration, 1)
      const easedProgress = easing(progress)
      const current = from + (to - from) * easedProgress

      setValue(parseFloat(current.toFixed(decimals)))

      if (progress < 1) {
        rafRef.current = requestAnimationFrame(animate)
      }
    }

    rafRef.current = requestAnimationFrame(animate)
    return () => {
      if (rafRef.current) cancelAnimationFrame(rafRef.current)
    }
  }, [to, from, duration, decimals, easing])

  return value
}
```

### Animated Number Component

```tsx
// components/ui/AnimatedNumber.tsx
'use client'

import { useCountUp } from '@/hooks/useCountUp'
import { useInView } from 'framer-motion'
import { useRef, useState, useEffect } from 'react'

interface AnimatedNumberProps {
  value: number
  prefix?: string
  suffix?: string
  decimals?: number
  duration?: number
  triggerOnView?: boolean
  className?: string
}

export function AnimatedNumber({
  value,
  prefix = '',
  suffix = '',
  decimals = 0,
  duration = 1200,
  triggerOnView = true,
  className,
}: AnimatedNumberProps) {
  const ref = useRef<HTMLSpanElement>(null)
  const isInView = useInView(ref, { once: true, margin: '-10% 0px' })
  const [started, setStarted] = useState(!triggerOnView)

  useEffect(() => {
    if (isInView && triggerOnView) setStarted(true)
  }, [isInView, triggerOnView])

  const display = useCountUp({
    from: 0,
    to: started ? value : 0,
    duration,
    decimals,
  })

  return (
    <span ref={ref} className={className} aria-label={`${prefix}${value}${suffix}`}>
      {prefix}
      {display.toLocaleString(undefined, {
        minimumFractionDigits: decimals,
        maximumFractionDigits: decimals,
      })}
      {suffix}
    </span>
  )
}
```

### Usage

```tsx
<div className="grid grid-cols-3 gap-8 text-center">
  <div>
    <AnimatedNumber value={12500} suffix="+" className="text-4xl font-bold" />
    <p className="text-sm text-zinc-500">Active users</p>
  </div>
  <div>
    <AnimatedNumber value={99.9} suffix="%" decimals={1} className="text-4xl font-bold" />
    <p className="text-sm text-zinc-500">Uptime</p>
  </div>
  <div>
    <AnimatedNumber value={4.8} decimals={1} className="text-4xl font-bold" />
    <p className="text-sm text-zinc-500">Avg rating</p>
  </div>
</div>
```

---

## 11. Drag and Drop Feedback

### Install

```bash
npm install @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
```

### Sortable List with Drag Feedback

```tsx
// components/SortableList.tsx
'use client'

import {
  DndContext,
  closestCenter,
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
  DragEndEvent,
  DragStartEvent,
} from '@dnd-kit/core'
import {
  SortableContext,
  sortableKeyboardCoordinates,
  useSortable,
  verticalListSortingStrategy,
  arrayMove,
} from '@dnd-kit/sortable'
import { CSS } from '@dnd-kit/utilities'
import { useState } from 'react'
import { GripVerticalIcon } from 'lucide-react'
import { cn } from '@/lib/utils'

interface Item {
  id: string
  label: string
}

function SortableItem({ item }: { item: Item }) {
  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
    isDragging,
    isOver,
  } = useSortable({ id: item.id })

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
  }

  return (
    <div
      ref={setNodeRef}
      style={style}
      className={cn(
        'flex items-center gap-3 rounded-lg border bg-white p-3',
        'select-none transition-shadow',
        isDragging
          ? 'border-blue-300 shadow-lg ring-2 ring-blue-200 opacity-90 z-10 relative'
          : 'border-zinc-200 shadow-sm hover:shadow-md',
        isOver && !isDragging && 'border-blue-200 bg-blue-50'
      )}
    >
      <button
        {...attributes}
        {...listeners}
        className={cn(
          'cursor-grab touch-none rounded p-1 text-zinc-400',
          'hover:bg-zinc-100 hover:text-zinc-600',
          'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-blue-500',
          'active:cursor-grabbing'
        )}
        aria-label={`Drag to reorder ${item.label}`}
      >
        <GripVerticalIcon className="h-4 w-4" />
      </button>

      <span className="text-sm font-medium text-zinc-900">{item.label}</span>
    </div>
  )
}

export function SortableList({ initialItems }: { initialItems: Item[] }) {
  const [items, setItems] = useState(initialItems)

  const sensors = useSensors(
    useSensor(PointerSensor, {
      activationConstraint: {
        distance: 8,
      },
    }),
    useSensor(KeyboardSensor, {
      coordinateGetter: sortableKeyboardCoordinates,
    })
  )

  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event

    if (over && active.id !== over.id) {
      setItems((current) => {
        const oldIndex = current.findIndex((i) => i.id === active.id)
        const newIndex = current.findIndex((i) => i.id === over.id)
        return arrayMove(current, oldIndex, newIndex)
      })
    }
  }

  return (
    <DndContext
      sensors={sensors}
      collisionDetection={closestCenter}
      onDragEnd={handleDragEnd}
    >
      <SortableContext items={items} strategy={verticalListSortingStrategy}>
        <div className="space-y-2">
          {items.map((item) => (
            <SortableItem key={item.id} item={item} />
          ))}
        </div>
      </SortableContext>
    </DndContext>
  )
}
```

---

## 12. Timing Reference Table

The most important skill in micro-interactions is choosing the right duration. Too fast = jarring. Too slow = sluggish.

| Interaction type | Duration | Easing | Notes |
|-----------------|----------|--------|-------|
| Button press / tap feedback | 80-120ms | ease-out | Must feel instant |
| Tooltip / popover appear | 120-150ms | ease-out | Fast in, barely noticeable |
| Tooltip / popover dismiss | 80-100ms | ease-in | Faster out than in |
| Toast / snackbar slide in | 250-300ms | ease-out-expo | Snappy entry |
| Toast / snackbar slide out | 200ms | ease-in | Quicker than entry |
| Modal / dialog open | 200-250ms | ease-out-expo | Surface rising |
| Modal / dialog close | 150-180ms | ease-in | Fade + scale down |
| Page / route transition | 200-300ms | ease-out | Cross-fade or slide |
| Skeleton to content | 200ms | ease-out | Crossfade, not hard swap |
| Accordion expand | 250-300ms | ease-out | Height + opacity |
| Drag ghost creation | 150ms | ease-out | Quick lift |
| Number count-up | 800-1500ms | ease-out-expo | Depends on magnitude |
| Progress bar fill | 400ms per step | ease-in-out | Smooth, not jumpy |
| Hover card open (after delay) | 150ms | ease-out | Delay=300ms, animation=150ms |
| Success state persist | 1800-2500ms | — | Time on screen before reset |
| Error state persist | 2000-3000ms | — | Longer — must be readable |

```ts
// lib/animation.ts — shared easing constants
export const ease = {
  outExpo: [0.16, 1, 0.3, 1] as const,
  outSine: [0.39, 0.575, 0.565, 1] as const,
  outBack: [0.34, 1.56, 0.64, 1] as const,
  inQuart: [0.895, 0.03, 0.685, 0.22] as const,
}

export const duration = {
  instant: 0.08,
  fast: 0.15,
  normal: 0.25,
  slow: 0.4,
  dramatic: 0.8,
} as const
```

---

## 13. Anti-Patterns

### Too Much Motion

Only animate on meaningful state change. Never loop animations on idle content.

### Inconsistent Feedback

Use one shared ActionButton component everywhere. Avoid mixing spinners, text changes, and disabled states ad hoc.

### Missing Loading State

Never render `null` while fetching. Use skeletons for known shapes, spinners for unknown.

### Animating Without `prefers-reduced-motion`

Always check `useReducedMotion()` before applying motion. Provide a zero-duration fallback.

### Blocking Interactions During Animation

Never disable form controls during decorative animations. Only disable during the actual async request.

### Toast Overload

Toast only for async outcomes the user cannot see directly (archive, upload, send). Never toast synchronous UI actions like checkbox toggles.
