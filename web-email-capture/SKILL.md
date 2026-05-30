---
name: web-email-capture
description: Email capture and newsletter integration patterns — opt-in forms, Resend transactional email, Beehiiv/Mailchimp newsletter sync, drip sequence wiring, and conversion-optimized signup flows.
origin: community
tags: [email, resend, beehiiv, mailchimp, newsletter, drip, opt-in, conversion]
---

# web-email-capture

Email capture and newsletter integration for Next.js 14 App Router. Covers Resend transactional email, Beehiiv/Mailchimp sync, double opt-in, drip sequences, popup capture, and conversion analytics.

---

## 1. When to Use Which

| Need | Tool | Why |
|------|------|-----|
| Transactional email (welcome, receipts, password reset) | **Resend** | Developer-first, React Email templates, reliable delivery |
| Newsletter + paid subscriptions | **Beehiiv** | Built-in monetization, referral program, analytics |
| Marketing automation + CRM | **Mailchimp** | Audience segmentation, A/B testing, large ecosystem |
| Creator-focused sequences + commerce | **ConvertKit** | Tag-based automation, creator tools, Stripe integration |
| Simple drip with code control | **Resend Broadcasts** | No extra vendor, full control, cheap |

**Decision rule:**
- Start with Resend for all transactional.
- Add Beehiiv if you want a hosted newsletter with growth tools.
- Add Mailchimp only if you need heavy segmentation or existing list.
- Use ConvertKit if you sell digital products to your list.
- Never use two newsletter platforms simultaneously — sync is fragile.

---

## 2. Resend Setup

```bash
npm install resend
```

```ts
// lib/resend.ts
import { Resend } from 'resend'

if (!process.env.RESEND_API_KEY) {
  throw new Error('RESEND_API_KEY is required')
}

export const resend = new Resend(process.env.RESEND_API_KEY)
```

```ts
// .env.local
RESEND_API_KEY=re_xxxxxxxxxxxxxxxxxxxx
RESEND_FROM_EMAIL=hello@yourdomain.com
RESEND_FROM_NAME=Your Brand
```

**Domain verification** — add these DNS records in your registrar:

```
Type   Name                          Value
TXT    resend._domainkey.yourdomain  (Resend provides this in dashboard)
MX     send.yourdomain               feedback-smtp.us-east-1.amazonses.com
TXT    @                             v=spf1 include:amazonses.com ~all
```

Verify ownership via the Resend dashboard under Domains. Transactional email requires a verified domain — never send from a free Gmail/Yahoo address.

---

## 3. Welcome Email Template

```bash
npm install @react-email/components @react-email/render
```

```tsx
// emails/WelcomeEmail.tsx
import {
  Body,
  Button,
  Container,
  Head,
  Heading,
  Hr,
  Html,
  Img,
  Link,
  Preview,
  Section,
  Text,
} from '@react-email/components'

interface WelcomeEmailProps {
  firstName: string
  confirmUrl?: string
  unsubscribeUrl: string
}

export function WelcomeEmail({
  firstName,
  confirmUrl,
  unsubscribeUrl,
}: WelcomeEmailProps) {
  return (
    <Html lang="en" dir="ltr">
      <Head />
      <Preview>Welcome — confirm your email to get started</Preview>
      <Body style={body}>
        <Container style={container}>
          <Section style={logoSection}>
            <Img
              src="https://yourdomain.com/logo.png"
              width="120"
              height="40"
              alt="Your Brand"
            />
          </Section>

          <Heading style={h1}>Welcome, {firstName}</Heading>

          <Text style={text}>
            Thanks for joining. You are one step away — confirm your email
            address so we can send you the good stuff.
          </Text>

          {confirmUrl && (
            <Section style={buttonSection}>
              <Button style={button} href={confirmUrl}>
                Confirm my email
              </Button>
            </Section>
          )}

          <Text style={mutedText}>
            Button not working?{' '}
            <Link href={confirmUrl} style={link}>
              Copy this link
            </Link>
          </Text>

          <Hr style={hr} />

          <Text style={footer}>
            You received this because you signed up at yourdomain.com.{' '}
            <Link href={unsubscribeUrl} style={link}>
              Unsubscribe
            </Link>
          </Text>
        </Container>
      </Body>
    </Html>
  )
}

// Inline styles — required for email clients. No Tailwind here.
const body: React.CSSProperties = {
  backgroundColor: '#f6f9fc',
  fontFamily:
    '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
  margin: 0,
  padding: 0,
}

const container: React.CSSProperties = {
  backgroundColor: '#ffffff',
  border: '1px solid #e6ebf1',
  borderRadius: '8px',
  margin: '40px auto',
  maxWidth: '560px',
  padding: '40px',
}

const logoSection: React.CSSProperties = {
  marginBottom: '32px',
}

const h1: React.CSSProperties = {
  color: '#1a1a1a',
  fontSize: '24px',
  fontWeight: '700',
  lineHeight: '1.3',
  margin: '0 0 16px',
}

const text: React.CSSProperties = {
  color: '#374151',
  fontSize: '16px',
  lineHeight: '1.6',
  margin: '0 0 24px',
}

const mutedText: React.CSSProperties = {
  ...text,
  color: '#6b7280',
  fontSize: '14px',
}

const buttonSection: React.CSSProperties = {
  margin: '0 0 24px',
  textAlign: 'center',
}

const button: React.CSSProperties = {
  backgroundColor: '#2563eb',
  borderRadius: '6px',
  color: '#ffffff',
  display: 'inline-block',
  fontSize: '16px',
  fontWeight: '600',
  padding: '12px 28px',
  textDecoration: 'none',
}

const hr: React.CSSProperties = {
  borderColor: '#e6ebf1',
  margin: '32px 0',
}

const footer: React.CSSProperties = {
  color: '#9ca3af',
  fontSize: '12px',
  lineHeight: '1.5',
}

const link: React.CSSProperties = {
  color: '#2563eb',
}
```

