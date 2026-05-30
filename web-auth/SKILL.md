---
name: web-auth
description: >
  Next.js 14 App Router authentication patterns. Covers NextAuth v5, Clerk,
  and Supabase Auth end-to-end: setup, protected routes, RBAC, OAuth, session
  handling in Server and Client Components, UI state, and security rules.
origin: community
tags: [nextjs, auth, nextauth, clerk, supabase, typescript, app-router]
---

# web-auth — Next.js 14 App Router Authentication

## 1. Decision Table — NextAuth vs Clerk vs Supabase Auth

| Criterion | NextAuth v5 | Clerk | Supabase Auth |
|---|---|---|---|
| **Cost** | Free (OSS) | Free tier; paid for orgs/MFA | Free tier; usage-based |
| **Setup time** | Medium (~1 hr) | Fast (~20 min) | Medium (~45 min) |
| **UI components** | None (bring your own) | Pre-built, customizable | None (bring your own) |
| **Database** | Any via adapter | Clerk-managed | Supabase Postgres |
| **Row-Level Security** | Manual | No | Native (RLS) |
| **Organizations / Teams** | Manual | Built-in | Manual |
| **SSR / RSC support** | Yes (v5) | Yes | Yes (SSR pkg) |
| **MFA / Passkeys** | Manual | Built-in | Built-in |
| **Best for** | Full control, custom DB | Rapid dev, B2B SaaS | Apps already on Supabase |

**Pick NextAuth v5** when you own the database and need full control over the auth flow.  
**Pick Clerk** when you need org management, device sessions, or polished pre-built UI fast.  
**Pick Supabase Auth** when your data already lives in Supabase and you want RLS.

---

## 2. NextAuth v5 — Full Setup

### Install

```bash
npm install next-auth@beta
```

### `auth.ts` — core config

```typescript
// auth.ts  (project root)
import NextAuth from 'next-auth'
import GitHub from 'next-auth/providers/github'
import Google from 'next-auth/providers/google'
import Credentials from 'next-auth/providers/credentials'
import { PrismaAdapter } from '@auth/prisma-adapter'
import { prisma } from '@/lib/prisma'
import bcrypt from 'bcryptjs'
import { z } from 'zod'

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: 'jwt' },
  pages: {
    signIn: '/login',
    error: '/login',
  },
  providers: [
    GitHub({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    }),
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    Credentials({
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const parsed = z.object({
          email: z.string().email(),
          password: z.string().min(8),
        }).safeParse(credentials)

        if (!parsed.success) return null

        const user = await prisma.user.findUnique({
          where: { email: parsed.data.email },
        })
        if (!user?.hashedPassword) return null

        const valid = await bcrypt.compare(parsed.data.password, user.hashedPassword)
        if (!valid) return null

        return { id: user.id, email: user.email, name: user.name, role: user.role }
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id
        token.role = (user as any).role ?? 'user'
      }
      return token
    },
    async session({ session, token }) {
      if (token) {
        session.user.id = token.id as string
        session.user.role = token.role as string
      }
      return session
    },
  },
})
```

### TypeScript augmentation — add `role` and `id` to session

```typescript
// types/next-auth.d.ts
import NextAuth from 'next-auth'

declare module 'next-auth' {
  interface User {
    role?: string
  }
  interface Session {
    user: {
      id: string
      role: string
      email: string
      name?: string | null
      image?: string | null
    }
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    id: string
    role: string
  }
}
```

### Route handler

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/auth'
export const { GET, POST } = handlers
```

### Middleware

```typescript
// middleware.ts
import { auth } from '@/auth'
import { NextResponse } from 'next/server'

export default auth((req) => {
  const { pathname } = req.nextUrl
  const isLoggedIn = !!req.auth

  const publicPaths = ['/login', '/register', '/api/auth']
  const isPublic = publicPaths.some((p) => pathname.startsWith(p))

  if (!isLoggedIn && !isPublic) {
    return NextResponse.redirect(new URL('/login', req.url))
  }
  return NextResponse.next()
})

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

### Login page with Server Action

