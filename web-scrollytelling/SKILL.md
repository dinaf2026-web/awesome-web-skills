---
name: web-scrollytelling
description: Scroll-driven narrative design for modern websites — sticky sections, progress-tied animations, pinned panels, horizontal scroll, and immersive storytelling sequences using GSAP ScrollTrigger and Framer Motion.
origin: community
tags: [scrollytelling, scroll-driven, parallax, sticky, storytelling, gsap, framer-motion]
---

# web-scrollytelling

Scroll-driven narrative design for modern websites. This skill covers the full implementation stack: sticky sections, progress-tied animations, pinned panels, horizontal scroll, and immersive storytelling using GSAP ScrollTrigger and Framer Motion.

---

## 1. When to Use Scrollytelling

### Use it when

- You need to guide users through a multi-step story or product reveal
- Data visualization benefits from progressive disclosure (charts that build as you scroll)
- Brand narratives require cinematic pacing (hero journeys, feature walkthroughs)
- You have a dedicated landing page where scroll position IS the primary interaction

### Do NOT use it when

- Your content is utility-first (dashboards, forms, documentation, e-commerce listings)
- Your audience is predominantly mobile-only with intermittent connectivity
- The animation adds no semantic value — decoration for decoration's sake
- You cannot afford the JS bundle cost (sub-80kb budget pages)
- Your team lacks the bandwidth to maintain reduced-motion fallbacks

### The cost calculus

| Factor | Impact |
|--------|--------|
| GSAP + ScrollTrigger | ~27kb gzipped |
| Framer Motion | ~47kb gzipped |
| R3F (Three.js) | ~160kb gzipped |
| ScrollTrigger scroll listeners | Throttled to rAF — safe |
| Pinned elements | Forces compositor layers — watch GPU RAM |

---

## 2. The Three Scrollytelling Patterns

### Pattern 1 — Pin + Scrub

The viewport locks to a section. As the user scrolls, an animation scrubs forward proportional to scroll distance. The page "unfreezes" when the animation completes.

**Best for:** Product reveals, data builds, step-by-step explanations

### Pattern 2 — Step-Based (Waypoint)

No pinning. Discrete scroll waypoints trigger state changes. The page scrolls normally; entering/leaving a viewport threshold fires events.

**Best for:** Long-form journalism, chapter transitions, annotation sequences

### Pattern 3 — Progress-Tied

A scroll value (0–1) drives a CSS variable or animation value continuously. No pinning, no steps — smooth and reactive.

**Best for:** Parallax headers, progress bars, continuous visual feedback

---

## 3. Pinned Panel Sequence

Text and visual content change as the user scrolls through a pinned section. The panel stays fixed while a timeline of content plays.

