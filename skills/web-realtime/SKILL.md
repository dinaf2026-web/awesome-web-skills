---
name: web-realtime
description: Real-time feature patterns for modern web apps — WebSockets, Server-Sent Events, Pusher/Ably, live counters, collaborative editing, and presence indicators.
origin: community
tags: [realtime, websockets, sse, pusher, ably, live-updates, presence, collaborative]
---

# web-realtime

Real-time feature patterns for modern web apps. Covers transport selection, server setup, client hooks, presence, collaboration, and production scale concerns.

---

## 1. When to Use Which

| Transport | Latency | Direction | Server Load | Best For |
|---|---|---|---|---|
| **SSE** | Low | Server → Client only | Low | Feeds, notifications, live scores |
| **WebSocket** | Very low | Bidirectional | Medium | Chat, games, collaborative editing |
| **Long Polling** | Medium | Server → Client | High | Legacy fallback, simple alerts |
| **Short Polling** | High | Client → Server | Very high | Last resort, infrequent updates |
| **Pusher/Ably** | Low | Bidirectional (managed) | None (offloaded) | Presence, auth channels, rapid prototyping |

**Decision rules:**

- Client only needs to receive → use **SSE**
- Client needs to send and receive → use **WebSocket** or **Pusher/Ably**
- You need auth'd private channels or presence out of the box → use **Pusher** or **Ably**
- You control infra and need zero vendor dependency → use **ws** (WebSocket)
- You need < 1 hour to ship → use **Pusher** or **Ably**

---

## 2. Server-Sent Events in Next.js

### Route Handler (App Router)

```ts
// app/api/stream/route.ts
import { NextRequest } from 'next/server'

export const runtime = 'edge' // or 'nodejs'

export async function GET(req: NextRequest) {
  const encoder = new TextEncoder()

  const stream = new ReadableStream({
    async start(controller) {
      const send = (event: string, data: unknown) => {
        const payload = `event: ${event}\ndata: ${JSON.stringify(data)}\n\n`
        controller.enqueue(encoder.encode(payload))
      }

      // Send initial state
      send('connected', { ts: Date.now() })

      // Example: push updates every 2 s (replace with your event source)
      const interval = setInterval(() => {
        send('update', { count: Math.floor(Math.random() * 100), ts: Date.now() })
      }, 2000)

      // Clean up when client disconnects
      req.signal.addEventListener('abort', () => {
        clearInterval(interval)
        controller.close()
      })
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache, no-transform',
      Connection: 'keep-alive',
      'X-Accel-Buffering': 'no', // Disable Nginx buffering
    },
  })
}
```

### EventSource Client Hook

```ts
// hooks/useSSE.ts
import { useEffect, useRef, useState } from 'react'

type SSEStatus = 'connecting' | 'open' | 'closed' | 'error'

interface SSEOptions {
  onMessage?: (event: MessageEvent) => void
  onEvent?: (type: string, data: unknown) => void
  withCredentials?: boolean
  retryInterval?: number
}

export function useSSE(url: string, options: SSEOptions = {}) {
  const { onMessage, onEvent, withCredentials = false, retryInterval = 3000 } = options
  const [status, setStatus] = useState<SSEStatus>('connecting')
  const esRef = useRef<EventSource | null>(null)
  const retryTimer = useRef<ReturnType<typeof setTimeout>>()

  useEffect(() => {
    let cancelled = false

    const connect = () => {
      if (cancelled) return
      const es = new EventSource(url, { withCredentials })
      esRef.current = es

      es.onopen = () => setStatus('open')
      es.onerror = () => {
        setStatus('error')
        es.close()
        retryTimer.current = setTimeout(connect, retryInterval)
      }
      es.onmessage = (e) => onMessage?.(e)

      // Forward named events
      const handler = (e: MessageEvent) => {
        try {
          onEvent?.(e.type, JSON.parse(e.data))
        } catch {
          onEvent?.(e.type, e.data)
        }
      }
      // Attach common named events — add yours here
      ;['update', 'notification', 'presence', 'cursor'].forEach((ev) =>
        es.addEventListener(ev, handler as EventListener)
      )
    }

    connect()

    return () => {
      cancelled = true
      clearTimeout(retryTimer.current)
      esRef.current?.close()
      setStatus('closed')
    }
  }, [url])

  return { status }
}
```

### Usage

```tsx
// components/LiveFeed.tsx
export function LiveFeed() {
  const [items, setItems] = useState<FeedItem[]>([])

  const { status } = useSSE('/api/stream', {
    onEvent(type, data) {
      if (type === 'update') {
        setItems((prev) => [data as FeedItem, ...prev].slice(0, 50))
      }
    },
  })

  return (
    <div>
      <ConnectionBadge status={status} />
      {items.map((item) => <FeedCard key={item.id} item={item} />)}
    </div>
  )
}
```