```ts
// lib/send-welcome.ts
import { render } from '@react-email/render'
import { resend } from './resend'
import { WelcomeEmail } from '@/emails/WelcomeEmail'

export async function sendWelcomeEmail({
  to,
  firstName,
  confirmToken,
}: {
  to: string
  firstName: string
  confirmToken: string
}) {
  const confirmUrl = `${process.env.NEXT_PUBLIC_BASE_URL}/confirm?token=${confirmToken}`
  const unsubscribeUrl = `${process.env.NEXT_PUBLIC_BASE_URL}/unsubscribe?email=${encodeURIComponent(to)}`

  const html = await render(
    WelcomeEmail({ firstName, confirmUrl, unsubscribeUrl }) as React.ReactElement
  )

  const { data, error } = await resend.emails.send({
    from: `${process.env.RESEND_FROM_NAME} <${process.env.RESEND_FROM_EMAIL}>`,
    to,
    subject: 'Confirm your email',
    html,
    headers: {
      'List-Unsubscribe': `<${unsubscribeUrl}>`,
      'List-Unsubscribe-Post': 'List-Unsubscribe=One-Click',
    },
  })

  if (error) {
    throw new Error(`Resend error: ${error.message}`)
  }

  return data
}
```

---

## 4. Opt-In Form (Server Action + Double Opt-In)

```ts
// lib/db/subscribers.ts
// Assumes Prisma — adapt to your ORM/DB
import { prisma } from '@/lib/prisma'
import crypto from 'crypto'

export async function createPendingSubscriber({
  email,
  firstName,
  source,
}: {
  email: string
  firstName: string
  source: string
}) {
  const token = crypto.randomBytes(32).toString('hex')
  const expiresAt = new Date(Date.now() + 24 * 60 * 60 * 1000) // 24h

  const subscriber = await prisma.subscriber.upsert({
    where: { email },
    update: {
      confirmToken: token,
      confirmTokenExpiresAt: expiresAt,
      status: 'PENDING',
      firstName,
    },
    create: {
      email,
      firstName,
      confirmToken: token,
      confirmTokenExpiresAt: expiresAt,
      status: 'PENDING',
      source,
    },
  })

  return { subscriber, token }
}

export async function confirmSubscriber(token: string) {
  const subscriber = await prisma.subscriber.findFirst({
    where: {
      confirmToken: token,
      confirmTokenExpiresAt: { gt: new Date() },
      status: 'PENDING',
    },
  })

  if (!subscriber) {
    return null
  }

  return prisma.subscriber.update({
    where: { id: subscriber.id },
    data: {
      status: 'ACTIVE',
      confirmedAt: new Date(),
      confirmToken: null,
      confirmTokenExpiresAt: null,
    },
  })
}
```

```ts
// app/actions/subscribe.ts
'use server'

import { z } from 'zod'
import { createPendingSubscriber } from '@/lib/db/subscribers'
import { sendWelcomeEmail } from '@/lib/send-welcome'

const schema = z.object({
  email: z.string().email('Enter a valid email address'),
  firstName: z.string().min(1, 'First name is required').max(50),
  source: z.string().optional().default('unknown'),
})

type SubscribeState = {
  success: boolean
  error?: string
  fieldErrors?: Record<string, string[]>
}

export async function subscribeAction(
  _prev: SubscribeState,
  formData: FormData
): Promise<SubscribeState> {
  const raw = {
    email: formData.get('email'),
    firstName: formData.get('firstName'),
    source: formData.get('source'),
  }

  const parsed = schema.safeParse(raw)

  if (!parsed.success) {
    return {
      success: false,
      fieldErrors: parsed.error.flatten().fieldErrors,
    }
  }

  const { email, firstName, source } = parsed.data

  try {
    const { token } = await createPendingSubscriber({ email, firstName, source })

    await sendWelcomeEmail({ to: email, firstName, confirmToken: token })

    return { success: true }
  } catch (err) {
    console.error('Subscribe error:', err)
    return { success: false, error: 'Something went wrong. Please try again.' }
  }
}
```