```tsx
// PinnedSequence.tsx
'use client'

import { useEffect, useRef } from 'react'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger)

interface Step {
  heading: string
  body: string
  accent: string
}

const STEPS: Step[] = [
  { heading: 'Step One', body: 'The problem emerges from decades of neglect.', accent: '#3b82f6' },
  { heading: 'Step Two', body: 'We mapped every affected node in the network.', accent: '#8b5cf6' },
  { heading: 'Step Three', body: 'A single intervention point changes everything.', accent: '#10b981' },
  { heading: 'Step Four', body: 'Results compound across the entire system.', accent: '#f59e0b' },
]

export function PinnedSequence() {
  const wrapperRef = useRef<HTMLDivElement>(null)
  const panelRef = useRef<HTMLDivElement>(null)
  const headingRef = useRef<HTMLHeadingElement>(null)
  const bodyRef = useRef<HTMLParagraphElement>(null)
  const accentRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const ctx = gsap.context(() => {
      const tl = gsap.timeline({
        scrollTrigger: {
          trigger: wrapperRef.current,
          start: 'top top',
          end: `+=${STEPS.length * 100}%`,
          scrub: 1,
          pin: panelRef.current,
          anticipatePin: 1,
        },
      })

      STEPS.forEach((step, i) => {
        if (i === 0) return // first step is the initial state

        const label = `step${i}`
        tl.addLabel(label, i - 1)

        tl.to(
          headingRef.current,
          { opacity: 0, y: -20, duration: 0.3 },
          label,
        )
        tl.to(
          bodyRef.current,
          { opacity: 0, y: -10, duration: 0.3 },
          label,
        )
        tl.call(
          () => {
            if (headingRef.current) headingRef.current.textContent = step.heading
            if (bodyRef.current) bodyRef.current.textContent = step.body
          },
          undefined,
          `${label}+=0.3`,
        )
        tl.to(
          headingRef.current,
          { opacity: 1, y: 0, duration: 0.3 },
          `${label}+=0.3`,
        )
        tl.to(
          bodyRef.current,
          { opacity: 1, y: 0, duration: 0.3 },
          `${label}+=0.3`,
        )
        tl.to(
          accentRef.current,
          { backgroundColor: step.accent, duration: 0.5 },
          label,
        )
      })
    }, wrapperRef)

    return () => ctx.revert()
  }, [])

  return (
    // Height determines how long the pin lasts
    <div ref={wrapperRef} style={{ height: `${STEPS.length * 100}vh` }}>
      <div
        ref={panelRef}
        className="flex h-screen items-center justify-center bg-gray-950"
      >
        <div
          ref={accentRef}
          className="absolute inset-0 opacity-5 transition-colors"
          style={{ backgroundColor: STEPS[0].accent }}
        />
        <div className="relative z-10 max-w-2xl px-8 text-center">
          <h2
            ref={headingRef}
            className="mb-6 text-5xl font-bold text-white"
          >
            {STEPS[0].heading}
          </h2>
          <p
            ref={bodyRef}
            className="text-xl leading-relaxed text-gray-300"
          >
            {STEPS[0].body}
          </p>
        </div>
      </div>
    </div>
  )
}
```

### Framer Motion step-based variant (no pin, waypoint triggers)

```tsx
// StepSequence.tsx — waypoint pattern, no pinning
'use client'

import { useInView } from 'framer-motion'
import { useRef } from 'react'
import { motion } from 'framer-motion'

interface StepCardProps {
  heading: string
  body: string
  index: number
}

function StepCard({ heading, body, index }: StepCardProps) {
  const ref = useRef<HTMLDivElement>(null)
  const isInView = useInView(ref, { margin: '-40% 0px -40% 0px', once: false })

  return (
    <div ref={ref} className="flex min-h-[60vh] items-center py-24">
      <motion.div
        animate={{
          opacity: isInView ? 1 : 0.2,
          x: isInView ? 0 : -32,
        }}
        transition={{ duration: 0.5, ease: [0.16, 1, 0.3, 1] }}
        className="max-w-xl"
      >
        <span className="mb-4 block text-sm font-mono text-blue-400">
          {String(index + 1).padStart(2, '0')}
        </span>
        <h3 className="mb-4 text-4xl font-bold text-white">{heading}</h3>
        <p className="text-lg text-gray-400">{body}</p>
      </motion.div>
    </div>
  )
}
```

---

## 4. Horizontal Scroll Section

A section that scrolls horizontally while the page scrolls vertically. GSAP translates a wide container leftward as scroll progresses.

```tsx
// HorizontalScroll.tsx
'use client'

import { useEffect, useRef } from 'react'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger)

interface Panel {
  id: string
  label: string
  bg: string
}

const PANELS: Panel[] = [
  { id: 'a', label: 'Chapter One', bg: 'bg-blue-950' },
  { id: 'b', label: 'Chapter Two', bg: 'bg-violet-950' },
  { id: 'c', label: 'Chapter Three', bg: 'bg-emerald-950' },
  { id: 'd', label: 'Chapter Four', bg: 'bg-amber-950' },
]

export function HorizontalScroll() {
  const sectionRef = useRef<HTMLDivElement>(null)
  const trackRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const ctx = gsap.context(() => {
      const track = trackRef.current!
      const panels = gsap.utils.toArray<HTMLElement>('.h-panel')

      gsap.to(track, {
        x: () => -(track.scrollWidth - window.innerWidth),
        ease: 'none',
        scrollTrigger: {
          trigger: sectionRef.current,
          start: 'top top',
          end: () => `+=${track.scrollWidth - window.innerWidth}`,
          scrub: 1,
          pin: true,
          anticipatePin: 1,
          invalidateOnRefresh: true,
        },
      })
    }, sectionRef)

    return () => ctx.revert()
  }, [])

  return (
    <section ref={sectionRef} className="overflow-hidden">
      <div
        ref={trackRef}
        className="flex will-change-transform"
        style={{ width: `${PANELS.length * 100}vw` }}
      >
        {PANELS.map((panel) => (
          <div
            key={panel.id}
            className={`h-panel flex h-screen w-screen shrink-0 items-center justify-center ${panel.bg}`}
          >
            <h2 className="text-6xl font-bold text-white">{panel.label}</h2>
          </div>
        ))}
      </div>
    </section>
  )
}
```

