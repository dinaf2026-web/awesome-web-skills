---
name: web-animations-gsap
description: GSAP animation patterns for modern websites — ScrollTrigger, timelines, text animations, SVG morphing, and scroll-driven storytelling. More powerful than CSS or Framer Motion for complex sequences.
origin: community
tags: [gsap, animation, scrolltrigger, timeline, svg, web-animation]
---

# GSAP Animation Patterns

## 1. When to Use GSAP vs Framer Motion vs CSS

| Scenario | Winner | Reason |
|----------|--------|--------|
| Simple hover/toggle transitions | CSS | Zero JS overhead |
| Component enter/exit animations | Framer Motion | React-idiomatic, AnimatePresence |
| Gesture-driven drag interactions | Framer Motion | Built-in gesture API |
| Multi-step sequenced timelines | GSAP | Precise control, labels, callbacks |
| Scroll-linked scrubbed animations | GSAP ScrollTrigger | Best-in-class scrub fidelity |
| SVG path drawing / morphing | GSAP | drawSVG + morphSVG plugins |
| Text character/word reveals | GSAP SplitText | Line-aware splitting with reflow |
| 60fps parallax layers | GSAP | Ticker + compositor-only transforms |
| Counter/number animations | GSAP | Built-in number tweening |
| Canvas / WebGL orchestration | GSAP | Ticker drives any JS value |
| Simple page-load fade | CSS or Framer Motion | Overkill for GSAP |
| Reduced-motion users | CSS | Use `prefers-reduced-motion` media query first |

**Rule of thumb:** If you need a scrubbed scroll timeline, a multi-step sequence with labels, or SVG manipulation — reach for GSAP. For component-level presence animations in React, Framer Motion wins on ergonomics.

---

## 2. Setup in Next.js

```bash
npm install gsap
# Club plugins (SplitText, DrawSVG, MorphSVG, ScrambleText) require GSAP Club membership
# npm install gsap@npm:@gsap/shockingly  # Club tier
```

```typescript
// lib/gsap.ts — centralized registration, SSR-safe
import { gsap } from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { SplitText } from 'gsap/SplitText'
import { DrawSVGPlugin } from 'gsap/DrawSVGPlugin'
import { MorphSVGPlugin } from 'gsap/MorphSVGPlugin'
import { ScrambleTextPlugin } from 'gsap/ScrambleTextPlugin'

// Register once — safe to call multiple times
if (typeof window !== 'undefined') {
  gsap.registerPlugin(
    ScrollTrigger,
    SplitText,
    DrawSVGPlugin,
    MorphSVGPlugin,
    ScrambleTextPlugin,
  )
}

export { gsap, ScrollTrigger, SplitText, DrawSVGPlugin, MorphSVGPlugin, ScrambleTextPlugin }
```

```typescript
// hooks/useGsap.ts — cleanup-safe animation hook
import { useEffect, useRef, DependencyList } from 'react'
import { gsap } from '@/lib/gsap'

/**
 * Drop-in replacement for useEffect that auto-reverts all GSAP
 * animations when the component unmounts or deps change.
 */
export function useGsap(
  callback: (context: gsap.Context) => void,
  deps: DependencyList = [],
) {
  const contextRef = useRef<gsap.Context | null>(null)

  useEffect(() => {
    contextRef.current = gsap.context(callback)
    return () => {
      contextRef.current?.revert()
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, deps)
}
```

```typescript
// Usage in any component
'use client'
import { useRef } from 'react'
import { gsap } from '@/lib/gsap'
import { useGsap } from '@/hooks/useGsap'

export function FadeInBox() {
  const boxRef = useRef<HTMLDivElement>(null)

  useGsap((ctx) => {
    ctx.add(() => {
      gsap.from(boxRef.current, { opacity: 0, y: 40, duration: 0.8 })
    })
  }, [])

  return <div ref={boxRef} className="box" />
}
```

---

## 3. ScrollTrigger Fundamentals