```typescript
// app/login/page.tsx
'use client'
import { signIn } from 'next-auth/react'
import { useRouter } from 'next/navigation'
import { useState } from 'react'

export default function LoginPage() {
  const router = useRouter()
  const [error, setError] = useState<string | null>(null)

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    const fd = new FormData(e.currentTarget)
    const result = await signIn('credentials', {
      email: fd.get('email'),
      password: fd.get('password'),
      redirect: false,
    })
    if (result?.error) {
      setError('Invalid credentials')
    } else {
      router.push('/dashboard')
    }
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4 max-w-sm mx-auto mt-16">
      {error && <p className="text-red-500 text-sm">{error}</p>}
      <input name="email" type="email" placeholder="Email" required className="w-full border px-3 py-2 rounded" />
      <input name="password" type="password" placeholder="Password" required className="w-full border px-3 py-2 rounded" />
      <button type="submit" className="w-full bg-blue-600 text-white py-2 rounded">Sign in</button>
      <button type="button" onClick={() => signIn('google')} className="w-full border py-2 rounded">
        Continue with Google
      </button>
    </form>
  )
}
```

### Session in Server Component

```typescript
// app/dashboard/page.tsx
import { auth } from '@/auth'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const session = await auth()
  if (!session) redirect('/login')

  return <h1>Welcome, {session.user.name}</h1>
}
```

### Required env vars

```bash
# .env.local
AUTH_SECRET=<openssl rand -base64 32>
AUTH_URL=http://localhost:3000
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
DATABASE_URL=postgresql://...
```

---

## 3. Clerk — Middleware, Provider, Components, Hooks

### Install

```bash
npm install @clerk/nextjs
```

### Middleware

```typescript
// middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server'

const isPublicRoute = createRouteMatcher([
  '/sign-in(.*)',
  '/sign-up(.*)',
  '/api/webhooks(.*)',
  '/',
])

export default clerkMiddleware((auth, req) => {
  if (!isPublicRoute(req)) {
    auth().protect()
  }
})

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

### Root layout — ClerkProvider

```typescript
// app/layout.tsx
import { ClerkProvider } from '@clerk/nextjs'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body>{children}</body>
      </html>
    </ClerkProvider>
  )
}
```

### Pre-built auth pages

```typescript
// app/sign-in/[[...sign-in]]/page.tsx
import { SignIn } from '@clerk/nextjs'
export default function Page() {
  return <SignIn />
}

// app/sign-up/[[...sign-up]]/page.tsx
import { SignUp } from '@clerk/nextjs'
export default function Page() {
  return <SignUp />
}
```

### UserButton and navigation

```typescript
// components/nav.tsx
'use client'
import { UserButton, SignInButton, SignedIn, SignedOut } from '@clerk/nextjs'

export function Nav() {
  return (
    <nav className="flex items-center justify-between p-4">
      <span className="font-bold">MyApp</span>
      <div className="flex items-center gap-4">
        <SignedOut>
          <SignInButton mode="modal">
            <button className="bg-blue-600 text-white px-4 py-2 rounded">Sign in</button>
          </SignInButton>
        </SignedOut>
        <SignedIn>
          <UserButton afterSignOutUrl="/" />
        </SignedIn>
      </div>
    </nav>
  )
}
```

### Client hooks — useAuth / useUser

```typescript
// components/profile-card.tsx
'use client'
import { useAuth, useUser } from '@clerk/nextjs'

export function ProfileCard() {
  const { isLoaded, isSignedIn, userId } = useAuth()
  const { user } = useUser()

  if (!isLoaded) return <div className="animate-pulse h-10 bg-gray-200 rounded" />
  if (!isSignedIn) return null

  return (
    <div className="flex items-center gap-3">
      <img src={user?.imageUrl} alt="" className="w-8 h-8 rounded-full" />
      <span>{user?.fullName}</span>
      <span className="text-xs text-gray-500">{userId}</span>
    </div>
  )
}
```

### currentUser() in Server Components

```typescript
// app/dashboard/page.tsx
import { currentUser } from '@clerk/nextjs/server'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const user = await currentUser()
  if (!user) redirect('/sign-in')

  const role = user.publicMetadata.role as string | undefined

  return (
    <div>
      <h1>Hello {user.firstName}</h1>
      {role === 'admin' && <AdminPanel />}
    </div>
  )
}
```

### Required env vars

```bash
# .env.local
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/dashboard
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/dashboard
```

---

## 4. Supabase Auth — SSR-Safe Clients, Middleware, Email, OAuth

### Install

```bash
npm install @supabase/supabase-js @supabase/ssr
```

### Server client — for Server Components and Route Handlers

```typescript
// lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import type { Database } from '@/types/supabase'

