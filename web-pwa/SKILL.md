---
name: web-pwa
description: Progressive Web App patterns for Next.js — service workers, offline mode, install prompts, push notifications, and app manifest. Make your web app installable and work without internet.
origin: community
tags: [pwa, service-worker, offline, push-notifications, manifest, installable]
---

# web-pwa — Progressive Web App Patterns for Next.js

## 1. When to Build a PWA (and When Native Is Better)

### Build a PWA when:
- Your core value works in a browser (forms, dashboards, content, e-commerce)
- You need installability but not deep OS integration
- Offline reading, caching, or background sync is sufficient
- You want one codebase across desktop, Android, and iOS
- App store distribution overhead is not worth it

### Choose native when:
- You need Bluetooth, NFC, USB, or ARKit/ARCore
- You need background audio/location that runs even when the screen is off
- Your app is heavily gesture-driven (swipe-heavy games, camera apps)
- iOS restrictions matter: WebKit blocks service workers in WKWebView; Apple's PWA support lags Chromium
- You need in-app purchase with App Store billing

### PWA support caveats (2024):
- iOS 16.4+ supports push notifications for PWAs installed to home screen only
- Safari blocks persistent storage without user gesture
- Background sync is Chromium-only (no Safari, limited Firefox)

---

## 2. Web App Manifest

### `/public/manifest.json`

```json
{
  "name": "My App — Full Name",
  "short_name": "MyApp",
  "description": "What your app does in one sentence.",
  "start_url": "/?source=pwa",
  "display": "standalone",
  "orientation": "portrait-primary",
  "background_color": "#ffffff",
  "theme_color": "#1a1a2e",
  "categories": ["productivity", "utilities"],
  "lang": "en-US",
  "icons": [
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-maskable-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ],
  "shortcuts": [
    {
      "name": "New Entry",
      "short_name": "New",
      "description": "Create a new entry",
      "url": "/new?source=shortcut",
      "icons": [{ "src": "/icons/shortcut-new.png", "sizes": "96x96" }]
    },
    {
      "name": "Dashboard",
      "short_name": "Dashboard",
      "url": "/dashboard?source=shortcut",
      "icons": [{ "src": "/icons/shortcut-dashboard.png", "sizes": "96x96" }]
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/desktop.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide",
      "label": "Dashboard view on desktop"
    },
    {
      "src": "/screenshots/mobile.png",
      "sizes": "390x844",
      "type": "image/png",
      "form_factor": "narrow",
      "label": "Dashboard view on mobile"
    }
  ],
  "related_applications": [],
  "prefer_related_applications": false
}
```

### Link manifest in `app/layout.tsx`

```tsx
// app/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  manifest: '/manifest.json',
  themeColor: '#1a1a2e',
  appleWebApp: {
    capable: true,
    statusBarStyle: 'default',
    title: 'MyApp',
  },
  formatDetection: {
    telephone: false,
  },
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <head>
        {/* iOS home screen icons — manifest icons don't apply on Safari */}
        <link rel="apple-touch-icon" href="/icons/apple-touch-icon.png" />
        <link rel="apple-touch-icon" sizes="152x152" href="/icons/icon-152x152.png" />
        <link rel="apple-touch-icon" sizes="180x180" href="/icons/icon-180x180.png" />
        <meta name="apple-mobile-web-app-capable" content="yes" />
        <meta name="apple-mobile-web-app-status-bar-style" content="default" />
        <meta name="apple-mobile-web-app-title" content="MyApp" />
        {/* Microsoft Tiles */}
        <meta name="msapplication-TileColor" content="#1a1a2e" />
        <meta name="msapplication-TileImage" content="/icons/icon-144x144.png" />
      </head>
      <body>{children}</body>
    </html>
  )
}
```

### Icon generation script (run once)

```bash
# Install sharp globally or as a dev dep
npm install -D sharp

# scripts/generate-icons.mjs
import sharp from 'sharp'
import { mkdirSync } from 'fs'

const sizes = [72, 96, 128, 144, 152, 180, 192, 384, 512]
mkdirSync('public/icons', { recursive: true })

for (const size of sizes) {
  await sharp('public/icon-source.png')
    .resize(size, size)
    .png()
    .toFile(`public/icons/icon-${size}x${size}.png`)
  console.log(`Generated ${size}x${size}`)
}

// Maskable — add safe zone padding (10% each side)
await sharp('public/icon-source.png')
  .resize(460, 460)
  .extend({ top: 26, bottom: 26, left: 26, right: 26, background: '#1a1a2e' })
  .resize(512, 512)
  .png()
  .toFile('public/icons/icon-maskable-512x512.png')
```

