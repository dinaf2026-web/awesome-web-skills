# awesome-web-skills — Claude Code Skill Suite

14 production-ready Claude Code skills for building modern websites. Install one or all.

## Skills Included

### Tier 1 — Core
- `/web-copywriting` — Headlines, CTAs, microcopy, value propositions that convert
- `/web-forms` — React Hook Form + Zod + Server Actions, multi-step, file upload
- `/web-animations-gsap` — GSAP ScrollTrigger, timelines, text animations, SVG
- `/web-scrollytelling` — Pinned panels, scroll-driven narratives, horizontal scroll
- `/web-micro-interactions` — Skeleton screens, toasts, optimistic UI, hover states

### Tier 2 — Integrations
- `/web-auth` — NextAuth v5, Clerk, Supabase Auth, RBAC, protected routes
- `/web-ecommerce` — Stripe Checkout, cart, webhooks, subscriptions
- `/web-email-capture` — Resend, Beehiiv, Mailchimp, drip sequences, opt-in forms
- `/web-analytics` — GA4, PostHog, Vercel Analytics, event tracking, funnels

### Tier 3 — Advanced
- `/web-og-images` — Dynamic OG images with @vercel/og, per-page social cards
- `/web-pwa` — Service workers, offline mode, install prompts, push notifications
- `/web-maps` — Mapbox GL, Leaflet, clustering, location search, geo-filtering
- `/web-realtime` — WebSockets, SSE, Pusher, presence indicators, live updates
- `/web-i18n` — next-intl, locale routing, RTL, pluralization, language switcher

### Master Skill
- `/web-complete` — All 14 skills in one hub. Start here.

## Install

```powershell
# Windows (PowerShell)
$dest = "$env:USERPROFILE\.claude\skills"
Get-ChildItem -Directory | ForEach-Object { Copy-Item $_.FullName $dest -Recurse -Force }
```

```bash
# macOS / Linux
cp -r web-* ~/.claude/skills/
```

## Stack
Next.js 14 App Router · TypeScript · Tailwind CSS · shadcn/ui · Framer Motion · GSAP

## Author
[@dinaf2026-web](https://github.com/dinaf2026-web)

## License
MIT