```typescript
'use client'
import { useRef } from 'react'
import { gsap } from '@/lib/gsap'
import { ScrollTrigger } from '@/lib/gsap'
import { useGsap } from '@/hooks/useGsap'

export function ScrollSection() {
  const sectionRef = useRef<HTMLElement>(null)
  const contentRef = useRef<HTMLDivElement>(null)

  useGsap(() => {
    // Pin + scrub — element stays fixed while scrolling through its own height
    gsap.from(contentRef.current, {
      x: 200,
      opacity: 0,
      duration: 1,
      scrollTrigger: {
        trigger: sectionRef.current,
        start: 'top 80%',      // when top of trigger hits 80% from top of viewport
        end: 'bottom 20%',     // when bottom of trigger hits 20% from top of viewport
        scrub: 1,              // smooth scrub — number = seconds of lag behind scroll
        pin: false,            // set true to pin the trigger element
        markers: process.env.NODE_ENV === 'development', // dev-only visual guides
        toggleActions: 'play none none reverse', // onEnter onLeave onEnterBack onLeaveBack
        onEnter: () => console.log('entered'),
        onLeave: () => console.log('left'),
      },
    })
  }, [])

  return (
    <section ref={sectionRef} className="min-h-screen">
      <div ref={contentRef}>Content</div>
    </section>
  )
}

// Pinned scrollytelling panel
export function PinnedStory() {
  const containerRef = useRef<HTMLDivElement>(null)
  const panelsRef = useRef<HTMLDivElement[]>([])

  useGsap(() => {
    const panels = panelsRef.current

    // Horizontal scroll pinned section
    gsap.to(panels, {
      xPercent: -100 * (panels.length - 1),
      ease: 'none',
      scrollTrigger: {
        trigger: containerRef.current,
        pin: true,
        scrub: 1,
        snap: 1 / (panels.length - 1),
        end: () => `+=${containerRef.current?.offsetWidth ?? 0}`,
      },
    })
  }, [])

  return (
    <div ref={containerRef} className="overflow-hidden flex w-[300vw]">
      {['Panel 1', 'Panel 2', 'Panel 3'].map((label, i) => (
        <div
          key={label}
          ref={(el) => { if (el) panelsRef.current[i] = el }}
          className="w-screen h-screen flex-shrink-0 flex items-center justify-center"
        >
          {label}
        </div>
      ))}
    </div>
  )
}
```

### start/end Syntax Reference

```
"top top"       → top of element meets top of viewport
"top center"    → top of element meets center of viewport
"top 80%"       → top of element meets 80% down from viewport top
"top +=200"     → top of element + 200px offset
"center center" → centers meet
"bottom bottom" → bottom of element meets bottom of viewport
"+=500"         → 500px after the start trigger (no element reference)
```

---

## 4. Timeline Sequences