---

## 3. next-pwa Setup

### Install

```bash
npm install next-pwa
npm install -D @types/next-pwa  # if using TypeScript
```

### `next.config.js`

```js
const withPWA = require('next-pwa')({
  dest: 'public',
  disable: process.env.NODE_ENV === 'development',  // no SW in dev
  register: true,       // auto-register SW in _app or layout
  skipWaiting: false,   // we handle this manually for update prompt UX
  reloadOnOnline: true, // reload page when network comes back

  // Custom service worker file — optional but recommended for full control
  // swSrc: 'src/sw.ts',

  runtimeCaching: [
    // Google Fonts — cache-first
    {
      urlPattern: /^https:\/\/fonts\.(?:googleapis|gstatic)\.com\/.*/i,
      handler: 'CacheFirst',
      options: {
        cacheName: 'google-fonts',
        expiration: { maxEntries: 4, maxAgeSeconds: 365 * 24 * 60 * 60 },
      },
    },
    // Static assets — cache-first
    {
      urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp|ico|woff2?)$/i,
      handler: 'CacheFirst',
      options: {
        cacheName: 'static-assets',
        expiration: { maxEntries: 64, maxAgeSeconds: 30 * 24 * 60 * 60 },
      },
    },
    // Next.js static chunks — stale-while-revalidate
    {
      urlPattern: /^\/_next\/static\/.*/i,
      handler: 'StaleWhileRevalidate',
      options: { cacheName: 'next-static' },
    },
    // API routes — network-first with offline fallback
    {
      urlPattern: /^\/api\/.*/i,
      handler: 'NetworkFirst',
      options: {
        cacheName: 'api-cache',
        networkTimeoutSeconds: 10,
        expiration: { maxEntries: 32, maxAgeSeconds: 24 * 60 * 60 },
      },
    },
    // Pages — stale-while-revalidate
    {
      urlPattern: ({ request }) => request.destination === 'document',
      handler: 'StaleWhileRevalidate',
      options: {
        cacheName: 'pages',
        expiration: { maxEntries: 32 },
      },
    },
  ],

  // Fallback to offline page when document fetch fails
  fallbacks: {
    document: '/offline',
  },

  buildExcludes: [/middleware-manifest\.json$/],
})

/** @type {import('next').NextConfig} */
const nextConfig = {
  // your other next config
}

module.exports = withPWA(nextConfig)
```

### `.gitignore` additions

```
# next-pwa generated files
public/sw.js
public/sw.js.map
public/workbox-*.js
public/workbox-*.js.map
public/fallback-*.js
public/fallback-*.js.map
```

---

## 4. Service Worker Patterns

### Cache-First

Best for: fonts, images, icons, vendor JS — things that change rarely.

```js
// Returns cached response immediately; fetches in background only if cache misses
{
  handler: 'CacheFirst',
  options: {
    cacheName: 'static-v1',
    expiration: {
      maxEntries: 60,
      maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
    },
    cacheableResponse: {
      statuses: [0, 200], // 0 = opaque responses (cross-origin)
    },
  },
}
```

### Network-First

Best for: API calls, authenticated data, anything that must be fresh.

```js
// Tries network; falls back to cache on failure
{
  handler: 'NetworkFirst',
  options: {
    cacheName: 'api-v1',
    networkTimeoutSeconds: 10,   // fall back to cache after 10s
    expiration: {
      maxEntries: 32,
      maxAgeSeconds: 60 * 60, // 1 hour
    },
  },
}
```

### Stale-While-Revalidate

Best for: pages, non-critical API data — serve instantly from cache, update silently.

```js
{
  handler: 'StaleWhileRevalidate',
  options: {
    cacheName: 'pages-v1',
    expiration: {
      maxEntries: 32,
      maxAgeSeconds: 24 * 60 * 60, // 1 day
    },
  },
}
```

### Custom service worker (`public/sw-custom.js`)

If you need logic beyond what next-pwa's config allows, use `swSrc`:

```js
// public/sw-custom.js
// This file is merged with the Workbox-generated SW

import { precacheAndRoute } from 'workbox-precaching'
import { registerRoute } from 'workbox-routing'
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies'
import { ExpirationPlugin } from 'workbox-expiration'
import { BackgroundSyncPlugin } from 'workbox-background-sync'

// Precache all assets from Next.js build
precacheAndRoute(self.__WB_MANIFEST || [])

// Custom route: user-specific data — never cache
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/user/'),
  new NetworkFirst({ networkTimeoutSeconds: 5 })
)

// Background sync for form submissions
const bgSyncPlugin = new BackgroundSyncPlugin('formQueue', {
  maxRetentionTime: 24 * 60, // retry for 24 hours
})

registerRoute(
  ({ url }) => url.pathname.startsWith('/api/submit'),
  new NetworkFirst({ plugins: [bgSyncPlugin] }),
  'POST'
)

// Listen for skip-waiting message from update prompt
self.addEventListener('message', (event) => {
  if (event.data?.type === 'SKIP_WAITING') {
    self.skipWaiting()
  }
})
```

