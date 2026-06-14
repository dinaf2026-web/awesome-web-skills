---
name: web-i18n
description: Internationalization patterns for Next.js — next-intl setup, locale routing, translation management, RTL support, pluralization, date/number formatting, and locale detection.
origin: community
tags: [i18n, internationalization, next-intl, localization, rtl, translation, multilingual]
---

# web-i18n — Internationalization for Next.js

## 1. When to Internationalize (and the cost of doing it late)

Internationalization is architectural. Adding it after launch means touching every component, every route, every metadata block, and every database string field. The longer you wait, the more expensive the retrofit.

**Add i18n from day one if any of these are true:**
- The product will serve users in more than one language
- The target market includes Arabic, Hebrew, Persian, or Urdu speakers (RTL)
- You need region-specific formatting for dates, currencies, or numbers
- SEO matters across language markets

**What you are actually building when you internationalize:**
- A routing layer that maps `/en/...`, `/ar/...`, `/fr/...` to the same pages
- A translation pipeline that maps message keys to locale-specific strings
- A formatting layer that uses `Intl.*` APIs for dates, numbers, and plurals
- An HTML `dir` attribute system for RTL languages
- A set of SEO `hreflang` tags so search engines index the right locale

The cost of doing it late: you will rename every route, audit every hardcoded string, restructure your `app/` directory, and patch every `<head>` block. Plan for a week of pure migration work on a medium-sized app.

---

## 2. next-intl Setup (App Router, middleware, routing, messages folder)

### Installation

```bash
npm install next-intl
```

### Project structure

```
app/
  [locale]/
    layout.tsx
    page.tsx
    about/
      page.tsx
messages/
  en.json
  ar.json
  fr.json
  es.json
middleware.ts
i18n.ts
next.config.ts
```

### `next.config.ts`

```ts
import createNextIntlPlugin from 'next-intl/plugin'

const withNextIntl = createNextIntlPlugin('./i18n.ts')

const nextConfig = {
  // your other config
}

export default withNextIntl(nextConfig)
```

### `i18n.ts` — request configuration

```ts
import { getRequestConfig } from 'next-intl/server'
import { routing } from './middleware'

export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale

  if (!locale || !routing.locales.includes(locale as 'en' | 'ar' | 'fr' | 'es')) {
    locale = routing.defaultLocale
  }

  return {
    locale,
    messages: (await import(`./messages/${locale}.json`)).default
  }
})
```

### `middleware.ts`

```ts
import createMiddleware from 'next-intl/middleware'

export const routing = createMiddleware({
  locales: ['en', 'ar', 'fr', 'es'],
  defaultLocale: 'en',
  localePrefix: 'always'
})

export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico|.*\\..*).*)'
  ]
}
```

### `app/[locale]/layout.tsx`

```tsx
import { NextIntlClientProvider } from 'next-intl'
import { getMessages, getLocale } from 'next-intl/server'
import { notFound } from 'next/navigation'

const locales = ['en', 'ar', 'fr', 'es']

export default async function LocaleLayout({
  children,
  params
}: {
  children: React.ReactNode
  params: Promise<{ locale: string }>
}) {
  const { locale } = await params

  if (!locales.includes(locale)) {
    notFound()
  }

  const messages = await getMessages()
  const isRTL = ['ar', 'he', 'fa', 'ur'].includes(locale)

  return (
    <html lang={locale} dir={isRTL ? 'rtl' : 'ltr'}>
      <body>
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  )
}
```

---

## 3. Translation Files Structure (JSON format, nested keys, namespacing)

### Design principles

- Namespace by feature or page, not by component
- Keep keys flat within a namespace — avoid nesting deeper than two levels
- Use descriptive keys, not positional ones (`hero.title` not `hero.text1`)
- Include context comments in a parallel `_comments` file for translators

### `messages/en.json`

```json
{
  "common": {
    "loading": "Loading…",
    "error": "Something went wrong",
    "retry": "Try again",
    "cancel": "Cancel",
    "save": "Save",
    "delete": "Delete",
    "confirm": "Confirm",
    "back": "Back",
    "next": "Next",
    "close": "Close"
  },
  "navigation": {
    "home": "Home",
    "about": "About",
    "pricing": "Pricing",
    "blog": "Blog",
    "contact": "Contact",
    "signIn": "Sign in",
    "signOut": "Sign out",
    "dashboard": "Dashboard"
  },
  "auth": {
    "signIn": {
      "title": "Sign in to your account",
      "emailLabel": "Email address",
      "passwordLabel": "Password",
      "submit": "Sign in",
      "forgotPassword": "Forgot your password?",
      "noAccount": "Don't have an account?",
      "createAccount": "Create one"
    },
    "errors": {
      "invalidCredentials": "Invalid email or password",
      "tooManyAttempts": "Too many attempts. Try again in {minutes} minutes.",
      "sessionExpired": "Your session has expired. Please sign in again."
    }
  },
  "dashboard": {
    "welcome": "Welcome back, {name}",
    "stats": {
      "totalOrders": "Total orders",
      "revenue": "Revenue",
      "activeUsers": "Active users",
      "conversion": "Conversion rate"
    }
  },
  "products": {
    "itemCount": "{count, plural, =0 {No items} one {# item} other {# items}}",
    "addToCart": "Add to cart",
    "outOfStock": "Out of stock",
    "price": "{price, number, currency}"
  },
  "errors": {
    "notFound": {
      "title": "Page not found",
      "description": "The page you are looking for does not exist.",
      "action": "Go home"
    },
    "serverError": {
      "title": "Server error",
      "description": "We are working on fixing this. Please try again shortly."
    }
  }
}
```