---

## 3. WebSocket with ws Library

### Server Setup (Node / Express)

```ts
// server/ws.ts
import { WebSocketServer, WebSocket } from 'ws'
import { IncomingMessage } from 'http'
import { parse } from 'url'

interface Client {
  ws: WebSocket
  id: string
  room: string
  userId?: string
}

const clients = new Map<string, Client>()

export function createWSServer(server: import('http').Server) {
  const wss = new WebSocketServer({ server, path: '/ws' })

  wss.on('connection', (ws: WebSocket, req: IncomingMessage) => {
    const { query } = parse(req.url ?? '', true)
    const room = (query.room as string) ?? 'default'
    const userId = query.userId as string | undefined
    const id = crypto.randomUUID()

    clients.set(id, { ws, id, room, userId })

    ws.on('message', (raw) => {
      try {
        const msg = JSON.parse(raw.toString())
        handleMessage(id, msg)
      } catch {
        // ignore malformed
      }
    })

    ws.on('close', () => {
      clients.delete(id)
      broadcast(room, { type: 'presence', userId, online: false })
    })

    ws.on('error', (err) => console.error('ws error', err))

    // Acknowledge connection
    send(ws, { type: 'connected', id })
    broadcast(room, { type: 'presence', userId, online: true }, id)
  })

  return wss
}

function handleMessage(senderId: string, msg: Record<string, unknown>) {
  const client = clients.get(senderId)
  if (!client) return

  switch (msg.type) {
    case 'message':
      broadcast(client.room, { ...msg, senderId })
      break
    case 'cursor':
      broadcast(client.room, { ...msg, senderId }, senderId)
      break
    case 'ping':
      send(client.ws, { type: 'pong' })
      break
  }
}

function broadcast(room: string, payload: unknown, excludeId?: string) {
  const data = JSON.stringify(payload)
  for (const [id, client] of clients) {
    if (id === excludeId) continue
    if (client.room !== room) continue
    if (client.ws.readyState === WebSocket.OPEN) {
      client.ws.send(data)
    }
  }
}

function send(ws: WebSocket, payload: unknown) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify(payload))
  }
}
```

### Client Hook with Reconnection

```ts
// hooks/useWebSocket.ts
import { useEffect, useRef, useState, useCallback } from 'react'

type WSStatus = 'connecting' | 'open' | 'closing' | 'closed'

interface UseWebSocketOptions {
  onMessage: (msg: unknown) => void
  reconnectDelay?: number
  maxReconnectDelay?: number
}

export function useWebSocket(url: string, options: UseWebSocketOptions) {
  const { onMessage, reconnectDelay = 1000, maxReconnectDelay = 30_000 } = options
  const [status, setStatus] = useState<WSStatus>('connecting')
  const wsRef = useRef<WebSocket | null>(null)
  const reconnectAttempts = useRef(0)
  const reconnectTimer = useRef<ReturnType<typeof setTimeout>>()
  const isMounted = useRef(true)

  const connect = useCallback(() => {
    if (!isMounted.current) return

    const ws = new WebSocket(url)
    wsRef.current = ws
    setStatus('connecting')

    ws.onopen = () => {
      setStatus('open')
      reconnectAttempts.current = 0
    }

    ws.onclose = () => {
      setStatus('closed')
      if (!isMounted.current) return

      const delay = Math.min(
        reconnectDelay * 2 ** reconnectAttempts.current,
        maxReconnectDelay
      )
      reconnectAttempts.current += 1
      reconnectTimer.current = setTimeout(connect, delay)
    }

    ws.onerror = () => {
      ws.close()
    }

    ws.onmessage = (e) => {
      try {
        onMessage(JSON.parse(e.data))
      } catch {
        onMessage(e.data)
      }
    }
  }, [url, onMessage, reconnectDelay, maxReconnectDelay])

  useEffect(() => {
    isMounted.current = true
    connect()
    return () => {
      isMounted.current = false
      clearTimeout(reconnectTimer.current)
      wsRef.current?.close()
    }
  }, [connect])

  const send = useCallback((payload: unknown) => {
    const ws = wsRef.current
    if (ws?.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(payload))
    }
  }, [])

  return { send, status }
}
```

---

## 4. Pusher Setup

### Install

```bash
npm install pusher pusher-js
```

### Server — Trigger Event