---

## 5. Offline Fallback Page

### `app/offline/page.tsx`

```tsx
'use client'

import { useEffect, useState } from 'react'

export default function OfflinePage() {
  const [isOnline, setIsOnline] = useState(false)

  useEffect(() => {
    setIsOnline(navigator.onLine)
    const handleOnline = () => setIsOnline(true)
    const handleOffline = () => setIsOnline(false)
    window.addEventListener('online', handleOnline)
    window.addEventListener('offline', handleOffline)
    return () => {
      window.removeEventListener('online', handleOnline)
      window.removeEventListener('offline', handleOffline)
    }
  }, [])

  const handleRetry = () => window.location.reload()

  return (
    <div style={{ display: 'flex', flexDirection: 'column', alignItems: 'center', justifyContent: 'center', minHeight: '100vh', gap: 16, padding: 24 }}>
      <svg width="64" height="64" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" aria-hidden="true">
        <path d="M1 1l22 22M16.72 11.06A10.94 10.94 0 0 1 19 12.55M5 12.55a10.94 10.94 0 0 1 5.17-2.39M10.71 5.05A16 16 0 0 1 22.56 9M1.42 9a15.91 15.91 0 0 1 4.7-2.88M8.53 16.11a6 6 0 0 1 6.95 0M12 20h.01" strokeLinecap="round" strokeLinejoin="round" />
      </svg>
      <h1 style={{ fontSize: 24, fontWeight: 700, margin: 0 }}>You're offline</h1>
      <p style={{ color: '#666', textAlign: 'center', maxWidth: 320, margin: 0 }}>
        {isOnline
          ? 'Connection restored. You can retry now.'
          : 'Check your internet connection and try again.'}
      </p>
      <button
        onClick={handleRetry}
        style={{ padding: '10px 24px', background: '#1a1a2e', color: '#fff', border: 'none', borderRadius: 8, cursor: 'pointer', fontSize: 16 }}
      >
        {isOnline ? 'Retry' : 'Try again'}
      </button>
    </div>
  )
}
```

### Online/offline hook for use anywhere in the app

```tsx
// hooks/useNetworkStatus.ts
import { useEffect, useState } from 'react'

export function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(true)
  const [wasOffline, setWasOffline] = useState(false)

  useEffect(() => {
    setIsOnline(navigator.onLine)

    const handleOnline = () => {
      setIsOnline(true)
      setWasOffline(true)
      // Reset the "was offline" flag after showing restored banner
      setTimeout(() => setWasOffline(false), 3000)
    }
    const handleOffline = () => {
      setIsOnline(false)
      setWasOffline(false)
    }

    window.addEventListener('online', handleOnline)
    window.addEventListener('offline', handleOffline)
    return () => {
      window.removeEventListener('online', handleOnline)
      window.removeEventListener('offline', handleOffline)
    }
  }, [])

  return { isOnline, wasOffline }
}
```

### Offline banner component

```tsx
// components/OfflineBanner.tsx
'use client'

import { useNetworkStatus } from '@/hooks/useNetworkStatus'

export function OfflineBanner() {
  const { isOnline, wasOffline } = useNetworkStatus()

  if (isOnline && !wasOffline) return null

  return (
    <div
      role="status"
      aria-live="polite"
      style={{
        position: 'fixed',
        top: 0,
        left: 0,
        right: 0,
        padding: '10px 16px',
        textAlign: 'center',
        fontSize: 14,
        fontWeight: 500,
        zIndex: 9999,
        background: isOnline ? '#16a34a' : '#dc2626',
        color: '#fff',
        transition: 'background 0.3s',
      }}
    >
      {isOnline ? 'Back online' : 'You are offline — some features may be unavailable'}
    </div>
  )
}
```

---

## 6. Install Prompt

### `hooks/usePwaInstall.ts`