### Framer Motion horizontal scroll (useScroll + useTransform)

```tsx
// HorizontalScrollFM.tsx
'use client'

import { useRef } from 'react'
import { motion, useScroll, useTransform } from 'framer-motion'

const PANELS = ['Prologue', 'Act I', 'Act II', 'Epilogue']

export function HorizontalScrollFM() {
  const containerRef = useRef<HTMLDivElement>(null)

  const { scrollYProgress } = useScroll({
    target: containerRef,
    offset: ['start start', 'end end'],
  })

  // scrollYProgress 0->1 maps to translateX 0 -> -(n-1)*100vw
  const x = useTransform(
    scrollYProgress,
    [0, 1],
    ['0vw', `-${(PANELS.length - 1) * 100}vw`],
  )

  return (
    // height creates the vertical scroll space
    <div ref={containerRef} style={{ height: `${PANELS.length * 100}vh` }}>
      <div className="sticky top-0 h-screen overflow-hidden">
        <motion.div className="flex h-full" style={{ x }}>
          {PANELS.map((label, i) => (
            <div
              key={i}
              className="flex h-full w-screen shrink-0 items-center justify-center bg-gray-900"
            >
              <h2 className="text-5xl font-bold text-white">{label}</h2>
            </div>
          ))}
        </motion.div>
      </div>
    </div>
  )
}
```

---

## 5. Progress Bar + Section Navigation

Scroll progress indicator with jump-link navigation. The bar fills as the user scrolls. Clicking a link smoothly scrolls to a section.

```tsx
// ScrollProgress.tsx
'use client'

import { useEffect, useRef, useState } from 'react'
import { motion, useScroll, useSpring } from 'framer-motion'

interface Section {
  id: string
  label: string
}

const SECTIONS: Section[] = [
  { id: 'intro', label: 'Intro' },
  { id: 'problem', label: 'Problem' },
  { id: 'solution', label: 'Solution' },
  { id: 'results', label: 'Results' },
]

export function ScrollProgress() {
  const { scrollYProgress } = useScroll()
  const scaleX = useSpring(scrollYProgress, { stiffness: 400, damping: 40 })
  const [active, setActive] = useState<string>('intro')

  useEffect(() => {
    const observers = SECTIONS.map(({ id }) => {
      const el = document.getElementById(id)
      if (!el) return null

      const observer = new IntersectionObserver(
        ([entry]) => { if (entry.isIntersecting) setActive(id) },
        { rootMargin: '-50% 0px -50% 0px' },
      )
      observer.observe(el)
      return observer
    })

    return () => observers.forEach((obs) => obs?.disconnect())
  }, [])

  function scrollTo(id: string) {
    document.getElementById(id)?.scrollIntoView({ behavior: 'smooth' })
  }

  return (
    <>
      {/* Progress bar — fixed top */}
      <motion.div
        className="fixed left-0 top-0 z-50 h-1 w-full origin-left bg-blue-500"
        style={{ scaleX }}
      />

      {/* Section nav — fixed right */}
      <nav
        className="fixed right-6 top-1/2 z-40 -translate-y-1/2"
        aria-label="Section navigation"
      >
        <ul className="flex flex-col gap-4">
          {SECTIONS.map(({ id, label }) => (
            <li key={id}>
              <button
                onClick={() => scrollTo(id)}
                aria-label={`Jump to ${label}`}
                className="group flex items-center gap-2"
              >
                <span
                  className={`block h-2 w-2 rounded-full transition-all duration-300 ${
                    active === id
                      ? 'scale-150 bg-blue-500'
                      : 'bg-gray-500 group-hover:bg-gray-300'
                  }`}
                />
                <span className="hidden text-xs text-gray-400 group-hover:inline">
                  {label}
                </span>
              </button>
            </li>
          ))}
        </ul>
      </nav>
    </>
  )
}
```