```ts
// lib/pusher-server.ts
import Pusher from 'pusher'

export const pusherServer = new Pusher({
  appId: process.env.PUSHER_APP_ID!,
  key: process.env.NEXT_PUBLIC_PUSHER_KEY!,
  secret: process.env.PUSHER_SECRET!,
  cluster: process.env.NEXT_PUBLIC_PUSHER_CLUSTER!,
  useTLS: true,
})
```

```ts
// app/api/notify/route.ts
import { pusherServer } from '@/lib/pusher-server'
import { NextRequest, NextResponse } from 'next/server'

export async function POST(req: NextRequest) {
  const body = await req.json()

  await pusherServer.trigger(
    `channel-${body.roomId}`,    // channel
    'new-message',               // event
    { text: body.text, userId: body.userId }
  )

  return NextResponse.json({ ok: true })
}
```

### Auth Endpoint (private / presence channels)

```ts
// app/api/pusher/auth/route.ts
import { pusherServer } from '@/lib/pusher-server'
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'

export async function POST(req: NextRequest) {
  const session = await getServerSession()
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const data = await req.formData()
  const socketId = data.get('socket_id') as string
  const channel = data.get('channel_name') as string

  const presenceData = {
    user_id: session.user.id,
    user_info: { name: session.user.name, avatar: session.user.image },
  }

  const authResponse = pusherServer.authorizeChannel(socketId, channel, presenceData)
  return NextResponse.json(authResponse)
}
```

### Client Hook

```ts
// hooks/usePusher.ts
import { useEffect, useRef } from 'react'
import Pusher, { Channel } from 'pusher-js'

let pusherClient: Pusher | null = null

function getPusherClient() {
  if (!pusherClient) {
    pusherClient = new Pusher(process.env.NEXT_PUBLIC_PUSHER_KEY!, {
      cluster: process.env.NEXT_PUBLIC_PUSHER_CLUSTER!,
      authEndpoint: '/api/pusher/auth',
    })
  }
  return pusherClient
}

export function usePusherChannel(
  channelName: string,
  events: Record<string, (data: unknown) => void>
) {
  const channelRef = useRef<Channel | null>(null)

  useEffect(() => {
    const client = getPusherClient()
    const channel = client.subscribe(channelName)
    channelRef.current = channel

    for (const [event, handler] of Object.entries(events)) {
      channel.bind(event, handler)
    }

    return () => {
      for (const event of Object.keys(events)) {
        channel.unbind(event)
      }
      client.unsubscribe(channelName)
    }
  }, [channelName])
}
```

### Usage

```tsx
// components/ChatRoom.tsx
export function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<Message[]>([])

  usePusherChannel(`presence-room-${roomId}`, {
    'new-message': (data) => setMessages((p) => [...p, data as Message]),
  })

  return <MessageList messages={messages} />
}
```

---

## 5. Ably Setup

### Install

```bash
npm install ably
```

### Token Auth Endpoint

```ts
// app/api/ably/token/route.ts
import Ably from 'ably'
import { NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'

const ably = new Ably.Rest(process.env.ABLY_API_KEY!)

export async function GET() {
  const session = await getServerSession()
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const tokenParams = {
    clientId: session.user.id,
    capability: { '*': ['subscribe', 'publish', 'presence'] },
    ttl: 3600_000, // 1 hour in ms
  }

  const tokenRequest = await ably.auth.createTokenRequest(tokenParams)
  return NextResponse.json(tokenRequest)
}
```

### Client Hook

```ts
// hooks/useAbly.ts
import { useEffect, useRef, useState } from 'react'
import Ably, { RealtimeChannel } from 'ably'

let ablyClient: Ably.Realtime | null = null

function getAblyClient(clientId: string) {
  if (!ablyClient) {
    ablyClient = new Ably.Realtime({
      authUrl: '/api/ably/token',
      clientId,
    })
  }
  return ablyClient
}

export function useAblyChannel(
  channelName: string,
  clientId: string,
  events: Record<string, (msg: Ably.Message) => void>
) {
  const channelRef = useRef<RealtimeChannel | null>(null)
  const [connectionState, setConnectionState] = useState<string>('initialized')

  useEffect(() => {
    const client = getAblyClient(clientId)

    client.connection.on((stateChange) => {
      setConnectionState(stateChange.current)
    })

    const channel = client.channels.get(channelName)
    channelRef.current = channel

    const subscriptions = Object.entries(events).map(([event, handler]) => {
      channel.subscribe(event, handler)
      return event
    })

    return () => {
      subscriptions.forEach((event) => channel.unsubscribe(event))
      channel.detach()
    }
  }, [channelName, clientId])

  const publish = (event: string, data: unknown) => {
    channelRef.current?.publish(event, data)
  }

  return { publish, connectionState }
}

// Presence hook
export function useAblyPresence(channelName: string, clientId: string, userInfo: object) {
  const [members, setMembers] = useState<Ably.PresenceMessage[]>([])

  useEffect(() => {
    const client = getAblyClient(clientId)
    const channel = client.channels.get(channelName)

    channel.presence.enter(userInfo)
    channel.presence.subscribe(() => {
      channel.presence.get().then(setMembers)
    })
    channel.presence.get().then(setMembers)

    return () => {
      channel.presence.leave()
      channel.presence.unsubscribe()
    }
  }, [channelName])

  return { members }
}
```