```tsx
import { useEffect, useState } from 'react'

interface BeforeInstallPromptEvent extends Event {
  prompt: () => Promise<void>
  userChoice: Promise<{ outcome: 'accepted' | 'dismissed' }>
}

export function usePwaInstall() {
  const [installPrompt, setInstallPrompt] = useState<BeforeInstallPromptEvent | null>(null)
  const [isInstalled, setIsInstalled] = useState(false)
  const [isDismissed, setIsDismissed] = useState(false)

  useEffect(() => {
    // Check if already installed
    const isStandalone =
      window.matchMedia('(display-mode: standalone)').matches ||
      (navigator as Navigator & { standalone?: boolean }).standalone === true

    if (isStandalone) {
      setIsInstalled(true)
      return
    }

    // Check if user dismissed recently
    const dismissedAt = localStorage.getItem('pwa-install-dismissed')
    if (dismissedAt) {
      const daysSinceDismissed = (Date.now() - Number(dismissedAt)) / (1000 * 60 * 60 * 24)
      if (daysSinceDismissed < 7) {
        setIsDismissed(true)
      }
    }

    const handleBeforeInstallPrompt = (e: Event) => {
      e.preventDefault()
      setInstallPrompt(e as BeforeInstallPromptEvent)
    }

    const handleAppInstalled = () => {
      setIsInstalled(true)
      setInstallPrompt(null)
    }

    window.addEventListener('beforeinstallprompt', handleBeforeInstallPrompt)
    window.addEventListener('appinstalled', handleAppInstalled)

    return () => {
      window.removeEventListener('beforeinstallprompt', handleBeforeInstallPrompt)
      window.removeEventListener('appinstalled', handleAppInstalled)
    }
  }, [])

  const install = async () => {
    if (!installPrompt) return false
    await installPrompt.prompt()
    const { outcome } = await installPrompt.userChoice
    if (outcome === 'accepted') {
      setIsInstalled(true)
    }
    setInstallPrompt(null)
    return outcome === 'accepted'
  }

  const dismiss = () => {
    setIsDismissed(true)
    localStorage.setItem('pwa-install-dismissed', String(Date.now()))
  }

  const canInstall = !!installPrompt && !isInstalled && !isDismissed

  return { canInstall, isInstalled, install, dismiss }
}
```

### Install button component

```tsx
// components/InstallPrompt.tsx
'use client'

import { usePwaInstall } from '@/hooks/usePwaInstall'

export function InstallPrompt() {
  const { canInstall, install, dismiss } = usePwaInstall()

  if (!canInstall) return null

  return (
    <div
      role="dialog"
      aria-label="Install app"
      style={{
        position: 'fixed',
        bottom: 24,
        left: '50%',
        transform: 'translateX(-50%)',
        background: '#fff',
        border: '1px solid #e5e7eb',
        borderRadius: 12,
        boxShadow: '0 4px 24px rgba(0,0,0,0.12)',
        padding: '16px 20px',
        display: 'flex',
        gap: 12,
        alignItems: 'center',
        maxWidth: 360,
        width: 'calc(100% - 48px)',
        zIndex: 9998,
      }}
    >
      <div style={{ flex: 1 }}>
        <p style={{ margin: 0, fontWeight: 600, fontSize: 15 }}>Install MyApp</p>
        <p style={{ margin: '2px 0 0', fontSize: 13, color: '#6b7280' }}>
          Add to your home screen for faster access
        </p>
      </div>
      <button
        onClick={dismiss}
        aria-label="Dismiss install prompt"
        style={{ background: 'none', border: 'none', cursor: 'pointer', color: '#9ca3af', padding: 4 }}
      >
        ✕
      </button>
      <button
        onClick={install}
        style={{ padding: '8px 16px', background: '#1a1a2e', color: '#fff', border: 'none', borderRadius: 8, cursor: 'pointer', fontWeight: 600, whiteSpace: 'nowrap' }}
      >
        Install
      </button>
    </div>
  )
}
```

### Inline install button (e.g., in settings page)

```tsx
// components/InstallButton.tsx
'use client'

import { usePwaInstall } from '@/hooks/usePwaInstall'

export function InstallButton() {
  const { canInstall, isInstalled, install } = usePwaInstall()

  if (isInstalled) {
    return <p style={{ color: '#16a34a' }}>App is installed</p>
  }

  if (!canInstall) {
    // Not installable (already installed, or browser doesn't support)
    return null
  }

  return (
    <button onClick={install} style={{ padding: '10px 20px', borderRadius: 8, cursor: 'pointer' }}>
      Add to Home Screen
    </button>
  )
}
```

---

## 7. Push Notifications

### Generate VAPID keys (one time)

