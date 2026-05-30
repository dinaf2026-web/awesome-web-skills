---
name: web-ecommerce
description: E-commerce patterns for modern web apps — Stripe Checkout and Payment Intents, cart state management, product pages, webhooks, and order management. Build a complete store in Next.js.
origin: community
tags: [ecommerce, stripe, checkout, cart, payments, webhooks, nextjs]
---

# web-ecommerce

E-commerce patterns for Next.js 14 App Router with Stripe. Covers Checkout Sessions, Payment Intents, cart state, webhooks, subscriptions, and order management.

---

## 1. Architecture Overview

Choose the right Stripe integration for your use case:

| Use Case | Integration | When to Use |
|----------|-------------|-------------|
| Simple buy now / cart checkout | **Checkout Session** | Fastest path to production; Stripe hosts the payment page |
| Custom payment UI embedded in your site | **Payment Intents + Elements** | You own the checkout UI; full design control |
| Subscriptions / recurring billing | **Subscription + Checkout Session** | SaaS, memberships, recurring plans |
| Marketplace / platform payments | **Connect + Payment Intents** | Multi-vendor, split payments |

**Decision rule:** Start with Checkout Session. Move to Payment Intents only when you need a fully embedded, custom-designed payment form. Never build custom card input without Stripe Elements — PCI scope explodes.

**Request flow (Checkout Session):**
```
Client (cart) → POST /api/checkout → Stripe API → session.url
→ redirect to Stripe-hosted page → webhook payment_intent.succeeded
→ fulfill order → redirect to /success?session_id=
```

**Request flow (Payment Intents):**
```
Client (cart) → POST /api/payment-intent → { clientSecret }
→ Stripe Elements confirmPayment() → webhook payment_intent.succeeded
→ fulfill order → client shows success state
```

---

## 2. Stripe Setup

### Install

```bash
npm install stripe @stripe/stripe-js @stripe/react-stripe-js zod
```

### Environment variables

```bash
# .env.local
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_BASE_URL=http://localhost:3000
```

### Server-side Stripe instance

```typescript
// lib/stripe/server.ts
import Stripe from 'stripe'

if (!process.env.STRIPE_SECRET_KEY) {
  throw new Error('STRIPE_SECRET_KEY is not set')
}

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: '2024-06-20',
  typescript: true,
})
```

### Client-side Stripe instance

```typescript
// lib/stripe/client.ts
import { loadStripe } from '@stripe/stripe-js'

const key = process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY

if (!key) {
  throw new Error('NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY is not set')
}

// Singleton — loadStripe must not be called inside a render
export const stripePromise = loadStripe(key)
```

---

## 3. Product Catalog

### Data model

```typescript
// lib/types/product.ts
export type Price = {
  id: string           // Stripe Price ID: price_...
  unitAmount: number   // cents
  currency: string     // 'usd'
  interval?: 'month' | 'year'  // only for subscriptions
}

export type Product = {
  id: string           // Stripe Product ID: prod_...
  name: string
  description: string
  images: string[]
  prices: Price[]
  metadata: Record<string, string>
}
```

### Fetch products from Stripe (no CMS)

```typescript
// lib/products.ts
import { stripe } from './stripe/server'
import type { Product } from './types/product'

export async function getProducts(): Promise<Product[]> {
  const { data: stripeProducts } = await stripe.products.list({
    active: true,
    expand: ['data.default_price'],
  })

  const products: Product[] = await Promise.all(
    stripeProducts.map(async (p) => {
      const { data: prices } = await stripe.prices.list({
        product: p.id,
        active: true,
      })

      return {
        id: p.id,
        name: p.name,
        description: p.description ?? '',
        images: p.images,
        prices: prices.map((price) => ({
          id: price.id,
          unitAmount: price.unit_amount ?? 0,
          currency: price.currency,
          interval: price.recurring?.interval as Price['interval'],
        })),
        metadata: p.metadata,
      }
    })
  )

  return products
}

export async function getProduct(id: string): Promise<Product | null> {
  try {
    const p = await stripe.products.retrieve(id, {
      expand: ['default_price'],
    })
    const { data: prices } = await stripe.prices.list({ product: id, active: true })

    return {
      id: p.id,
      name: p.name,
      description: p.description ?? '',
      images: p.images,
      prices: prices.map((price) => ({
        id: price.id,
        unitAmount: price.unit_amount ?? 0,
        currency: price.currency,
        interval: price.recurring?.interval as Price['interval'],
      })),
      metadata: p.metadata,
    }
  } catch {
    return null
  }
}
```

