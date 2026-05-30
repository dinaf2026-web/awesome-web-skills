---
name: web-complete
description: The complete web development skill suite — all 14 specialized skills in one. Covers copywriting, forms, GSAP animations, scrollytelling, micro-interactions, auth, ecommerce, email capture, analytics, OG images, PWA, maps, real-time features, and i18n. Use this as your entry point, then drill into specific skills as needed.
origin: community
tags: [web, complete, full-stack, landing-page, ecommerce, pwa, i18n, realtime, gsap, auth]
---

# web-complete — Master Web Development Skill Suite

## 1. Overview

This skill is the hub for all 14 web development sub-skills organized across three tiers. Use it to orient yourself, then invoke the specific sub-skill for deep patterns.

**How to use this suite:**
- Start here to identify which skills apply to your current build
- Use the Quick Reference Table to pick the right sub-skill
- Follow the Stack Decision Guide when combining skills for a specific project type
- Each skill section below gives 2-3 actionable patterns — the full treatment lives in the named sub-skill

**The three tiers:**
- **Tier 1 — Core:** Patterns every web project needs (copy, forms, motion, scroll)
- **Tier 2 — Integrations:** Third-party systems that require deliberate wiring (auth, payments, email, analytics)
- **Tier 3 — Advanced:** Capabilities that differentiate production apps (PWA, maps, realtime, i18n, OG images)

---

## 2. Quick Reference Table

| Skill | One-line description | Reach for it when... |
|---|---|---|
| web-copywriting | Conversion-first copy patterns | You're writing headlines, CTAs, value props, or pricing copy |
| web-forms | Accessible, validated forms | You need inputs, multi-step flows, or server-side validation |
| web-gsap | GSAP animation system | You want timeline-driven, scroll-triggered, or SVG animations |
| web-scrollytelling | Scroll-driven narrative UI | You're building a story section pinned to scroll progress |
| web-micro-interactions | Hover, focus, feedback states | Buttons, toggles, loaders, skeleton screens need polish |
| web-auth | Authentication flows | Login, signup, OAuth, session management, protected routes |
| web-ecommerce | Cart, checkout, product UI | You're building a store, product page, or checkout flow |
| web-email-capture | Lead capture and opt-in flows | Newsletter signups, waitlists, lead magnets |
| web-analytics | Event tracking and funnels | You need pageview tracking, conversion events, or A/B setup |
| web-og-images | Dynamic Open Graph image generation | Social sharing previews need to be generated per-page |
| web-pwa | Progressive Web App patterns | Offline support, install prompt, service worker, push notifications |
| web-maps | Interactive maps and geolocation | You need embedded maps, markers, clustering, or routing |
| web-realtime | WebSocket and live update patterns | Chat, live scores, collaborative editing, presence indicators |
| web-i18n | Internationalization and localization | Multi-language support, locale routing, RTL layout |

---

## 3. Tier 1 — Core Skills

### web-copywriting

**Goal:** Write copy that converts — not copy that describes.

**Key patterns:**

1. **Problem-first headline structure**
   Lead with the pain, not the product. "Stop losing clients to slow proposals" outperforms "Proposal software for agencies." Follow with the outcome in the subheadline.

2. **CTA specificity rule**
   Replace vague verbs (Get started, Learn more) with outcome verbs tied to the offer. "Send my first proposal free" converts higher than "Sign up."

3. **Pricing page anchoring**
   Always show the highest tier first. Users anchor to the first price they see — everything below feels like a deal.

See `web-copywriting` for full patterns: benefit ladders, objection handling, microcopy for error states, and testimonial placement rules.

---

### web-forms

**Goal:** Forms that are accessible, validate correctly, and feel fast.

**Key patterns:**

1. **Inline validation timing**
   Validate on `blur` for most fields, not on `input`. Exception: password strength meters validate live. Never validate on submit-only — users hate discovering errors late.

2. **Error message anatomy**
   Every error needs three parts: what went wrong, why it matters, how to fix it. "Invalid email" fails all three. "Email must include @ — check for typos like gmail,com" passes.

3. **Multi-step form state**
   Persist step data to `sessionStorage` on every field change. Users lose progress on accidental back-navigation — this is a silent conversion killer.

See `web-forms` for full patterns: fieldset grouping, ARIA live regions, file upload UX, and server-side validation error mapping.