---

## 6. Live Counter

### Online User Count (SSE-based)

```ts
// lib/presence-store.ts — in-memory (use Redis for multi-instance)
const rooms = new Map<string, Set<string>>()

export function join(room: string, id: string) {
  if (!rooms.has(room)) rooms.set(room, new Set())
  rooms.get(room)!.add(id)
}

export function leave(room: string, id: string) {
  rooms.get(room)?.delete(id)
}

export function count(room: string) {
  return rooms.get(room)?.size ?? 0
}
```

```ts
// app/api/counter/route.ts
import { NextRequest } from 'next/server'
import { join, leave, count } from '@/lib/presence-store'

const subscribers = new Map<string, Set<(n: number) => void>>()

function notifyRoom(room: string) {
  subscribers.get(room)?.forEach((fn) => fn(count(room)))
}

export async function GET(req: NextRequest) {
  const room = req.nextUrl.searchParams.get('room') ?? 'default'
  const clientId = crypto.randomUUID()
  const encoder = new TextEncoder()

  join(room, clientId)
  notifyRoom(room)

  const stream = new ReadableStream({
    start(controller) {
      const send = (n: number) => {
        controller.enqueue(encoder.encode(`data: ${n}\n\n`))
      }

      send(count(room))

      if (!subscribers.has(room)) subscribers.set(room, new Set())
      subscribers.get(room)!.add(send)

      req.signal.addEventListener('abort', () => {
        subscribers.get(room)?.delete(send)
        leave(room, clientId)
        notifyRoom(room)
        controller.close()
      })
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    },
  })
}
```

### Live Likes / Votes Hook

```ts
// hooks/useLiveCounter.ts
import { useState, useEffect } from 'react'

export function useLiveCounter(room: string) {
  const [count, setCount] = useState<number | null>(null)

  useEffect(() => {
    const es = new EventSource(`/api/counter?room=${room}`)
    es.onmessage = (e) => setCount(Number(e.data))
    es.onerror = () => es.close()
    return () => es.close()
  }, [room])

  const increment = async () => {
    await fetch(`/api/counter/vote`, {
      method: 'POST',
      body: JSON.stringify({ room, delta: 1 }),
      headers: { 'Content-Type': 'application/json' },
    })
  }

  return { count, increment }
}
```

```tsx
// components/LikeButton.tsx
export function LikeButton({ postId }: { postId: string }) {
  const { count, increment } = useLiveCounter(`post-${postId}`)
  const [optimistic, setOptimistic] = useState(0)

  const handleClick = () => {
    setOptimistic((n) => n + 1)
    increment()
  }

  return (
    <button onClick={handleClick} className="flex items-center gap-2">
      <HeartIcon />
      <span>{(count ?? 0) + optimistic}</span>
    </button>
  )
}
```

---

## 7. Presence Indicators

### Who's Online

```ts
// hooks/usePresence.ts (Pusher presence channel)
import { useEffect, useState } from 'react'
import { usePusherChannel } from './usePusher'
import type { Members } from 'pusher-js'

interface PresenceMember {
  id: string
  info: { name: string; avatar: string }
}

export function usePresence(roomId: string) {
  const [members, setMembers] = useState<PresenceMember[]>([])

  usePusherChannel(`presence-room-${roomId}`, {
    'pusher:subscription_succeeded': (data: { members: Members }) => {
      setMembers(Object.entries(data.members).map(([id, info]) => ({ id, info: info as PresenceMember['info'] })))
    },
    'pusher:member_added': (member: PresenceMember) => {
      setMembers((prev) => [...prev.filter((m) => m.id !== member.id), member])
    },
    'pusher:member_removed': (member: { id: string }) => {
      setMembers((prev) => prev.filter((m) => m.id !== member.id))
    },
  })

  return { members, onlineCount: members.length }
}
```

### Typing Indicators