```typescript
'use client'
import { useRef } from 'react'
import { gsap } from '@/lib/gsap'
import { ScrollTrigger } from '@/lib/gsap'
import { useGsap } from '@/hooks/useGsap'

export function HeroTimeline() {
  const containerRef = useRef<HTMLDivElement>(null)

  useGsap(() => {
    const tl = gsap.timeline({
      defaults: { ease: 'power3.out', duration: 0.8 },
      scrollTrigger: {
        trigger: containerRef.current,
        start: 'top 70%',
        once: true, // play once — don't reverse on scroll back
      },
    })

    // Labels as named anchor points
    tl.addLabel('start')
      .from('.hero-eyebrow', { y: 20, opacity: 0 }, 'start')
      .from('.hero-headline', { y: 40, opacity: 0 }, 'start+=0.15')
      .from('.hero-subline', { y: 30, opacity: 0 }, 'start+=0.3')
      .addLabel('cta')
      .from('.hero-cta', { scale: 0.9, opacity: 0, duration: 0.5 }, 'cta')
      .from('.hero-image', { x: 60, opacity: 0, duration: 1 }, 'start+=0.2')

    return () => tl.kill()
  }, [])

  return (
    <div ref={containerRef}>
      <p className="hero-eyebrow">Eyebrow</p>
      <h1 className="hero-headline">Headline</h1>
      <p className="hero-subline">Subline copy</p>
      <button className="hero-cta">Get Started</button>
      <img className="hero-image" src="/hero.jpg" alt="" />
    </div>
  )
}

// Stagger — animate a collection with offset delay
export function StaggerCards() {
  const listRef = useRef<HTMLUListElement>(null)

  useGsap(() => {
    const tl = gsap.timeline({
      scrollTrigger: {
        trigger: listRef.current,
        start: 'top 75%',
        once: true,
      },
    })

    tl.from(listRef.current?.querySelectorAll('.card') ?? [], {
      y: 50,
      opacity: 0,
      stagger: {
        each: 0.1,          // 100ms between each
        from: 'start',      // 'start' | 'end' | 'center' | 'random' | index
        ease: 'power2.inOut',
      },
      duration: 0.6,
    })
  }, [])

  return (
    <ul ref={listRef} className="grid grid-cols-3 gap-6">
      {Array.from({ length: 6 }).map((_, i) => (
        <li key={i} className="card bg-white rounded-xl p-6 shadow">Card {i + 1}</li>
      ))}
    </ul>
  )
}

// Nested timelines for modular composition
function buildCardTimeline(el: HTMLElement): gsap.core.Timeline {
  return gsap.timeline()
    .from(el.querySelector('.card-image'), { scale: 1.1, duration: 0.6 })
    .from(el.querySelector('.card-title'), { y: 20, opacity: 0, duration: 0.4 }, '-=0.2')
    .from(el.querySelector('.card-body'), { y: 10, opacity: 0, duration: 0.3 }, '-=0.1')
}

export function FeatureSection() {
  const sectionRef = useRef<HTMLElement>(null)

  useGsap(() => {
    const master = gsap.timeline({
      scrollTrigger: { trigger: sectionRef.current, start: 'top 60%', once: true },
    })

    sectionRef.current?.querySelectorAll('.feature-card').forEach((card, i) => {
      master.add(buildCardTimeline(card as HTMLElement), i * 0.15)
    })
  }, [])

  return <section ref={sectionRef}>{/* cards */}</section>
}
```

---

## 5. Text Animations

```typescript
'use client'
import { useRef } from 'react'
import { gsap, SplitText } from '@/lib/gsap'
import { ScrollTrigger } from '@/lib/gsap'
import { useGsap } from '@/hooks/useGsap'

// Word reveal — mask each word with overflow:hidden wrapper
export function WordReveal({ text }: { text: string }) {
  const headingRef = useRef<HTMLHeadingElement>(null)

  useGsap(() => {
    const split = new SplitText(headingRef.current, {
      type: 'words,lines',
      linesClass: 'line-wrapper overflow-hidden', // mask container
    })

    const tl = gsap.timeline({
      scrollTrigger: {
        trigger: headingRef.current,
        start: 'top 85%',
        once: true,
      },
    })

    tl.from(split.words, {
      y: '110%',          // slide up from below the line mask
      opacity: 0,
      duration: 0.7,
      ease: 'power4.out',
      stagger: 0.04,
    })

    return () => split.revert() // restore original DOM
  }, [])

  return <h2 ref={headingRef}>{text}</h2>
}

// Character reveal — cinematic letter-by-letter
export function CharReveal({ text }: { text: string }) {
  const ref = useRef<HTMLHeadingElement>(null)

  useGsap(() => {
    const split = new SplitText(ref.current, { type: 'chars' })

    gsap.from(split.chars, {
      opacity: 0,
      y: 20,
      rotateX: -90,
      stagger: 0.02,
      duration: 0.5,
      ease: 'back.out(1.7)',
      scrollTrigger: { trigger: ref.current, start: 'top 80%', once: true },
    })

    return () => split.revert()
  }, [])

  return <h1 ref={ref} style={{ perspective: '600px' }}>{text}</h1>
}

// Typewriter — cursor blink effect
export function Typewriter({ phrases }: { phrases: string[] }) {
  const elRef = useRef<HTMLSpanElement>(null)

  useGsap(() => {
    let phraseIndex = 0

    const type = () => {
      const phrase = phrases[phraseIndex % phrases.length]
      phraseIndex++

      gsap.to(elRef.current, {
        duration: phrase.length * 0.06,
        text: { value: phrase, delimiter: '' },
        ease: 'none',
        onComplete: () => {
          gsap.delayedCall(1.5, erase)
        },
      })
    }

    const erase = () => {
      gsap.to(elRef.current, {
        duration: 0.4,
        text: { value: '', delimiter: '' },
        ease: 'none',
        onComplete: () => gsap.delayedCall(0.3, type),
      })
    }

    type()
  }, [])

  return (
    <span>
      <span ref={elRef} />
      <span className="animate-blink">|</span>
    </span>
  )
}

// Scramble text (Club plugin)
export function ScrambleReveal({ text }: { text: string }) {
  const ref = useRef<HTMLParagraphElement>(null)

  useGsap(() => {
    gsap.to(ref.current, {
      duration: 1.5,
      scrambleText: {
        text,
        chars: 'upperCase',
        revealDelay: 0.3,
        speed: 0.4,
      },
      scrollTrigger: { trigger: ref.current, start: 'top 80%', once: true },
    })
  }, [])

  return <p ref={ref}>{text}</p>
}
```