---

### web-gsap

**Goal:** Timeline-driven animations with scroll integration.

**Key patterns:**

1. **Timeline registration**
   Always register plugins at the top of the module, not inside components or effects. `gsap.registerPlugin(ScrollTrigger, SplitText)` called twice causes duplicate registration warnings.

2. **ScrollTrigger cleanup**
   In React/Vue, kill all ScrollTrigger instances in the cleanup function. Use `ScrollTrigger.getAll().forEach(t => t.kill())` or scope to a context: `ctx.revert()`.

3. **Stagger from center**
   For grid reveals, `stagger: { from: "center", amount: 0.6 }` reads more intentional than sequential stagger. Combine with `autoAlpha` (not `opacity`) so hidden elements don't take up interaction area.

See `web-gsap` for full patterns: SVG path drawing, morphing, flip animations, and matchMedia for reduced-motion.

---

### web-scrollytelling

**Goal:** Scroll-driven narrative where the story advances as the user scrolls.

**Key patterns:**

1. **Pin + scrub anatomy**
   The trigger element is the section container. The pinned element is the sticky visual. `pin: true`, `scrub: 1`, `start: "top top"`, `end: "+=300%"`. The `scrub` value (1-2) controls lag — lower feels snappier.

2. **Progress-driven state**
   Map `self.progress` (0-1) to discrete story beats with `gsap.utils.mapRange`. Avoid too-fine-grained updates — 4-6 beats per section is readable; 20 beats is overwhelming.

3. **Mobile fallback**
   Pinning on mobile causes layout thrash. Use `ScrollTrigger.matchMedia` to swap scroll-driven animations for static reveals below 768px.

See `web-scrollytelling` for full patterns: horizontal scroll, canvas-based sequences, video scrubbing, and performance profiling.

---

### web-micro-interactions

**Goal:** Every interactive element has a designed hover, focus, active, and feedback state.

**Key patterns:**

1. **Button states as a system**
   Define all five states in CSS custom properties: default, hover, focus-visible, active, disabled. Disabled state must not use `pointer-events: none` alone — add `aria-disabled` and handle keyboard.

2. **Skeleton screens over spinners**
   Spinners communicate waiting with no information. Skeletons communicate the shape of what's loading. Use `animation: shimmer 1.5s infinite` on a gradient background-position shift.

3. **Optimistic UI pattern**
   On form submit, immediately update the UI to the success state. Roll back with an error toast if the request fails. Never make users wait for confirmation of low-risk actions.

See `web-micro-interactions` for full patterns: toggle animations, drag handles, progress indicators, and notification badge behavior.

---

## 4. Tier 2 — Integration Skills

### web-auth

**Goal:** Secure, usable authentication without reinventing session management.

**Key patterns:**

1. **Use a provider, not homebrew**
   Clerk, Auth.js, Supabase Auth, or Firebase Auth. Rolling your own session tokens, password hashing, and refresh logic is a security liability — not a feature.

2. **Protected route pattern**
   Check session server-side on every protected request, not just on client mount. Client-side guards are UX only — they are not security.

3. **OAuth error handling**
   Always handle `error` and `error_description` query params on the callback route. Users who deny OAuth consent land back on your app — if you don't handle the error param, they see a broken page.

See `web-auth` for full patterns: magic links, passkeys, role-based access, JWT rotation, and middleware guard patterns.

---

### web-ecommerce

**Goal:** Cart, product, and checkout flows that convert.

**Key patterns:**

1. **Cart persistence**
   Sync cart to `localStorage` and to a server-side session if the user is logged in. Anonymous carts must survive page reload. Merge anonymous + logged-in carts on sign-in.

2. **Checkout field minimalism**
   Every additional field reduces conversion. Collect only what you need to ship. Defer account creation to post-purchase — "guest checkout" is not optional.

3. **Product image zoom**
   Mobile: pinch-to-zoom on the native image. Desktop: CSS `transform: scale()` on hover with `overflow: hidden` on the container. Never implement custom touch handlers for zoom.

See `web-ecommerce` for full patterns: Stripe integration, inventory checks, shipping calculator, upsell placement, and abandoned cart recovery.

---

### web-email-capture

**Goal:** High-converting lead capture with deliverability hygiene.