```tsx
// components/SubscribeForm.tsx
'use client'

import { useActionState } from 'react'
import { subscribeAction } from '@/app/actions/subscribe'

const initialState = { success: false }

interface SubscribeFormProps {
  source?: string
  className?: string
}

export function SubscribeForm({ source = 'inline', className }: SubscribeFormProps) {
  const [state, action, isPending] = useActionState(subscribeAction, initialState)

  if (state.success) {
    return (
      <div className={className} role="status">
        <p className="text-green-700 font-medium">
          Check your inbox — confirm your email to complete signup.
        </p>
      </div>
    )
  }

  return (
    <form action={action} className={className} noValidate>
      <input type="hidden" name="source" value={source} />

      <div className="space-y-3">
        <div>
          <label htmlFor="firstName" className="block text-sm font-medium text-gray-700 mb-1">
            First name
          </label>
          <input
            id="firstName"
            name="firstName"
            type="text"
            autoComplete="given-name"
            required
            disabled={isPending}
            className="w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:opacity-50"
          />
          {state.fieldErrors?.firstName && (
            <p className="mt-1 text-xs text-red-600" role="alert">
              {state.fieldErrors.firstName[0]}
            </p>
          )}
        </div>

        <div>
          <label htmlFor="email" className="block text-sm font-medium text-gray-700 mb-1">
            Email address
          </label>
          <input
            id="email"
            name="email"
            type="email"
            autoComplete="email"
            required
            disabled={isPending}
            className="w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:opacity-50"
          />
          {state.fieldErrors?.email && (
            <p className="mt-1 text-xs text-red-600" role="alert">
              {state.fieldErrors.email[0]}
            </p>
          )}
        </div>

        {state.error && (
          <p className="text-sm text-red-600" role="alert">
            {state.error}
          </p>
        )}

        <button
          type="submit"
          disabled={isPending}
          className="w-full rounded-md bg-blue-600 px-4 py-2 text-sm font-semibold text-white hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          {isPending ? 'Subscribing…' : 'Get free access'}
        </button>

        <p className="text-xs text-gray-500 text-center">
          No spam. Unsubscribe anytime.
        </p>
      </div>
    </form>
  )
}
```

```ts
// app/confirm/route.ts — double opt-in confirmation endpoint
import { type NextRequest, NextResponse } from 'next/server'
import { confirmSubscriber } from '@/lib/db/subscribers'

export async function GET(request: NextRequest) {
  const token = request.nextUrl.searchParams.get('token')

  if (!token) {
    return NextResponse.redirect(new URL('/confirmed?error=missing', request.url))
  }

  const subscriber = await confirmSubscriber(token)

  if (!subscriber) {
    return NextResponse.redirect(new URL('/confirmed?error=invalid', request.url))
  }

  // Optionally sync to Beehiiv/Mailchimp here after confirmation
  await syncToNewsletter(subscriber.email, subscriber.firstName ?? '')

  return NextResponse.redirect(new URL('/confirmed', request.url))
}

async function syncToNewsletter(email: string, firstName: string) {
  // Call Beehiiv or Mailchimp sync — see sections 5 and 6
}
```

---

## 5. Beehiiv Integration

```ts
// lib/beehiiv.ts
const BEEHIIV_API_KEY = process.env.BEEHIIV_API_KEY
const BEEHIIV_PUBLICATION_ID = process.env.BEEHIIV_PUBLICATION_ID

if (!BEEHIIV_API_KEY || !BEEHIIV_PUBLICATION_ID) {
  throw new Error('BEEHIIV_API_KEY and BEEHIIV_PUBLICATION_ID are required')
}

const BASE_URL = `https://api.beehiiv.com/v2/publications/${BEEHIIV_PUBLICATION_ID}`

interface BeehiivSubscribeOptions {
  email: string
  firstName?: string
  tags?: string[]
  utmSource?: string
  utmMedium?: string
  utmCampaign?: string
  referringSite?: string
}

export async function beehiivSubscribe({
  email,
  firstName,
  tags = [],
  utmSource,
  utmMedium,
  utmCampaign,
  referringSite,
}: BeehiivSubscribeOptions) {
  const body: Record<string, unknown> = {
    email,
    reactivate_existing: false,
    send_welcome_email: false, // We send our own via Resend
    double_opt_override: 'disabled', // We handle double opt-in ourselves
  }

  if (firstName) {
    body.custom_fields = [{ name: 'First Name', value: firstName }]
  }

  if (tags.length > 0) {
    body.tags = tags
  }

  if (utmSource) {
    body.utm_source = utmSource
    body.utm_medium = utmMedium
    body.utm_campaign = utmCampaign
    body.referring_site = referringSite
  }

  const response = await fetch(`${BASE_URL}/subscriptions`, {
    method: 'POST',
    headers: {
      Accept: 'application/json',
      'Content-Type': 'application/json',
      Authorization: `Bearer ${BEEHIIV_API_KEY}`,
    },
    body: JSON.stringify(body),
  })

  if (!response.ok) {
    const text = await response.text()
    throw new Error(`Beehiiv subscribe failed ${response.status}: ${text}`)
  }

  return response.json()
}