```bash
npx web-push generate-vapid-keys

# Output:
# Public Key: BEl62i...
# Private Key: 4bq...

# Save to .env.local:
NEXT_PUBLIC_VAPID_PUBLIC_KEY=BEl62i...
VAPID_PRIVATE_KEY=4bq...
VAPID_SUBJECT=mailto:you@example.com
```

### Subscribe (client)

```tsx
// lib/pushNotifications.ts
export async function subscribeToPush(): Promise<PushSubscription | null> {
  if (!('serviceWorker' in navigator) || !('PushManager' in window)) {
    console.warn('Push not supported')
    return null
  }

  const permission = await Notification.requestPermission()
  if (permission !== 'granted') return null

  const registration = await navigator.serviceWorker.ready

  // Check for existing subscription
  const existing = await registration.pushManager.getSubscription()
  if (existing) return existing

  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(
      process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!
    ),
  })

  // Send subscription to your server
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription),
  })

  return subscription
}

export async function unsubscribeFromPush(): Promise<void> {
  const registration = await navigator.serviceWorker.ready
  const subscription = await registration.pushManager.getSubscription()
  if (!subscription) return

  await subscription.unsubscribe()

  await fetch('/api/push/unsubscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ endpoint: subscription.endpoint }),
  })
}

// Utility — browsers need Uint8Array not base64 string
function urlBase64ToUint8Array(base64String: string): Uint8Array {
  const padding = '='.repeat((4 - (base64String.length % 4)) % 4)
  const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/')
  const rawData = window.atob(base64)
  return Uint8Array.from([...rawData].map((char) => char.charCodeAt(0)))
}
```

### Push notification component

```tsx
// components/PushNotificationToggle.tsx
'use client'

import { useEffect, useState } from 'react'
import { subscribeToPush, unsubscribeFromPush } from '@/lib/pushNotifications'

export function PushNotificationToggle() {
  const [status, setStatus] = useState<'unsupported' | 'denied' | 'subscribed' | 'unsubscribed'>('unsubscribed')
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    if (!('Notification' in window)) {
      setStatus('unsupported')
      return
    }
    if (Notification.permission === 'denied') {
      setStatus('denied')
      return
    }
    navigator.serviceWorker.ready.then((reg) =>
      reg.pushManager.getSubscription().then((sub) => {
        setStatus(sub ? 'subscribed' : 'unsubscribed')
      })
    )
  }, [])

  const toggle = async () => {
    setLoading(true)
    if (status === 'subscribed') {
      await unsubscribeFromPush()
      setStatus('unsubscribed')
    } else {
      const sub = await subscribeToPush()
      setStatus(sub ? 'subscribed' : 'unsubscribed')
    }
    setLoading(false)
  }

  if (status === 'unsupported') return <p>Push notifications not supported in this browser.</p>
  if (status === 'denied') return <p>Push notifications are blocked. Enable them in browser settings.</p>

  return (
    <button onClick={toggle} disabled={loading}>
      {loading ? 'Working...' : status === 'subscribed' ? 'Disable notifications' : 'Enable notifications'}
    </button>
  )
}
```

### Server: subscribe endpoint (`app/api/push/subscribe/route.ts`)

```ts
import { NextRequest, NextResponse } from 'next/server'
import webpush from 'web-push'

webpush.setVapidDetails(
  process.env.VAPID_SUBJECT!,
  process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!,
  process.env.VAPID_PRIVATE_KEY!
)

export async function POST(req: NextRequest) {
  const subscription: PushSubscription = await req.json()

  // Save subscription to your database
  // await db.pushSubscriptions.upsert({ endpoint: subscription.endpoint, subscription })

  return NextResponse.json({ success: true })
}
```

### Server: send notification (`app/api/push/send/route.ts`)

```ts
import { NextRequest, NextResponse } from 'next/server'
import webpush from 'web-push'

webpush.setVapidDetails(
  process.env.VAPID_SUBJECT!,
  process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!,
  process.env.VAPID_PRIVATE_KEY!
)

export async function POST(req: NextRequest) {
  const { title, body, url } = await req.json()

  // Fetch all subscriptions from your database
  // const subscriptions = await db.pushSubscriptions.findAll()

  const payload = JSON.stringify({ title, body, url, icon: '/icons/icon-192x192.png' })

  const results = await Promise.allSettled(
    subscriptions.map((sub) => webpush.sendNotification(sub.subscription, payload))
  )

  const failed = results.filter((r) => r.status === 'rejected')
  // Remove invalid subscriptions (410 Gone = user unsubscribed)
  // await db.pushSubscriptions.deleteMany(failed.map(f => f.endpoint))

  return NextResponse.json({ sent: results.length - failed.length, failed: failed.length })
}
```

