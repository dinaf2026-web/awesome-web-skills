---
name: web-analytics
description: Analytics and event tracking patterns for modern web apps — GA4, PostHog, Vercel Analytics, and custom event tracking. Know what users actually do on your site.
origin: community
tags: [analytics, ga4, posthog, vercel-analytics, event-tracking, funnels, heatmaps]
---

# Web Analytics

Analytics and event tracking patterns for modern web apps. GA4, PostHog, Vercel Analytics, and custom event systems — set up once, trust forever.

---

## 1. When to Use Which

| Tool | Best For | Pricing | Self-Host | Autocapture | Session Recording |
|------|----------|---------|-----------|-------------|-------------------|
| **Vercel Analytics** | Next.js perf + Web Vitals, zero-config | Free tier generous | No | No | No |
| **GA4** | Marketing attribution, Google Ads, SEO reporting | Free | No | Limited | No |
| **PostHog** | Product analytics, funnels, feature flags, session replay | Generous free tier | Yes | Yes | Yes |
| **Plausible** | Privacy-first page stats, GDPR out of the box | $9/mo | Yes | No | No |

**Decision rules:**

- Solo project / side project → Vercel Analytics + Plausible. Done.
- SaaS product that needs funnels, retention, feature flags → PostHog.
- Marketing site with Google Ads spend → GA4. You need conversion attribution.
- Enterprise with strict data residency → PostHog self-hosted.
- Use **PostHog + Vercel Analytics** together. They solve different problems.

---

## 2. Vercel Analytics Setup

Zero-config Web Vitals. Add it and forget it.

```bash
npm install @vercel/analytics @vercel/speed-insights
```

### Root Layout

```tsx
// app/layout.tsx
import { Analytics } from '@vercel/analytics/react'
import { SpeedInsights } from '@vercel/speed-insights/next'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Analytics />
        <SpeedInsights />
      </body>
    </html>
  )
}
```

### Custom Events

```tsx
// lib/analytics/vercel.ts
import { track } from '@vercel/analytics'

export function trackVercel(event: string, properties?: Record<string, string | number | boolean>) {
  track(event, properties)
}
```

```tsx
// Usage in a component
import { trackVercel } from '@/lib/analytics/vercel'

function SignupButton() {
  return (
    <button
      onClick={() => {
        trackVercel('signup_clicked', { plan: 'pro', source: 'hero' })
      }}
    >
      Get Started
    </button>
  )
}
```

Vercel Analytics custom events appear in your Vercel dashboard under the Events tab. Properties must be strings, numbers, or booleans — no nested objects.

---

## 3. GA4 Setup in Next.js (App Router)

### Install

No package needed. GA4 loads via Google's script tag.

### Environment Variable

```bash
# .env.local
NEXT_PUBLIC_GA_MEASUREMENT_ID=G-XXXXXXXXXX
```

### Script Component — Root Layout

```tsx
// app/layout.tsx
import Script from 'next/script'

const GA_ID = process.env.NEXT_PUBLIC_GA_MEASUREMENT_ID

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <head>
        {GA_ID && (
          <>
            <Script
              src={`https://www.googletagmanager.com/gtag/js?id=${GA_ID}`}
              strategy="afterInteractive"
            />
            <Script id="google-analytics" strategy="afterInteractive">
              {`
                window.dataLayer = window.dataLayer || [];
                function gtag(){dataLayer.push(arguments);}
                gtag('js', new Date());
                gtag('config', '${GA_ID}', {
                  page_path: window.location.pathname,
                  anonymize_ip: true,
                  cookie_flags: 'SameSite=None;Secure'
                });
              `}
            </Script>
          </>
        )}
      </head>
      <body>{children}</body>
    </html>
  )
}
```

### GA4 Wrapper

```tsx
// lib/analytics/ga4.ts
declare global {
  interface Window {
    gtag: (...args: unknown[]) => void
    dataLayer: unknown[]
  }
}

const GA_ID = process.env.NEXT_PUBLIC_GA_MEASUREMENT_ID

export function pageview(url: string) {
  if (!GA_ID || typeof window === 'undefined') return
  window.gtag('config', GA_ID, { page_path: url })
}