---

## 6. Image Sequence / Frame-by-Frame

Swap images on scroll to create a product reveal or animation sequence. Preloads all frames; shows only the frame matching the current scroll position.

```tsx
// ImageSequence.tsx
'use client'

import { useEffect, useRef } from 'react'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger)

// Generate frame URLs — replace with your actual asset pattern
function frameUrl(index: number) {
  return `/frames/product-${String(index).padStart(4, '0')}.webp`
}

const FRAME_COUNT = 60

interface ImageSequenceProps {
  frameCount?: number
}

export function ImageSequence({ frameCount = FRAME_COUNT }: ImageSequenceProps) {
  const wrapperRef = useRef<HTMLDivElement>(null)
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const framesRef = useRef<HTMLImageElement[]>([])
  const stateRef = useRef({ frame: 0 })

  useEffect(() => {
    const canvas = canvasRef.current!
    const ctx2d = canvas.getContext('2d')!

    // Preload all frames
    let loaded = 0
    const frames: HTMLImageElement[] = Array.from({ length: frameCount }, (_, i) => {
      const img = new Image()
      img.src = frameUrl(i)
      img.onload = () => {
        loaded++
        if (loaded === 1) {
          // Draw first frame immediately
          canvas.width = img.naturalWidth
          canvas.height = img.naturalHeight
          ctx2d.drawImage(img, 0, 0)
        }
      }
      return img
    })
    framesRef.current = frames

    function render(frame: number) {
      const img = frames[Math.round(frame)]
      if (img?.complete) ctx2d.drawImage(img, 0, 0)
    }

    const st = ScrollTrigger.create({
      trigger: wrapperRef.current,
      start: 'top top',
      end: `+=${frameCount * 20}px`,
      pin: true,
      scrub: 0.5,
      onUpdate: (self) => {
        const frame = Math.floor(self.progress * (frameCount - 1))
        if (frame !== stateRef.current.frame) {
          stateRef.current.frame = frame
          render(frame)
        }
      },
    })

    return () => {
      st.kill()
      frames.forEach((img) => { img.src = '' })
    }
  }, [frameCount])

  return (
    <div
      ref={wrapperRef}
      style={{ height: `${FRAME_COUNT * 20}px` }}
      className="bg-black"
    >
      <canvas
        ref={canvasRef}
        className="mx-auto h-screen w-auto object-contain"
        aria-label="Product animation sequence"
        role="img"
      />
    </div>
  )
}
```

### Lightweight variant — CSS-only image swap via Framer Motion

```tsx
// ImageSwap.tsx — small frame counts (< 20), no canvas needed
'use client'

import { useRef } from 'react'
import { useScroll, useTransform, motion, MotionValue } from 'framer-motion'

interface ImageSwapProps {
  frames: string[] // array of image URLs
}

export function ImageSwap({ frames }: ImageSwapProps) {
  const ref = useRef<HTMLDivElement>(null)

  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start start', 'end end'],
  })

  const frameIndex = useTransform(
    scrollYProgress,
    [0, 1],
    [0, frames.length - 1],
  )

  // Derive a rounded integer MotionValue for frame comparison
  const currentFrame = useTransform(frameIndex, (v) => Math.round(v))

  return (
    <div ref={ref} style={{ height: `${frames.length * 80}vh` }}>
      <div className="sticky top-0 flex h-screen items-center justify-center">
        {frames.map((src, i) => (
          <FrameImage key={src} src={src} index={i} currentFrame={currentFrame} />
        ))}
      </div>
    </div>
  )
}

interface FrameImageProps {
  src: string
  index: number
  currentFrame: MotionValue<number>
}

function FrameImage({ src, index, currentFrame }: FrameImageProps) {
  const opacity = useTransform(currentFrame, (v) => (v === index ? 1 : 0))
  return (
    <motion.img
      src={src}
      alt={`Frame ${index + 1}`}
      className="absolute h-full w-full object-cover"
      style={{ opacity }}
    />
  )
}
```