export function createClient() {
  const cookieStore = cookies()
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll()
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            )
          } catch {
            // Ignore: Server Component can't set cookies (middleware handles it)
          }
        },
      },
    }
  )
}
```

### Browser client — for Client Components

```typescript
// lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr'
import type { Database } from '@/types/supabase'

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

### Middleware — refresh session on every request

```typescript
// middleware.ts
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll()
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value))
          supabaseResponse = NextResponse.next({ request })
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          )
        },
      },
    }
  )

  // Refresh session — MUST do this before any auth checks
  const { data: { user } } = await supabase.auth.getUser()

  const { pathname } = request.nextUrl
  const publicPaths = ['/login', '/auth/callback', '/register']
  const isPublic = publicPaths.some((p) => pathname.startsWith(p))

  if (!user && !isPublic) {
    const url = request.nextUrl.clone()
    url.pathname = '/login'
    return NextResponse.redirect(url)
  }

  return supabaseResponse
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)'],
}
```

### Auth callback route — handles OAuth and magic link redirects

```typescript
// app/auth/callback/route.ts
import { createClient } from '@/lib/supabase/server'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export async function GET(request: NextRequest) {
  const { searchParams, origin } = new URL(request.url)
  const code = searchParams.get('code')
  const next = searchParams.get('next') ?? '/dashboard'

  if (code) {
    const supabase = createClient()
    const { error } = await supabase.auth.exchangeCodeForSession(code)
    if (!error) {
      return NextResponse.redirect(`${origin}${next}`)
    }
  }

  return NextResponse.redirect(`${origin}/login?error=auth_callback_failed`)
}
```

### Email / Password sign-up and sign-in (Server Actions)

```typescript
// app/login/actions.ts
'use server'
import { createClient } from '@/lib/supabase/server'
import { redirect } from 'next/navigation'
import { revalidatePath } from 'next/cache'
import { z } from 'zod'

const credentialsSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export async function signIn(formData: FormData) {
  const parsed = credentialsSchema.safeParse({
    email: formData.get('email'),
    password: formData.get('password'),
  })
  if (!parsed.success) return { error: 'Invalid input' }

  const supabase = createClient()
  const { error } = await supabase.auth.signInWithPassword(parsed.data)
  if (error) return { error: error.message }

  revalidatePath('/', 'layout')
  redirect('/dashboard')
}

export async function signUp(formData: FormData) {
  const parsed = credentialsSchema.safeParse({
    email: formData.get('email'),
    password: formData.get('password'),
  })
  if (!parsed.success) return { error: 'Invalid input' }

  const supabase = createClient()
  const { error } = await supabase.auth.signUp({
    ...parsed.data,
    options: { emailRedirectTo: `${process.env.NEXT_PUBLIC_SITE_URL}/auth/callback` },
  })
  if (error) return { error: error.message }

  return { success: 'Check your email to confirm your account.' }
}

export async function signOut() {
  const supabase = createClient()
  await supabase.auth.signOut()
  redirect('/login')
}
```

### OAuth — Google sign-in (Client Component)

```typescript
// components/oauth-buttons.tsx
'use client'
import { createClient } from '@/lib/supabase/client'

export function GoogleSignInButton() {
  async function handleGoogleSignIn() {
    const supabase = createClient()
    await supabase.auth.signInWithOAuth({
      provider: 'google',
      options: {
        redirectTo: `${window.location.origin}/auth/callback`,
        queryParams: { access_type: 'offline', prompt: 'consent' },
      },
    })
  }

  return (
    <button onClick={handleGoogleSignIn} className="flex items-center gap-2 border px-4 py-2 rounded">
      <GoogleIcon />
      Continue with Google
    </button>
  )
}
```

### Get authenticated user in Server Component

```typescript
// app/dashboard/page.tsx
import { createClient } from '@/lib/supabase/server'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const supabase = createClient()
  // Use getUser() — not getSession() — to verify JWT with Supabase servers
  const { data: { user }, error } = await supabase.auth.getUser()

  if (error || !user) redirect('/login')

  const { data: profile } = await supabase
    .from('profiles')
    .select('*')
    .eq('id', user.id)
    .single()

  return <div>Welcome, {profile?.display_name}</div>
}
```