export function gtagEvent(
  action: string,
  params: {
    category?: string
    label?: string
    value?: number
    [key: string]: unknown
  } = {}
) {
  if (!GA_ID || typeof window === 'undefined') return
  window.gtag('event', action, params)
}
```

### Pageview Tracking on Route Changes (App Router)

App Router handles navigation differently from Pages Router. Use a client component that listens to pathname changes.

```tsx
// components/analytics/GoogleAnalytics.tsx
'use client'

import { usePathname, useSearchParams } from 'next/navigation'
import { useEffect } from 'react'
import { pageview } from '@/lib/analytics/ga4'

export function GoogleAnalytics() {
  const pathname = usePathname()
  const searchParams = useSearchParams()

  useEffect(() => {
    const url = pathname + (searchParams.toString() ? `?${searchParams.toString()}` : '')
    pageview(url)
  }, [pathname, searchParams])

  return null
}
```

```tsx
// app/layout.tsx — add inside <body>
import { Suspense } from 'react'
import { GoogleAnalytics } from '@/components/analytics/GoogleAnalytics'

// Wrap in Suspense — useSearchParams requires it in App Router
<Suspense fallback={null}>
  <GoogleAnalytics />
</Suspense>
```

---

## 4. PostHog Setup

Full product analytics: autocapture, funnels, session recording, feature flags.

```bash
npm install posthog-js posthog-node
```

### Environment Variables

```bash
NEXT_PUBLIC_POSTHOG_KEY=phc_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
NEXT_PUBLIC_POSTHOG_HOST=https://us.i.posthog.com
# For server-side: use the same key, PostHog Node uses it directly
```

### Client Provider

```tsx
// components/analytics/PostHogProvider.tsx
'use client'

import posthog from 'posthog-js'
import { PostHogProvider as PHProvider } from 'posthog-js/react'
import { useEffect } from 'react'

export function PostHogProvider({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
      api_host: process.env.NEXT_PUBLIC_POSTHOG_HOST,
      person_profiles: 'identified_only', // only create profiles for identified users
      capture_pageview: false, // handle manually for App Router
      capture_pageleave: true,
      session_recording: {
        maskAllInputs: true, // mask form inputs in recordings
        maskInputOptions: { password: true },
      },
      loaded: (ph) => {
        if (process.env.NODE_ENV === 'development') {
          ph.opt_out_capturing() // don't pollute prod data with dev events
        }
      },
    })
  }, [])

  return <PHProvider client={posthog}>{children}</PHProvider>
}
```

### Pageview Tracking

```tsx
// components/analytics/PostHogPageview.tsx
'use client'

import { usePathname, useSearchParams } from 'next/navigation'
import { useEffect } from 'react'
import { usePostHog } from 'posthog-js/react'

export function PostHogPageview() {
  const pathname = usePathname()
  const searchParams = useSearchParams()
  const posthog = usePostHog()

  useEffect(() => {
    if (!posthog) return
    const url = window.origin + pathname + (searchParams.toString() ? `?${searchParams.toString()}` : '')
    posthog.capture('$pageview', { $current_url: url })
  }, [pathname, searchParams, posthog])

  return null
}
```

### Root Layout Integration

```tsx
// app/layout.tsx
import { Suspense } from 'react'
import { PostHogProvider } from '@/components/analytics/PostHogProvider'
import { PostHogPageview } from '@/components/analytics/PostHogPageview'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <PostHogProvider>
          <Suspense fallback={null}>
            <PostHogPageview />
          </Suspense>
          {children}
        </PostHogProvider>
      </body>
    </html>
  )
}
```

### Server-Side PostHog (API Routes, Server Actions)

```tsx
// lib/analytics/posthog-server.ts
import { PostHog } from 'posthog-node'

let _client: PostHog | null = null

export function getPostHogServer(): PostHog {
  if (!_client) {
    _client = new PostHog(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
      host: process.env.NEXT_PUBLIC_POSTHOG_HOST,
      flushAt: 1,      // flush immediately in serverless
      flushInterval: 0,
    })
  }
  return _client
}