```ts
// hooks/useTyping.ts
import { useState, useCallback, useRef } from 'react'

export function useTyping(send: (payload: unknown) => void) {
  const [typingUsers, setTypingUsers] = useState<string[]>([])
  const typingTimers = useRef<Map<string, ReturnType<typeof setTimeout>>>(new Map())
  const myTypingTimer = useRef<ReturnType<typeof setTimeout>>()
  const isTyping = useRef(false)

  const onKeyDown = useCallback(() => {
    if (!isTyping.current) {
      isTyping.current = true
      send({ type: 'typing_start' })
    }

    clearTimeout(myTypingTimer.current)
    myTypingTimer.current = setTimeout(() => {
      isTyping.current = false
      send({ type: 'typing_stop' })
    }, 2000)
  }, [send])

  const handleRemoteTyping = useCallback((userId: string, isTyping: boolean) => {
    if (isTyping) {
      setTypingUsers((prev) => [...new Set([...prev, userId])])

      clearTimeout(typingTimers.current.get(userId))
      typingTimers.current.set(
        userId,
        setTimeout(() => {
          setTypingUsers((prev) => prev.filter((u) => u !== userId))
        }, 3000)
      )
    } else {
      clearTimeout(typingTimers.current.get(userId))
      setTypingUsers((prev) => prev.filter((u) => u !== userId))
    }
  }, [])

  return { typingUsers, onKeyDown, handleRemoteTyping }
}
```

```tsx
// components/TypingIndicator.tsx
export function TypingIndicator({ users }: { users: string[] }) {
  if (users.length === 0) return null

  const label =
    users.length === 1
      ? `${users[0]} is typing...`
      : users.length === 2
      ? `${users[0]} and ${users[1]} are typing...`
      : `${users.length} people are typing...`

  return (
    <div className="flex items-center gap-2 text-sm text-muted-foreground">
      <span className="flex gap-0.5">
        {[0, 1, 2].map((i) => (
          <span
            key={i}
            className="h-1.5 w-1.5 rounded-full bg-current animate-bounce"
            style={{ animationDelay: `${i * 150}ms` }}
          />
        ))}
      </span>
      {label}
    </div>
  )
}
```

### Last Seen

```ts
// lib/last-seen.ts (Redis-backed)
import { redis } from './redis'

export async function updateLastSeen(userId: string) {
  await redis.set(`last-seen:${userId}`, Date.now(), { ex: 86400 })
}

export async function getLastSeen(userId: string): Promise<Date | null> {
  const ts = await redis.get<number>(`last-seen:${userId}`)
  return ts ? new Date(ts) : null
}

export function formatLastSeen(date: Date | null): string {
  if (!date) return 'Never'
  const diff = Date.now() - date.getTime()
  if (diff < 60_000) return 'Just now'
  if (diff < 3_600_000) return `${Math.floor(diff / 60_000)}m ago`
  if (diff < 86_400_000) return `${Math.floor(diff / 3_600_000)}h ago`
  return date.toLocaleDateString()
}
```

---

## 8. Live Notifications Feed

### Notification Store + SSE

```ts
// app/api/notifications/stream/route.ts
import { NextRequest } from 'next/server'
import { getServerSession } from 'next-auth'

// Shared emitter registry — use Redis pub/sub for multi-instance
const emitters = new Map<string, Set<(n: Notification) => void>>()

export function emitNotification(userId: string, notification: Notification) {
  emitters.get(userId)?.forEach((fn) => fn(notification))
}

export async function GET(req: NextRequest) {
  const session = await getServerSession()
  if (!session) return new Response('Unauthorized', { status: 401 })

  const userId = session.user.id
  const encoder = new TextEncoder()

  const stream = new ReadableStream({
    start(controller) {
      const send = (notification: Notification) => {
        controller.enqueue(
          encoder.encode(`event: notification\ndata: ${JSON.stringify(notification)}\n\n`)
        )
      }

      if (!emitters.has(userId)) emitters.set(userId, new Set())
      emitters.get(userId)!.add(send)

      // Heartbeat to keep connection alive through proxies
      const heartbeat = setInterval(() => {
        controller.enqueue(encoder.encode(': heartbeat\n\n'))
      }, 15_000)

      req.signal.addEventListener('abort', () => {
        clearInterval(heartbeat)
        emitters.get(userId)?.delete(send)
        controller.close()
      })
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    },
  })
}
```

### Client Notification Hook