---

## 6. Scroll-Driven Counters

```typescript
'use client'
import { useRef } from 'react'
import { gsap } from '@/lib/gsap'
import { ScrollTrigger } from '@/lib/gsap'
import { useGsap } from '@/hooks/useGsap'

interface Stat {
  label: string
  end: number
  prefix?: string
  suffix?: string
  decimals?: number
}

const STATS: Stat[] = [
  { label: 'Projects shipped', end: 142, suffix: '+' },
  { label: 'Uptime', end: 99.97, suffix: '%', decimals: 2 },
  { label: 'Revenue generated', end: 4.2, prefix: '$', suffix: 'M' },
]

export function StatsCounter() {
  const sectionRef = useRef<HTMLElement>(null)
  const valueRefs = useRef<HTMLSpanElement[]>([])

  useGsap(() => {
    STATS.forEach((stat, i) => {
      const el = valueRefs.current[i]
      if (!el) return

      const obj = { value: 0 }

      gsap.to(obj, {
        value: stat.end,
        duration: 2,
        ease: 'power2.out',
        onUpdate() {
          const formatted = obj.value.toFixed(stat.decimals ?? 0)
          el.textContent = `${stat.prefix ?? ''}${formatted}${stat.suffix ?? ''}`
        },
        scrollTrigger: {
          trigger: el,
          start: 'top 85%',
          once: true,
        },
      })
    })
  }, [])

  return (
    <section ref={sectionRef} className="grid grid-cols-3 gap-12 py-24">
      {STATS.map((stat, i) => (
        <div key={stat.label} className="text-center">
          <div className="text-6xl font-bold tabular-nums">
            <span
              ref={(el) => { if (el) valueRefs.current[i] = el }}
            >
              {stat.prefix ?? ''}0{stat.suffix ?? ''}
            </span>
          </div>
          <p className="mt-2 text-sm text-gray-500">{stat.label}</p>
        </div>
      ))}
    </section>
  )
}
```

---

## 7. Parallax Layers