// Always shut down after serverless invocation
export async function shutdownPostHog() {
  if (_client) {
    await _client.shutdown()
    _client = null
  }
}
```

---

## 5. Custom Event Tracking Pattern

Type-safe, centralized event system. One place to change event names. No magic strings scattered across components.

### Define Your Event Catalog

```tsx
// lib/analytics/events.ts

// All events your app can emit — add new ones here only
export type AnalyticsEvent =
  // Auth
  | { name: 'signup_started'; properties: { source: string; plan?: string } }
  | { name: 'signup_completed'; properties: { plan: string; method: 'email' | 'google' | 'github' } }
  | { name: 'login'; properties: { method: 'email' | 'google' | 'github' } }
  | { name: 'logout'; properties: Record<string, never> }
  // Billing
  | { name: 'upgrade_clicked'; properties: { from_plan: string; to_plan: string; trigger: string } }
  | { name: 'checkout_started'; properties: { plan: string; price: number; currency: string } }
  | { name: 'payment_completed'; properties: { plan: string; price: number; currency: string } }
  | { name: 'payment_failed'; properties: { plan: string; error_code: string } }
  // Feature usage
  | { name: 'feature_used'; properties: { feature: string; context?: string } }
  | { name: 'export_triggered'; properties: { format: 'csv' | 'pdf' | 'json'; record_count: number } }
  // Engagement
  | { name: 'onboarding_step_completed'; properties: { step: number; step_name: string } }
  | { name: 'search_performed'; properties: { query: string; results_count: number } }
  | { name: 'error_encountered'; properties: { error_type: string; page: string } }
```

### Centralized track() Function

```tsx
// lib/analytics/track.ts
import posthog from 'posthog-js'
import { gtagEvent } from './ga4'
import { trackVercel } from './vercel'
import type { AnalyticsEvent } from './events'

export function track<T extends AnalyticsEvent>(
  event: T['name'],
  properties: Extract<T, { name: T['name'] }>['properties']
) {
  // PostHog
  if (typeof window !== 'undefined') {
    posthog.capture(event, properties)
  }

  // GA4 — map to GA4 event structure
  gtagEvent(event, properties as Record<string, unknown>)

  // Vercel — flatten to primitives only
  const flatProps = flattenForVercel(properties)
  trackVercel(event, flatProps)
}

// Vercel only accepts string | number | boolean values
function flattenForVercel(
  obj: Record<string, unknown>
): Record<string, string | number | boolean> {
  return Object.fromEntries(
    Object.entries(obj)
      .filter(([, v]) => ['string', 'number', 'boolean'].includes(typeof v))
      .map(([k, v]) => [k, v as string | number | boolean])
  )
}
```

### Usage

```tsx
// Anywhere in your app — fully typed, autocompletes
import { track } from '@/lib/analytics/track'

// TypeScript enforces correct properties for each event name
track('signup_completed', {
  plan: 'pro',
  method: 'google',
})

track('export_triggered', {
  format: 'csv',
  record_count: 1482,
})

// Type error — 'method' is required for signup_completed
// track('signup_completed', { plan: 'pro' }) // ❌ TS error
```

---

## 6. Funnel Tracking

Funnels only work if every step fires the right event with consistent properties. Define the funnel events before building the feature.

### Signup Funnel

```
Landing page visit → CTA clicked → Signup form viewed → Email entered →
Plan selected → Payment entered → Account created → Onboarding step 1 → ... → Activated
```

```tsx
// lib/analytics/funnels/signup.ts
import { track } from '@/lib/analytics/track'

export const signupFunnel = {
  ctaClicked: (source: string, plan?: string) =>
    track('signup_started', { source, plan }),

  formViewed: () =>
    track('signup_started', { source: 'direct' }),

  completed: (plan: string, method: 'email' | 'google' | 'github') =>
    track('signup_completed', { plan, method }),

  onboardingStep: (step: number, stepName: string) =>
    track('onboarding_step_completed', { step, step_name: stepName }),
}
```

```tsx
// Usage — clean call sites
import { signupFunnel } from '@/lib/analytics/funnels/signup'

// Hero CTA
<button onClick={() => signupFunnel.ctaClicked('hero', 'pro')}>
  Start Free Trial
</button>