### Required env vars

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...  # server-only, never expose to client
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

---

## 5. Protected Routes — Matcher Patterns, Role-Aware Redirects

### Matcher patterns

```typescript
// middleware.ts — matcher examples

// Protect everything except static assets and public pages
export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|images|fonts).*)',
  ],
}

// Explicit protected prefix list (alternative)
export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*', '/settings/:path*', '/api/((?!auth|public).*)'],
}
```

### Role-aware redirect logic (NextAuth example)

```typescript
// middleware.ts
import { auth } from '@/auth'
import { NextResponse } from 'next/server'

const ROLE_HOME: Record<string, string> = {
  admin: '/admin',
  moderator: '/moderation',
  user: '/dashboard',
}

export default auth((req) => {
  const { pathname } = req.nextUrl
  const session = req.auth
  const role = session?.user?.role ?? null

  // Unauthenticated — redirect to login
  if (!session && !pathname.startsWith('/login') && !pathname.startsWith('/api/auth')) {
    return NextResponse.redirect(new URL(`/login?callbackUrl=${encodeURIComponent(pathname)}`, req.url))
  }

  // Authenticated user hitting login — redirect to role home
  if (session && pathname === '/login') {
    const home = ROLE_HOME[role ?? 'user'] ?? '/dashboard'
    return NextResponse.redirect(new URL(home, req.url))
  }

  // Admin-only zone
  if (pathname.startsWith('/admin') && role !== 'admin') {
    return NextResponse.redirect(new URL('/dashboard?error=forbidden', req.url))
  }

  return NextResponse.next()
})
```

### Protect in Server Component (belt + suspenders)

```typescript
// lib/auth-utils.ts
import { auth } from '@/auth'
import { redirect } from 'next/navigation'

export async function requireAuth() {
  const session = await auth()
  if (!session) redirect('/login')
  return session
}

export async function requireRole(role: string) {
  const session = await requireAuth()
  if (session.user.role !== role) redirect('/dashboard?error=forbidden')
  return session
}

// Usage in a page
export default async function AdminPage() {
  const session = await requireRole('admin')
  return <div>Admin panel for {session.user.email}</div>
}
```

---

## 6. RBAC — Permission Map, Helpers, RoleGuard Component

### Permission map

```typescript
// lib/permissions.ts
export type Role = 'admin' | 'editor' | 'viewer' | 'user'
export type Permission =
  | 'post:create' | 'post:edit' | 'post:delete' | 'post:publish'
  | 'user:manage' | 'settings:edit' | 'analytics:view'

const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin: [
    'post:create', 'post:edit', 'post:delete', 'post:publish',
    'user:manage', 'settings:edit', 'analytics:view',
  ],
  editor: ['post:create', 'post:edit', 'post:publish', 'analytics:view'],
  viewer: ['analytics:view'],
  user: ['post:create'],
}

export function hasPermission(role: Role, permission: Permission): boolean {
  return ROLE_PERMISSIONS[role]?.includes(permission) ?? false
}

export function requirePermission(role: Role, permission: Permission): void {
  if (!hasPermission(role, permission)) {
    throw new Error(`Role '${role}' does not have permission '${permission}'`)
  }
}
```

### Helpers for server-side checks

```typescript
// lib/rbac.ts
import { auth } from '@/auth'
import { redirect } from 'next/navigation'
import { hasPermission, type Permission, type Role } from './permissions'

export async function checkPermission(permission: Permission) {
  const session = await auth()
  if (!session) redirect('/login')

  const role = session.user.role as Role
  if (!hasPermission(role, permission)) {
    redirect('/dashboard?error=forbidden')
  }
  return session
}

// In a Server Action
export async function deletePost(postId: string) {
  const session = await auth()
  if (!session) throw new Error('Unauthenticated')

  const role = session.user.role as Role
  if (!hasPermission(role, 'post:delete')) {
    throw new Error('Forbidden')
  }

  await prisma.post.delete({ where: { id: postId } })
}
```

### RoleGuard — async Server Component