```ts
// hooks/useNotifications.ts
import { useState, useEffect } from 'react'

interface Notification {
  id: string
  type: string
  title: string
  body: string
  href?: string
  readAt?: string
  createdAt: string
}

export function useNotifications() {
  const [notifications, setNotifications] = useState<Notification[]>([])
  const [unreadCount, setUnreadCount] = useState(0)

  useEffect(() => {
    const es = new EventSource('/api/notifications/stream')

    es.addEventListener('notification', (e) => {
      const notification = JSON.parse(e.data) as Notification
      setNotifications((prev) => [notification, ...prev])
      setUnreadCount((n) => n + 1)

      // Browser notification if permitted
      if (Notification.permission === 'granted') {
        new Notification(notification.title, { body: notification.body })
      }
    })

    return () => es.close()
  }, [])

  const markRead = async (id: string) => {
    await fetch(`/api/notifications/${id}/read`, { method: 'PATCH' })
    setNotifications((prev) =>
      prev.map((n) => (n.id === id ? { ...n, readAt: new Date().toISOString() } : n))
    )
    setUnreadCount((n) => Math.max(0, n - 1))
  }

  const markAllRead = async () => {
    await fetch('/api/notifications/read-all', { method: 'PATCH' })
    setNotifications((prev) => prev.map((n) => ({ ...n, readAt: new Date().toISOString() })))
    setUnreadCount(0)
  }

  return { notifications, unreadCount, markRead, markAllRead }
}
```

---

## 9. Collaborative Cursor Presence

### Cursor Broadcast Hook

```ts
// hooks/useCursorPresence.ts
import { useEffect, useState, useCallback, useRef } from 'react'
import { useWebSocket } from './useWebSocket'

interface CursorPosition {
  x: number
  y: number
  userId: string
  name: string
  color: string
}

const USER_COLORS = ['#ef4444', '#f97316', '#eab308', '#22c55e', '#3b82f6', '#8b5cf6']

export function useCursorPresence(roomId: string, currentUser: { id: string; name: string }) {
  const [cursors, setCursors] = useState<Map<string, CursorPosition>>(new Map())
  const myColor = USER_COLORS[parseInt(currentUser.id, 36) % USER_COLORS.length]
  const throttleRef = useRef<ReturnType<typeof setTimeout>>()

  const { send } = useWebSocket(`/ws?room=${roomId}&userId=${currentUser.id}`, {
    onMessage(msg: unknown) {
      const data = msg as { type: string; x: number; y: number; senderId: string; name: string; color: string }
      if (data.type === 'cursor') {
        setCursors((prev) => {
          const next = new Map(prev)
          next.set(data.senderId, {
            x: data.x,
            y: data.y,
            userId: data.senderId,
            name: data.name,
            color: data.color,
          })
          return next
        })
      }
      if (data.type === 'presence' && !(msg as { online: boolean }).online) {
        setCursors((prev) => {
          const next = new Map(prev)
          next.delete(data.senderId)
          return next
        })
      }
    },
  })

  const onMouseMove = useCallback(
    (e: React.MouseEvent<HTMLElement>) => {
      clearTimeout(throttleRef.current)
      throttleRef.current = setTimeout(() => {
        const rect = e.currentTarget.getBoundingClientRect()
        send({
          type: 'cursor',
          x: (e.clientX - rect.left) / rect.width,
          y: (e.clientY - rect.top) / rect.height,
          name: currentUser.name,
          color: myColor,
        })
      }, 16) // ~60fps throttle
    },
    [send, currentUser.name, myColor]
  )

  return { cursors, onMouseMove }
}
```

### Cursor Overlay Component

```tsx
// components/CursorOverlay.tsx
interface Cursor {
  x: number
  y: number
  userId: string
  name: string
  color: string
}

export function CursorOverlay({ cursors }: { cursors: Map<string, Cursor> }) {
  return (
    <div className="pointer-events-none absolute inset-0 overflow-hidden">
      {Array.from(cursors.values()).map((cursor) => (
        <div
          key={cursor.userId}
          className="absolute transition-all duration-75"
          style={{
            left: `${cursor.x * 100}%`,
            top: `${cursor.y * 100}%`,
            transform: 'translate(-2px, -2px)',
          }}
        >
          {/* Cursor SVG */}
          <svg width="16" height="20" viewBox="0 0 16 20" fill={cursor.color}>
            <path d="M0 0l6 16 2.5-6L14 8z" />
          </svg>
          {/* Name tag */}
          <span
            className="absolute left-4 top-0 whitespace-nowrap rounded px-1.5 py-0.5 text-xs font-medium text-white"
            style={{ backgroundColor: cursor.color }}
          >
            {cursor.name}
          </span>
        </div>
      ))}
    </div>
  )
}
```

### Usage