```typescript
'use client'
import { useRef } from 'react'
import { gsap } from '@/lib/gsap'
import { ScrollTrigger } from '@/lib/gsap'
import { useGsap } from '@/hooks/useGsap'

// Multi-speed depth parallax
export function ParallaxHero() {
  const sectionRef = useRef<HTMLElement>(null)

  useGsap(() => {
    const layers = [
      { selector: '.parallax-bg', speed: -0.5 },     // slowest — background
      { selector: '.parallax-mid', speed: -0.25 },   // midground
      { selector: '.parallax-fg', speed: 0.1 },      // foreground — moves opposite
      { selector: '.parallax-text', speed: -0.15 },  // text
    ]

    layers.forEach(({ selector, speed }) => {
      const el = sectionRef.current?.querySelector(selector)
      if (!el) return

      gsap.to(el, {
        yPercent: speed * 100,
        ease: 'none',
        scrollTrigger: {
          trigger: sectionRef.current,
          start: 'top bottom',
          end: 'bottom top',
          scrub: true, // boolean true = instant, number = lag seconds
        },
      })
    })
  }, [])

  return (
    <section ref={sectionRef} className="relative h-screen overflow-hidden">
      <div className="parallax-bg absolute inset-0 scale-125">
        <img src="/bg.jpg" alt="" className="w-full h-full object-cover" />
      </div>
      <div className="parallax-mid absolute inset-0 flex items-center justify-center">
        <img src="/midground.png" alt="" />
      </div>
      <div className="parallax-fg absolute bottom-0 w-full">
        <img src="/foreground.png" alt="" className="w-full" />
      </div>
      <div className="parallax-text absolute inset-0 flex items-center justify-center">
        <h1 className="text-white text-8xl font-black">Parallax</h1>
      </div>
    </section>
  )
}

// Card grid with individual parallax offset
export function ParallaxGrid({ items }: { items: string[] }) {
  const gridRef = useRef<HTMLDivElement>(null)

  useGsap(() => {
    // Odd columns shift up, even columns shift down for staggered depth
    const cards = gridRef.current?.querySelectorAll('.para-card')
    cards?.forEach((card, i) => {
      const direction = i % 2 === 0 ? -1 : 1
      gsap.to(card, {
        yPercent: direction * 15,
        ease: 'none',
        scrollTrigger: {
          trigger: gridRef.current,
          start: 'top bottom',
          end: 'bottom top',
          scrub: 1,
        },
      })
    })
  }, [])

  return (
    <div ref={gridRef} className="grid grid-cols-3 gap-6">
      {items.map((item) => (
        <div key={item} className="para-card bg-white rounded-2xl p-8 shadow-lg">
          {item}
        </div>
      ))}
    </div>
  )
}
```

---

## 8. SVG Path Drawing