```typescript
// components/role-guard.tsx
import { auth } from '@/auth'
import { hasPermission, type Permission, type Role } from '@/lib/permissions'

interface RoleGuardProps {
  permission: Permission
  children: React.ReactNode
  fallback?: React.ReactNode
}

export async function RoleGuard({ permission, children, fallback = null }: RoleGuardProps) {
  const session = await auth()
  if (!session) return fallback

  const role = session.user.role as Role
  if (!hasPermission(role, permission)) return fallback

  return <>{children}</>
}

// Usage
export default async function PostPage({ params }: { params: { id: string } }) {
  return (
    <div>
      <PostContent id={params.id} />
      <RoleGuard permission="post:edit">
        <EditButton id={params.id} />
      </RoleGuard>
      <RoleGuard permission="post:delete" fallback={<p className="text-xs text-gray-400">No delete access</p>}>
        <DeleteButton id={params.id} />
      </RoleGuard>
    </div>
  )
}
```

### RBAC check in API Route Handler

```typescript
// app/api/posts/[id]/route.ts
import { auth } from '@/auth'
import { hasPermission, type Role } from '@/lib/permissions'
import { NextResponse } from 'next/server'

export async function DELETE(req: Request, { params }: { params: { id: string } }) {
  const session = await auth()
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const role = session.user.role as Role
  if (!hasPermission(role, 'post:delete')) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }

  await prisma.post.delete({ where: { id: params.id } })
  return NextResponse.json({ success: true })
}
```

---

## 7. OAuth Providers — Google and GitHub for All Three Solutions

### NextAuth v5

```typescript
// auth.ts — add providers
import GitHub from 'next-auth/providers/github'
import Google from 'next-auth/providers/google'

providers: [
  GitHub({
    clientId: process.env.GITHUB_CLIENT_ID!,
    clientSecret: process.env.GITHUB_CLIENT_SECRET!,
  }),
  Google({
    clientId: process.env.GOOGLE_CLIENT_ID!,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    authorization: {
      params: {
        prompt: 'consent',
        access_type: 'offline',
        response_type: 'code',
      },
    },
  }),
]
```