### `messages/ar.json`

```json
{
  "common": {
    "loading": "جارٍ التحميل…",
    "error": "حدث خطأ ما",
    "retry": "حاول مرة أخرى",
    "cancel": "إلغاء",
    "save": "حفظ",
    "delete": "حذف",
    "confirm": "تأكيد",
    "back": "رجوع",
    "next": "التالي",
    "close": "إغلاق"
  },
  "navigation": {
    "home": "الرئيسية",
    "about": "عن الشركة",
    "pricing": "الأسعار",
    "blog": "المدونة",
    "contact": "تواصل معنا",
    "signIn": "تسجيل الدخول",
    "signOut": "تسجيل الخروج",
    "dashboard": "لوحة التحكم"
  },
  "products": {
    "itemCount": "{count, plural, =0 {لا توجد عناصر} one {عنصر واحد} two {عنصران} few {# عناصر} many {# عنصرًا} other {# عنصر}}",
    "addToCart": "أضف إلى السلة",
    "outOfStock": "نفد المخزون",
    "price": "{price, number, currency}"
  }
}
```

### Key naming conventions

```
// Page-scoped
"home.hero.title"      → Home > Hero section > title
"home.hero.subtitle"   → Home > Hero section > subtitle
"home.cta.primary"     → Home > CTA > primary button

// Feature-scoped
"auth.signIn.title"
"auth.signUp.title"
"auth.resetPassword.title"

// Shared
"common.loading"
"common.error"
"errors.notFound.title"
```

---

## 4. useTranslations in Client and Server Components

### Server Component (default — preferred)

```tsx
// app/[locale]/page.tsx
import { useTranslations } from 'next-intl'

export default function HomePage() {
  const t = useTranslations('home')

  return (
    <main>
      <h1>{t('hero.title')}</h1>
      <p>{t('hero.subtitle')}</p>
      <a href="/pricing">{t('cta.primary')}</a>
    </main>
  )
}
```

### Server Component with dynamic values

```tsx
// app/[locale]/dashboard/page.tsx
import { useTranslations } from 'next-intl'
import { getCurrentUser } from '@/lib/auth'

export default async function DashboardPage() {
  const t = useTranslations('dashboard')
  const user = await getCurrentUser()

  return (
    <div>
      <h1>{t('welcome', { name: user.firstName })}</h1>
    </div>
  )
}
```

### Client Component — must receive translations as props or use hook inside provider

```tsx
'use client'

import { useTranslations } from 'next-intl'

interface CartButtonProps {
  productId: string
  disabled?: boolean
}

export function CartButton({ productId, disabled }: CartButtonProps) {
  const t = useTranslations('products')

  return (
    <button
      disabled={disabled}
      onClick={() => addToCart(productId)}
      aria-label={t('addToCart')}
    >
      {disabled ? t('outOfStock') : t('addToCart')}
    </button>
  )
}
```

### Server Component with `getTranslations` (async)

```tsx
// Use when you need translations outside a component render
import { getTranslations } from 'next-intl/server'

export async function generateMetadata({
  params
}: {
  params: Promise<{ locale: string }>
}) {
  const { locale } = await params
  const t = await getTranslations({ locale, namespace: 'home' })

  return {
    title: t('meta.title'),
    description: t('meta.description')
  }
}
```

### Accessing multiple namespaces

```tsx
import { useTranslations } from 'next-intl'

export default function CheckoutPage() {
  const tCommon = useTranslations('common')
  const tCheckout = useTranslations('checkout')
  const tErrors = useTranslations('errors')

  return (
    <form>
      <h1>{tCheckout('title')}</h1>
      <button type="submit">{tCheckout('placeOrder')}</button>
      <button type="button">{tCommon('cancel')}</button>
    </form>
  )
}
```

---

## 5. Locale Routing (locale prefix, domain-based, automatic detection)

### Option A: Prefix-based (recommended for most apps)

```
/en/about
/ar/about
/fr/about
/es/about
```

```ts
// middleware.ts
export const routing = createMiddleware({
  locales: ['en', 'ar', 'fr', 'es'],
  defaultLocale: 'en',
  localePrefix: 'always'        // always prefix, including default locale
  // localePrefix: 'as-needed'  // omit prefix for default locale only
})
```

### Option B: Domain-based routing

```
en.example.com/about
ar.example.com/about
fr.example.com/about
```