export async function beehiivAddTags(email: string, tags: string[]) {
  // First look up subscriber ID
  const searchRes = await fetch(
    `${BASE_URL}/subscriptions?email=${encodeURIComponent(email)}&limit=1`,
    {
      headers: {
        Authorization: `Bearer ${BEEHIIV_API_KEY}`,
        Accept: 'application/json',
      },
    }
  )

  if (!searchRes.ok) return

  const { data } = await searchRes.json()
  const subscriber = data?.[0]
  if (!subscriber) return

  await fetch(`${BASE_URL}/subscriptions/${subscriber.id}`, {
    method: 'PATCH',
    headers: {
      Accept: 'application/json',
      'Content-Type': 'application/json',
      Authorization: `Bearer ${BEEHIIV_API_KEY}`,
    },
    body: JSON.stringify({ tags }),
  })
}
```

**Embed vs custom form:**

```tsx
// Option A: Beehiiv embed (fast, no backend needed)
export function BeehiivEmbed() {
  return (
    <iframe
      src="https://embeds.beehiiv.com/YOUR_EMBED_ID"
      data-test-id="beehiiv-embed"
      width="100%"
      style={{ height: '52px', margin: 0, borderRadius: '0px' }}
      frameBorder="0"
      scrolling="no"
    />
  )
}

// Option B: Custom form syncing to Beehiiv (full control)
// Use SubscribeForm from section 4, then call beehiivSubscribe in
// confirmSubscriber callback or the Server Action.
```

---

## 6. Mailchimp Integration

```bash
npm install @mailchimp/mailchimp_marketing
```

```ts
// lib/mailchimp.ts
import mailchimp from '@mailchimp/mailchimp_marketing'
import crypto from 'crypto'

mailchimp.setConfig({
  apiKey: process.env.MAILCHIMP_API_KEY!,
  server: process.env.MAILCHIMP_SERVER_PREFIX!, // e.g. "us21"
})

const LIST_ID = process.env.MAILCHIMP_LIST_ID!

function emailHash(email: string) {
  return crypto.createHash('md5').update(email.toLowerCase()).digest('hex')
}

interface MailchimpSubscribeOptions {
  email: string
  firstName?: string
  lastName?: string
  tags?: string[]
  mergeFields?: Record<string, string>
}

export async function mailchimpSubscribe({
  email,
  firstName,
  lastName,
  tags = [],
  mergeFields = {},
}: MailchimpSubscribeOptions) {
  try {
    await mailchimp.lists.addListMember(LIST_ID, {
      email_address: email,
      status: 'subscribed',
      merge_fields: {
        FNAME: firstName ?? '',
        LNAME: lastName ?? '',
        ...mergeFields,
      },
      tags,
    })
  } catch (err: unknown) {
    // Mailchimp throws on 4xx — handle "already subscribed" gracefully
    if (isMailchimpError(err) && err.status === 400) {
      const detail = err.response?.body?.detail ?? ''

      if (detail.includes('is already a list member')) {
        // Update existing member instead
        await mailchimpUpdateMember({ email, tags, mergeFields })
        return
      }
    }
    throw err
  }
}

export async function mailchimpUpdateMember({
  email,
  tags,
  mergeFields,
}: Pick<MailchimpSubscribeOptions, 'email' | 'tags' | 'mergeFields'>) {
  const hash = emailHash(email)

  if (mergeFields && Object.keys(mergeFields).length > 0) {
    await mailchimp.lists.updateListMember(LIST_ID, hash, {
      merge_fields: mergeFields,
    })
  }

  if (tags && tags.length > 0) {
    await mailchimp.lists.updateListMemberTags(LIST_ID, hash, {
      tags: tags.map((name) => ({ name, status: 'active' })),
    })
  }
}

export async function mailchimpUnsubscribe(email: string) {
  const hash = emailHash(email)
  await mailchimp.lists.updateListMember(LIST_ID, hash, {
    status: 'unsubscribed',
  })
}

function isMailchimpError(err: unknown): err is {
  status: number
  response: { body: { detail: string } }
} {
  return typeof err === 'object' && err !== null && 'status' in err
}
```

---

## 7. Lead Magnet Flow

```
User sees offer → fills form → gets confirmation email → clicks confirm →
receives email with download link → synced to newsletter list
```

```ts
// lib/send-lead-magnet.ts
import { render } from '@react-email/render'
import { resend } from './resend'
import { LeadMagnetEmail } from '@/emails/LeadMagnetEmail'

export async function sendLeadMagnet({
  to,
  firstName,
  magnetTitle,
  downloadUrl,
}: {
  to: string
  firstName: string
  magnetTitle: string
  downloadUrl: string
}) {
  const html = await render(
    LeadMagnetEmail({ firstName, magnetTitle, downloadUrl }) as React.ReactElement
  )

  return resend.emails.send({
    from: `${process.env.RESEND_FROM_NAME} <${process.env.RESEND_FROM_EMAIL}>`,
    to,
    subject: `Your ${magnetTitle} is here`,
    html,
  })
}
```

```ts
// lib/send-lead-magnet-confirm.ts
import { render } from '@react-email/render'
import { resend } from './resend'
import { WelcomeEmail } from '@/emails/WelcomeEmail'