### Product page with image optimization

```typescript
// app/products/[id]/page.tsx
import Image from 'next/image'
import { getProduct } from '@/lib/products'
import { AddToCartButton } from '@/components/cart/AddToCartButton'
import { notFound } from 'next/navigation'
import { formatCurrency } from '@/lib/utils/currency'

type Props = { params: { id: string } }

export default async function ProductPage({ params }: Props) {
  const product = await getProduct(params.id)
  if (!product) notFound()

  const price = product.prices[0]

  return (
    <main className="max-w-5xl mx-auto px-4 py-12 grid grid-cols-1 md:grid-cols-2 gap-12">
      <div className="relative aspect-square rounded-xl overflow-hidden bg-neutral-100">
        {product.images[0] ? (
          <Image
            src={product.images[0]}
            alt={product.name}
            fill
            sizes="(max-width: 768px) 100vw, 50vw"
            priority
            className="object-cover"
          />
        ) : (
          <div className="flex items-center justify-center h-full text-neutral-400">
            No image
          </div>
        )}
      </div>

      <div className="flex flex-col gap-6">
        <h1 className="text-3xl font-bold">{product.name}</h1>
        <p className="text-2xl font-semibold">
          {price ? formatCurrency(price.unitAmount, price.currency) : 'Contact for pricing'}
        </p>
        <p className="text-neutral-600 leading-relaxed">{product.description}</p>
        {price && (
          <AddToCartButton
            product={{ id: product.id, name: product.name, priceId: price.id, unitAmount: price.unitAmount, currency: price.currency, image: product.images[0] }}
          />
        )}
      </div>
    </main>
  )
}
```

```typescript
// lib/utils/currency.ts
export function formatCurrency(amount: number, currency: string): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: currency.toUpperCase(),
  }).format(amount / 100)
}
```

---

## 4. Cart State (Zustand)

```typescript
// lib/store/cart.ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

export type CartItem = {
  productId: string
  priceId: string
  name: string
  unitAmount: number
  currency: string
  image?: string
  quantity: number
}

type CartState = {
  items: CartItem[]
  addItem: (item: Omit<CartItem, 'quantity'>) => void
  removeItem: (priceId: string) => void
  updateQuantity: (priceId: string, quantity: number) => void
  clearCart: () => void
  totalItems: () => number
  totalAmount: () => number
}

export const useCartStore = create<CartState>()(
  persist(
    (set, get) => ({
      items: [],

      addItem: (incoming) => {
        set((state) => {
          const existing = state.items.find((i) => i.priceId === incoming.priceId)
          if (existing) {
            return {
              items: state.items.map((i) =>
                i.priceId === incoming.priceId
                  ? { ...i, quantity: i.quantity + 1 }
                  : i
              ),
            }
          }
          return { items: [...state.items, { ...incoming, quantity: 1 }] }
        })
      },

      removeItem: (priceId) => {
        set((state) => ({
          items: state.items.filter((i) => i.priceId !== priceId),
        }))
      },

      updateQuantity: (priceId, quantity) => {
        if (quantity <= 0) {
          get().removeItem(priceId)
          return
        }
        set((state) => ({
          items: state.items.map((i) =>
            i.priceId === priceId ? { ...i, quantity } : i
          ),
        }))
      },

      clearCart: () => set({ items: [] }),

      totalItems: () => get().items.reduce((sum, i) => sum + i.quantity, 0),

      totalAmount: () =>
        get().items.reduce((sum, i) => sum + i.unitAmount * i.quantity, 0),
    }),
    {
      name: 'cart-storage',
      storage: createJSONStorage(() => localStorage),
    }
  )
)
```

### AddToCartButton component

```typescript
// components/cart/AddToCartButton.tsx
'use client'

import { useCartStore } from '@/lib/store/cart'
import type { CartItem } from '@/lib/store/cart'

type Props = {
  product: Omit<CartItem, 'quantity'>
}

export function AddToCartButton({ product }: Props) {
  const addItem = useCartStore((s) => s.addItem)

  return (
    <button
      onClick={() => addItem(product)}
      className="rounded-lg bg-black text-white px-6 py-3 font-medium hover:bg-neutral-800 transition-colors"
    >
      Add to Cart
    </button>
  )
}
```