---

## 7. Text Reveal Sequences

Words and lines appear as the user scrolls. Two variants: word-by-word (dramatic) and line-by-line (editorial).

```tsx
// WordReveal.tsx — GSAP SplitText approach (manual split for no-plugin version)
'use client'

import { useEffect, useRef } from 'react'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger)

interface WordRevealProps {
  text: string
  className?: string
}

export function WordReveal({ text, className = '' }: WordRevealProps) {
  const ref = useRef<HTMLParagraphElement>(null)

  useEffect(() => {
    const el = ref.current!
    const words = el.querySelectorAll<HTMLSpanElement>('.word')

    const ctx = gsap.context(() => {
      gsap.fromTo(
        words,
        { opacity: 0.1, y: 8 },
        {
          opacity: 1,
          y: 0,
          duration: 0.4,
          stagger: 0.04,
          ease: 'power2.out',
          scrollTrigger: {
            trigger: el,
            start: 'top 80%',
            end: 'bottom 30%',
            scrub: 0.5,
          },
        },
      )
    }, el)

    return () => ctx.revert()
  }, [])

  const wordSpans = text.split(' ').map((word, i) => (
    <span key={i} className="word mr-[0.25em] inline-block">
      {word}
    </span>
  ))

  return (
    <p ref={ref} className={className}>
      {wordSpans}
    </p>
  )
}
```

```tsx
// LineReveal.tsx — Framer Motion, line by line
'use client'

import { useRef } from 'react'
import { motion, useInView } from 'framer-motion'

interface LineRevealProps {
  lines: string[]
  className?: string
}

const LINE_VARIANTS = {
  hidden: { opacity: 0, y: 24 },
  visible: (i: number) => ({
    opacity: 1,
    y: 0,
    transition: {
      delay: i * 0.12,
      duration: 0.6,
      ease: [0.16, 1, 0.3, 1],
    },
  }),
}

export function LineReveal({ lines, className = '' }: LineRevealProps) {
  const ref = useRef<HTMLDivElement>(null)
  const isInView = useInView(ref, { once: true, margin: '0px 0px -100px 0px' })

  return (
    <div ref={ref} className={className} aria-label={lines.join(' ')}>
      {lines.map((line, i) => (
        <div key={i} className="overflow-hidden">
          <motion.p
            custom={i}
            variants={LINE_VARIANTS}
            initial="hidden"
            animate={isInView ? 'visible' : 'hidden'}
            aria-hidden="true"
            className="leading-tight"
          >
            {line}
          </motion.p>
        </div>
      ))}
    </div>
  )
}
```

---

## 8. Before/After Scroll Reveal

A split-panel comparison where the divider position is driven by scroll progress. No drag required — the reveal happens automatically as the user scrolls.

```tsx
// BeforeAfter.tsx
'use client'

import { useRef } from 'react'
import { motion, useScroll, useTransform, useSpring } from 'framer-motion'

interface BeforeAfterProps {
  before: { src: string; label: string }
  after: { src: string; label: string }
}

export function BeforeAfter({ before, after }: BeforeAfterProps) {
  const ref = useRef<HTMLDivElement>(null)

  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start 0.8', 'end 0.2'],
  })

  // Smooth the raw scroll value
  const smoothProgress = useSpring(scrollYProgress, {
    stiffness: 200,
    damping: 30,
  })

  // 0% progress -> divider at 10% (almost fully "before")
  // 1% progress -> divider at 90% (almost fully "after")
  const clipPercent = useTransform(smoothProgress, [0, 1], [10, 90])
  const clipPath = useTransform(
    clipPercent,
    (v) => `inset(0 ${100 - v}% 0 0)`,
  )

  return (
    <div
      ref={ref}
      style={{ height: '300vh' }}
      className="relative"
      role="img"
      aria-label={`Before: ${before.label}. After: ${after.label}.`}
    >
      <div className="sticky top-0 h-screen overflow-hidden">
        {/* Before layer — always visible */}
        <img
          src={before.src}
          alt=""
          className="absolute inset-0 h-full w-full object-cover"
        />

        {/* Before label */}
        <span className="absolute left-4 top-4 rounded bg-black/60 px-2 py-1 text-sm text-white">
          {before.label}
        </span>

        {/* After layer — clipped by scroll */}
        <motion.div
          className="absolute inset-0"
          style={{ clipPath }}
        >
          <img
            src={after.src}
            alt=""
            className="h-full w-full object-cover"
          />
          <span className="absolute right-4 top-4 rounded bg-white/80 px-2 py-1 text-sm text-gray-900">
            {after.label}
          </span>
        </motion.div>

        {/* Divider line */}
        <motion.div
          className="absolute inset-y-0 w-0.5 bg-white shadow-lg"
          style={{ left: useTransform(clipPercent, (v) => `${v}%`) }}
          aria-hidden="true"
        />
      </div>
    </div>
  )
}
```