### Service worker: handle push event

```js
// Add to your custom SW file (public/sw-custom.js or swSrc)
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {}
  const { title = 'MyApp', body = '', url = '/', icon = '/icons/icon-192x192.png' } = data

  event.waitUntil(
    self.registration.showNotification(title, {
      body,
      icon,
      badge: '/icons/badge-72x72.png',
      data: { url },
      actions: [
        { action: 'open', title: 'Open' },
        { action: 'dismiss', title: 'Dismiss' },
      ],
      vibrate: [200, 100, 200],
    })
  )
})

self.addEventListener('notificationclick', (event) => {
  event.notification.close()
  if (event.action === 'dismiss') return

  const url = event.notification.data?.url ?? '/'
  event.waitUntil(
    clients.matchAll({ type: 'window', includeUncontrolled: true }).then((clientList) => {
      const existing = clientList.find((c) => c.url === url && 'focus' in c)
      if (existing) return existing.focus()
      return clients.openWindow(url)
    })
  )
})
```

---

## 8. Background Sync

Background sync queues failed requests and retries them when connectivity returns. Chromium only.

### Setup in next.config.js

```js
// In runtimeCaching array:
{
  urlPattern: ({ url }) => url.pathname.startsWith('/api/submit'),
  handler: 'NetworkOnly',
  method: 'POST',
  options: {
    backgroundSync: {
      name: 'submitQueue',
      options: {
        maxRetentionTime: 24 * 60, // minutes — retry for 24 hours
      },
    },
  },
}
```

### Manual background sync (custom SW)

```js
// In sw-custom.js
import { BackgroundSyncPlugin } from 'workbox-background-sync'
import { registerRoute } from 'workbox-routing'
import { NetworkOnly } from 'workbox-strategies'

const bgSyncPlugin = new BackgroundSyncPlugin('formQueue', {
  maxRetentionTime: 24 * 60,
  onSync: async ({ queue }) => {
    let entry
    while ((entry = await queue.shiftRequest())) {
      try {
        await fetch(entry.request)
      } catch {
        await queue.unshiftRequest(entry)
        throw new Error('Sync failed, retrying later')
      }
    }
  },
})

registerRoute(
  ({ url }) => url.pathname === '/api/forms/submit',
  new NetworkOnly({ plugins: [bgSyncPlugin] }),
  'POST'
)
```

### IndexedDB queue for complex offline actions

```ts
// lib/offlineQueue.ts
interface QueuedAction {
  id: string
  type: string
  payload: unknown
  createdAt: number
}

const DB_NAME = 'offlineQueue'
const STORE_NAME = 'actions'

async function openDb(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(DB_NAME, 1)
    request.onupgradeneeded = () => {
      request.result.createObjectStore(STORE_NAME, { keyPath: 'id' })
    }
    request.onsuccess = () => resolve(request.result)
    request.onerror = () => reject(request.error)
  })
}

export async function enqueueAction(type: string, payload: unknown): Promise<void> {
  const db = await openDb()
  const action: QueuedAction = { id: crypto.randomUUID(), type, payload, createdAt: Date.now() }
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readwrite')
    tx.objectStore(STORE_NAME).add(action)
    tx.oncomplete = () => resolve()
    tx.onerror = () => reject(tx.error)
  })
}

export async function flushQueue(processor: (action: QueuedAction) => Promise<void>): Promise<void> {
  const db = await openDb()
  const actions: QueuedAction[] = await new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readonly')
    const req = tx.objectStore(STORE_NAME).getAll()
    req.onsuccess = () => resolve(req.result)
    req.onerror = () => reject(req.error)
  })

  for (const action of actions) {
    await processor(action)
    // Remove after successful processing
    await new Promise<void>((resolve, reject) => {
      const tx = db.transaction(STORE_NAME, 'readwrite')
      tx.objectStore(STORE_NAME).delete(action.id)
      tx.oncomplete = () => resolve()
      tx.onerror = () => reject(tx.error)
    })
  }
}
```

### Flush on reconnect

```tsx
// In your root layout or _app
useEffect(() => {
  const handleOnline = async () => {
    await flushQueue(async (action) => {
      await fetch('/api/sync', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(action),
      })
    })
  }
  window.addEventListener('online', handleOnline)
  return () => window.removeEventListener('online', handleOnline)
}, [])
```

---

## 9. App Shortcuts

Shortcuts appear on desktop (right-click app icon in taskbar) and mobile (long-press home screen icon). Already defined in the manifest — here is guidance on making them work well.