```ts
// middleware.ts
export const routing = createMiddleware({
  locales: ['en', 'ar', 'fr'],
  defaultLocale: 'en',
  domains: [
    {
      domain: 'en.example.com',
      defaultLocale: 'en',
      locales: ['en']
    },
    {
      domain: 'ar.example.com',
      defaultLocale: 'ar',
      locales: ['ar']
    },
    {
      domain: 'fr.example.com',
      defaultLocale: 'fr',
      locales: ['fr']
    }
  ]
})
```

### Automatic locale detection from browser

next-intl's middleware reads the `Accept-Language` header automatically. To customize:

```ts
// middleware.ts
import createMiddleware from 'next-intl/middleware'
import Negotiator from 'negotiator'
import { match } from '@formatjs/intl-localematcher'

const locales = ['en', 'ar', 'fr', 'es']
const defaultLocale = 'en'

function detectLocale(request: Request): string {
  const acceptLanguage = request.headers.get('accept-language') ?? ''
  const languages = new Negotiator({
    headers: { 'accept-language': acceptLanguage }
  }).languages()

  try {
    return match(languages, locales, defaultLocale)
  } catch {
    return defaultLocale
  }
}

export const routing = createMiddleware({
  locales,
  defaultLocale,
  localeDetection: true   // default; reads Accept-Language header
})
```

### Typed navigation with next-intl

```ts
// navigation.ts — create typed navigation helpers
import { createNavigation } from 'next-intl/navigation'

export const { Link, redirect, usePathname, useRouter } = createNavigation({
  locales: ['en', 'ar', 'fr', 'es'],
  defaultLocale: 'en'
})
```

```tsx
// Use typed Link — locale is handled automatically
import { Link } from '@/navigation'

export function Nav() {
  return (
    <nav>
      <Link href="/about">About</Link>
      <Link href="/pricing">Pricing</Link>
      <Link href={{ pathname: '/products/[id]', params: { id: '42' } }}>
        Product
      </Link>
    </nav>
  )
}
```

---

## 6. Pluralization (one/other/zero forms, count-based messages)

next-intl uses ICU message format for pluralization. The `Intl.PluralRules` API drives the plural category selection per locale automatically.

### ICU plural syntax

```
{count, plural, =0 {zero form} one {singular} other {plural}}
```

### Translation file entries

```json
{
  "items": {
    "count": "{count, plural, =0 {No items} one {# item} other {# items}}",
    "selected": "{count, plural, =0 {Nothing selected} one {# item selected} other {# items selected}}",
    "remaining": "{count, plural, =0 {All done} one {# remaining} other {# remaining}}"
  },
  "notifications": {
    "unread": "{count, plural, =0 {No unread messages} one {1 unread message} other {# unread messages}}",
    "newFollowers": "{count, plural, one {# new follower} other {# new followers}}"
  },
  "time": {
    "daysAgo": "{count, plural, =0 {today} one {yesterday} other {# days ago}}",
    "hoursAgo": "{count, plural, one {# hour ago} other {# hours ago}}"
  }
}
```

### Arabic plural forms (6 categories)

```json
{
  "items": {
    "count": "{count, plural, =0 {لا توجد عناصر} one {عنصر واحد} two {عنصران} few {# عناصر} many {# عنصرًا} other {# عنصر}}"
  }
}
```

### Component usage

```tsx
import { useTranslations } from 'next-intl'

interface ItemListHeaderProps {
  count: number
  selectedCount: number
}

export function ItemListHeader({ count, selectedCount }: ItemListHeaderProps) {
  const t = useTranslations('items')

  return (
    <div className="flex items-center justify-between">
      <span>{t('count', { count })}</span>
      {selectedCount > 0 && (
        <span>{t('selected', { count: selectedCount })}</span>
      )}
    </div>
  )
}
```

### Ordinals (first, second, third…)

```json
{
  "rank": {
    "position": "{position, selectordinal, one {#st} two {#nd} few {#rd} other {#th}}"
  }
}
```

```tsx
const t = useTranslations('rank')
t('position', { position: 1 })  // "1st"
t('position', { position: 2 })  // "2nd"
t('position', { position: 11 }) // "11th"
```

---

## 7. Date, Time, and Number Formatting

### `useFormatter` hook

```tsx
'use client'

import { useFormatter, useLocale } from 'next-intl'

export function OrderSummary({ order }: { order: Order }) {
  const format = useFormatter()
  const locale = useLocale()

  return (
    <div>
      <p>
        {format.dateTime(new Date(order.createdAt), {
          year: 'numeric',
          month: 'long',
          day: 'numeric'
        })}
      </p>
      <p>
        {format.number(order.total, {
          style: 'currency',
          currency: order.currency
        })}
      </p>
      <p>{format.relativeTime(new Date(order.updatedAt))}</p>
    </div>
  )
}
```

### Server-side formatting with `getFormatter`