### Cart drawer component

```typescript
// components/cart/CartDrawer.tsx
'use client'

import { useCartStore } from '@/lib/store/cart'
import { formatCurrency } from '@/lib/utils/currency'
import { CheckoutButton } from './CheckoutButton'

export function CartDrawer() {
  const { items, removeItem, updateQuantity, totalAmount } = useCartStore()

  if (items.length === 0) {
    return <p className="text-neutral-500 p-6">Your cart is empty.</p>
  }

  return (
    <div className="flex flex-col gap-4 p-6">
      {items.map((item) => (
        <div key={item.priceId} className="flex items-center gap-4">
          <div className="flex-1">
            <p className="font-medium">{item.name}</p>
            <p className="text-sm text-neutral-500">
              {formatCurrency(item.unitAmount, item.currency)} each
            </p>
          </div>
          <div className="flex items-center gap-2">
            <button
              onClick={() => updateQuantity(item.priceId, item.quantity - 1)}
              className="w-8 h-8 rounded border flex items-center justify-center"
            >
              −
            </button>
            <span className="w-6 text-center">{item.quantity}</span>
            <button
              onClick={() => updateQuantity(item.priceId, item.quantity + 1)}
              className="w-8 h-8 rounded border flex items-center justify-center"
            >
              +
            </button>
          </div>
          <button
            onClick={() => removeItem(item.priceId)}
            className="text-red-500 text-sm hover:underline"
          >
            Remove
          </button>
        </div>
      ))}

      <div className="border-t pt-4 flex items-center justify-between">
        <p className="font-semibold text-lg">
          Total: {formatCurrency(totalAmount(), items[0]?.currency ?? 'usd')}
        </p>
        <CheckoutButton />
      </div>
    </div>
  )
}
```

---

## 5. Stripe Checkout Session

### API route — create session

```typescript
// app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { stripe } from '@/lib/stripe/server'
import { z } from 'zod'

const LineItemSchema = z.object({
  priceId: z.string().startsWith('price_'),
  quantity: z.number().int().min(1).max(100),
})

const CheckoutSchema = z.object({
  items: z.array(LineItemSchema).min(1).max(50),
})

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const { items } = CheckoutSchema.parse(body)

    const baseUrl = process.env.NEXT_PUBLIC_BASE_URL

    const session = await stripe.checkout.sessions.create({
      mode: 'payment',
      line_items: items.map(({ priceId, quantity }) => ({
        price: priceId,
        quantity,
      })),
      success_url: `${baseUrl}/checkout/success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${baseUrl}/cart`,
      automatic_tax: { enabled: true },
      shipping_address_collection: {
        allowed_countries: ['US', 'CA', 'GB'],
      },
      metadata: {
        source: 'web-store',
      },
    })

    return NextResponse.json({ url: session.url })
  } catch (err) {
    if (err instanceof z.ZodError) {
      return NextResponse.json({ error: err.issues }, { status: 400 })
    }
    console.error('Checkout error:', err)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

### CheckoutButton component

```typescript
// components/cart/CheckoutButton.tsx
'use client'

import { useState } from 'react'
import { useCartStore } from '@/lib/store/cart'
import { useRouter } from 'next/navigation'

export function CheckoutButton() {
  const { items, clearCart } = useCartStore()
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)
  const router = useRouter()

  const handleCheckout = async () => {
    setLoading(true)
    setError(null)

    try {
      const res = await fetch('/api/checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          items: items.map((i) => ({ priceId: i.priceId, quantity: i.quantity })),
        }),
      })

      if (!res.ok) throw new Error('Checkout failed')

      const { url } = await res.json()
      clearCart()
      router.push(url)
    } catch {
      setError('Something went wrong. Please try again.')
    } finally {
      setLoading(false)
    }
  }

  return (
    <div>
      {error && <p className="text-red-500 text-sm mb-2">{error}</p>}
      <button
        onClick={handleCheckout}
        disabled={loading || items.length === 0}
        className="rounded-lg bg-black text-white px-6 py-3 font-medium hover:bg-neutral-800 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
      >
        {loading ? 'Redirecting...' : 'Checkout'}
      </button>
    </div>
  )
}
```

### Success page

```typescript
// app/checkout/success/page.tsx
import { stripe } from '@/lib/stripe/server'
import { redirect } from 'next/navigation'
import { formatCurrency } from '@/lib/utils/currency'