```tsx
// components/CollaborativeCanvas.tsx
export function CollaborativeCanvas({ roomId }: { roomId: string }) {
  const { cursors, onMouseMove } = useCursorPresence(roomId, currentUser)

  return (
    <div className="relative h-full w-full" onMouseMove={onMouseMove}>
      <Canvas />
      <CursorOverlay cursors={cursors} />
    </div>
  )
}
```

---

## 10. Optimistic Updates + Real-time Sync

### Merge Strategy

```ts
// lib/realtime-merge.ts

interface Item {
  id: string
  updatedAt: number
  [key: string]: unknown
}

type OperationType = 'insert' | 'update' | 'delete'

interface Operation {
  type: OperationType
  item: Item
  clientId: string
}

/**
 * Last-Write-Wins merge by updatedAt timestamp.
 * For collaborative text, use CRDT (e.g. Yjs) instead.
 */
export function mergeItems(local: Item[], remote: Operation): Item[] {
  switch (remote.type) {
    case 'insert': {
      const exists = local.some((i) => i.id === remote.item.id)
      return exists ? local : [...local, remote.item]
    }
    case 'update': {
      return local.map((item) => {
        if (item.id !== remote.item.id) return item
        // Remote wins if it's newer
        return item.updatedAt > remote.item.updatedAt ? item : { ...item, ...remote.item }
      })
    }
    case 'delete': {
      return local.filter((item) => item.id !== remote.item.id)
    }
  }
}
```

### Optimistic Update Hook

```ts
// hooks/useOptimisticList.ts
import { useState, useCallback } from 'react'

interface OptimisticItem {
  id: string
  _optimistic?: boolean
  _failed?: boolean
  [key: string]: unknown
}

export function useOptimisticList<T extends OptimisticItem>(initial: T[]) {
  const [items, setItems] = useState<T[]>(initial)

  const optimisticInsert = useCallback(
    async (tempItem: T, persist: () => Promise<T>) => {
      // 1. Show immediately
      setItems((prev) => [{ ...tempItem, _optimistic: true }, ...prev])

      try {
        // 2. Persist to server
        const saved = await persist()

        // 3. Replace temp with real
        setItems((prev) =>
          prev.map((item) => (item.id === tempItem.id ? { ...saved, _optimistic: false } : item))
        )
      } catch {
        // 4. Mark as failed — let user retry
        setItems((prev) =>
          prev.map((item) =>
            item.id === tempItem.id ? { ...item, _optimistic: false, _failed: true } : item
          )
        )
      }
    },
    []
  )

  const applyRemote = useCallback((op: { type: string; item: T }) => {
    setItems((prev) => {
      // Skip remote echoes of our own optimistic items
      if (op.type === 'insert' && prev.some((i) => i.id === op.item.id && !i._optimistic)) {
        return prev
      }
      // Apply merge
      if (op.type === 'insert') return [...prev.filter((i) => i.id !== op.item.id), op.item]
      if (op.type === 'update') return prev.map((i) => (i.id === op.item.id ? { ...i, ...op.item } : i))
      if (op.type === 'delete') return prev.filter((i) => i.id !== op.item.id)
      return prev
    })
  }, [])

  return { items, optimisticInsert, applyRemote }
}
```

---

## 11. Connection State UI

### Connection Badge Component

```tsx
// components/ConnectionBadge.tsx
type Status = 'connecting' | 'open' | 'closed' | 'error' | 'reconnecting'

const CONFIG: Record<Status, { label: string; color: string; pulse: boolean }> = {
  connecting: { label: 'Connecting', color: 'bg-yellow-400', pulse: true },
  open: { label: 'Live', color: 'bg-green-400', pulse: false },
  reconnecting: { label: 'Reconnecting', color: 'bg-orange-400', pulse: true },
  closed: { label: 'Offline', color: 'bg-gray-400', pulse: false },
  error: { label: 'Error', color: 'bg-red-400', pulse: false },
}

export function ConnectionBadge({ status }: { status: Status }) {
  const cfg = CONFIG[status]
  return (
    <div className="flex items-center gap-1.5">
      <span className="relative flex h-2 w-2">
        {cfg.pulse && (
          <span className={`absolute inline-flex h-full w-full animate-ping rounded-full ${cfg.color} opacity-75`} />
        )}
        <span className={`relative inline-flex h-2 w-2 rounded-full ${cfg.color}`} />
      </span>
      <span className="text-xs text-muted-foreground">{cfg.label}</span>
    </div>
  )
}
```

### Offline Banner