```typescript
'use client'
import { useRef } from 'react'
import { gsap, DrawSVGPlugin, MorphSVGPlugin } from '@/lib/gsap'
import { ScrollTrigger } from '@/lib/gsap'
import { useGsap } from '@/hooks/useGsap'

// DrawSVG — animate stroke-dashoffset to reveal paths
export function PathReveal() {
  const svgRef = useRef<SVGSVGElement>(null)

  useGsap(() => {
    // Set all paths to 0% drawn
    gsap.set(svgRef.current?.querySelectorAll('path, circle, line') ?? [], {
      drawSVG: '0%',
    })

    const tl = gsap.timeline({
      scrollTrigger: {
        trigger: svgRef.current,
        start: 'top 70%',
        once: true,
      },
    })

    // Draw from 0% to 100% — left to right
    tl.to('.svg-stroke-1', { drawSVG: '100%', duration: 1.2, ease: 'power2.inOut' })
      .to('.svg-stroke-2', { drawSVG: '100%', duration: 0.8 }, '-=0.4')
      // Draw only a segment: start% to end%
      .to('.svg-stroke-3', { drawSVG: '30% 70%', duration: 0.6 }, '-=0.2')
  }, [])

  return (
    <svg ref={svgRef} viewBox="0 0 400 300">
      <path className="svg-stroke-1" d="M 0 150 Q 200 0 400 150" fill="none" stroke="currentColor" strokeWidth="2" />
      <circle className="svg-stroke-2" cx="200" cy="150" r="60" fill="none" stroke="currentColor" strokeWidth="2" />
      <line className="svg-stroke-3" x1="0" y1="200" x2="400" y2="200" stroke="currentColor" strokeWidth="2" />
    </svg>
  )
}

// MorphSVG — shape-shift between paths
export function MorphIcon({ isActive }: { isActive: boolean }) {
  const pathRef = useRef<SVGPathElement>(null)

  // Path data for two states
  const hamburgerPath = 'M 5 10 L 35 10 M 5 20 L 35 20 M 5 30 L 35 30'
  const closePath = 'M 8 8 L 32 32 M 32 8 L 8 32'

  useGsap(() => {
    if (!pathRef.current) return

    gsap.to(pathRef.current, {
      morphSVG: isActive ? closePath : hamburgerPath,
      duration: 0.4,
      ease: 'power2.inOut',
    })
  }, [isActive])

  return (
    <svg viewBox="0 0 40 40" width="40" height="40">
      <path
        ref={pathRef}
        d={hamburgerPath}
        fill="none"
        stroke="currentColor"
        strokeWidth="2"
        strokeLinecap="round"
      />
    </svg>
  )
}

// Animated SVG line chart reveal
export function ChartReveal() {
  const svgRef = useRef<SVGSVGElement>(null)

  useGsap(() => {
    const paths = svgRef.current?.querySelectorAll('.chart-line')

    gsap.set(paths ?? [], { drawSVG: '0%' })

    gsap.to(paths ?? [], {
      drawSVG: '100%',
      duration: 1.5,
      ease: 'power1.inOut',
      stagger: 0.2,
      scrollTrigger: {
        trigger: svgRef.current,
        start: 'top 75%',
        once: true,
      },
    })
  }, [])

  return (
    <svg ref={svgRef} viewBox="0 0 600 300">
      <polyline
        className="chart-line"
        points="0,250 100,180 200,200 300,100 400,120 500,60 600,80"
        fill="none"
        stroke="#6366f1"
        strokeWidth="3"
      />
    </svg>
  )
}
```

---

## 9. Page Transitions

```typescript
// components/PageTransition.tsx
'use client'
import { useEffect, useRef } from 'react'
import { usePathname } from 'next/navigation'
import { gsap } from '@/lib/gsap'

export function PageTransition({ children }: { children: React.ReactNode }) {
  const wrapperRef = useRef<HTMLDivElement>(null)
  const curtainRef = useRef<HTMLDivElement>(null)
  const pathname = usePathname()

  // Enter animation on mount
  useEffect(() => {
    const ctx = gsap.context(() => {
      const tl = gsap.timeline()

      // Curtain wipes away to reveal page
      tl.set(curtainRef.current, { scaleX: 1, transformOrigin: 'right center' })
        .to(curtainRef.current, {
          scaleX: 0,
          duration: 0.6,
          ease: 'power3.inOut',
        })
        .from(
          wrapperRef.current,
          { opacity: 0, y: 20, duration: 0.4, ease: 'power2.out' },
          '-=0.2',
        )
    })

    return () => ctx.revert()
  }, [pathname])

  return (
    <div className="relative">
      {/* Curtain overlay */}
      <div
        ref={curtainRef}
        className="fixed inset-0 z-50 bg-black pointer-events-none page-curtain"
        style={{ transformOrigin: 'left center' }}
      />
      <div ref={wrapperRef}>{children}</div>
    </div>
  )
}

// hooks/usePageLeave.ts — trigger exit animation before navigation
import { useRouter } from 'next/navigation'
import { gsap } from '@/lib/gsap'
import { useCallback } from 'react'

export function usePageLeave() {
  const router = useRouter()

  const navigateTo = useCallback((href: string) => {
    const curtain = document.querySelector('.page-curtain') as HTMLElement
    if (!curtain) {
      router.push(href)
      return
    }

    const tl = gsap.timeline({
      onComplete: () => router.push(href),
    })

    tl.set(curtain, { scaleX: 0, transformOrigin: 'left center' })
      .to(curtain, { scaleX: 1, duration: 0.5, ease: 'power3.inOut' })
  }, [router])

  return { navigateTo }
}

// Usage: replace <Link href> with navigateTo for animated transitions
// const { navigateTo } = usePageLeave()
// <button onClick={() => navigateTo('/about')}>About</button>
```