**Key patterns:**

1. **Exit-intent timing**
   Trigger the modal on `mouseleave` targeting the `document` (not the window). Add a 3-second page-time minimum and a 30-day cookie to suppress repeat shows.

2. **Double opt-in flow**
   Send confirmation immediately on signup. Confirmation email subject line: "Confirm your email — one click" — not "Welcome." Unconfirmed subscribers degrade sender reputation.

3. **Lead magnet specificity**
   "Free guide" converts less than "The 7-step checklist we use to cut proposal time in half — PDF." Specificity signals value.

See `web-email-capture` for full patterns: Mailchimp/ConvertKit/Resend integration, inline vs. modal placement testing, and GDPR consent handling.

---

### web-analytics

**Goal:** Accurate event tracking that connects to business outcomes.

**Key patterns:**

1. **Event naming convention**
   Use `object_action` format: `signup_completed`, `checkout_started`, `video_played`. Avoid vague names like `click` or `button_pressed` — they are unactionable in reports.

2. **Funnel instrumentation order**
   Track the exit point before the entry point. If checkout completion is the goal, instrument `checkout_completed` first, then work backward through `checkout_started`, `cart_viewed`, `product_viewed`.

3. **First-party over third-party**
   Plausible, PostHog, or Umami over GA4 for privacy-first projects. If using GA4, proxy the script through your own domain to survive adblocker blocking.

See `web-analytics` for full patterns: custom dimensions, cohort tracking, A/B test result reading, and Segment integration.

---

## 5. Tier 3 — Advanced Skills

### web-og-images

**Goal:** Per-page Open Graph images generated dynamically at build or request time.

**Key patterns:**

1. **Vercel OG / @vercel/og**
   Use JSX-in-edge-function approach for Next.js. Generate at the edge, cache with `Cache-Control: public, max-age=31536000, immutable`. Regenerate only when content changes.

2. **Design constraints**
   OG images render at 1200x630. Use minimum 48px font size — smaller type disappears in feed previews. Test in the Twitter Card Validator and LinkedIn Post Inspector before shipping.

3. **Fallback image**
   Always define a static `/og-default.png` in `<meta>` as the fallback. Dynamic generation can fail; a missing OG image makes shared links look broken.

See `web-og-images` for full patterns: Satori for non-Next.js, font loading in edge functions, screenshot-based generation with Playwright, and cache invalidation.

---

### web-pwa

**Goal:** Installable, offline-capable web apps with push notification support.

**Key patterns:**

1. **Service worker scope**
   Register from the root (`/sw.js`) so the scope covers the entire app. A service worker at `/app/sw.js` only controls routes under `/app/`.

2. **Cache strategy by resource type**
   - HTML: network-first (always fresh shell)
   - JS/CSS with hash: cache-first (immutable assets)
   - API responses: stale-while-revalidate
   - Images: cache-first with 30-day expiry

3. **Install prompt deferral**
   Capture `beforeinstallprompt`, suppress the browser default, and show your own install CTA at a moment of demonstrated value (after 3 visits, after a key action). Do not show it on first page load.

See `web-pwa` for full patterns: Workbox configuration, push notification subscription flow, background sync, and iOS Safari install prompt workarounds.

---

### web-maps

**Goal:** Performant, accessible maps with markers and optional routing.

**Key patterns:**

1. **Lazy-load the map library**
   Mapbox GL JS is 280kb gzipped. Load it only when the map container enters the viewport using IntersectionObserver. Placeholder: a static image of the map area via the Static Tiles API.

2. **Marker clustering**
   Above 50 markers, clustering is required for performance and readability. Use `supercluster` for Mapbox or the built-in clustering for Google Maps. Uncluster at zoom 14+.

3. **Accessibility**
   Maps are not keyboard-navigable by default. Provide a text alternative: a list of locations with addresses. Use `role="application"` on the map container and `aria-label` describing what the map shows.

See `web-maps` for full patterns: Mapbox vs. Google Maps decision matrix, custom marker design, geolocation, routing via Directions API, and choropleth overlays.

---

### web-realtime

**Goal:** Live updates, presence, and collaborative features with minimal backend complexity.

**Key patterns:**