// After account creation
await createAccount(data)
signupFunnel.completed(selectedPlan, 'email')
```

### Checkout Funnel

```tsx
// lib/analytics/funnels/checkout.ts
import { track } from '@/lib/analytics/track'

export const checkoutFunnel = {
  upgradeClicked: (fromPlan: string, toPlan: string, trigger: string) =>
    track('upgrade_clicked', { from_plan: fromPlan, to_plan: toPlan, trigger }),

  started: (plan: string, price: number) =>
    track('checkout_started', { plan, price, currency: 'USD' }),

  completed: (plan: string, price: number) =>
    track('payment_completed', { plan, price, currency: 'USD' }),

  failed: (plan: string, errorCode: string) =>
    track('payment_failed', { plan, error_code: errorCode }),
}
```

### PostHog Funnel Configuration

In PostHog UI: Insights → Funnels → add these events in order:

```
signup_started → signup_completed → onboarding_step_completed (step=1) →
onboarding_step_completed (step=3) → feature_used
```

Set the conversion window to 7 days. Filter by `plan = 'pro'` to see paid vs free behavior.

---

## 7. User Identification

Identify users immediately after login. Traits flow through to every subsequent event.

### Client-Side Identification

```tsx
// lib/analytics/identify.ts
import posthog from 'posthog-js'
import { gtagEvent } from './ga4'

interface UserTraits {
  email: string
  name?: string
  plan?: string
  created_at?: string
  company?: string
  company_size?: number
  role?: string
}

export function identifyUser(userId: string, traits: UserTraits) {
  // PostHog
  posthog.identify(userId, {
    email: traits.email,
    name: traits.name,
    plan: traits.plan,
    created_at: traits.created_at,
    // Company data as group (PostHog Groups feature)
    ...(traits.company && { company: traits.company }),
  })

  // GA4 — set user properties
  if (typeof window !== 'undefined' && window.gtag) {
    window.gtag('set', 'user_properties', {
      plan: traits.plan,
      user_id: userId,
    })
  }
}

export function setCompany(companyId: string, companyTraits: { name: string; plan?: string; size?: number }) {
  // PostHog Groups — associates user with a company for B2B analytics
  posthog.group('company', companyId, {
    name: companyTraits.name,
    plan: companyTraits.plan,
    size: companyTraits.size,
  })
}

export function resetUser() {
  posthog.reset()
}
```

### Call After Login

```tsx
// app/actions/auth.ts (Server Action)
'use server'

import { redirect } from 'next/navigation'

export async function loginAction(formData: FormData) {
  const user = await authenticateUser(formData)
  // ... session setup
  return { userId: user.id, traits: { email: user.email, plan: user.plan } }
}
```

```tsx
// components/auth/LoginForm.tsx
'use client'

import { identifyUser } from '@/lib/analytics/identify'
import { track } from '@/lib/analytics/track'

export function LoginForm() {
  async function handleLogin(formData: FormData) {
    const result = await loginAction(formData)
    if (result) {
      identifyUser(result.userId, result.traits)
      track('login', { method: 'email' })
    }
  }

  return <form action={handleLogin}>...</form>
}
```

### Server-Side Identification (API Routes)

```tsx
// app/api/webhooks/stripe/route.ts
import { getPostHogServer, shutdownPostHog } from '@/lib/analytics/posthog-server'

export async function POST(request: Request) {
  const event = await parseStripeWebhook(request)

  if (event.type === 'customer.subscription.created') {
    const ph = getPostHogServer()
    ph.capture({
      distinctId: event.data.object.metadata.userId,
      event: 'payment_completed',
      properties: {
        plan: event.data.object.metadata.plan,
        price: event.data.object.plan.amount / 100,
        currency: 'USD',
      },
    })
    await shutdownPostHog()
  }

  return Response.json({ received: true })
}
```

---

## 8. Feature Flag Integration

Feature flags in PostHog. Same system as your analytics — no extra SDK.

### Client-Side Flags

```tsx
// lib/analytics/flags.ts
import { useFeatureFlagEnabled, useFeatureFlagVariantKey } from 'posthog-js/react'

// Simple boolean flag
export function useFlag(flagKey: string): boolean {
  return useFeatureFlagEnabled(flagKey) ?? false
}