---

## 10. Performance Rules

```typescript
// GOOD: Force GPU layer — use sparingly on elements that actually animate
gsap.set(element, { force3D: true })       // promotes to composite layer
gsap.set(element, { force3D: 'auto' })    // default — GSAP decides

// GOOD: Remove layer hint when animation is done
gsap.to(element, {
  x: 100,
  onComplete() {
    gsap.set(element, { force3D: false, clearProps: 'transform' })
  },
})

// GOOD: Use clearProps to remove inline styles after animation
gsap.from('.fade', {
  opacity: 0,
  y: 30,
  clearProps: 'opacity,y',  // or 'all' — lets CSS take over again
})

// GOOD: Batch ScrollTrigger refresh for layout recalculation
ScrollTrigger.batch('.card', {
  onEnter: (elements) => {
    gsap.from(elements, { opacity: 0, y: 30, stagger: 0.1 })
  },
  start: 'top 85%',
  once: true,
})

// GOOD: Kill all on unmount via gsap.context (preferred)
useEffect(() => {
  const ctx = gsap.context(() => {
    // all animations here are tracked
  })
  return () => ctx.revert() // kills + reverts everything
}, [])

// GOOD: Kill specific ScrollTriggers to avoid memory leaks in SPAs
useEffect(() => {
  const triggers: ScrollTrigger[] = []

  triggers.push(ScrollTrigger.create({
    trigger: '.el',
    onEnter: () => gsap.to('.el', { opacity: 1 }),
  }))

  return () => {
    triggers.forEach((t) => t.kill())
  }
}, [])

// GOOD: Ticker cleanup for custom RAF loops
const tickerCallback = () => { /* frame logic */ }
gsap.ticker.add(tickerCallback)
// cleanup:
gsap.ticker.remove(tickerCallback)

// GOOD: Refresh ScrollTrigger after dynamic content loads
useEffect(() => {
  ScrollTrigger.refresh()
}, [items]) // re-run when data changes

// NEVER: animate layout properties (causes reflow every frame)
// BAD:
gsap.to(el, { width: 200, height: 100, top: 50, left: 100 }) // causes layout thrash

// GOOD: use transform equivalents
gsap.to(el, { x: 100, y: 50, scaleX: 1.2, scaleY: 0.8 }) // compositor only
```

---

## 11. prefers-reduced-motion Integration

```typescript
// hooks/useReducedMotion.ts
import { useState, useEffect } from 'react'

export function useReducedMotion(): boolean {
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

// Global GSAP config — set once at app root (app/layout.tsx or _app.tsx)
import { gsap } from '@/lib/gsap'

if (typeof window !== 'undefined') {
  const mq = window.matchMedia('(prefers-reduced-motion: reduce)')
  if (mq.matches) {
    // Set duration to near-zero — animations still complete but instantly
    gsap.globalTimeline.timeScale(1000)
    // Or: disable all motion entirely
    gsap.defaults({ duration: 0 })
  }
}

// Component-level — conditional animation
'use client'
import { useRef } from 'react'
import { gsap } from '@/lib/gsap'
import { useGsap } from '@/hooks/useGsap'
import { useReducedMotion } from '@/hooks/useReducedMotion'

export function AccessibleReveal({ children }: { children: React.ReactNode }) {
  const ref = useRef<HTMLDivElement>(null)
  const prefersReduced = useReducedMotion()

  useGsap(() => {
    if (prefersReduced) {
      // Skip animation — element is already visible
      gsap.set(ref.current, { opacity: 1 })
      return
    }

    gsap.from(ref.current, {
      opacity: 0,
      y: 40,
      duration: 0.8,
      ease: 'power3.out',
    })
  }, [prefersReduced])

  return <div ref={ref} style={{ opacity: prefersReduced ? 1 : 0 }}>{children}</div>
}

// ScrollTrigger — disable scrub for reduced motion
export function AccessibleScrollSection() {
  const ref = useRef<HTMLDivElement>(null)
  const prefersReduced = useReducedMotion()

  useGsap(() => {
    gsap.from(ref.current, {
      opacity: 0,
      scrollTrigger: {
        trigger: ref.current,
        start: 'top 80%',
        scrub: prefersReduced ? false : 1,  // no scrub for reduced motion
        once: prefersReduced ? true : false, // fire once for reduced motion
      },
    })
  }, [prefersReduced])

  return <div ref={ref}>Content</div>
}
```

