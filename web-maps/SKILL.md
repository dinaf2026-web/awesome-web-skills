# web-maps Skill

Interactive map integration for web apps — provider selection, setup, markers, search, geolocation, GeoJSON layers, clustering, geo-filtering, routing, mobile, and performance.

---

## 1. Provider Decision Table

| | Mapbox GL JS | Leaflet | Google Maps JS |
|---|---|---|---|
| **Bundle (gzip)** | ~250 KB | ~40 KB | ~100 KB (CDN only) |
| **Pricing** | 50K loads/mo free, then ~$0.50/1K | Free (tile costs separate) | $200/mo credit, then pay-per-use |
| **Vector tiles** | Yes (native) | Plugin required | Yes |
| **Custom styles** | Mapbox Studio | Limited | Limited |
| **3D / terrain** | Yes | No | Yes (paid) |
| **Clustering** | Built-in | Plugin | Plugin |
| **React lib** | react-map-gl | react-leaflet | @react-google-maps/api |
| **Best for** | Rich custom maps, data viz | Lightweight, OSM, open source | Google Places integration |

**Rule of thumb:** Use Mapbox when you need custom styles, clustering, or GeoJSON expressions. Use Leaflet when bundle size is critical or you need OSM tiles for free. Use Google Maps only when you need Places Autocomplete or Street View.

---

## 2. Next.js Setup (Mapbox)

```bash
npm install mapbox-gl react-map-gl
```

```ts
// next.config.ts
const nextConfig = {
  transpilePackages: ['react-map-gl', 'mapbox-gl'],
}
export default nextConfig
```

```tsx
// app/map-page/page.tsx — SSR-safe dynamic import
import dynamic from 'next/dynamic'

const MapView = dynamic(() => import('@/components/MapView'), { ssr: false })

export default function MapPage() {
  return <MapView />
}
```

```tsx
// components/MapView.tsx — CSS import must be client-side only
import 'mapbox-gl/dist/mapbox-gl.css'
```

```env
# .env.local
NEXT_PUBLIC_MAPBOX_TOKEN=pk.eyJ1IjoieW91cnVzZXIi...
```

---

## 3. Base Map Component

```tsx
'use client'
import { useRef, useEffect } from 'react'
import mapboxgl from 'mapbox-gl'
import 'mapbox-gl/dist/mapbox-gl.css'

mapboxgl.accessToken = process.env.NEXT_PUBLIC_MAPBOX_TOKEN!

interface BaseMapProps {
  center?: [number, number]  // [lng, lat]
  zoom?: number
  style?: string
  className?: string
}

export function BaseMap({
  center = [-118.2437, 34.0522],
  zoom = 11,
  style = 'mapbox://styles/mapbox/streets-v12',
  className = 'w-full h-96',
}: BaseMapProps) {
  const containerRef = useRef<HTMLDivElement>(null)
  const mapRef = useRef<mapboxgl.Map | null>(null)

  useEffect(() => {
    if (!containerRef.current || mapRef.current) return

    const map = new mapboxgl.Map({
      container: containerRef.current,
      style,
      center,
      zoom,
    })

    map.addControl(new mapboxgl.NavigationControl(), 'top-right')

    // Fix blank map when container resizes (e.g. sidebar toggle)
    const observer = new ResizeObserver(() => map.resize())
    observer.observe(containerRef.current)

    mapRef.current = map

    return () => {
      observer.disconnect()
      map.remove()
      mapRef.current = null
    }
  }, [])  // run once — center/zoom/style are init values only

  return <div ref={containerRef} className={className} />
}
```

---

## 4. Custom Markers

### DOM marker (SVG pin)

```tsx
import { useEffect, useRef } from 'react'
import mapboxgl from 'mapbox-gl'

function addCustomMarker(map: mapboxgl.Map, lngLat: [number, number], label: string) {
  const el = document.createElement('div')
  el.className = 'custom-marker'
  el.innerHTML = `
    <svg width="32" height="40" viewBox="0 0 32 40" fill="none">
      <path d="M16 0C7.16 0 0 7.16 0 16c0 10.67 16 24 16 24s16-13.33 16-24C32 7.16 24.84 0 16 0z" fill="#6366f1"/>
      <circle cx="16" cy="16" r="6" fill="white"/>
      <!-- transparent hit area -->
      <rect x="0" y="0" width="32" height="40" fill="transparent"/>
    </svg>
  `

  const popup = new mapboxgl.Popup({ offset: 25 }).setText(label)

  new mapboxgl.Marker({ element: el })
    .setLngLat(lngLat)
    .setPopup(popup)
    .addTo(map)
}
```