export async function sendLeadMagnetConfirm({
  to,
  firstName,
  magnetTitle,
  token,
}: {
  to: string
  firstName: string
  magnetTitle: string
  token: string
}) {
  const confirmUrl = `${process.env.NEXT_PUBLIC_BASE_URL}/confirm?token=${token}&magnet=${encodeURIComponent(magnetTitle)}`
  const unsubscribeUrl = `${process.env.NEXT_PUBLIC_BASE_URL}/unsubscribe?email=${encodeURIComponent(to)}`

  const html = await render(
    WelcomeEmail({ firstName, confirmUrl, unsubscribeUrl }) as React.ReactElement
  )

  return resend.emails.send({
    from: `${process.env.RESEND_FROM_NAME} <${process.env.RESEND_FROM_EMAIL}>`,
    to,
    subject: `Confirm your email to get ${magnetTitle}`,
    html,
    headers: {
      'List-Unsubscribe': `<${unsubscribeUrl}>`,
      'List-Unsubscribe-Post': 'List-Unsubscribe=One-Click',
    },
  })
}
```

```tsx
// emails/LeadMagnetEmail.tsx
import { Body, Button, Container, Head, Html, Preview, Text } from '@react-email/components'

export function LeadMagnetEmail({
  firstName,
  magnetTitle,
  downloadUrl,
}: {
  firstName: string
  magnetTitle: string
  downloadUrl: string
}) {
  return (
    <Html>
      <Head />
      <Preview>Your {magnetTitle} download is ready</Preview>
      <Body style={{ backgroundColor: '#f6f9fc', fontFamily: 'sans-serif' }}>
        <Container style={{ backgroundColor: '#fff', padding: 40, maxWidth: 560, margin: '40px auto', borderRadius: 8 }}>
          <Text style={{ fontSize: 16, color: '#374151' }}>
            Hi {firstName},
          </Text>
          <Text style={{ fontSize: 16, color: '#374151' }}>
            Here is your copy of <strong>{magnetTitle}</strong>.
          </Text>
          <Button
            href={downloadUrl}
            style={{
              backgroundColor: '#2563eb',
              color: '#fff',
              padding: '12px 28px',
              borderRadius: 6,
              fontWeight: 600,
              textDecoration: 'none',
              display: 'inline-block',
            }}
          >
            Download now
          </Button>
          <Text style={{ fontSize: 13, color: '#9ca3af', marginTop: 32 }}>
            You are now subscribed to our newsletter. Unsubscribe anytime.
          </Text>
        </Container>
      </Body>
    </Html>
  )
}
```

```tsx
// components/LeadMagnetForm.tsx
'use client'

import { useActionState } from 'react'
import { leadMagnetAction } from '@/app/actions/lead-magnet'

const initialState = { success: false }

export function LeadMagnetForm({ magnetTitle }: { magnetTitle: string }) {
  const [state, action, isPending] = useActionState(leadMagnetAction, initialState)

  if (state.success) {
    return (
      <p className="text-green-700 font-medium">
        Check your inbox for your download link.
      </p>
    )
  }

  return (
    <form action={action} className="space-y-3">
      <input type="hidden" name="magnetTitle" value={magnetTitle} />
      <p className="text-lg font-semibold">Get your free {magnetTitle}</p>
      <input name="firstName" type="text" placeholder="First name" required className="w-full border rounded-md px-3 py-2 text-sm" />
      <input name="email" type="email" placeholder="Email address" required className="w-full border rounded-md px-3 py-2 text-sm" />
      {state.error && <p className="text-red-600 text-sm">{state.error}</p>}
      <button type="submit" disabled={isPending} className="w-full bg-blue-600 text-white rounded-md py-2 text-sm font-semibold disabled:opacity-50">
        {isPending ? 'Sending…' : 'Send me the free guide'}
      </button>
    </form>
  )
}
```

```ts
// app/actions/lead-magnet.ts
'use server'

import { z } from 'zod'
import { createPendingSubscriber } from '@/lib/db/subscribers'
import { sendLeadMagnetConfirm } from '@/lib/send-lead-magnet-confirm'

const schema = z.object({
  email: z.string().email(),
  firstName: z.string().min(1).max(50),
  magnetTitle: z.string(),
})

export async function leadMagnetAction(_prev: unknown, formData: FormData) {
  const parsed = schema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) {
    return { success: false, error: 'Please fill in all fields correctly.' }
  }

  const { email, firstName, magnetTitle } = parsed.data

  try {
    const { token } = await createPendingSubscriber({
      email,
      firstName,
      source: `lead-magnet:${magnetTitle}`,
    })

    // Confirmation email carries the magnet title so we deliver after confirm
    await sendLeadMagnetConfirm({ to: email, firstName, magnetTitle, token })

    return { success: true }
  } catch {
    return { success: false, error: 'Something went wrong. Please try again.' }
  }
}
```

On confirmation (`/confirm` route), detect `source` starts with `lead-magnet:` and send the actual magnet email via `sendLeadMagnet`.

---

## 8. Popup / Exit-Intent Capture

```tsx
// components/ExitIntentPopup.tsx
'use client'

import { useEffect, useRef, useState } from 'react'
import { SubscribeForm } from './SubscribeForm'

const SUPPRESS_KEY = 'exit_popup_dismissed'
const SUPPRESS_DAYS = 7

function isSuppressed(): boolean {
  if (typeof window === 'undefined') return false
  const val = localStorage.getItem(SUPPRESS_KEY)
  if (!val) return false
  return Date.now() < parseInt(val, 10)
}