---

## 12. Anti-Patterns

```typescript
// ANTI-PATTERN: Missing cleanup causes memory leaks and ghost animations
useEffect(() => {
  gsap.to('.element', { opacity: 1, duration: 1 })
  // No return cleanup — ScrollTriggers and tweens pile up on each render
}, [])

// CORRECT: Always return cleanup via gsap.context
useEffect(() => {
  const ctx = gsap.context(() => {
    gsap.to('.element', { opacity: 1, duration: 1 })
  })
  return () => ctx.revert()
}, [])


// ANTI-PATTERN: Animating layout-triggering properties
gsap.to(card, { width: '+=100', height: '+=50', margin: 20 }) // forces reflow every frame

// CORRECT: Compositor-only properties
gsap.to(card, { scaleX: 1.1, scaleY: 1.1 }) // GPU only


// ANTI-PATTERN: Forgetting to revert SplitText — orphaned spans stay in DOM
const split = new SplitText(heading, { type: 'chars' })
gsap.from(split.chars, { opacity: 0 })
// Missing: split.revert()

// CORRECT: Revert in cleanup
useGsap(() => {
  const split = new SplitText(headingRef.current, { type: 'chars' })
  gsap.from(split.chars, { opacity: 0 })
  return () => split.revert()
}, [])


// ANTI-PATTERN: Calling ScrollTrigger.refresh() in a scroll handler
window.addEventListener('scroll', () => {
  ScrollTrigger.refresh() // runs layout recalculation on every scroll event
})

// CORRECT: Refresh only on resize or content change
window.addEventListener('resize', () => ScrollTrigger.refresh())


// ANTI-PATTERN: Selecting elements before they exist (SSR / hydration timing)
useEffect(() => {
  gsap.from(document.querySelectorAll('.card'), { opacity: 0 }) // may select 0 elements
}, [])

// CORRECT: Use refs or scope selection to the component's DOM subtree
useGsap(() => {
  gsap.from(containerRef.current?.querySelectorAll('.card') ?? [], { opacity: 0 })
}, [])


// ANTI-PATTERN: z-fighting — animating z-index causes stacking context repaints
gsap.to(el, { zIndex: 10, duration: 0.3 }) // triggers full paint, never interpolates cleanly

// CORRECT: Set z-index with gsap.set (no tween) or via CSS class toggle
gsap.set(el, { zIndex: 10 })


// ANTI-PATTERN: will-change left on permanently
// .card { will-change: transform; } — permanently elevates to composite layer, wastes VRAM

// CORRECT: Apply/remove will-change dynamically around animations
gsap.to(el, {
  onStart() { (el as HTMLElement).style.willChange = 'transform' },
  x: 100,
  onComplete() { (el as HTMLElement).style.willChange = 'auto' },
})


// ANTI-PATTERN: Using GSAP inside SSR render path without guard
// This throws on the server: "document is not defined"
// const split = new SplitText('h1') — top-level module code, no window check

// CORRECT: Only run GSAP in useEffect / client components / browser-only code
// All GSAP calls must live inside useEffect, useGsap, or event handlers
```