### react-map-gl Marker + Popup

```tsx
import { Marker, Popup } from 'react-map-gl'
import { useState } from 'react'

interface Location { id: string; lng: number; lat: number; name: string }

function LocationMarker({ loc }: { loc: Location }) {
  const [open, setOpen] = useState(false)

  return (
    <>
      <Marker longitude={loc.lng} latitude={loc.lat} anchor="bottom" onClick={() => setOpen(true)}>
        <div className="w-8 h-8 bg-indigo-500 rounded-full border-2 border-white shadow cursor-pointer" />
      </Marker>
      {open && (
        <Popup longitude={loc.lng} latitude={loc.lat} anchor="top" onClose={() => setOpen(false)}>
          <p className="font-medium text-sm">{loc.name}</p>
        </Popup>
      )}
    </>
  )
}
```

---

## 5. Location Search (Geocoding)

```tsx
'use client'
import { useState, useCallback, useRef } from 'react'

interface GeocodingFeature {
  id: string
  place_name: string
  center: [number, number]
}

interface LocationSearchProps {
  onSelect: (center: [number, number], name: string) => void
}

export function LocationSearch({ onSelect }: LocationSearchProps) {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState<GeocodingFeature[]>([])
  const debounceRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  const search = useCallback((q: string) => {
    if (debounceRef.current) clearTimeout(debounceRef.current)
    if (q.length < 3) { setResults([]); return }

    debounceRef.current = setTimeout(async () => {
      const token = process.env.NEXT_PUBLIC_MAPBOX_TOKEN
      const url = `https://api.mapbox.com/geocoding/v5/mapbox.places/${encodeURIComponent(q)}.json?access_token=${token}&limit=5`
      const res = await fetch(url)
      const data = await res.json()
      setResults(data.features ?? [])
    }, 300)
  }, [])

  return (
    <div className="relative">
      <input
        type="search"
        value={query}
        onChange={(e) => { setQuery(e.target.value); search(e.target.value) }}
        placeholder="Search location..."
        className="w-full px-3 py-2 border rounded-lg text-sm"
      />
      {results.length > 0 && (
        <ul className="absolute z-50 w-full mt-1 bg-white border rounded-lg shadow-lg max-h-60 overflow-auto">
          {results.map((f) => (
            <li key={f.id}>
              <button
                className="w-full text-left px-3 py-2 text-sm hover:bg-gray-50"
                onClick={() => {
                  onSelect(f.center, f.place_name)
                  setQuery(f.place_name)
                  setResults([])
                }}
              >
                {f.place_name}
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}

// In parent map component:
// map.flyTo({ center, zoom: 14, duration: 1200 })
```

---

## 6. User Geolocation

```tsx
import { useState, useEffect } from 'react'

interface GeolocationState {
  coords: GeolocationCoordinates | null
  error: string | null
  loading: boolean
}

export function useGeolocation(): GeolocationState {
  const [state, setState] = useState<GeolocationState>({ coords: null, error: null, loading: false })

  useEffect(() => {
    if (!navigator.geolocation) {
      setState((s) => ({ ...s, error: 'Geolocation not supported' }))
      return
    }

    setState((s) => ({ ...s, loading: true }))

    const id = navigator.geolocation.watchPosition(
      (pos) => setState({ coords: pos.coords, error: null, loading: false }),
      (err) => {
        const messages: Record<number, string> = {
          1: 'Location permission denied',
          2: 'Location unavailable',
          3: 'Location request timed out',
        }
        setState({ coords: null, error: messages[err.code] ?? 'Unknown error', loading: false })
      },
      { enableHighAccuracy: true, timeout: 10000 }
    )

    return () => navigator.geolocation.clearWatch(id)
  }, [])

  return state
}

// Pulsing dot on Mapbox map
function addUserDot(map: mapboxgl.Map, lngLat: [number, number]) {
  const el = document.createElement('div')
  el.className = 'user-location-dot'
  // CSS: blue circle with pulsing ring via @keyframes
  new mapboxgl.Marker({ element: el }).setLngLat(lngLat).addTo(map)
}
```

```css
/* globals.css */
.user-location-dot {
  width: 16px;
  height: 16px;
  background: #3b82f6;
  border: 2px solid white;
  border-radius: 50%;
  box-shadow: 0 0 0 0 rgba(59, 130, 246, 0.5);
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%   { box-shadow: 0 0 0 0 rgba(59, 130, 246, 0.5); }
  70%  { box-shadow: 0 0 0 12px rgba(59, 130, 246, 0); }
  100% { box-shadow: 0 0 0 0 rgba(59, 130, 246, 0); }
}
```

---

## 7. GeoJSON Layers (Choropleth + Hover)

```tsx
map.on('load', () => {
  // Add GeoJSON source
  map.addSource('regions', {
    type: 'geojson',
    data: '/data/regions.geojson',
    generateId: true,  // required for feature-state
  })

  // Choropleth fill layer
  map.addLayer({
    id: 'regions-fill',
    type: 'fill',
    source: 'regions',
    paint: {
      'fill-color': [
        'interpolate', ['linear'],
        ['get', 'value'],
        0,   '#f1f5f9',
        25,  '#bfdbfe',
        50,  '#60a5fa',
        75,  '#2563eb',
        100, '#1e3a8a',
      ],
      // Dim non-hovered features using feature-state
      'fill-opacity': ['case', ['boolean', ['feature-state', 'hover'], false], 0.9, 0.6],
    },
  })

  // Outline layer
  map.addLayer({
    id: 'regions-outline',
    type: 'line',
    source: 'regions',
    paint: { 'line-color': '#94a3b8', 'line-width': 1 },
  })

  // Hover interaction
  let hoveredId: number | string | null = null

  map.on('mousemove', 'regions-fill', (e) => {
    if (!e.features?.length) return
    map.getCanvas().style.cursor = 'pointer'
    if (hoveredId !== null) map.setFeatureState({ source: 'regions', id: hoveredId }, { hover: false })
    hoveredId = e.features[0].id ?? null
    if (hoveredId !== null) map.setFeatureState({ source: 'regions', id: hoveredId }, { hover: true })
  })

  map.on('mouseleave', 'regions-fill', () => {
    map.getCanvas().style.cursor = ''
    if (hoveredId !== null) map.setFeatureState({ source: 'regions', id: hoveredId }, { hover: false })
    hoveredId = null
  })

  // Popup on click
  const popup = new mapboxgl.Popup({ closeButton: false })
  map.on('click', 'regions-fill', (e) => {
    if (!e.features?.length) return
    const props = e.features[0].properties ?? {}
    popup.setLngLat(e.lngLat).setHTML(`<strong>${props.name}</strong><br/>Value: ${props.value}`).addTo(map)
  })
})
```

---

## 8. Clustering

```tsx
map.on('load', () => {
  map.addSource('points', {
    type: 'geojson',
    data: '/data/points.geojson',
    cluster: true,
    clusterMaxZoom: 14,
    clusterRadius: 50,
  })

  // Cluster circles
  map.addLayer({
    id: 'clusters',
    type: 'circle',
    source: 'points',
    filter: ['has', 'point_count'],
    paint: {
      'circle-color': ['step', ['get', 'point_count'], '#6366f1', 10, '#4f46e5', 50, '#3730a3'],
      'circle-radius': ['step', ['get', 'point_count'], 20, 10, 28, 50, 36],
    },
  })

  // Count badges
  map.addLayer({
    id: 'cluster-count',
    type: 'symbol',
    source: 'points',
    filter: ['has', 'point_count'],
    layout: {
      'text-field': ['get', 'point_count_abbreviated'],
      'text-size': 13,
      'text-font': ['DIN Offc Pro Medium', 'Arial Unicode MS Bold'],
    },
    paint: { 'text-color': '#ffffff' },
  })

  // Individual points
  map.addLayer({
    id: 'unclustered-point',
    type: 'circle',
    source: 'points',
    filter: ['!', ['has', 'point_count']],
    paint: { 'circle-color': '#6366f1', 'circle-radius': 6, 'circle-stroke-width': 2, 'circle-stroke-color': '#fff' },
  })

  // Expand cluster on click
  map.on('click', 'clusters', (e) => {
    const features = map.queryRenderedFeatures(e.point, { layers: ['clusters'] })
    if (!features.length) return
    const clusterId = features[0].properties?.cluster_id
    ;(map.getSource('points') as mapboxgl.GeoJSONSource).getClusterExpansionZoom(clusterId, (err, zoom) => {
      if (err) return
      map.easeTo({ center: (features[0].geometry as GeoJSON.Point).coordinates as [number, number], zoom: zoom! })
    })
  })

  // Individual point popup
  map.on('click', 'unclustered-point', (e) => {
    if (!e.features?.length) return
    const props = e.features[0].properties ?? {}
    const coords = (e.features[0].geometry as GeoJSON.Point).coordinates as [number, number]
    new mapboxgl.Popup().setLngLat(coords).setHTML(`<p>${props.name ?? 'Point'}</p>`).addTo(map)
  })

  map.on('mouseenter', 'clusters', () => { map.getCanvas().style.cursor = 'pointer' })
  map.on('mouseleave', 'clusters', () => { map.getCanvas().style.cursor = '' })
})
```

---

## 9. Geo-Filtering (Map Bounds to List Sync)

```tsx
import { useCallback, useEffect, useRef, useState } from 'react'
import mapboxgl from 'mapbox-gl'

interface BoundsFilterState {
  bounds: mapboxgl.LngLatBounds | null
  isFiltering: boolean
}

export function useMapBoundsFilter(map: mapboxgl.Map | null) {
  const [state, setState] = useState<BoundsFilterState>({ bounds: null, isFiltering: false })
  const debounceRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  useEffect(() => {
    if (!map) return

    const handleMoveEnd = () => {
      if (debounceRef.current) clearTimeout(debounceRef.current)
      debounceRef.current = setTimeout(() => {
        setState({ bounds: map.getBounds(), isFiltering: true })
      }, 200)
    }

    map.on('moveend', handleMoveEnd)
    return () => { map.off('moveend', handleMoveEnd) }
  }, [map])

  const clearFilter = useCallback(() => {
    setState({ bounds: null, isFiltering: false })
  }, [])

  return { ...state, clearFilter }
}

// Filter items by map bounds
function filterByBounds<T extends { lng: number; lat: number }>(
  items: T[],
  bounds: mapboxgl.LngLatBounds | null
): T[] {
  if (!bounds) return items
  return items.filter((item) => bounds.contains([item.lng, item.lat]))
}

// Fly to item when clicked in list
function flyToItem(map: mapboxgl.Map, lng: number, lat: number) {
  map.flyTo({ center: [lng, lat], zoom: 15, duration: 800 })
}
```

---

## 10. Route Display (Directions API)

```tsx
interface RouteResult {
  distance: number  // meters
  duration: number  // seconds
  geometry: GeoJSON.LineString
}

async function fetchRoute(
  origin: [number, number],
  destination: [number, number],
  profile: 'driving' | 'walking' | 'cycling' = 'driving'
): Promise<RouteResult | null> {
  const token = process.env.NEXT_PUBLIC_MAPBOX_TOKEN
  const coords = `${origin.join(',')};${destination.join(',')}`
  const url = `https://api.mapbox.com/directions/v5/mapbox/${profile}/${coords}?geometries=geojson&access_token=${token}`
  const res = await fetch(url)
  const data = await res.json()
  const route = data.routes?.[0]
  if (!route) return null
  return { distance: route.distance, duration: route.duration, geometry: route.geometry }
}

function displayRoute(map: mapboxgl.Map, geometry: GeoJSON.LineString) {
  // Remove existing route if any
  if (map.getLayer('route')) map.removeLayer('route')
  if (map.getSource('route')) map.removeSource('route')

  map.addSource('route', { type: 'geojson', data: { type: 'Feature', geometry, properties: {} } })
  map.addLayer({
    id: 'route',
    type: 'line',
    source: 'route',
    layout: { 'line-join': 'round', 'line-cap': 'round' },
    paint: { 'line-color': '#6366f1', 'line-width': 5, 'line-opacity': 0.85 },
  })

  // Fit map to route
  const coords = geometry.coordinates as [number, number][]
  const bounds = coords.reduce(
    (b, c) => b.extend(c),
    new mapboxgl.LngLatBounds(coords[0], coords[0])
  )
  map.fitBounds(bounds, { padding: 60 })
}

// Usage
// const route = await fetchRoute(origin, destination)
// if (route) {
//   displayRoute(map, route.geometry)
//   const km = (route.distance / 1000).toFixed(1)
//   const min = Math.round(route.duration / 60)
// }
```

---

## 11. Mobile

```tsx
// cooperativeGestures prevents accidental scroll hijack
const map = new mapboxgl.Map({
  container: containerRef.current,
  style,
  center,
  zoom,
  cooperativeGestures: true,  // shows "use two fingers" overlay on mobile
})
```

```css
/* Fullscreen map wrapper */
.map-fullscreen {
  position: fixed;
  inset: 0;
  z-index: 40;
}

/* 44px minimum touch targets for all map controls */
.mapboxgl-ctrl button {
  min-width: 44px;
  min-height: 44px;
}

/* Prevent iOS bounce interfering with map */
.map-container {
  touch-action: none;
  overscroll-behavior: none;
}
```

---

## 12. Performance

### Lazy-load map on scroll into view

```tsx
import { useEffect, useRef, useState } from 'react'

export function LazyMap() {
  const wrapperRef = useRef<HTMLDivElement>(null)
  const [load, setLoad] = useState(false)

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => { if (entry.isIntersecting) { setLoad(true); observer.disconnect() } },
      { rootMargin: '200px' }
    )
    if (wrapperRef.current) observer.observe(wrapperRef.current)
    return () => observer.disconnect()
  }, [])

  return (
    <div ref={wrapperRef} className="h-96 w-full bg-slate-100">
      {load && <BaseMap />}
    </div>
  )
}
```

### DOM marker limit

```
Rule: Never add more than 200 DOM markers simultaneously.
Over 200 — switch to a symbol layer or clustering source.
DOM markers trigger layout/paint on every map move; symbol layers are GPU-rendered.
```

### Debounce moveend handlers

```tsx
// 150-300ms debounce on any moveend/zoomend callbacks that trigger data fetching
const debounceRef = useRef<ReturnType<typeof setTimeout> | null>(null)
map.on('moveend', () => {
  if (debounceRef.current) clearTimeout(debounceRef.current)
  debounceRef.current = setTimeout(() => fetchData(map.getBounds()), 250)
})
```

### Map instance in ref, not state

```tsx
// CORRECT — no re-renders when map moves
const mapRef = useRef<mapboxgl.Map | null>(null)

// WRONG — triggers component re-render on every setState call
const [map, setMap] = useState<mapboxgl.Map | null>(null)
```

### Tile cache headers

Configure your tile server or CDN with:
```
Cache-Control: public, max-age=86400
```
Mapbox-hosted tiles handle this automatically. For self-hosted tiles (MBTiles, pmtiles), set cache headers on the static file server.

---

## Quick Reference

| Task | Key API / Pattern |
|---|---|
| Init map | `new mapboxgl.Map({ container, style, center, zoom })` |
| SSR-safe Next.js | `dynamic(() => import(...), { ssr: false })` |
| Custom marker | `new mapboxgl.Marker({ element }).setLngLat().addTo(map)` |
| Popup | `new mapboxgl.Popup().setLngLat().setHTML().addTo(map)` |
| Geocode search | Mapbox Geocoding API v5 + debounced fetch |
| Fly to location | `map.flyTo({ center, zoom, duration })` |
| GeoJSON layer | `addSource + addLayer` with paint expressions |
| Hover state | `generateId: true` + `setFeatureState` |
| Clustering | `cluster: true` on GeoJSON source |
| Directions | Mapbox Directions API v5 + `fitBounds` |
| Bounds filter | `map.getBounds()` on `moveend` + debounce |
| Mobile scroll | `cooperativeGestures: true` |
| Lazy load | `IntersectionObserver` with 200px rootMargin |
| Max DOM markers | 200 — above this use symbol layers |