type Props = {
  searchParams: { session_id?: string }
}

export default async function SuccessPage({ searchParams }: Props) {
  const sessionId = searchParams.session_id
  if (!sessionId) redirect('/')

  const session = await stripe.checkout.sessions.retrieve(sessionId, {
    expand: ['line_items', 'payment_intent'],
  })

  if (session.payment_status !== 'paid') {
    redirect('/cart')
  }

  const amount = session.amount_total ?? 0
  const currency = session.currency ?? 'usd'

  return (
    <main className="max-w-lg mx-auto px-4 py-20 text-center">
      <div className="text-5xl mb-6">✓</div>
      <h1 className="text-3xl font-bold mb-4">Order Confirmed</h1>
      <p className="text-neutral-600 mb-2">
        Thank you for your purchase of{' '}
        <strong>{formatCurrency(amount, currency)}</strong>.
      </p>
      <p className="text-sm text-neutral-500 mb-8">
        Order ID: {session.id}
      </p>
      <a
        href="/products"
        className="inline-block bg-black text-white px-6 py-3 rounded-lg hover:bg-neutral-800 transition-colors"
      >
        Continue Shopping
      </a>
    </main>
  )
}
```

---

## 6. Payment Intents (Embedded Checkout)

Use this when you want full control over the payment UI.

### API route — create Payment Intent

```typescript
// app/api/payment-intent/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { stripe } from '@/lib/stripe/server'
import { z } from 'zod'

const Schema = z.object({
  items: z.array(
    z.object({
      priceId: z.string().startsWith('price_'),
      quantity: z.number().int().min(1),
    })
  ).min(1),
})

async function computeAmount(items: { priceId: string; quantity: number }[]): Promise<number> {
  // ALWAYS compute price server-side — never trust the client
  let total = 0
  for (const { priceId, quantity } of items) {
    const price = await stripe.prices.retrieve(priceId)
    if (!price.active) throw new Error(`Price ${priceId} is not active`)
    total += (price.unit_amount ?? 0) * quantity
  }
  return total
}

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const { items } = Schema.parse(body)

    const amount = await computeAmount(items)

    const paymentIntent = await stripe.paymentIntents.create({
      amount,
      currency: 'usd',
      automatic_payment_methods: { enabled: true },
      metadata: {
        items: JSON.stringify(items),
      },
    })

    return NextResponse.json({ clientSecret: paymentIntent.client_secret })
  } catch (err) {
    if (err instanceof z.ZodError) {
      return NextResponse.json({ error: err.issues }, { status: 400 })
    }
    console.error('Payment intent error:', err)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

### Custom payment form with Stripe Elements

```typescript
// components/checkout/PaymentForm.tsx
'use client'

import { useState } from 'react'
import {
  PaymentElement,
  useStripe,
  useElements,
} from '@stripe/react-stripe-js'

type Props = {
  onSuccess: () => void
}

export function PaymentForm({ onSuccess }: Props) {
  const stripe = useStripe()
  const elements = useElements()
  const [loading, setLoading] = useState(false)
  const [errorMessage, setErrorMessage] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (!stripe || !elements) return

    setLoading(true)
    setErrorMessage(null)

    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/checkout/success`,
      },
      redirect: 'if_required',
    })

    if (error) {
      setErrorMessage(error.message ?? 'Payment failed')
      setLoading(false)
    } else {
      onSuccess()
    }
  }

  return (
    <form onSubmit={handleSubmit} className="flex flex-col gap-6">
      <PaymentElement />
      {errorMessage && (
        <p className="text-red-500 text-sm">{errorMessage}</p>
      )}
      <button
        type="submit"
        disabled={!stripe || loading}
        className="w-full bg-black text-white py-3 rounded-lg font-medium hover:bg-neutral-800 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
      >
        {loading ? 'Processing...' : 'Pay Now'}
      </button>
    </form>
  )
}
```

### Checkout page with Elements provider

```typescript
// app/checkout/page.tsx
'use client'