---

## 9. 3D Object Rotation on Scroll

React Three Fiber canvas with a model that rotates based on scroll position. ScrollTrigger drives the rotation value; R3F reads it via a ref.

```tsx
// ScrollObject3D.tsx
'use client'

import { useEffect, useRef, Suspense } from 'react'
import { Canvas, useFrame } from '@react-three/fiber'
import { useGLTF, Environment, ContactShadows } from '@react-three/drei'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import * as THREE from 'three'

gsap.registerPlugin(ScrollTrigger)

// Shared scroll state — avoids passing through React state (avoids re-renders)
const scrollState = { rotationY: 0, rotationX: 0 }

interface ModelProps {
  url: string
}

function Model({ url }: ModelProps) {
  const { scene } = useGLTF(url)
  const groupRef = useRef<THREE.Group>(null)

  useFrame(() => {
    if (!groupRef.current) return
    groupRef.current.rotation.y = THREE.MathUtils.lerp(
      groupRef.current.rotation.y,
      scrollState.rotationY,
      0.05,
    )
    groupRef.current.rotation.x = THREE.MathUtils.lerp(
      groupRef.current.rotation.x,
      scrollState.rotationX,
      0.05,
    )
  })

  return <primitive ref={groupRef} object={scene} />
}

interface ScrollObject3DProps {
  modelUrl: string
}

export function ScrollObject3D({ modelUrl }: ScrollObject3DProps) {
  const wrapperRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const st = ScrollTrigger.create({
      trigger: wrapperRef.current,
      start: 'top top',
      end: '+=300%',
      pin: true,
      scrub: 1,
      onUpdate: (self) => {
        // Full 360 degree Y rotation + 30 degree X tilt over the scroll range
        scrollState.rotationY = self.progress * Math.PI * 2
        scrollState.rotationX = Math.sin(self.progress * Math.PI) * 0.5
      },
    })

    return () => st.kill()
  }, [])

  return (
    <div ref={wrapperRef} style={{ height: '400vh' }}>
      <div className="sticky top-0 h-screen bg-gray-950">
        <Canvas
          camera={{ position: [0, 0, 3], fov: 45 }}
          dpr={[1, 1.5]}
          gl={{ antialias: true, powerPreference: 'high-performance' }}
        >
          <Suspense fallback={null}>
            <ambientLight intensity={0.4} />
            <directionalLight position={[5, 5, 5]} intensity={1} />
            <Model url={modelUrl} />
            <Environment preset="studio" />
            <ContactShadows
              position={[0, -1.5, 0]}
              blur={2}
              opacity={0.4}
            />
          </Suspense>
        </Canvas>
      </div>
    </div>
  )
}
```

---

## 10. Mobile Considerations

### Touch scroll and pinning

GSAP ScrollTrigger pins work on iOS/Android but require careful setup. Always test on real devices.

```tsx
// MobileScrollConfig.tsx — configure ScrollTrigger for mobile
import { useEffect } from 'react'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger)

export function MobileScrollConfig() {
  useEffect(() => {
    // Normalize scroll — required for smooth scrub on iOS
    ScrollTrigger.normalizeScroll(true)

    // Refresh on orientation change
    function handleResize() {
      ScrollTrigger.refresh()
    }
    window.addEventListener('orientationchange', handleResize)

    return () => {
      window.removeEventListener('orientationchange', handleResize)
      ScrollTrigger.normalizeScroll(false)
    }
  }, [])

  return null
}
```

