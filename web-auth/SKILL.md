Skill written to `C:\Users\farha\.claude\skills\web-auth\SKILL.md`.

The file covers all 11 requested sections with complete, copy-paste-ready TypeScript code targeting Next.js 14 App Router:

1. **Decision table** — NextAuth vs Clerk vs Supabase Auth with clear rules for when to pick each.
2. **NextAuth v5** — full setup: `auth.ts`, route handler, server actions, login page, env vars, TypeScript augmentation for custom `role` field.
3. **Clerk** — middleware with `createRouteMatcher`, `ClerkProvider`, `UserButton`, `useAuth`/`useUser`, `currentUser()` in Server Components, organization switcher.
4. **Supabase Auth** — SSR-safe server/browser clients, middleware session refresh, email/password actions, OAuth flow, auth callback route.
5. **Protected routes** — matcher patterns, role-aware redirect logic covering unauthenticated, authenticated-wrong-role, and auth-page redirect cases.
6. **RBAC** — permission map, `hasPermission`/`requireRole` helpers, async `RoleGuard` server component, checks in Server Components and API routes.
7. **OAuth providers** — Google and GitHub config for all three solutions with dashboard setup steps.
8. **Session in Server vs Client** — `auth()` / `getUser()` on server, `useSession()` / `onAuthStateChange` on client, prop-passing pattern to avoid redundant fetches.
9. **Auth state in UI** — conditional nav, three-state loading skeleton, `SessionProvider` wrapper.
10. **Security rules** — httpOnly cookie verification, CSRF origin check, JWT rotation with refresh token, global sign-out for Supabase, session revocation for NextAuth.
11. **Anti-patterns** — localStorage tokens, over-exposing session, trusting client role claims, `getSession()` vs `getUser()`, skipping middleware, hardcoded secrets.