// A/B test variant
export function useVariant(flagKey: string): string | null {
  const variant = useFeatureFlagVariantKey(flagKey)
  return typeof variant === 'string' ? variant : null
}
```

```tsx
// components/Pricing.tsx
'use client'

import { useVariant } from '@/lib/analytics/flags'

export function Pricing() {
  const pricingVariant = useVariant('pricing_page_test')
  // 'control' | 'variant_annual_first' | 'variant_monthly_first' | null

  if (pricingVariant === 'variant_annual_first') {
    return <AnnualFirstPricing />
  }

  return <DefaultPricing />
}
```

### Server-Side Flags (App Router Server Components)

```tsx
// app/dashboard/page.tsx
import { getPostHogServer, shutdownPostHog } from '@/lib/analytics/posthog-server'
import { auth } from '@/lib/auth'

export default async function DashboardPage() {
  const session = await auth()
  const ph = getPostHogServer()

  const showNewDashboard = await ph.isFeatureEnabled(
    'new_dashboard',
    session.user.id
  )

  await shutdownPostHog()

  return showNewDashboard ? <NewDashboard /> : <LegacyDashboard />
}
```

### Track Experiment Exposure

PostHog autocaptures `$feature_flag_called` events. For manual tracking:

```tsx
import { useEffect } from 'react'
import { usePostHog } from 'posthog-js/react'
import { useVariant } from '@/lib/analytics/flags'

function ExperimentComponent() {
  const posthog = usePostHog()
  const variant = useVariant('checkout_experiment')

  useEffect(() => {
    if (variant) {
      posthog.capture('experiment_viewed', {
        experiment: 'checkout_experiment',
        variant,
      })
    }
  }, [variant, posthog])

  return <div>...</div>
}
```

---

## 9. Privacy Compliance

### Cookie Consent Gate

Don't fire analytics until the user consents. PostHog and GA4 both support deferred initialization.

```tsx
// lib/analytics/consent.ts
const CONSENT_KEY = 'analytics_consent'

export type ConsentState = 'granted' | 'denied' | 'pending'

export function getConsent(): ConsentState {
  if (typeof window === 'undefined') return 'pending'
  return (localStorage.getItem(CONSENT_KEY) as ConsentState) ?? 'pending'
}

export function setConsent(state: 'granted' | 'denied') {
  localStorage.setItem(CONSENT_KEY, state)
  window.dispatchEvent(new CustomEvent('consent_updated', { detail: state }))
}
```

```tsx
// components/analytics/ConsentBanner.tsx
'use client'

import { useState, useEffect } from 'react'
import posthog from 'posthog-js'
import { getConsent, setConsent } from '@/lib/analytics/consent'

export function ConsentBanner() {
  const [visible, setVisible] = useState(false)

  useEffect(() => {
    if (getConsent() === 'pending') setVisible(true)
  }, [])

  function accept() {
    setConsent('granted')
    posthog.opt_in_capturing()
    // GA4 — update consent mode
    window.gtag?.('consent', 'update', {
      analytics_storage: 'granted',
      ad_storage: 'denied', // deny ads unless specifically needed
    })
    setVisible(false)
  }

  function decline() {
    setConsent('denied')
    posthog.opt_out_capturing()
    window.gtag?.('consent', 'update', {
      analytics_storage: 'denied',
      ad_storage: 'denied',
    })
    setVisible(false)
  }

  if (!visible) return null

  return (
    <div role="dialog" aria-label="Cookie consent" className="consent-banner">
      <p>We use analytics to improve your experience.</p>
      <button onClick={accept}>Accept</button>
      <button onClick={decline}>Decline</button>
    </div>
  )
}
```

### GA4 Consent Mode Default

Set consent defaults before GA4 loads — required for GDPR markets.

```tsx
// app/layout.tsx — Script BEFORE the GA4 script tag
<Script id="consent-defaults" strategy="beforeInteractive">
  {`
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('consent', 'default', {
      analytics_storage: 'denied',
      ad_storage: 'denied',
      wait_for_update: 500
    });
  `}