function suppress() {
  const expiry = Date.now() + SUPPRESS_DAYS * 24 * 60 * 60 * 1000
  localStorage.setItem(SUPPRESS_KEY, String(expiry))
}

export function ExitIntentPopup() {
  const [open, setOpen] = useState(false)
  const triggered = useRef(false)

  useEffect(() => {
    // No popup on mobile — exit intent is desktop only
    if (window.innerWidth < 768) return
    if (isSuppressed()) return

    const handleMouseLeave = (e: MouseEvent) => {
      if (triggered.current) return
      // Trigger when cursor leaves toward top of viewport
      if (e.clientY <= 0) {
        triggered.current = true
        setOpen(true)
      }
    }

    // Also trigger after 45 seconds of reading (scroll depth signal)
    const timer = setTimeout(() => {
      if (triggered.current || isSuppressed()) return
      triggered.current = true
      setOpen(true)
    }, 45_000)

    document.addEventListener('mouseleave', handleMouseLeave)

    return () => {
      document.removeEventListener('mouseleave', handleMouseLeave)
      clearTimeout(timer)
    }
  }, [])

  const handleClose = () => {
    suppress()
    setOpen(false)
  }

  if (!open) return null

  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-label="Newsletter signup"
      className="fixed inset-0 z-50 flex items-center justify-center p-4"
    >
      {/* Backdrop */}
      <div
        className="absolute inset-0 bg-black/50"
        onClick={handleClose}
        aria-hidden="true"
      />

      {/* Modal */}
      <div className="relative z-10 w-full max-w-md rounded-xl bg-white p-8 shadow-2xl">
        <button
          onClick={handleClose}
          className="absolute right-4 top-4 text-gray-400 hover:text-gray-600"
          aria-label="Close"
        >
          x
        </button>

        <h2 className="mb-2 text-xl font-bold text-gray-900">
          Before you go...
        </h2>
        <p className="mb-6 text-sm text-gray-600">
          Get our weekly tips delivered to your inbox. No spam.
        </p>

        <SubscribeForm source="exit-intent-popup" />
      </div>
    </div>
  )
}
```

```tsx
// app/layout.tsx — add popup to root layout
import { ExitIntentPopup } from '@/components/ExitIntentPopup'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <ExitIntentPopup />
      </body>
    </html>
  )
}
```

**Mobile alternative** — time-based scroll popup instead of exit intent:

```tsx
// components/ScrollDepthPopup.tsx
'use client'

import { useEffect, useRef, useState } from 'react'
import { SubscribeForm } from './SubscribeForm'

const SUPPRESS_KEY = 'scroll_popup_dismissed'
const SUPPRESS_DAYS = 7

function isSuppressed(): boolean {
  if (typeof window === 'undefined') return false
  const val = localStorage.getItem(SUPPRESS_KEY)
  if (!val) return false
  return Date.now() < parseInt(val, 10)
}

function suppress() {
  const expiry = Date.now() + SUPPRESS_DAYS * 24 * 60 * 60 * 1000
  localStorage.setItem(SUPPRESS_KEY, String(expiry))
}

export function ScrollDepthPopup({ threshold = 0.6 }: { threshold?: number }) {
  const [open, setOpen] = useState(false)
  const triggered = useRef(false)

  useEffect(() => {
    if (isSuppressed()) return

    const handleScroll = () => {
      if (triggered.current) return
      const scrolled = window.scrollY / (document.body.scrollHeight - window.innerHeight)
      if (scrolled >= threshold) {
        triggered.current = true
        setOpen(true)
      }
    }

    window.addEventListener('scroll', handleScroll, { passive: true })
    return () => window.removeEventListener('scroll', handleScroll)
  }, [threshold])

  if (!open) return null

  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-label="Newsletter signup"
      className="fixed inset-0 z-50 flex items-center justify-center p-4"
    >
      <div
        className="absolute inset-0 bg-black/50"
        onClick={() => { suppress(); setOpen(false) }}
        aria-hidden="true"
      />
      <div className="relative z-10 w-full max-w-md rounded-xl bg-white p-8 shadow-2xl">
        <button
          onClick={() => { suppress(); setOpen(false) }}
          className="absolute right-4 top-4 text-gray-400 hover:text-gray-600"
          aria-label="Close"
        >
          ✕
        </button>
        <h2 className="mb-2 text-xl font-bold text-gray-900">Enjoying this?</h2>
        <p className="mb-6 text-sm text-gray-600">
          Get our best content delivered weekly. No spam.
        </p>
        <SubscribeForm source="scroll-depth-popup" />
      </div>
    </div>
  )
}
```

---

## 9. Embedded Inline Form Placements

```tsx
// components/InlineCapture.tsx — composable placement wrapper
import { SubscribeForm } from './SubscribeForm'

interface InlineCaptureProps {
  placement: 'hero' | 'mid-content' | 'end-of-post' | 'sidebar'
  headline?: string
  subtext?: string
}

const placementStyles: Record<string, string> = {
  hero: 'bg-blue-600 text-white rounded-2xl p-8 my-8',
  'mid-content': 'bg-gray-50 border border-gray-200 rounded-xl p-6 my-10',
  'end-of-post': 'bg-gradient-to-r from-blue-50 to-indigo-50 rounded-xl p-8 mt-12',
  sidebar: 'bg-white border border-gray-200 rounded-xl p-5 sticky top-6',
}