### Reduce or disable scrollytelling on small screens

```tsx
// useScrollytelling.ts — disable on mobile/reduced-motion
import { useEffect, useState } from 'react'

export function useScrollytelling() {
  const [enabled, setEnabled] = useState(false)

  useEffect(() => {
    const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches
    const isMobileViewport = window.innerWidth < 768

    // Skip scrollytelling on mobile OR if user prefers reduced motion
    setEnabled(!prefersReduced && !isMobileViewport)
  }, [])

  return enabled
}
```

```tsx
// Usage: gracefully degrade to static content on mobile
function HeroSection() {
  const scrollyEnabled = useScrollytelling()

  if (!scrollyEnabled) {
    return <StaticHero />
  }

  return <AnimatedHero />
}
```

### Performance budget for mobile

| Asset | Mobile limit |
|-------|-------------|
| Total JS for scroll animations | < 50kb gzipped |
| Image frames (preloaded) | < 2MB total |
| Canvas operations | Avoid on low-end; use CSS instead |
| Active ScrollTriggers | < 10 simultaneously |
| GPU layers (will-change) | < 5 at once |

---

## 11. Performance Rules

### Only animate compositor-friendly properties

```ts
// GOOD — GPU composited, no layout recalc
gsap.to(el, { x: 100, y: 50, opacity: 0.5, scale: 1.2 })

// BAD — triggers layout recalculation on every frame
gsap.to(el, { width: 200, height: 100, top: 50, marginLeft: 20 })
```

### Use will-change sparingly

```css
/* Apply before animation starts, remove after it ends */
.animating {
  will-change: transform, opacity;
}

/* Never apply globally */
* { will-change: transform; } /* WRONG — destroys performance */
```

In GSAP, use force3D:
```ts
gsap.set(el, { force3D: true }) // promote to GPU layer before animation
// ... animation ...
gsap.set(el, { force3D: false }) // demote after
```

### IntersectionObserver vs ScrollTrigger — choose by use case

```ts
// Use IntersectionObserver for: enter/leave triggers, lazy loading, simple reveals
const io = new IntersectionObserver(
  ([entry]) => {
    if (entry.isIntersecting) el.classList.add('revealed')
  },
  { threshold: 0.2 },
)
io.observe(el)

// Use ScrollTrigger for: scrub animations, pinning, progress-tied values
// ScrollTrigger is heavier — don't use it for simple "fade in on enter"
ScrollTrigger.create({
  trigger: el,
  start: 'top 80%',
  onEnter: () => el.classList.add('revealed'),
})
```

### Kill all ScrollTriggers and timelines on unmount

```tsx
useEffect(() => {
  const ctx = gsap.context(() => {
    // All gsap.to / ScrollTrigger.create calls here are scoped
  }, rootRef)

  return () => {
    ctx.revert() // kills all animations, ScrollTriggers, and event listeners
  }
}, [])
```

### Batch ScrollTrigger.refresh calls

```ts
// WRONG — refresh on every resize event
window.addEventListener('resize', () => ScrollTrigger.refresh())

// CORRECT — debounce it
let rafId: number
window.addEventListener('resize', () => {
  cancelAnimationFrame(rafId)
  rafId = requestAnimationFrame(() => ScrollTrigger.refresh())
})
```

### Framer Motion performance tips

```tsx
// Use layout prop sparingly — triggers layout calculations
// AVOID for scroll-driven animations
// <motion.div layout> — not for scrollytelling

// PREFER direct style transforms
// <motion.div style={{ x, opacity }}> — correct

// Use useMotionValue + useTransform — no React re-renders
const x = useMotionValue(0)
const opacity = useTransform(x, [0, 200], [1, 0])
// These update on the animation thread, not the JS thread
```

---

## 12. Accessibility

### CSS hook — the foundation

```css
/* globals.css — apply this universally */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

### React hook — check once, memoize

```ts
// useReducedMotion.ts
import { useEffect, useState } from 'react'