```tsx
import { getFormatter, getLocale } from 'next-intl/server'

export default async function InvoicePage({ invoiceId }: { invoiceId: string }) {
  const format = await getFormatter()
  const invoice = await getInvoice(invoiceId)

  const formattedDate = format.dateTime(new Date(invoice.date), {
    dateStyle: 'full'
  })

  const formattedAmount = format.number(invoice.amount, {
    style: 'currency',
    currency: 'USD',
    minimumFractionDigits: 2
  })

  return (
    <div>
      <p>Invoice date: {formattedDate}</p>
      <p>Amount due: {formattedAmount}</p>
    </div>
  )
}
```

### Date formatting patterns

```tsx
const format = useFormatter()
const date = new Date('2024-03-15T14:30:00Z')

// Short date
format.dateTime(date, { dateStyle: 'short' })
// en: "3/15/24"   ar: "١٥‏/٣‏/٢٠٢٤"   fr: "15/03/2024"

// Long date
format.dateTime(date, { dateStyle: 'long' })
// en: "March 15, 2024"   ar: "١٥ مارس ٢٠٢٤"   fr: "15 mars 2024"

// Date + time
format.dateTime(date, { dateStyle: 'medium', timeStyle: 'short' })
// en: "Mar 15, 2024, 2:30 PM"

// Relative time
format.relativeTime(new Date(Date.now() - 60 * 60 * 1000))
// en: "1 hour ago"   ar: "منذ ساعة"   fr: "il y a 1 heure"
```

### Number formatting patterns

```tsx
const format = useFormatter()

// Currency
format.number(1234.56, { style: 'currency', currency: 'EUR' })
// en: "€1,234.56"   ar: "١٬٢٣٤٫٥٦ €"   fr: "1 234,56 €"

// Percentage
format.number(0.847, { style: 'percent', maximumFractionDigits: 1 })
// en: "84.7%"   ar: "٨٤٫٧٪"   fr: "84,7 %"

// Compact (large numbers)
format.number(1_500_000, { notation: 'compact' })
// en: "1.5M"   ar: "١٫٥ مليون"   fr: "1,5 M"

// Plain with separators
format.number(9876543, { useGrouping: true })
// en: "9,876,543"   ar: "٩٬٨٧٦٬٥٤٣"   fr: "9 876 543"
```

### Locale-aware currency selection

```ts
// lib/currency.ts
const localeCurrencyMap: Record<string, string> = {
  en: 'USD',
  'en-GB': 'GBP',
  fr: 'EUR',
  de: 'EUR',
  ar: 'SAR',
  'ar-AE': 'AED',
  ja: 'JPY',
  zh: 'CNY'
}

export function getCurrencyForLocale(locale: string): string {
  return localeCurrencyMap[locale] ?? localeCurrencyMap[locale.split('-')[0]] ?? 'USD'
}
```

---

## 8. RTL Support

### HTML `dir` attribute

Set at the `<html>` element — handled in the locale layout (shown in section 2). The browser propagates direction to all descendants automatically.

### RTL-aware locale detection

```ts
// lib/locale.ts
export const RTL_LOCALES = new Set(['ar', 'he', 'fa', 'ur', 'ps', 'sd'])

export function isRTL(locale: string): boolean {
  return RTL_LOCALES.has(locale.split('-')[0])
}

export function getDirection(locale: string): 'ltr' | 'rtl' {
  return isRTL(locale) ? 'rtl' : 'ltr'
}
```

### Tailwind CSS RTL utilities

Tailwind v3+ includes `rtl:` and `ltr:` variants. Use logical properties by default.

```tsx
// Avoid physical properties — these break RTL
<div className="ml-4 text-left border-l-2">...</div>

// Use logical properties — work correctly in both directions
<div className="ms-4 text-start border-s-2">...</div>
```

**Tailwind logical property mapping:**

| Physical | Logical | Tailwind class |
|----------|---------|----------------|
| `margin-left` | `margin-inline-start` | `ms-4` |
| `margin-right` | `margin-inline-end` | `me-4` |
| `padding-left` | `padding-inline-start` | `ps-4` |
| `padding-right` | `padding-inline-end` | `pe-4` |
| `text-left` | `text-start` | `text-start` |
| `text-right` | `text-end` | `text-end` |
| `border-left` | `border-inline-start` | `border-s-2` |
| `border-right` | `border-inline-end` | `border-e-2` |
| `left-0` | `inset-inline-start: 0` | `start-0` |
| `right-0` | `inset-inline-end: 0` | `end-0` |
| `rounded-l-*` | `rounded-s-*` | `rounded-s-md` |
| `rounded-r-*` | `rounded-e-*` | `rounded-e-md` |

### RTL-specific overrides with `rtl:` variant

```tsx
// Use rtl: variant only when logical properties cannot express the intent
<div className="flex flex-row rtl:flex-row-reverse">
  <Icon />
  <span>Label</span>
</div>

// Icon that should mirror in RTL (arrows, chevrons)
<ChevronRightIcon className="rtl:rotate-180 transition-transform" />

// Dropdown that opens to the right in LTR, left in RTL
<ul className="absolute start-0 top-full">
  {items.map(item => <li key={item.id}>{item.label}</li>)}
</ul>
```