const headlineDefaults: Record<string, string> = {
  hero: 'Get actionable tips every week',
  'mid-content': 'Enjoying this? Get more like it.',
  'end-of-post': 'Found this useful?',
  sidebar: 'Weekly newsletter',
}

export function InlineCapture({
  placement,
  headline,
  subtext = 'Join thousands of readers. Unsubscribe anytime.',
}: InlineCaptureProps) {
  const isHero = placement === 'hero'

  return (
    <section className={placementStyles[placement]} aria-label="Newsletter signup">
      <h2 className={`font-bold mb-1 ${isHero ? 'text-2xl text-white' : 'text-lg text-gray-900'}`}>
        {headline ?? headlineDefaults[placement]}
      </h2>
      <p className={`text-sm mb-5 ${isHero ? 'text-blue-100' : 'text-gray-600'}`}>
        {subtext}
      </p>
      <SubscribeForm source={`inline-${placement}`} />
    </section>
  )
}
```

```tsx
// Usage in a blog post layout
import { InlineCapture } from '@/components/InlineCapture'

export default function BlogPost() {
  return (
    <article>
      {/* Above-fold hero */}
      <InlineCapture placement="hero" />

      <p>Post content...</p>

      {/* Mid-content — after ~300 words */}
      <InlineCapture placement="mid-content" headline="Want the full playbook?" />

      <p>More content...</p>

      {/* End of post */}
      <InlineCapture placement="end-of-post" />
    </article>
  )
}
```

---

## 10. Drip Sequence Pattern

Resend does not have built-in scheduling — use a job queue or cron. Pattern shown with `node-cron` + your DB, but works identically with Vercel Cron or any queue (BullMQ, Trigger.dev).

```ts
// lib/drip/templates.ts
export const DRIP_SEQUENCE = [
  { dayOffset: 0,  subject: 'Welcome — start here',          template: 'drip-day-0'  },
  { dayOffset: 3,  subject: 'The one thing most people miss', template: 'drip-day-3'  },
  { dayOffset: 7,  subject: 'Quick win for this week',        template: 'drip-day-7'  },
  { dayOffset: 14, subject: 'Where do you want to go next?',  template: 'drip-day-14' },
] as const
```

```ts
// lib/drip/scheduler.ts
import { prisma } from '@/lib/prisma'
import { DRIP_SEQUENCE } from './templates'