import { useEffect, useState } from 'react'
import { Elements } from '@stripe/react-stripe-js'
import { stripePromise } from '@/lib/stripe/client'
import { PaymentForm } from '@/components/checkout/PaymentForm'
import { useCartStore } from '@/lib/store/cart'
import { useRouter } from 'next/navigation'

export default function CheckoutPage() {
  const { items, clearCart } = useCartStore()
  const [clientSecret, setClientSecret] = useState<string | null>(null)
  const [error, setError] = useState<string | null>(null)
  const router = useRouter()

  useEffect(() => {
    if (items.length === 0) return

    fetch('/api/payment-intent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        items: items.map((i) => ({ priceId: i.priceId, quantity: i.quantity })),
      }),
    })
      .then((r) => r.json())
      .then(({ clientSecret, error }) => {
        if (error) setError('Failed to initialize payment')
        else setClientSecret(clientSecret)
      })
      .catch(() => setError('Network error'))
  }, [items])

  const handleSuccess = () => {
    clearCart()
    router.push('/checkout/success')
  }

  if (items.length === 0) {
    return <p className="text-center py-20 text-neutral-500">Your cart is empty.</p>
  }

  if (error) {
    return <p className="text-center py-20 text-red-500">{error}</p>
  }

  if (!clientSecret) {
    return <p className="text-center py-20 text-neutral-400">Loading payment form...</p>
  }

  return (
    <main className="max-w-md mx-auto px-4 py-12">
      <h1 className="text-2xl font-bold mb-8">Complete Your Purchase</h1>
      <Elements stripe={stripePromise} options={{ clientSecret }}>
        <PaymentForm onSuccess={handleSuccess} />
      </Elements>
    </main>
  )
}
```

---

## 7. Webhooks

Webhooks are how Stripe tells your server a payment succeeded. Never fulfill orders based on client-side signals — only webhooks.

### Webhook API route

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { stripe } from '@/lib/stripe/server'
import { fulfillOrder } from '@/lib/orders/fulfill'
import { handleSubscriptionChange } from '@/lib/subscriptions/handler'
import Stripe from 'stripe'

// Required: disable body parsing so we can verify raw signature
export const dynamic = 'force-dynamic'

export async function POST(req: NextRequest) {
  const body = await req.text()
  const signature = req.headers.get('stripe-signature')

  if (!signature) {
    return NextResponse.json({ error: 'Missing signature' }, { status: 400 })
  }

  let event: Stripe.Event

  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    console.error('Webhook signature verification failed:', err)
    return NextResponse.json({ error: 'Invalid signature' }, { status: 400 })
  }

  try {
    switch (event.type) {
      case 'payment_intent.succeeded': {
        const pi = event.data.object as Stripe.PaymentIntent
        await fulfillOrder(pi)
        break
      }

      case 'checkout.session.completed': {
        const session = event.data.object as Stripe.CheckoutSession
        if (session.payment_status === 'paid') {
          // Retrieve payment intent for fulfillment details
          if (typeof session.payment_intent === 'string') {
            const pi = await stripe.paymentIntents.retrieve(session.payment_intent)
            await fulfillOrder(pi, session)
          }
        }
        break
      }

      case 'customer.subscription.created':
      case 'customer.subscription.updated':
      case 'customer.subscription.deleted': {
        const subscription = event.data.object as Stripe.Subscription
        await handleSubscriptionChange(event.type, subscription)
        break
      }

      case 'invoice.payment_failed': {
        const invoice = event.data.object as Stripe.Invoice
        console.warn('Payment failed for invoice:', invoice.id)
        // Send dunning email here
        break
      }

      default:
        console.log(`Unhandled event type: ${event.type}`)
    }

    return NextResponse.json({ received: true })
  } catch (err) {
    console.error(`Error handling event ${event.type}:`, err)
    // Return 200 so Stripe does not retry — you handle idempotency internally
    return NextResponse.json({ error: 'Handler failed' }, { status: 500 })
  }
}
```

### Order fulfillment