### CSS logical properties in plain CSS

```css
.card {
  padding-inline: 1.5rem;       /* horizontal padding, direction-aware */
  padding-block: 1rem;          /* vertical padding */
  margin-inline-start: 1rem;    /* left in LTR, right in RTL */
  border-inline-start: 2px solid currentColor;
  text-align: start;            /* left in LTR, right in RTL */
}

.icon-before {
  margin-inline-end: 0.5rem;    /* gap between icon and text */
}
```

### Font considerations for RTL

```tsx
// app/[locale]/layout.tsx
import { Cairo } from 'next/font/google'  // Arabic font
import { Inter } from 'next/font/google'

const inter = Inter({ subsets: ['latin'] })
const cairo = Cairo({ subsets: ['arabic'] })

export default async function LocaleLayout({ children, params }) {
  const { locale } = await params
  const isRTL = ['ar', 'he', 'fa', 'ur'].includes(locale)
  const fontClass = isRTL ? cairo.className : inter.className

  return (
    <html lang={locale} dir={isRTL ? 'rtl' : 'ltr'}>
      <body className={fontClass}>
        {children}
      </body>
    </html>
  )
}
```

---

## 9. Dynamic Content Translation

### Pattern 1: Database-stored translations

```ts
// schema (Prisma example)
model Product {
  id          String               @id
  slug        String               @unique
  translations ProductTranslation[]
}

model ProductTranslation {
  id          String  @id
  productId   String
  locale      String  // "en" | "ar" | "fr"
  name        String
  description String
  product     Product @relation(fields: [productId], references: [id])

  @@unique([productId, locale])
}
```

```ts
// lib/products.ts
import { getLocale } from 'next-intl/server'
import { db } from './db'

export async function getProduct(slug: string) {
  const locale = await getLocale()

  const product = await db.product.findUnique({
    where: { slug },
    include: {
      translations: {
        where: { locale },
        take: 1
      }
    }
  })

  if (!product) return null

  // Fall back to English if requested locale is not available
  const translation = product.translations[0]
    ?? await db.productTranslation.findFirst({
      where: { productId: product.id, locale: 'en' }
    })

  return {
    id: product.id,
    slug: product.slug,
    name: translation?.name ?? '',
    description: translation?.description ?? ''
  }
}
```

### Pattern 2: API-fetched content with locale parameter

```ts
// lib/cms.ts
import { getLocale } from 'next-intl/server'

interface CMSPost {
  id: string
  title: string
  body: string
  locale: string
}

export async function fetchPost(slug: string): Promise<CMSPost | null> {
  const locale = await getLocale()

  const res = await fetch(
    `${process.env.CMS_URL}/posts/${slug}?locale=${locale}`,
    {
      headers: { Authorization: `Bearer ${process.env.CMS_TOKEN}` },
      next: { revalidate: 3600 }
    }
  )

  if (!res.ok) return null
  return res.json()
}
```

### Pattern 3: User-generated content with machine translation fallback

```ts
// lib/translate.ts
interface TranslationCache {
  [key: string]: { [locale: string]: string }
}

const cache: TranslationCache = {}

export async function translateUserContent(
  text: string,
  targetLocale: string,
  sourceLocale = 'en'
): Promise<string> {
  if (targetLocale === sourceLocale) return text

  const cacheKey = `${text}:${targetLocale}`
  if (cache[cacheKey]?.[targetLocale]) return cache[cacheKey][targetLocale]

  // Call your translation API (DeepL, Google Translate, etc.)
  const translated = await callTranslationAPI(text, sourceLocale, targetLocale)

  cache[cacheKey] = { [targetLocale]: translated }
  return translated
}

async function callTranslationAPI(
  text: string,
  source: string,
  target: string
): Promise<string> {
  const res = await fetch('https://api.deepl.com/v2/translate', {
    method: 'POST',
    headers: {
      'Authorization': `DeepL-Auth-Key ${process.env.DEEPL_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      text: [text],
      source_lang: source.toUpperCase(),
      target_lang: target.toUpperCase()
    })
  })

  const data = await res.json()
  return data.translations[0]?.text ?? text
}
```

---

## 10. Language Switcher Component

### `LanguageSwitcher` with path preservation

```tsx
'use client'

import { useLocale, useTranslations } from 'next-intl'
import { usePathname, useRouter } from '@/navigation'
import { useTransition } from 'react'

const LOCALES = [
  { code: 'en', label: 'English', flag: '🇺🇸' },
  { code: 'ar', label: 'العربية', flag: '🇸🇦' },
  { code: 'fr', label: 'Français', flag: '🇫🇷' },
  { code: 'es', label: 'Español', flag: '🇪🇸' }
] as const

type LocaleCode = typeof LOCALES[number]['code']