### Rules for shortcuts:
- Maximum 4 shortcuts (browsers cap it)
- Each needs a dedicated icon at 96x96 (some browsers also use 192x192)
- URLs must be within the same origin as `start_url`
- Use `?source=shortcut` to track usage in analytics

### Dynamic shortcuts via Service Worker (Chrome 96+)

```js
// In sw-custom.js — update shortcuts based on user state
self.addEventListener('activate', (event) => {
  event.waitUntil(
    (async () => {
      if ('shortcuts' in navigator) {
        await navigator.shortcuts.set([
          {
            name: 'Recent Items',
            url: '/recent',
            icons: [{ src: '/icons/shortcut-recent.png', sizes: '96x96' }],
          },
        ])
      }
    })()
  )
})
```

### Shortcut icon generation

```bash
# Create 96x96 icons for each shortcut
for shortcut in new dashboard settings search; do
  sharp public/icons/icon-source.png \
    --resize 96 96 \
    -o public/icons/shortcut-${shortcut}.png
done
```

---

## 10. Update Flow

When a new service worker is deployed, the old SW keeps running until all tabs are closed. A good UX prompts the user to reload.

### `hooks/useServiceWorkerUpdate.ts`

```tsx
import { useEffect, useState } from 'react'

export function useServiceWorkerUpdate() {
  const [updateAvailable, setUpdateAvailable] = useState(false)
  const [registration, setRegistration] = useState<ServiceWorkerRegistration | null>(null)

  useEffect(() => {
    if (!('serviceWorker' in navigator)) return

    navigator.serviceWorker.getRegistration().then((reg) => {
      if (!reg) return
      setRegistration(reg)

      // SW already waiting (e.g., user had another tab open)
      if (reg.waiting) {
        setUpdateAvailable(true)
        return
      }

      // New SW installing
      reg.addEventListener('updatefound', () => {
        const newWorker = reg.installing
        if (!newWorker) return
        newWorker.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            setUpdateAvailable(true)
          }
        })
      })
    })

    // Reload all clients once new SW takes control
    let refreshing = false
    navigator.serviceWorker.addEventListener('controllerchange', () => {
      if (refreshing) return
      refreshing = true
      window.location.reload()
    })
  }, [])

  const applyUpdate = () => {
    if (!registration?.waiting) return
    registration.waiting.postMessage({ type: 'SKIP_WAITING' })
  }

  return { updateAvailable, applyUpdate }
}
```

### Update banner

```tsx
// components/UpdateBanner.tsx
'use client'

import { useServiceWorkerUpdate } from '@/hooks/useServiceWorkerUpdate'

export function UpdateBanner() {
  const { updateAvailable, applyUpdate } = useServiceWorkerUpdate()

  if (!updateAvailable) return null

  return (
    <div
      role="alert"
      style={{
        position: 'fixed',
        bottom: 24,
        left: '50%',
        transform: 'translateX(-50%)',
        background: '#1a1a2e',
        color: '#fff',
        borderRadius: 10,
        padding: '12px 20px',
        display: 'flex',
        gap: 16,
        alignItems: 'center',
        zIndex: 9999,
        boxShadow: '0 4px 20px rgba(0,0,0,0.3)',
        maxWidth: 380,
        width: 'calc(100% - 48px)',
      }}
    >
      <p style={{ margin: 0, fontSize: 14 }}>A new version is available.</p>
      <button
        onClick={applyUpdate}
        style={{ padding: '6px 14px', background: '#fff', color: '#1a1a2e', border: 'none', borderRadius: 6, cursor: 'pointer', fontWeight: 600, whiteSpace: 'nowrap', fontSize: 13 }}
      >
        Refresh
      </button>
    </div>
  )
}
```

### Wire it all together in root layout

```tsx
// app/layout.tsx additions
import { OfflineBanner } from '@/components/OfflineBanner'
import { InstallPrompt } from '@/components/InstallPrompt'
import { UpdateBanner } from '@/components/UpdateBanner'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <OfflineBanner />
        {children}
        <InstallPrompt />
        <UpdateBanner />
      </body>
    </html>
  )
}
```

---

## 11. Testing

### Chrome DevTools — Application panel

1. Open DevTools → Application tab
2. **Manifest** — check name, icons, theme_color; look for validation errors
3. **Service Workers** — verify SW is registered and active; use "Update on reload" during development; test "Offline" checkbox
4. **Cache Storage** — inspect what's cached per cache name; delete entries to test cache misses
5. **Background Sync** — see queued sync events under "Background Sync"
6. **Push Messaging** — send a test push without a server using "Push" button
7. **Storage** — quota usage and persistent storage status

### Lighthouse PWA audit