</Script>
```

### PostHog — Default Opt Out Until Consent

```tsx
// In PostHog init:
posthog.init(key, {
  opt_out_capturing_by_default: true, // start opted out
  // After user grants consent:
  // posthog.opt_in_capturing()
})
```

### IP Anonymization and Data Retention

```tsx
// GA4 config
gtag('config', GA_ID, {
  anonymize_ip: true,              // truncates last IP octet
  storage: 'none',                 // disable cookies if consent denied
  client_storage: 'none',
})

// PostHog — disable geo from IP
posthog.init(key, {
  ip: false, // don't capture IP address
  persistence: 'memory', // session only, no localStorage until consent
})
```

---

## 10. Dashboard Setup

### SaaS — Key Metrics

**Acquisition**
- `signup_started` by source (UTM params) — where are signups coming from?
- `signup_completed` conversion rate — what % who start actually finish?
- Time to signup from first pageview

**Activation**
- `onboarding_step_completed` funnel — where do users drop off?
- Time to first `feature_used` after signup
- % of signups who reach "activated" state within 7 days

**Retention**
- DAU/WAU/MAU (PostHog Trends)
- `feature_used` events per user per week
- Cohort retention curves (PostHog → Retention)

**Revenue**
- `checkout_started` → `payment_completed` conversion
- `upgrade_clicked` by trigger — which prompts drive upgrades?
- MRR by plan (pull from Stripe, cross-reference with PostHog user properties)

**Health**
- `error_encountered` rate and breakdown by `error_type`
- `payment_failed` rate and `error_code` breakdown

### Ecommerce — Key Metrics

```
product_viewed → add_to_cart → checkout_started → payment_completed
```

Track with these events:
- `product_viewed` — `{ product_id, category, price }`
- `add_to_cart` — `{ product_id, quantity, cart_total }`
- `checkout_started` — `{ cart_total, item_count }`
- `payment_completed` — `{ order_id, total, items_count }`
- `abandoned_cart` — fire via server-side job 30 min after `add_to_cart` without `checkout_started`

### Content Site — Key Metrics

- Pageviews, unique visitors, scroll depth (PostHog heatmaps)
- Time on page by content category
- `search_performed` — what are people looking for?
- Bounce rate by traffic source
- CTA click rate per article

---

## 11. Anti-Patterns

**Tracking everything without a plan.**
Fifty events with no defined funnels, no one looks at them, you learn nothing. Start with 10 events that answer specific questions. Add more when a specific question requires a new event.

**Magic strings everywhere.**
```tsx
// Bad — typo-prone, impossible to rename, no autocomplete
posthog.capture('singup_completed', { plan: 'pro' }) // typo

// Good — typed event catalog, one place to change
track('signup_completed', { plan: 'pro', method: 'email' })
```

**No user identification.**
Anonymous events are largely useless for product analytics. If you can't connect an event to a user, you can't build cohorts, funnels, or retention curves. Identify immediately after login.

**Firing analytics in development.**
Dev events pollute your prod dashboards and inflate counts. PostHog's `opt_out_capturing()` in dev mode (shown above) handles this. For GA4, filter internal traffic by IP in GA4 Admin → Data Streams → Configure tag settings → Define internal traffic.

**Ignoring mobile and touch.**
Test your event tracking on mobile. Click events on non-button elements often don't fire on iOS. Use `onClick` on real `<button>` elements. PostHog session recording on mobile will catch dead clicks.

**Blocking render on analytics.**
Analytics must never block your UI. All tracking calls are fire-and-forget. Never `await` a `track()` call in a render path.

```tsx
// Bad — analytics blocking checkout
const handleCheckout = async () => {
  await track('checkout_started', { plan })  // never await analytics
  await startCheckout()
}

// Good
const handleCheckout = async () => {
  track('checkout_started', { plan })  // fire and forget
  await startCheckout()
}
```

**Tracking PII in event properties.**
Never put email addresses, names, phone numbers, or payment data in event properties. Use your `userId` (an opaque identifier) and look up user details in your database when needed. PostHog stores event properties long-term and they're harder to purge than user profiles.

**One giant analytics file.**
Split by concern: `ga4.ts`, `posthog-client.ts`, `posthog-server.ts`, `identify.ts`, `events.ts`, `track.ts`. Import only what you need. Tree-shaking works better, and you can swap providers without touching every call site.