export function LanguageSwitcher() {
  const locale = useLocale()
  const router = useRouter()
  const pathname = usePathname()
  const [isPending, startTransition] = useTransition()
  const t = useTranslations('common')

  function switchLocale(nextLocale: LocaleCode) {
    if (nextLocale === locale) return

    startTransition(() => {
      router.replace(pathname, { locale: nextLocale })
    })
  }

  return (
    <div className="relative">
      <select
        value={locale}
        onChange={(e) => switchLocale(e.target.value as LocaleCode)}
        disabled={isPending}
        aria-label={t('selectLanguage')}
        className="
          appearance-none
          rounded-md
          border border-neutral-200
          bg-white
          px-3 py-1.5
          pe-8
          text-sm
          focus:outline-none focus:ring-2 focus:ring-blue-500
          disabled:opacity-50
          dark:border-neutral-700
          dark:bg-neutral-900
        "
      >
        {LOCALES.map(({ code, label, flag }) => (
          <option key={code} value={code}>
            {flag} {label}
          </option>
        ))}
      </select>
      <span className="pointer-events-none absolute end-2 top-1/2 -translate-y-1/2 text-neutral-400">
        ▾
      </span>
    </div>
  )
}
```

### Dropdown variant with flags and native names

```tsx
'use client'

import { useLocale } from 'next-intl'
import { usePathname, useRouter } from '@/navigation'
import { useTransition, useState, useRef, useEffect } from 'react'

const LOCALES = [
  { code: 'en', nativeName: 'English', flag: '🇺🇸', dir: 'ltr' },
  { code: 'ar', nativeName: 'العربية', flag: '🇸🇦', dir: 'rtl' },
  { code: 'fr', nativeName: 'Français', flag: '🇫🇷', dir: 'ltr' },
  { code: 'es', nativeName: 'Español', flag: '🇪🇸', dir: 'ltr' }
] as const

type LocaleCode = typeof LOCALES[number]['code']