```bash
# CLI audit
npx lighthouse https://your-app.com \
  --only-categories=pwa \
  --output=html \
  --output-path=pwa-report.html \
  --chrome-flags="--headless"

# Or in DevTools: Lighthouse tab → select PWA category → Analyze
```

**Lighthouse PWA checklist (installable):**
- [ ] Has valid manifest with required fields
- [ ] Has maskable icon
- [ ] Registered service worker
- [ ] Responds with 200 when offline
- [ ] Sets `<meta name="viewport">`
- [ ] HTTPS (or localhost)

### Playwright PWA tests

```ts
// tests/pwa.spec.ts
import { test, expect } from '@playwright/test'

test('offline page shows when network is blocked', async ({ page, context }) => {
  await page.goto('/')
  // Navigate to a page not in cache
  await context.setOffline(true)
  await page.goto('/uncached-page', { waitUntil: 'networkidle' })
  await expect(page.getByRole('heading', { name: /offline/i })).toBeVisible()
})

test('manifest is valid', async ({ request }) => {
  const res = await request.get('/manifest.json')
  expect(res.status()).toBe(200)
  const manifest = await res.json()
  expect(manifest.name).toBeTruthy()
  expect(manifest.icons).toHaveLength(expect.any(Number))
  expect(manifest.start_url).toBeTruthy()
})

test('service worker registers', async ({ page }) => {
  await page.goto('/')
  const swRegistered = await page.evaluate(() =>
    navigator.serviceWorker.getRegistration().then((reg) => !!reg)
  )
  expect(swRegistered).toBe(true)
})
```

### Manual install testing

1. Open app in Chrome on Android via USB debug or real URL
2. Should see install banner after enough engagement signals (visiting 2+ times, 30+ seconds)
3. Or: DevTools → Application → Manifest → "Add to homescreen" button
4. Verify shortcut icons, splash screen, and theme_color on launch

---

## 12. Anti-Patterns

### Caching everything blindly

```js
// BAD — caches authenticated API responses for all users
{
  urlPattern: /^\/api\/.*/,
  handler: 'CacheFirst',  // Wrong for auth endpoints
}

// GOOD — network-first for auth, cache-first only for public static
{
  urlPattern: /^\/api\/user\/.*/,
  handler: 'NetworkOnly',  // Never cache user-specific data
}
{
  urlPattern: /^\/api\/public\/.*/,
  handler: 'StaleWhileRevalidate',
  options: { expiration: { maxAgeSeconds: 3600 } },
}
```

### No offline fallback

```js
// BAD — next-pwa config with no fallback
// When the user is offline and visits a page not in cache, they get browser's generic error

// GOOD — always set a fallback document
fallbacks: {
  document: '/offline',
}
```

### Ignoring the update flow

```js
// BAD — skipWaiting: true with no user notification
// The page reloads silently while the user is typing — data loss risk

// GOOD — skipWaiting: false + prompt user with UpdateBanner (see Section 10)
skipWaiting: false,
```

### Caching POST requests without background sync

```js
// BAD — caching POST responses means stale mutations
{
  urlPattern: /^\/api\/submit/,
  handler: 'CacheFirst',  // Never cache POST
  method: 'POST',
}

// GOOD — use NetworkOnly + BackgroundSyncPlugin for POST
{
  urlPattern: /^\/api\/submit/,
  handler: 'NetworkOnly',
  method: 'POST',
  options: { backgroundSync: { name: 'submitQueue' } },
}
```

### Requesting push permission on page load

```tsx
// BAD — browsers will block sites that ask immediately
useEffect(() => {
  Notification.requestPermission()  // Instant block — user hasn't opted in
}, [])

// GOOD — ask only after explicit user action
<button onClick={() => subscribeToPush()}>
  Enable notifications
</button>
```

### Missing maskable icon

```json
// BAD — only 'any' purpose icons
{ "src": "/icon-512.png", "sizes": "512x512", "purpose": "any" }

// GOOD — both any and maskable
{ "src": "/icon-512.png", "sizes": "512x512", "purpose": "any" },
{ "src": "/icon-maskable-512.png", "sizes": "512x512", "purpose": "maskable" }
// Without maskable, Android shows your icon in a white circle/square
```

### Assuming Safari supports everything

```ts
// Check before using Chromium-only APIs
if ('BackgroundSync' in self) {
  // Background sync — Chromium only
}

if ('PushManager' in window) {
  // Push — supported, but iOS requires app to be installed to home screen
}

if ('periodicSync' in registration) {
  // Periodic background sync — Chromium only, requires site engagement score
}
```