```tsx
// components/OfflineBanner.tsx
import { useEffect, useState } from 'react'

export function OfflineBanner() {
  const [offline, setOffline] = useState(!navigator.onLine)

  useEffect(() => {
    const goOnline = () => setOffline(false)
    const goOffline = () => setOffline(true)
    window.addEventListener('online', goOnline)
    window.addEventListener('offline', goOffline)
    return () => {
      window.removeEventListener('online', goOnline)
      window.removeEventListener('offline', goOffline)
    }
  }, [])

  if (!offline) return null

  return (
    <div className="fixed inset-x-0 top-0 z-50 bg-yellow-500 px-4 py-2 text-center text-sm font-medium text-black">
      You are offline — changes will sync when you reconnect
    </div>
  )
}
```

---

## 12. Performance + Scale

### Channel and Rate Limits

| Provider | Free tier channels | Messages/sec | Connections |
|---|---|---|---|
| Pusher Sandbox | 100 | 200/s | 100 concurrent |
| Ably Free | Unlimited | 6M/mo | 100 concurrent |
| Self-hosted ws | Unlimited | CPU-bound | Memory-bound |

### Fallback to Polling

```ts
// hooks/useRealtimeWithFallback.ts
import { useEffect, useRef, useState } from 'react'

interface Options<T> {
  sseUrl: string
  pollUrl: string
  pollInterval?: number
  onData: (data: T) => void
}

export function useRealtimeWithFallback<T>({
  sseUrl,
  pollUrl,
  pollInterval = 5000,
  onData,
}: Options<T>) {
  const [transport, setTransport] = useState<'sse' | 'polling'>('sse')
  const pollRef = useRef<ReturnType<typeof setInterval>>()

  useEffect(() => {
    if (!('EventSource' in window)) {
      // Browser doesn't support SSE — fall back immediately
      startPolling()
      return
    }

    const es = new EventSource(sseUrl)
    let failCount = 0

    es.onmessage = (e) => {
      failCount = 0
      onData(JSON.parse(e.data))
    }

    es.onerror = () => {
      failCount += 1
      if (failCount >= 3) {
        es.close()
        startPolling()
      }
    }

    function startPolling() {
      setTransport('polling')
      pollRef.current = setInterval(async () => {
        try {
          const res = await fetch(pollUrl)
          if (res.ok) onData(await res.json())
        } catch {
          // ignore
        }
      }, pollInterval)
    }

    return () => {
      es.close()
      clearInterval(pollRef.current)
    }
  }, [sseUrl, pollUrl, pollInterval])

  return { transport }
}
```

### Redis Pub/Sub for Multi-Instance SSE

```ts
// lib/redis-pubsub.ts — for horizontally scaled deployments
import { createClient } from 'redis'

const publisher = createClient({ url: process.env.REDIS_URL })
const subscriber = createClient({ url: process.env.REDIS_URL })

await publisher.connect()
await subscriber.connect()

type Handler = (message: string) => void
const handlers = new Map<string, Set<Handler>>()

await subscriber.pSubscribe('realtime:*', (message, channel) => {
  handlers.get(channel)?.forEach((fn) => fn(message))
})

export function subscribeToChannel(channel: string, handler: Handler) {
  const key = `realtime:${channel}`
  if (!handlers.has(key)) handlers.set(key, new Set())
  handlers.get(key)!.add(handler)
  return () => handlers.get(key)?.delete(handler)
}

export async function publishToChannel(channel: string, data: unknown) {
  await publisher.publish(`realtime:${channel}`, JSON.stringify(data))
}
```

### Message Batching (reduce noise at scale)

```ts
// lib/message-batcher.ts
export function createBatcher<T>(
  flush: (items: T[]) => void,
  delay = 50 // ms
) {
  let queue: T[] = []
  let timer: ReturnType<typeof setTimeout> | undefined

  return {
    add(item: T) {
      queue.push(item)
      if (!timer) {
        timer = setTimeout(() => {
          flush(queue)
          queue = []
          timer = undefined
        }, delay)
      }
    },
    flushNow() {
      clearTimeout(timer)
      flush(queue)
      queue = []
      timer = undefined
    },
  }
}
```

### Scale Checklist

- Use Redis pub/sub (or a managed broker) when running more than one server instance
- Send diffs/patches, not full state, for large documents
- Throttle cursor events to 16ms (60fps) on the client before sending
- Use presence channels only for rooms with < 1,000 members — above that switch to a counter
- Set a heartbeat interval of 15-25s to detect stale connections through load balancers
- Set socket timeout < your load balancer's idle timeout (typically LBs close at 60s)
- For collaborative text editing, use Yjs or Automerge (CRDT) instead of LWW merging
- Monitor connection churn — rapid connect/disconnect storms exhaust file descriptors