```typescript
// lib/orders/fulfill.ts
import Stripe from 'stripe'
import { db } from '@/lib/db'
import { sendOrderConfirmationEmail } from '@/lib/email/order-confirmation'

export async function fulfillOrder(
  paymentIntent: Stripe.PaymentIntent,
  session?: Stripe.CheckoutSession
) {
  // Idempotency: skip if already fulfilled
  const existing = await db.order.findUnique({
    where: { stripePaymentIntentId: paymentIntent.id },
  })
  if (existing) return

  const order = await db.order.create({
    data: {
      stripePaymentIntentId: paymentIntent.id,
      stripeSessionId: session?.id,
      amount: paymentIntent.amount,
      currency: paymentIntent.currency,
      status: 'paid',
      customerEmail: session?.customer_details?.email ?? '',
      customerName: session?.customer_details?.name ?? '',
      shippingAddress: session?.shipping_details?.address
        ? JSON.stringify(session.shipping_details.address)
        : null,
      items: paymentIntent.metadata.items ?? '[]',
    },
  })

  await sendOrderConfirmationEmail({
    to: order.customerEmail,
    orderId: order.id,
    amount: order.amount,
    currency: order.currency,
  })

  console.log(`Order fulfilled: ${order.id}`)
}
```

### Local webhook testing

```bash
# Install Stripe CLI and listen to local server
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# Trigger test events
stripe trigger payment_intent.succeeded
stripe trigger checkout.session.completed
```

---

## 8. Subscription Billing

### Create subscription Checkout Session

```typescript
// app/api/subscribe/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { stripe } from '@/lib/stripe/server'
import { z } from 'zod'

const Schema = z.object({
  priceId: z.string().startsWith('price_'),
  customerId: z.string().optional(),
  trialDays: z.number().int().min(0).max(30).optional(),
})

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const { priceId, customerId, trialDays } = Schema.parse(body)

    const baseUrl = process.env.NEXT_PUBLIC_BASE_URL

    const session = await stripe.checkout.sessions.create({
      mode: 'subscription',
      customer: customerId,
      line_items: [{ price: priceId, quantity: 1 }],
      subscription_data: trialDays
        ? { trial_period_days: trialDays }
        : undefined,
      success_url: `${baseUrl}/account/subscription?success=true&session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${baseUrl}/pricing`,
      allow_promotion_codes: true,
      billing_address_collection: 'required',
    })

    return NextResponse.json({ url: session.url })
  } catch (err) {
    if (err instanceof z.ZodError) {
      return NextResponse.json({ error: err.issues }, { status: 400 })
    }
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

### Customer portal (manage subscription)

```typescript
// app/api/customer-portal/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { stripe } from '@/lib/stripe/server'
import { z } from 'zod'