**Dashboard setup:**
- Google: [console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials → OAuth 2.0 Client ID → Authorized redirect URIs: `http://localhost:3000/api/auth/callback/google`
- GitHub: github.com → Settings → Developer Settings → OAuth Apps → Callback: `http://localhost:3000/api/auth/callback/github`

### Clerk

Clerk manages OAuth entirely from its dashboard:

1. [dashboard.clerk.com](https://dashboard.clerk.com) → Configure → Social connections
2. Toggle Google and/or GitHub
3. For production: provide your own Client ID and Secret (recommended for branding)
4. No code changes needed — Clerk handles the callback automatically

```typescript
// Trigger OAuth from Client Component
import { useSignIn } from '@clerk/nextjs'

function OAuthButtons() {
  const { signIn } = useSignIn()

  async function signInWithGoogle() {
    await signIn?.authenticateWithRedirect({
      strategy: 'oauth_google',
      redirectUrl: '/sso-callback',
      redirectUrlComplete: '/dashboard',
    })
  }

  async function signInWithGitHub() {
    await signIn?.authenticateWithRedirect({
      strategy: 'oauth_github',
      redirectUrl: '/sso-callback',
      redirectUrlComplete: '/dashboard',
    })
  }

  return (
    <div className="flex flex-col gap-2">
      <button onClick={signInWithGoogle} className="border px-4 py-2 rounded">Google</button>
      <button onClick={signInWithGitHub} className="border px-4 py-2 rounded">GitHub</button>
    </div>
  )
}
```

### Supabase Auth

```typescript
// Trigger OAuth — Client Component
import { createClient } from '@/lib/supabase/client'

export function OAuthButtons() {
  const supabase = createClient()

  async function handleOAuth(provider: 'google' | 'github') {
    await supabase.auth.signInWithOAuth({
      provider,
      options: {
        redirectTo: `${window.location.origin}/auth/callback`,
        // Google-specific
        ...(provider === 'google' && {
          queryParams: { access_type: 'offline', prompt: 'consent' },
        }),
      },
    })
  }

  return (
    <div className="flex flex-col gap-2">
      <button onClick={() => handleOAuth('google')} className="border px-4 py-2 rounded">Google</button>
      <button onClick={() => handleOAuth('github')} className="border px-4 py-2 rounded">GitHub</button>
    </div>
  )
}
```

**Supabase dashboard setup:**
- [supabase.com/dashboard](https://supabase.com/dashboard) → Authentication → Providers
- Enable Google / GitHub, paste Client ID and Secret
- Add to OAuth app callback: `https://<project>.supabase.co/auth/v1/callback`

---

## 8. Session in Server vs Client Components

### Server Components

```typescript
// NextAuth — auth() is a server-only function
import { auth } from '@/auth'

export default async function ServerComponent() {
  const session = await auth()
  // session is fully resolved — no loading state needed
  if (!session) return null
  return <div>Hello {session.user.name} ({session.user.role})</div>
}

// Supabase — getUser() verifies JWT with the Auth server (preferred over getSession())
import { createClient } from '@/lib/supabase/server'

export default async function ServerComponent() {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return null
  return <div>Hello {user.email}</div>
}

// Clerk — currentUser() is the server equivalent
import { currentUser } from '@clerk/nextjs/server'

export default async function ServerComponent() {
  const user = await currentUser()
  if (!user) return null
  return <div>Hello {user.firstName}</div>
}
```

### Client Components

```typescript
// NextAuth — useSession() from SessionProvider
'use client'
import { useSession } from 'next-auth/react'

export function ClientComponent() {
  const { data: session, status } = useSession()

  if (status === 'loading') return <Skeleton />
  if (status === 'unauthenticated') return <SignInPrompt />

  return <div>Hello {session.user.name}</div>
}

// Supabase — onAuthStateChange for real-time session
'use client'
import { createClient } from '@/lib/supabase/client'
import { useEffect, useState } from 'react'
import type { User } from '@supabase/supabase-js'

export function ClientComponent() {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const supabase = createClient()

    supabase.auth.getUser().then(({ data: { user } }) => {
      setUser(user)
      setLoading(false)
    })

    const { data: { subscription } } = supabase.auth.onAuthStateChange((_, session) => {
      setUser(session?.user ?? null)
    })

    return () => subscription.unsubscribe()
  }, [])

  if (loading) return <Skeleton />
  if (!user) return <SignInPrompt />
  return <div>Hello {user.email}</div>
}
```

### Avoid redundant fetches — pass session as prop

```typescript
// app/dashboard/layout.tsx
import { auth } from '@/auth'
import { DashboardNav } from '@/components/dashboard-nav'

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const session = await auth()
  // Fetch once at layout level, pass down to avoid N server round-trips
  return (
    <div>
      <DashboardNav user={session?.user} />
      {children}
    </div>
  )
}

// components/dashboard-nav.tsx — receives user as prop, no auth() call needed
interface DashboardNavProps {
  user?: { name?: string | null; role: string }
}
export function DashboardNav({ user }: DashboardNavProps) {
  return <nav>{user?.name}</nav>
}
```

---

## 9. Auth State in UI — Conditional Nav, Loading Skeleton, SessionProvider

### SessionProvider setup (NextAuth)

```typescript
// app/providers.tsx
'use client'
import { SessionProvider } from 'next-auth/react'

export function Providers({ children }: { children: React.ReactNode }) {
  return <SessionProvider>{children}</SessionProvider>
}

// app/layout.tsx
import { Providers } from './providers'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

### Conditional navigation — all three states

```typescript
// components/header.tsx (NextAuth)
'use client'
import { useSession, signIn, signOut } from 'next-auth/react'
import Link from 'next/link'

export function Header() {
  const { data: session, status } = useSession()

  return (
    <header className="flex items-center justify-between px-6 py-4 border-b">
      <Link href="/" className="font-bold text-xl">MyApp</Link>

      <div className="flex items-center gap-4">
        {status === 'loading' && (
          // Loading skeleton — prevents layout shift
          <div className="flex items-center gap-3">
            <div className="h-8 w-24 bg-gray-200 animate-pulse rounded" />
            <div className="h-8 w-8 bg-gray-200 animate-pulse rounded-full" />
          </div>
        )}

        {status === 'unauthenticated' && (
          <button
            onClick={() => signIn()}
            className="bg-blue-600 text-white px-4 py-2 rounded text-sm"
          >
            Sign in
          </button>
        )}

        {status === 'authenticated' && session && (
          <div className="flex items-center gap-3">
            <span className="text-sm text-gray-600">{session.user.name}</span>
            {session.user.image && (
              <img src={session.user.image} alt="" className="w-8 h-8 rounded-full" />
            )}
            <button
              onClick={() => signOut()}
              className="text-sm text-gray-500 hover:text-gray-900"
            >
              Sign out
            </button>
          </div>
        )}
      </div>
    </header>
  )
}
```

### Auth-aware nav (Supabase with real-time listener)

```typescript
// components/nav.tsx
'use client'
import { createClient } from '@/lib/supabase/client'
import { useEffect, useState } from 'react'
import type { User } from '@supabase/supabase-js'
import Link from 'next/link'
import { useRouter } from 'next/navigation'

export function Nav() {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const router = useRouter()
  const supabase = createClient()

  useEffect(() => {
    const { data: { subscription } } = supabase.auth.onAuthStateChange((event, session) => {
      setUser(session?.user ?? null)
      setLoading(false)
      if (event === 'SIGNED_OUT') router.push('/login')
    })

    supabase.auth.getUser().then(({ data: { user } }) => {
      setUser(user)
      setLoading(false)
    })

    return () => subscription.unsubscribe()
  }, [])

  if (loading) return <NavSkeleton />

  return (
    <nav className="flex items-center justify-between p-4">
      <Link href="/">Logo</Link>
      {user ? (
        <div className="flex items-center gap-4">
          <span>{user.email}</span>
          <button onClick={() => supabase.auth.signOut()}>Sign out</button>
        </div>
      ) : (
        <Link href="/login">Sign in</Link>
      )}
    </nav>
  )
}

function NavSkeleton() {
  return (
    <nav className="flex items-center justify-between p-4">
      <div className="h-6 w-16 bg-gray-200 animate-pulse rounded" />
      <div className="h-8 w-20 bg-gray-200 animate-pulse rounded" />
    </nav>
  )
}
```

---

## 10. Security Rules

### httpOnly cookies — verify in middleware

```typescript
// Verify the session token lives in httpOnly cookies, not localStorage
// NextAuth v5 sets auth cookies as httpOnly by default via iron-session
// Supabase SSR pkg also uses httpOnly cookies via the server client
// Clerk uses httpOnly session cookies automatically

// To confirm: inspect application cookies in DevTools — the token cookie
// must have HttpOnly checked and NOT be readable via document.cookie
```

### CSRF — verify origin on state-changing requests

```typescript
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server'

function verifyCsrf(req: NextRequest): boolean {
  const origin = req.headers.get('origin')
  const host = req.headers.get('host')
  if (!origin || !host) return false
  try {
    const originHost = new URL(origin).host
    return originHost === host
  } catch {
    return false
  }
}

export async function POST(req: NextRequest) {
  if (!verifyCsrf(req)) {
    return NextResponse.json({ error: 'CSRF validation failed' }, { status: 403 })
  }
  // ... handle request
}
```

### JWT rotation — refresh tokens with NextAuth

```typescript
// auth.ts — add refresh token rotation
callbacks: {
  async jwt({ token, user, account }) {
    // Initial sign-in: store access and refresh tokens
    if (account) {
      token.accessToken = account.access_token
      token.refreshToken = account.refresh_token
      token.expiresAt = account.expires_at
      return token
    }

    // Not expired — return token as-is
    if (Date.now() < (token.expiresAt as number) * 1000) {
      return token
    }

    // Expired — rotate using refresh token
    try {
      const response = await fetch('https://oauth2.googleapis.com/token', {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: new URLSearchParams({
          client_id: process.env.GOOGLE_CLIENT_ID!,
          client_secret: process.env.GOOGLE_CLIENT_SECRET!,
          grant_type: 'refresh_token',
          refresh_token: token.refreshToken as string,
        }),
      })
      const refreshed = await response.json()
      if (!response.ok) throw refreshed

      return {
        ...token,
        accessToken: refreshed.access_token,
        expiresAt: Math.floor(Date.now() / 1000 + refreshed.expires_in),
        refreshToken: refreshed.refresh_token ?? token.refreshToken,
      }
    } catch {
      return { ...token, error: 'RefreshTokenError' }
    }
  },
},
```

### Session revocation — global sign-out

```typescript
// NextAuth — invalidate all sessions by incrementing a version stored in DB
// (JWT strategy doesn't have built-in server-side revocation; use database strategy
//  or store a jti in the token and maintain a denylist)

// Supabase — admin sign-out all sessions for a user
import { createClient } from '@supabase/supabase-js'

const supabaseAdmin = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
  { auth: { autoRefreshToken: false, persistSession: false } }
)

export async function revokeAllSessions(userId: string) {
  const { error } = await supabaseAdmin.auth.admin.signOut(userId, 'global')
  if (error) throw error
}

// Clerk — revoke all sessions for a user
import { clerkClient } from '@clerk/nextjs/server'

export async function revokeAllClerkSessions(userId: string) {
  await clerkClient.users.deleteUser(userId) // Or use session revocation endpoint
  // Per-session: await clerkClient.sessions.revokeSession(sessionId)
}
```

---

## 11. Anti-Patterns

### Never store tokens in localStorage

```typescript
// WRONG — XSS can steal the token
localStorage.setItem('auth_token', token)
const token = localStorage.getItem('auth_token')

// CORRECT — httpOnly cookie, set server-side
// NextAuth, Clerk, and Supabase SSR all handle this automatically
// Never manually handle tokens in browser storage
```

### Never over-expose session data to the client

```typescript
// WRONG — exposes hashed password and internal fields to client
callbacks: {
  async session({ session, token }) {
    session.user = token.user as any  // dumps entire DB row into session
    return session
  },
},

// CORRECT — only forward what the client needs
callbacks: {
  async session({ session, token }) {
    session.user.id = token.id as string
    session.user.role = token.role as string
    // Do not attach hashedPassword, internalNotes, stripeCustomerId, etc.
    return session
  },
},
```

### Never trust client-supplied role claims

```typescript
// WRONG — role from request body is attacker-controlled
export async function POST(req: Request) {
  const { role } = await req.json()
  if (role === 'admin') {
    // ... grant admin action
  }
}

// CORRECT — always read role from the verified session
import { auth } from '@/auth'

export async function POST(req: Request) {
  const session = await auth()
  if (!session) return new Response('Unauthorized', { status: 401 })

  const role = session.user.role  // from JWT, signed and verified server-side
  if (role !== 'admin') return new Response('Forbidden', { status: 403 })
  // ... grant admin action
}
```

### Supabase: getUser() not getSession() on the server

```typescript
// WRONG — getSession() reads from cookie without re-verifying with Supabase
const { data: { session } } = await supabase.auth.getSession()
const user = session?.user  // untrusted — cookie may be forged

// CORRECT — getUser() sends JWT to Supabase Auth server for verification
const { data: { user }, error } = await supabase.auth.getUser()
if (error || !user) redirect('/login')
```

### Don't skip middleware for API routes

```typescript
// WRONG — matcher excludes /api routes, leaving them unprotected
export const config = {
  matcher: ['/dashboard/:path*'],  // /api routes are wide open
}

// CORRECT — include API routes (with explicit exclusions for public endpoints)
export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico).*)',
    '/api/((?!auth|public|webhooks).*)',
  ],
}
```

### Never hardcode secrets

```typescript
// WRONG
const secret = 'my-super-secret-key-123'

// CORRECT — always from environment, validated at startup
const secret = process.env.AUTH_SECRET
if (!secret) throw new Error('AUTH_SECRET is not set')
```

### Don't call auth() / getUser() in every child component

```typescript
// WRONG — triggers N separate auth calls per render tree
export default async function Page() {
  return (
    <div>
      <HeaderComponent />   {/* calls auth() internally */}
      <SidebarComponent />  {/* calls auth() internally */}
      <ContentComponent />  {/* calls auth() internally */}
    </div>
  )
}

// CORRECT — call once at the layout or page level, pass as prop or use React cache()
import { cache } from 'react'
import { auth } from '@/auth'

export const getSession = cache(auth)  // deduplicates within a single request

export default async function Page() {
  const session = await getSession()
  return (
    <div>
      <HeaderComponent session={session} />
      <SidebarComponent session={session} />
      <ContentComponent session={session} />
    </div>
  )
}
```