1. **WebSocket vs. SSE decision**
   Use Server-Sent Events (SSE) for one-way server-to-client streams (notifications, live scores, feed updates). Use WebSocket for bidirectional (chat, collaborative editing, presence). SSE works through HTTP/2 multiplexing — no upgrade required.

2. **Reconnection with exponential backoff**
   Never reconnect immediately on disconnect. Start at 1s, double each attempt, cap at 30s, add jitter. Without backoff, a server restart causes a thundering herd of simultaneous reconnections.

3. **Optimistic UI + conflict resolution**
   For collaborative features, apply local changes immediately. On server conflict (someone else edited the same field), show a non-blocking inline diff — never silently discard the user's change.

See `web-realtime` for full patterns: Supabase Realtime, Pusher/Ably integration, CRDT basics, presence indicators, and typing state broadcasting.

---

### web-i18n

**Goal:** Multi-language support with locale routing, RTL layout, and number/date formatting.

**Key patterns:**

1. **Locale routing strategy**
   Prefer path-based routing (`/en/`, `/fr/`) over subdomain (`en.site.com`) for SEO and caching simplicity. Use `hreflang` tags on every page pointing to all locale variants.

2. **Translation key naming**
   Use dot-notated namespaces: `checkout.summary.total`, `nav.links.about`. Flat keys (`checkout_summary_total`) become unmanageable past 200 strings. Group by page or feature, not by component.

3. **RTL layout**
   Use CSS logical properties (`margin-inline-start` not `margin-left`, `padding-block-end` not `padding-bottom`) from the start. RTL retrofit of physical properties is expensive. Set `dir="rtl"` on `<html>` and test Hebrew or Arabic early.

See `web-i18n` for full patterns: next-intl / i18next / Paraglide setup, pluralization rules, date/currency formatting with `Intl`, and translation file management with Crowdin or Localazy.

---

## 6. Stack Decision Guide

### Landing page (marketing, SaaS, product)

**Required:** web-copywriting, web-forms, web-gsap, web-micro-interactions, web-og-images, web-analytics, web-email-capture

**Optional:** web-scrollytelling (hero story section), web-i18n (multi-market)

**Avoid overbuilding:** auth, maps, realtime — none of these belong on a landing page unless the product requires them.

---

### SaaS app (logged-in product)

**Required:** web-auth, web-forms, web-analytics, web-micro-interactions, web-realtime (if collaborative), web-pwa (if mobile-critical)

**Optional:** web-i18n (if selling internationally), web-og-images (if user-shareable content), web-email-capture (for in-app upsell)

**Skip:** web-scrollytelling, web-gsap (unless you have a marketing page attached)

---

### Ecommerce store

**Required:** web-ecommerce, web-forms, web-analytics, web-micro-interactions, web-og-images, web-email-capture

**Optional:** web-auth (accounts vs. guest checkout tradeoff — see web-auth), web-maps (store locator), web-pwa (repeat buyer mobile experience)

**Skip:** web-realtime, web-scrollytelling (unless brand storytelling is a differentiator)

---

### Portfolio / creative site

**Required:** web-gsap, web-scrollytelling, web-micro-interactions, web-og-images, web-copywriting

**Optional:** web-forms (contact form), web-analytics (lightweight — Plausible)

**Skip:** auth, ecommerce, realtime, i18n — unless the brief specifically calls for them.

---

### Content site / blog / publication

**Required:** web-copywriting, web-analytics, web-email-capture, web-og-images, web-i18n (if multi-language), web-pwa (if offline reading matters)

**Optional:** web-forms (comments, contact), web-maps (if location content)

**Skip:** ecommerce, realtime, auth (unless subscriber-gated content)

---

## 7. Related Skills

- **ui-ux-pro-max** — Design system patterns, component hierarchy, and UX decision frameworks. Use this when you need to design before you build.
- **frontend-design-direction** — Visual direction setting: choosing a style, building a palette, typography pairing. Use this before writing any CSS.
- **motion-advanced** — Deep GSAP, Framer Motion, and CSS animation patterns beyond what web-gsap covers. Use for complex timeline choreography or physics-based motion.
- **remotion-video-creation** — Programmatic video generation with React. Use when you need rendered video exports, not just in-browser animation.
- **awesome-websites** — Curated references and teardowns of exceptional web work. Use for inspiration and benchmarking before starting a high-polish project.