const Schema = z.object({
  customerId: z.string().startsWith('cus_'),
})

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const { customerId } = Schema.parse(body)

    const portalSession = await stripe.billingPortal.sessions.create({
      customer: customerId,
      return_url: `${process.env.NEXT_PUBLIC_BASE_URL}/account`,
    })

    return NextResponse.json({ url: portalSession.url })
  } catch (err) {
    if (err instanceof z.ZodError) {
      return NextResponse.json({ error: err.issues }, { status: 400 })
    }
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

### Subscription event handler

```typescript
// lib/subscriptions/handler.ts
import Stripe from 'stripe'
import { db } from '@/lib/db'

type SubscriptionEvent =
  | 'customer.subscription.created'
  | 'customer.subscription.updated'
  | 'customer.subscription.deleted'

export async function handleSubscriptionChange(
  eventType: SubscriptionEvent,
  subscription: Stripe.Subscription
) {
  const customerId =
    typeof subscription.customer === 'string'
      ? subscription.customer
      : subscription.customer.id

  const status = subscription.status
  const priceId = subscription.items.data[0]?.price.id
  const currentPeriodEnd = new Date(subscription.current_period_end * 1000)

  await db.subscription.upsert({
    where: { stripeSubscriptionId: subscription.id },
    update: {
      status,
      priceId,
      currentPeriodEnd,
      cancelAtPeriodEnd: subscription.cancel_at_period_end,
    },
    create: {
      stripeSubscriptionId: subscription.id,
      stripeCustomerId: customerId,
      status,
      priceId,
      currentPeriodEnd,
      cancelAtPeriodEnd: subscription.cancel_at_period_end,
    },
  })
}
```

---

## 9. Order Management

### Order confirmation email

```typescript
// lib/email/order-confirmation.ts
// Uses Resend — swap for any transactional email provider
import { Resend } from 'resend'
import { formatCurrency } from '@/lib/utils/currency'

const resend = new Resend(process.env.RESEND_API_KEY)

type OrderEmailParams = {
  to: string
  orderId: string
  amount: number
  currency: string
}

export async function sendOrderConfirmationEmail({
  to,
  orderId,
  amount,
  currency,
}: OrderEmailParams) {
  if (!to) {
    console.warn(`No email address for order ${orderId}`)
    return
  }

  await resend.emails.send({
    from: 'orders@yourdomain.com',
    to,
    subject: `Order Confirmed — ${orderId}`,
    html: `
      <h1>Your order is confirmed</h1>
      <p>Order ID: <strong>${orderId}</strong></p>
      <p>Total: <strong>${formatCurrency(amount, currency)}</strong></p>
      <p>Thank you for your purchase. We'll send a shipping notification when your order ships.</p>
    `,
  })
}
```

### Order history page

```typescript
// app/account/orders/page.tsx
import { db } from '@/lib/db'
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'
import { formatCurrency } from '@/lib/utils/currency'

export default async function OrderHistoryPage() {
  const session = await auth()
  if (!session?.user?.email) redirect('/login')

  const orders = await db.order.findMany({
    where: { customerEmail: session.user.email },
    orderBy: { createdAt: 'desc' },
    take: 50,
  })

  return (
    <main className="max-w-3xl mx-auto px-4 py-12">
      <h1 className="text-2xl font-bold mb-8">Order History</h1>

      {orders.length === 0 ? (
        <p className="text-neutral-500">No orders yet.</p>
      ) : (
        <div className="flex flex-col gap-4">
          {orders.map((order) => (
            <div
              key={order.id}
              className="border rounded-xl p-6 flex items-center justify-between"
            >
              <div>
                <p className="font-medium font-mono text-sm text-neutral-500">
                  {order.id}
                </p>
                <p className="text-lg font-semibold mt-1">
                  {formatCurrency(order.amount, order.currency)}
                </p>
                <p className="text-sm text-neutral-400">
                  {new Date(order.createdAt).toLocaleDateString()}
                </p>
              </div>
              <span
                className={`px-3 py-1 rounded-full text-sm font-medium ${
                  order.status === 'paid'
                    ? 'bg-green-100 text-green-800'
                    : 'bg-neutral-100 text-neutral-600'
                }`}
              >
                {order.status}
              </span>
            </div>
          ))}
        </div>
      )}
    </main>
  )
}
```

---

## 10. Tax + Shipping

### Stripe Tax (automatic)

Enable in Checkout Session — Stripe calculates and collects tax automatically based on customer location.

```typescript
// In your checkout session creation:
const session = await stripe.checkout.sessions.create({
  mode: 'payment',
  line_items: [...],
  automatic_tax: { enabled: true },           // auto-calculate tax
  tax_id_collection: { enabled: true },       // allow business buyers to enter VAT/EIN
  success_url: '...',
  cancel_url: '...',
})
```

Prerequisite: Enable Stripe Tax in the Dashboard and configure tax registrations for each jurisdiction where you have nexus.

### Dynamic shipping rates

```typescript
// In your checkout session creation:
const session = await stripe.checkout.sessions.create({
  mode: 'payment',
  line_items: [...],
  shipping_address_collection: {
    allowed_countries: ['US', 'CA', 'GB', 'AU'],
  },
  shipping_options: [
    {
      shipping_rate_data: {
        type: 'fixed_amount',
        fixed_amount: { amount: 0, currency: 'usd' },
        display_name: 'Free shipping',
        delivery_estimate: {
          minimum: { unit: 'business_day', value: 5 },
          maximum: { unit: 'business_day', value: 7 },
        },
      },
    },
    {
      shipping_rate_data: {
        type: 'fixed_amount',
        fixed_amount: { amount: 1500, currency: 'usd' },
        display_name: 'Express shipping',
        delivery_estimate: {
          minimum: { unit: 'business_day', value: 1 },
          maximum: { unit: 'business_day', value: 2 },
        },
      },
    },
  ],
  automatic_tax: { enabled: true },
  success_url: '...',
  cancel_url: '...',
})
```

Pre-create shipping rates in the Dashboard or via API and reference their IDs with `shipping_rate: 'shr_...'` instead of inline `shipping_rate_data` for reuse across sessions.

---

## 11. Security Rules

### Never trust client-side price

```typescript
// WRONG — client sends amount, server blindly uses it
export async function POST(req: NextRequest) {
  const { amount } = await req.json()
  const pi = await stripe.paymentIntents.create({ amount, currency: 'usd' })
  // ...
}

// CORRECT — server fetches price from Stripe using priceId
export async function POST(req: NextRequest) {
  const { priceId, quantity } = await req.json()
  const price = await stripe.prices.retrieve(priceId)
  const amount = (price.unit_amount ?? 0) * quantity
  const pi = await stripe.paymentIntents.create({ amount, currency: price.currency })
  // ...
}
```

### Always verify webhook signatures

```typescript
// WRONG — no signature verification
export async function POST(req: NextRequest) {
  const event = await req.json()
  await processEvent(event)
}

// CORRECT — verify before trusting
export async function POST(req: NextRequest) {
  const body = await req.text()
  const sig = req.headers.get('stripe-signature')!
  const event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!)
  await processEvent(event)
}
```

### Idempotent order fulfillment

```typescript
// Always check before creating to handle Stripe retries
const existing = await db.order.findUnique({
  where: { stripePaymentIntentId: paymentIntent.id },
})
if (existing) return  // already fulfilled — safe to skip
```

### Rate limit your checkout API

```typescript
// app/api/checkout/route.ts
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'),
})

export async function POST(req: NextRequest) {
  const ip = req.headers.get('x-forwarded-for') ?? 'unknown'
  const { success } = await ratelimit.limit(ip)

  if (!success) {
    return NextResponse.json({ error: 'Too many requests' }, { status: 429 })
  }
  // ...
}
```

---

## 12. Anti-Patterns

### Storing card data

```typescript
// NEVER — PCI DSS violation, catastrophic liability
await db.user.update({
  where: { id: userId },
  data: { cardNumber: '4242...', cvv: '123' },
})

// CORRECT — store only the Stripe customer ID and payment method ID
await db.user.update({
  where: { id: userId },
  data: {
    stripeCustomerId: 'cus_...',
    defaultPaymentMethodId: 'pm_...',
  },
})
```

### Client-side price calculation

```typescript
// NEVER — user can modify prices in DevTools
const total = cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0)
await fetch('/api/checkout', { body: JSON.stringify({ total }) })

// CORRECT — send only priceIds and quantities; server computes total from Stripe
await fetch('/api/checkout', {
  body: JSON.stringify({
    items: cartItems.map((i) => ({ priceId: i.priceId, quantity: i.quantity })),
  }),
})
```

### Not handling webhook retries (duplicate fulfillment)

```typescript
// WRONG — creates duplicate orders on Stripe retries
export async function fulfillOrder(pi: Stripe.PaymentIntent) {
  await db.order.create({ data: { stripePaymentIntentId: pi.id, ... } })
}

// CORRECT — idempotency check first
export async function fulfillOrder(pi: Stripe.PaymentIntent) {
  const existing = await db.order.findUnique({ where: { stripePaymentIntentId: pi.id } })
  if (existing) return
  await db.order.create({ data: { stripePaymentIntentId: pi.id, ... } })
}
```

### Fulfilling orders on client redirect alone

```typescript
// WRONG — user can navigate directly to /success without paying
// app/checkout/success/page.tsx
export default function SuccessPage() {
  clearCart()
  createOrder()  // NO — no payment verification
  return <div>Thanks!</div>
}

// CORRECT — fulfill only from webhook; success page just shows confirmation UI
// Webhook: payment_intent.succeeded → fulfill
// Success page: display order details from DB, do not create orders
```

### Hardcoding currency and price in frontend

```typescript
// WRONG — price can drift from Stripe dashboard
const PRODUCT_PRICE = 2999  // hardcoded cents

// CORRECT — fetch from Stripe, display from the price object
const price = await stripe.prices.retrieve('price_...')
const displayPrice = formatCurrency(price.unit_amount!, price.currency)
```

### Not setting API version

```typescript
// WRONG — Stripe API version from dashboard; breaks on account upgrades
const stripe = new Stripe(key)

// CORRECT — pin the API version in code
const stripe = new Stripe(key, { apiVersion: '2024-06-20' })
```