export function LocaleSwitcher() {
  const locale = useLocale()
  const router = useRouter()
  const pathname = usePathname()
  const [isPending, startTransition] = useTransition()
  const [isOpen, setIsOpen] = useState(false)
  const ref = useRef<HTMLDivElement>(null)

  const current = LOCALES.find(l => l.code === locale)!

  useEffect(() => {
    function handleClickOutside(e: MouseEvent) {
      if (ref.current && !ref.current.contains(e.target as Node)) {
        setIsOpen(false)
      }
    }
    document.addEventListener('mousedown', handleClickOutside)
    return () => document.removeEventListener('mousedown', handleClickOutside)
  }, [])

  function switchLocale(nextLocale: LocaleCode) {
    setIsOpen(false)
    if (nextLocale === locale) return

    startTransition(() => {
      router.replace(pathname, { locale: nextLocale })
    })
  }

  return (
    <div ref={ref} className="relative">
      <button
        onClick={() => setIsOpen(!isOpen)}
        disabled={isPending}
        className="
          flex items-center gap-2
          rounded-md
          border border-neutral-200
          px-3 py-1.5
          text-sm
          hover:bg-neutral-50
          focus:outline-none focus:ring-2 focus:ring-blue-500
          disabled:opacity-50
        "
        aria-haspopup="listbox"
        aria-expanded={isOpen}
      >
        <span aria-hidden="true">{current.flag}</span>
        <span>{current.nativeName}</span>
        <span aria-hidden="true" className={`transition-transform ${isOpen ? 'rotate-180' : ''}`}>▾</span>
      </button>

      {isOpen && (
        <ul
          role="listbox"
          className="
            absolute
            start-0
            top-full
            z-50
            mt-1
            w-48
            rounded-md
            border border-neutral-200
            bg-white
            py-1
            shadow-lg
            dark:border-neutral-700
            dark:bg-neutral-900
          "
        >
          {LOCALES.map(({ code, nativeName, flag, dir }) => (
            <li key={code} role="option" aria-selected={code === locale}>
              <button
                onClick={() => switchLocale(code)}
                dir={dir}
                className={`
                  flex w-full items-center gap-2
                  px-3 py-2
                  text-sm
                  hover:bg-neutral-100
                  dark:hover:bg-neutral-800
                  ${code === locale ? 'font-medium text-blue-600' : ''}
                `}
              >
                <span aria-hidden="true">{flag}</span>
                <span>{nativeName}</span>
                {code === locale && (
                  <span className="ms-auto text-blue-600" aria-hidden="true">✓</span>
                )}
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

---

## 11. SEO for i18n

### `generateMetadata` with locale

```tsx
// app/[locale]/about/page.tsx
import { getTranslations } from 'next-intl/server'
import type { Metadata } from 'next'

const BASE_URL = process.env.NEXT_PUBLIC_BASE_URL ?? 'https://example.com'
const LOCALES = ['en', 'ar', 'fr', 'es']

export async function generateMetadata({
  params
}: {
  params: Promise<{ locale: string }>
}): Promise<Metadata> {
  const { locale } = await params
  const t = await getTranslations({ locale, namespace: 'about.meta' })

  return {
    title: t('title'),
    description: t('description'),
    alternates: {
      canonical: `${BASE_URL}/${locale}/about`,
      languages: Object.fromEntries(
        LOCALES.map(l => [l, `${BASE_URL}/${l}/about`])
      )
    },
    openGraph: {
      title: t('title'),
      description: t('description'),
      locale,
      alternateLocale: LOCALES.filter(l => l !== locale)
    }
  }
}
```

### `hreflang` link tags via `<Head>` or metadata API

The `alternates.languages` object in Next.js 13+ `Metadata` generates `<link rel="alternate" hreflang="...">` tags automatically.

```tsx
// Produces:
// <link rel="alternate" hreflang="en" href="https://example.com/en/about" />
// <link rel="alternate" hreflang="ar" href="https://example.com/ar/about" />
// <link rel="alternate" hreflang="fr" href="https://example.com/fr/about" />
// <link rel="alternate" hreflang="es" href="https://example.com/es/about" />
// <link rel="alternate" hreflang="x-default" href="https://example.com/en/about" />
```

### Including `x-default` for the default locale

```tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  const { locale } = await params

  return {
    alternates: {
      canonical: `${BASE_URL}/${locale}/about`,
      languages: {
        'x-default': `${BASE_URL}/en/about`,
        ...Object.fromEntries(
          LOCALES.map(l => [l, `${BASE_URL}/${l}/about`])
        )
      }
    }
  }
}
```

### Sitemap with all locales

```ts
// app/sitemap.ts
import { MetadataRoute } from 'next'

const BASE_URL = process.env.NEXT_PUBLIC_BASE_URL ?? 'https://example.com'
const LOCALES = ['en', 'ar', 'fr', 'es']

const STATIC_ROUTES = ['', '/about', '/pricing', '/blog', '/contact']

export default function sitemap(): MetadataRoute.Sitemap {
  const entries: MetadataRoute.Sitemap = []

  for (const route of STATIC_ROUTES) {
    for (const locale of LOCALES) {
      entries.push({
        url: `${BASE_URL}/${locale}${route}`,
        lastModified: new Date(),
        changeFrequency: 'weekly',
        priority: route === '' ? 1.0 : 0.8,
        alternates: {
          languages: Object.fromEntries(
            LOCALES.map(l => [l, `${BASE_URL}/${l}${route}`])
          )
        }
      })
    }
  }

  return entries
}
```

---

## 12. Translation Management

### Option A: Simple JSON workflow (small teams, ≤5 locales)

```
messages/
  en.json          ← source of truth
  ar.json
  fr.json
  es.json
  _template.json   ← key reference with empty values (for translators)
  _missing.ts      ← script to find missing keys
```

```ts
// scripts/find-missing-keys.ts
import en from '../messages/en.json'
import ar from '../messages/ar.json'
import fr from '../messages/fr.json'

function getKeys(obj: object, prefix = ''): string[] {
  return Object.entries(obj).flatMap(([key, value]) => {
    const fullKey = prefix ? `${prefix}.${key}` : key
    return typeof value === 'object' && value !== null
      ? getKeys(value, fullKey)
      : [fullKey]
  })
}

const enKeys = new Set(getKeys(en))

for (const [locale, messages] of [['ar', ar], ['fr', fr]] as const) {
  const localeKeys = new Set(getKeys(messages))
  const missing = [...enKeys].filter(k => !localeKeys.has(k))

  if (missing.length > 0) {
    console.log(`\n[${locale}] Missing ${missing.length} keys:`)
    missing.forEach(k => console.log(`  ${k}`))
  } else {
    console.log(`\n[${locale}] Complete ✓`)
  }
}
```

### Option B: Crowdin integration

```yaml
# crowdin.yml
project_id: "your-project-id"
api_token_env: CROWDIN_PERSONAL_TOKEN
preserve_hierarchy: true
base_path: "."
base_url: "https://api.crowdin.com"

files:
  - source: /messages/en.json
    translation: /messages/%locale%.json
    type: i18next_multivalue_json
    update_option: update_as_unapproved
```

```bash
# Pull latest translations
npx crowdin download

# Push source strings
npx crowdin upload sources

# Check translation progress
npx crowdin status
```

### Option C: Lokalise workflow

```ts
// scripts/sync-translations.ts
import { LokaliseApi } from '@lokalise/node-api'
import { readFileSync, writeFileSync } from 'fs'

const lokalise = new LokaliseApi({ apiKey: process.env.LOKALISE_API_KEY! })
const PROJECT_ID = process.env.LOKALISE_PROJECT_ID!

async function pullTranslations() {
  const response = await lokalise.files().download(PROJECT_ID, {
    format: 'json',
    original_filenames: false,
    bundle_structure: '%LANG_ISO%.json',
    export_empty_as: 'base'
  })

  console.log('Download URL:', response.bundle_url)
  // Unzip and copy to messages/
}

async function pushSourceStrings() {
  const source = readFileSync('./messages/en.json', 'utf-8')

  await lokalise.files().upload(PROJECT_ID, {
    data: Buffer.from(source).toString('base64'),
    filename: 'en.json',
    lang_iso: 'en'
  })

  console.log('Uploaded en.json to Lokalise')
}
```

### Translation review checklist before shipping a new locale

- [ ] All keys from `en.json` exist in the target locale file
- [ ] Pluralization rules are correct for that language's plural categories
- [ ] RTL text direction is correct if applicable
- [ ] Numbers, dates, and currencies format correctly
- [ ] Character encoding is UTF-8
- [ ] Long strings do not overflow UI containers (test at 200% text length)
- [ ] Font supports all required characters (Arabic, CJK, etc.)

---

## 13. Anti-Patterns

### Anti-pattern 1: Hardcoded strings

```tsx
// WRONG
export function Header() {
  return (
    <header>
      <h1>Welcome to our store</h1>
      <nav>
        <a href="/about">About</a>
        <a href="/contact">Contact us</a>
      </nav>
    </header>
  )
}

// CORRECT
export function Header() {
  const t = useTranslations('navigation')
  const tHome = useTranslations('home')

  return (
    <header>
      <h1>{tHome('title')}</h1>
      <nav>
        <Link href="/about">{t('about')}</Link>
        <Link href="/contact">{t('contact')}</Link>
      </nav>
    </header>
  )
}
```

### Anti-pattern 2: Concatenating translated strings

```tsx
// WRONG — word order differs between languages
const t = useTranslations()
const message = t('youHave') + ' ' + count + ' ' + t('messages')

// CORRECT — use ICU interpolation
// messages/en.json: { "inbox": "You have {count} messages" }
// messages/ar.json: { "inbox": "لديك {count} رسائل" }
const message = t('inbox', { count })
```

### Anti-pattern 3: No RTL testing

```tsx
// WRONG — never tested in RTL
<div className="flex items-center gap-2">
  <img src={avatar} className="rounded-full w-8 h-8 mr-3" />
  <span className="text-left">{user.name}</span>
</div>

// CORRECT — logical properties, tested in both directions
<div className="flex items-center gap-2">
  <img src={avatar} className="rounded-full w-8 h-8 me-3" />
  <span className="text-start">{user.name}</span>
</div>
```

### Anti-pattern 4: Missing fallback locale

```ts
// WRONG — crashes if locale file is missing a key
const t = useTranslations('products')
return t('someNewKey')  // throws if not in ar.json

// CORRECT — next-intl falls back to defaultLocale automatically
// Ensure i18n.ts is configured with a fallback:
export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale
  if (!locale || !locales.includes(locale)) {
    locale = defaultLocale  // ← always fall back
  }
  return { locale, messages: await import(`./messages/${locale}.json`) }
})
```

### Anti-pattern 5: Using browser locale for server rendering

```tsx
// WRONG — breaks SSR, hydration mismatch
'use client'
export function PriceDisplay({ amount }: { amount: number }) {
  const locale = navigator.language  // undefined on server
  return <span>{new Intl.NumberFormat(locale).format(amount)}</span>
}

// CORRECT — use next-intl's useFormatter which reads the server-set locale
'use client'
import { useFormatter } from 'next-intl'

export function PriceDisplay({ amount }: { amount: number }) {
  const format = useFormatter()
  return <span>{format.number(amount, { style: 'currency', currency: 'USD' })}</span>
}
```

### Anti-pattern 6: Locale in component state

```tsx
// WRONG — locale stored in component state, lost on navigation
const [locale, setLocale] = useState('en')

// CORRECT — locale is part of the URL, managed by the router
const router = useRouter()
const pathname = usePathname()
startTransition(() => {
  router.replace(pathname, { locale: nextLocale })
})
```

### Anti-pattern 7: Not generating `hreflang` tags

```tsx
// WRONG — search engines have no signal for locale variants
export async function generateMetadata() {
  return { title: 'About Us' }
}

// CORRECT — always include alternates
export async function generateMetadata({ params }) {
  const { locale } = await params
  return {
    title: t('meta.title'),
    alternates: {
      languages: Object.fromEntries(
        LOCALES.map(l => [l, `${BASE_URL}/${l}/about`])
      )
    }
  }
}
```

### Anti-pattern 8: One massive translation file

```
// WRONG
messages/
  en.json   ← 4000 lines, everything in one file

// CORRECT — split by domain, merge at build time if needed
messages/
  en/
    common.json
    auth.json
    products.json
    dashboard.json
    errors.json
```

```ts
// i18n.ts — merge namespace files
export default getRequestConfig(async ({ requestLocale }) => {
  const locale = (await requestLocale) ?? 'en'

  const [common, auth, products, dashboard, errors] = await Promise.all([
    import(`./messages/${locale}/common.json`),
    import(`./messages/${locale}/auth.json`),
    import(`./messages/${locale}/products.json`),
    import(`./messages/${locale}/dashboard.json`),
    import(`./messages/${locale}/errors.json`)
  ])

  return {
    locale,
    messages: {
      common: common.default,
      auth: auth.default,
      products: products.default,
      dashboard: dashboard.default,
      errors: errors.default
    }
  }
})
```