export function useReducedMotion(): boolean {
  const [reduced, setReduced] = useState(() =>
    typeof window !== 'undefined'
      ? window.matchMedia('(prefers-reduced-motion: reduce)').matches
      : false,
  )

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)')
    const handler = (e: MediaQueryListEvent) => setReduced(e.matches)
    mq.addEventListener('change', handler)
    return () => mq.removeEventListener('change', handler)
  }, [])

  return reduced
}
```

### GSAP — respect reduced motion

```tsx
useEffect(() => {
  const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches

  if (prefersReduced) {
    // Show final state immediately — no animation
    gsap.set(elements, { opacity: 1, y: 0 })
    return
  }

  const ctx = gsap.context(() => {
    gsap.from(elements, {
      opacity: 0,
      y: 30,
      stagger: 0.1,
      scrollTrigger: { trigger: container, start: 'top 80%' },
    })
  }, containerRef)

  return () => ctx.revert()
}, [])
```

### Framer Motion — built-in reduced motion

```tsx
import { useReducedMotion } from 'framer-motion'

function AnimatedCard({ children }: { children: React.ReactNode }) {
  const prefersReduced = useReducedMotion()

  return (
    <motion.div
      initial={prefersReduced ? false : { opacity: 0, y: 24 }}
      whileInView={prefersReduced ? {} : { opacity: 1, y: 0 }}
      transition={{ duration: prefersReduced ? 0 : 0.6 }}
    >
      {children}
    </motion.div>
  )
}
```

### Skip link — always provide one for pinned sections

```tsx
// SkipLink.tsx
export function SkipLink({ targetId, label }: { targetId: string; label: string }) {
  return (
    <a
      href={`#${targetId}`}
      className={[
        'fixed left-4 top-4 z-[9999] rounded bg-white px-4 py-2',
        'text-sm font-semibold text-gray-900 shadow-lg',
        'translate-y-[-200%] focus:translate-y-0',
        'transition-transform duration-200',
      ].join(' ')}
    >
      {label}
    </a>
  )
}

// Usage — place before every pinned sequence
// <SkipLink targetId="after-product-sequence" label="Skip product animation" />
// <PinnedSequence />
// <div id="after-product-sequence" />
```

### ARIA for sequential content

```tsx
// Pinned sequence — screen readers should read the final state, not each step
// <div aria-live="polite" aria-atomic="true" aria-label="Product story sequence">
//   <span className="sr-only">{currentStep.heading}: {currentStep.body}</span>
// </div>

// Canvas-based image sequences — always need a static description
// <canvas
//   aria-label="Animation showing the product assembling from components into finished form"
//   role="img"
// />
```

### Keyboard alternative for horizontal scroll

```tsx
// Provide arrow key navigation for horizontal panels
function HorizontalScrollWithKeyboard() {
  const [activePanel, setActivePanel] = useState(0)

  function handleKeyDown(e: React.KeyboardEvent) {
    if (e.key === 'ArrowRight') setActivePanel((p) => Math.min(p + 1, PANELS.length - 1))
    if (e.key === 'ArrowLeft') setActivePanel((p) => Math.max(p - 1, 0))
  }

  return (
    <div
      role="region"
      aria-label="Feature panels"
      tabIndex={0}
      onKeyDown={handleKeyDown}
      aria-roledescription="carousel"
    >
      <div aria-live="polite" className="sr-only">
        Panel {activePanel + 1} of {PANELS.length}: {PANELS[activePanel].label}
      </div>
      {/* visual scroll component */}
    </div>
  )
}
```

---

## Quick Reference: Pattern Decision Matrix

| Goal | Pattern | Library |
|------|---------|---------|
| Product reveal, multi-step story | Pin + Scrub | GSAP ScrollTrigger |
| Long-form article, chapters | Step-based (waypoint) | Framer Motion useInView |
| Parallax header | Progress-tied | Framer Motion useScroll |
| Horizontal feature tour | Horizontal scroll | Either |
| Frame-by-frame product animation | Image sequence (canvas) | GSAP ScrollTrigger |
| Editorial text entrance | Line/word reveal | Either |
| Photo comparison | Before/After | Framer Motion |
| 3D model rotate | R3F + ScrollTrigger | GSAP + R3F |

## Install

```bash
# GSAP (ScrollTrigger is included)
npm install gsap

# Framer Motion
npm install framer-motion

# R3F stack (for 3D)
npm install @react-three/fiber @react-three/drei three
npm install -D @types/three
```