export async function scheduleDrip(subscriberId: string, confirmedAt: Date) {
  const jobs = DRIP_SEQUENCE.map((step) => ({
    subscriberId,
    template: step.template,
    subject: step.subject,
    sendAfter: new Date(confirmedAt.getTime() + step.dayOffset * 24 * 60 * 60 * 1000),
    status: 'PENDING' as const,
  }))

  await prisma.emailJob.createMany({ data: jobs, skipDuplicates: true })
}
```

```ts
// app/api/cron/drip/route.ts — Vercel Cron (runs hourly)
import { type NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'
import { resend } from '@/lib/resend'
import { renderDripEmail } from '@/lib/drip/render'

export const runtime = 'nodejs'

export async function GET(request: NextRequest) {
  // Verify cron secret to prevent unauthorized calls
  const auth = request.headers.get('authorization')
  if (auth !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const jobs = await prisma.emailJob.findMany({
    where: {
      status: 'PENDING',
      sendAfter: { lte: new Date() },
    },
    include: { subscriber: true },
    take: 50, // Process in batches
  })

  const results = await Promise.allSettled(
    jobs.map(async (job) => {
      if (job.subscriber.status !== 'ACTIVE') {
        await prisma.emailJob.update({
          where: { id: job.id },
          data: { status: 'SKIPPED' },
        })
        return
      }

      const html = await renderDripEmail(job.template, {
        firstName: job.subscriber.firstName ?? 'there',
      })

      await resend.emails.send({
        from: `${process.env.RESEND_FROM_NAME} <${process.env.RESEND_FROM_EMAIL}>`,
        to: job.subscriber.email,
        subject: job.subject,
        html,
        headers: {
          'List-Unsubscribe': `<${process.env.NEXT_PUBLIC_BASE_URL}/unsubscribe?email=${encodeURIComponent(job.subscriber.email)}>`,
        },
      })

      await prisma.emailJob.update({
        where: { id: job.id },
        data: { status: 'SENT', sentAt: new Date() },
      })
    })
  )

  const failed = results.filter((r) => r.status === 'rejected').length

  return NextResponse.json({ processed: jobs.length, failed })
}
```

```ts
// lib/drip/render.ts
import { render } from '@react-email/render'
import { DripDay0 } from '@/emails/drip/DripDay0'
import { DripDay3 } from '@/emails/drip/DripDay3'
import { DripDay7 } from '@/emails/drip/DripDay7'
import { DripDay14 } from '@/emails/drip/DripDay14'

const templates: Record<string, React.ComponentType<{ firstName: string }>> = {
  'drip-day-0':  DripDay0,
  'drip-day-3':  DripDay3,
  'drip-day-7':  DripDay7,
  'drip-day-14': DripDay14,
}

export async function renderDripEmail(
  template: string,
  props: { firstName: string }
): Promise<string> {
  const Component = templates[template]
  if (!Component) throw new Error(`Unknown drip template: ${template}`)
  return render(Component(props) as React.ReactElement)
}
```

```json
// vercel.json — Vercel Cron config
{
  "crons": [
    {
      "path": "/api/cron/drip",
      "schedule": "0 * * * *"
    }
  ]
}
```

---

## 11. Analytics

```ts
// lib/analytics/capture.ts
import { prisma } from '@/lib/prisma'

export async function trackSignup({
  email,
  source,
  placement,
  page,
}: {
  email: string
  source: string
  placement?: string
  page?: string
}) {
  await prisma.signupEvent.create({
    data: {
      email,
      source,
      placement,
      page,
      createdAt: new Date(),
    },
  })
}
```

```ts
// lib/analytics/reports.ts — query conversion rates by placement
import { prisma } from '@/lib/prisma'

export async function conversionBySource(days = 30) {
  const since = new Date(Date.now() - days * 24 * 60 * 60 * 1000)

  const signups = await prisma.signupEvent.groupBy({
    by: ['source'],
    where: { createdAt: { gte: since } },
    _count: { _all: true },
    orderBy: { _count: { source: 'desc' } },
  })

  const confirmed = await prisma.subscriber.groupBy({
    by: ['source'],
    where: { confirmedAt: { gte: since }, status: 'ACTIVE' },
    _count: { _all: true },
  })

  const confirmedMap = Object.fromEntries(
    confirmed.map((r) => [r.source, r._count._all])
  )

  return signups.map((row) => ({
    source: row.source,
    signups: row._count._all,
    confirmed: confirmedMap[row.source] ?? 0,
    rate: confirmedMap[row.source]
      ? Math.round((confirmedMap[row.source] / row._count._all) * 100)
      : 0,
  }))
}

export async function unsubscribeRate(days = 30) {
  const since = new Date(Date.now() - days * 24 * 60 * 60 * 1000)

  const [active, unsubscribed] = await Promise.all([
    prisma.subscriber.count({ where: { status: 'ACTIVE' } }),
    prisma.subscriber.count({
      where: { status: 'UNSUBSCRIBED', updatedAt: { gte: since } },
    }),
  ])

  return {
    active,
    unsubscribedLast30d: unsubscribed,
    rate: active > 0 ? Math.round((unsubscribed / active) * 100 * 10) / 10 : 0,
  }
}
```

**Key metrics to track:**

| Metric | Healthy range | Action if outside |
|--------|---------------|-------------------|
| Signup to confirm rate | 40-65% | Rewrite confirmation email subject line |
| Confirm to open (drip day 0) | 50-70% | Fix from name or subject |
| Unsubscribe rate (monthly) | < 2% | Reduce send frequency or improve targeting |
| Popup conversion | 1-4% | Adjust timing, offer, or suppress logic |
| Lead magnet conversion | 15-40% | Improve offer clarity |

---

## 12. Anti-Patterns

**No double opt-in**
Skipping confirmation means bounces, spam complaints, and deliverability damage. Always send a confirmation email and only activate the subscriber after they click. The CAN-SPAM / GDPR risk is secondary — the deliverability damage is immediate.

**No unsubscribe mechanism**
Every email must include a working one-click unsubscribe link. Use the `List-Unsubscribe` header. Hiding the unsubscribe link increases spam reports, which tanks your sender reputation faster than unsubscribes ever will.

**Buying email lists**
Purchased lists fail every engagement metric. ISPs have already flagged most of those addresses as spam traps. One purchased list can get your domain blacklisted permanently.

**Vague incentives**
"Sign up for updates" converts at 0.1-0.5%. A specific, tangible offer ("Get the 12-step launch checklist") converts at 5-20x that. Always name what the subscriber gets and when.

**Sending from a free inbox (Gmail, Yahoo)**
Gmail and Yahoo enforce DMARC in 2024+. Email from `yourname@gmail.com` via Resend will be rejected or marked spam. Verify a real domain.

**No suppression list sync across providers**
If you use Resend for transactional and Beehiiv for newsletters, unsubscribes from one must be reflected in the other. Build a webhook or periodic sync. Sending to someone who unsubscribed from your newsletter via your transactional provider is a GDPR violation.

**Sending drip to unconfirmed subscribers**
Only send drip sequences to subscribers with `status: 'ACTIVE'`. The cron route in section 10 checks this — do not remove that check.

**No rate limiting on the subscribe endpoint**
Server Actions are public. Add rate limiting by IP to prevent abuse:

```ts
// app/actions/subscribe.ts — add at top of action
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'
import { headers } from 'next/headers'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, '10 m'), // 5 signups per IP per 10 min
})

export async function subscribeAction(_prev: SubscribeState, formData: FormData) {
  const ip = (await headers()).get('x-forwarded-for') ?? 'unknown'
  const { success } = await ratelimit.limit(ip)

  if (!success) {
    return { success: false, error: 'Too many requests. Try again later.' }
  }

  // ... rest of action
}
```
