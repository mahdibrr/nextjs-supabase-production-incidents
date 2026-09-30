# INC-022: A user is signed in as someone else — cached `Set-Cookie` or a shared server client

Last verified: 2026-09-30
Pinned to: `@supabase/ssr` cookie-based auth in a Next.js App Router app (a `proxy.ts` on Next.js 16, `middleware.ts` on 15 and earlier) deployed behind a CDN, with ISR, or on a runtime that reuses warm instances (Vercel Fluid compute). The cache-header argument to `setAll` exists from `@supabase/ssr` v0.10.0 (release notes, 2026-03-30).

> Related: [INC-005 session disappears after refresh](../incident-index/README.md#inc-005-session-disappears-after-refresh-in-ssr) is the opposite symptom (a session is lost). [INC-011](../incident-index/README.md#inc-011-server-actions-write-succeeds-but-ui-reads-old-state) covers stale *data* from a shared cache. This incident is a shared *session*: one user's auth token reaches another user's browser.

## Symptom

Rare, not reproducible on demand, and reported by users rather than caught by monitoring:

- A user opens the app and is signed in as a different account. Their browser holds the other user's access token.
- Server logs show a user's API requests switching to a different user's JWT mid-session, with no logout or login in between.
- It disappears after a refresh or a sign-out, and nobody can reproduce it locally.

Both reports behind this incident describe it that way. In [supabase/supabase-js#1682](https://github.com/supabase/supabase-js/issues/1682) (Nuxt), the reporter found another user's token in their own browser after opening their laptop; a second reporter on the same issue (Next.js App Router + `@supabase/ssr` on AWS Amplify) confirmed from CloudWatch logs that the JWT on one user's requests changed to another user's token during an active session, and that user then read and changed data under the other account. [supabase/ssr#94](https://github.com/supabase/ssr/issues/94) reports the same symptom; the reporter says it stopped once they removed ISR, but the root cause was never confirmed in that thread.

## Impact

Account takeover without an attacker: user A acts as user B, with B's RLS permissions, for as long as B's token is valid. No error is thrown and nothing fails, so error-rate alerts stay quiet. It is intermittent because it needs a token refresh and a cache hit on the same response, and that is why it survives code review and local testing.

## Root cause — three independent paths, each documented by Supabase

The contributor who investigated #1682 (and later authored the fix in supabase/ssr#176) concluded there are "two distinct causes for this symptom, and they are independent of each other": ISR, and CDN caching of SSR responses. The Supabase advanced SSR guide documents both, plus a third (a shared client on warm instances).

### Cause 1 — ISR caches a response that refreshed a session

`@supabase/ssr` refreshes an expired access token on the server and writes the new token to the response as a `Set-Cookie` header. The Supabase guide states: "If you use ISR on pages that trigger a Supabase session refresh, the cached response will include the `Set-Cookie` header containing the refreshed JWT. When that cached response is served to a subsequent user, their browser stores the token and they are signed in as the wrong person."

Repro shape:

```tsx
// app/dashboard/page.tsx — WRONG: ISR on a page that reads the session
export const revalidate = 60 // cached and served to every visitor for 60s

export default async function Dashboard() {
  const supabase = await createClient()        // may refresh the token → Set-Cookie
  const { data } = await supabase.auth.getClaims()
  // ...
}
```

### Cause 2 — A CDN or reverse proxy caches the `Set-Cookie` response

Same mechanism without ISR: the proxy (middleware) refreshes the token, the response carries `Set-Cookie`, and a CDN caches it. Supabase: "If your CDN (e.g. Vercel Edge, Cloudflare) caches that response and serves it to a different user, that user's browser will store the cached token and be signed in as the wrong person."

Before `@supabase/ssr` v0.10.0 the library set no cache headers on that response. The v0.10.0 release note reads "pass cache headers to setAll to prevent CDN caching of auth responses" ([supabase/ssr#176](https://github.com/supabase/ssr/pull/176)). From that version on, `setAll(cookiesToSet, headers)` receives `Cache-Control`, `Expires` and `Pragma` whenever a token refresh occurs, **but only an implementation that applies them to the response is protected**. PR #176 says so directly: TypeScript does not flag a `setAll` that declares only the first parameter, and such implementations "will silently miss applying the headers". Every `setAll` copied from an older quickstart is in that state.

```ts
// WRONG after upgrading to @supabase/ssr >= 0.10.0: the second argument is ignored
setAll(cookiesToSet) {
  cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value))
  response = NextResponse.next({ request })
  cookiesToSet.forEach(({ name, value, options }) => response.cookies.set(name, value, options))
}
```

The `Cache-Control` header does not decide caching on every CDN. The guide warns that CloudFront can still cache the response and its `Set-Cookie` "if its cache policy has a Minimum TTL greater than 0, or if cookies and the `Set-Cookie` header are not forwarded to the origin", and that managed platforms such as AWS Amplify "may similarly not fully respect `Cache-Control` headers set in your application". The #1682 Amplify reporter hit exactly this: they added `Cache-Control: private, no-store` but could not verify it took effect, because Amplify manages the CloudFront distribution.

### Cause 3 — A Supabase client kept at module scope on warm instances

Supabase: "Vercel's Fluid compute model can keep server instances warm and reuse them across requests. In some cases this means a Supabase client initialized in module scope — or stored in a shared variable — may be reused across requests from different users, causing one user's session to leak into another user's request." A commenter on [supabase/ssr#94](https://github.com/supabase/ssr/issues/94) gave the same diagnosis ("you are sharing a global variable in the server with your client session in it"); the reporter disputed it, so treat #94 as unresolved.

```ts
// lib/supabase/server.ts — WRONG: the client is cached in a variable that outlives the request
let client: SupabaseClient | undefined

export async function getClient() {
  if (!client) {
    const cookieStore = await cookies()   // the FIRST request's cookies
    client = createServerClient(URL, KEY, {
      cookies: { getAll: () => cookieStore.getAll(), setAll: () => {} },
    })
  }
  return client                          // every later request on this instance gets it
}
```

> Not a contradiction with [INC-017](./connection-pool-exhaustion-postmortem.md): a module-scope **Postgres pool** holds no user identity and is the recommended pattern there. A module-scope **Supabase auth client** is bound to one user's cookies, so it must be created per request. The official example says so in a comment: "With Fluid compute, don't put this client in a global environment variable. Always create a new one on each request."

## Detection (run these now)

### Check 1 — What do the auth responses tell a cache?

Use a throwaway test user. Replaying an old session cookie can trip refresh-token reuse detection, which Supabase says "revokes the whole session". Force a token refresh (or wait for the access token to expire), then inspect a response that carries `Set-Cookie`:

```bash
curl -sI https://your-domain.com/dashboard \
  -H "Cookie: sb-<project-ref>-auth-token=<expired-session-cookie>" \
  | grep -iE "set-cookie|cache-control|age|x-vercel-cache|cf-cache-status|x-cache"
```

- **Healthy:** every response with `Set-Cookie: sb-…-auth-token` also has `Cache-Control: private, no-cache, no-store, must-revalidate, max-age=0` (the value `@supabase/ssr` passes to `setAll`, in `src/cookies.ts`) or at least `private, no-store`, and no cache-hit indicator.
- **Incident:** a response carrying `Set-Cookie` that was served from cache — `age` greater than 0, `x-vercel-cache: HIT`, `cf-cache-status: HIT`, or `x-cache: Hit…`. A missing or public/`s-maxage` `Cache-Control` on such a response is the precondition. Do not treat an identical `Set-Cookie` on two requests as proof of caching: Supabase's reuse handling can legitimately return the active token again when its parent token is presented.

### Check 2 — Does your `setAll` use the second argument?

```bash
grep -rn "setAll(" --include=*.ts --include=*.tsx --include=*.js . | grep -v node_modules
npm ls @supabase/ssr
```

- **Healthy:** `@supabase/ssr` ≥ 0.10.0 and the proxy/middleware `setAll(cookiesToSet, headers)` applies `headers` to the response it returns.
- **Incident:** a single-parameter `setAll` in the proxy/middleware, or an older `@supabase/ssr` with no manual `Cache-Control`.

### Check 3 — ISR or static rendering on authenticated routes

```bash
grep -rnE "export const (revalidate|dynamic)" app/ | grep -v node_modules
```

- **Incident:** `revalidate = <number>` or `dynamic = 'force-static'` on a route that creates a Supabase server client or reads the session.

### Check 4 — Module-scope clients

```bash
# module-level variables that may hold a client, and the assignments that fill them
grep -rnE "^(export )?(let|var|const) \w+(: [A-Za-z<>| ]+)?( =|;|$)" --include=*.ts --include=*.tsx lib/ utils/ src/ 2>/dev/null | grep -iE "supabase|client"
grep -rnE "^\s*\w+ = (await )?create(Server|Browser)?Client\(" --include=*.ts --include=*.tsx . | grep -v node_modules
```

A hit is a finding when the server client (or anything built from `cookies()`) is stored in a module-level variable, or created at module scope, and returned to more than one request (Cause 3). `createBrowserClient` in the browser is a deliberate singleton and not this incident.

## Fix

### 1. Apply the cache headers in the proxy's `setAll` (`@supabase/ssr` ≥ 0.10.0)

This matches the official Supabase Next.js example (`examples/auth/nextjs/lib/supabase/proxy.ts` in `supabase/supabase`):

```ts
setAll(cookiesToSet, headers) {
  cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value))
  supabaseResponse = NextResponse.next({ request })
  cookiesToSet.forEach(({ name, value, options }) =>
    supabaseResponse.cookies.set(name, value, options)
  )
  Object.entries(headers).forEach(([key, value]) =>
    supabaseResponse.headers.set(key, value)
  )
},
```

If you return a different response (a redirect, a rewrite), copy the cookies **and** the cache headers onto it, as the Supabase guide shows:

```ts
const myNewResponse = NextResponse.next({ request })
myNewResponse.cookies.setAll(supabaseResponse.cookies.getAll())
for (const header of ['cache-control', 'expires', 'pragma']) {
  const value = supabaseResponse.headers.get(header)
  if (value) myNewResponse.headers.set(header, value)
}
return myNewResponse
```

On an older `@supabase/ssr`, set `Cache-Control: private, no-store` yourself on every response from a route that handles authentication.

### 2. No ISR where a session can refresh

Supabase: "Do not enable ISR on any route where authentication is handled or where a session refresh can occur. [...] In Next.js, use `export const dynamic = 'force-dynamic'` on pages that require authentication."

### 3. Make the CDN obey

For CloudFront, the guide lists three options: set Minimum TTL to 0 in the cache policy, send `Cache-Control: no-cache="Set-Cookie"`, or disable caching for authenticated routes (a TTL-0 policy or the managed `CachingDisabled` policy). On any managed CDN, verify with Check 1 that a response carrying `Set-Cookie` is never served from cache. If you cache SSR pages at all, cache only routes that never write `Set-Cookie`, and include the refresh-token cookie in the cache key for any route that serves user-specific content.

### 4. Create the Supabase server client inside the request

```ts
// lib/supabase/server.ts — create per call, never export an instance
export async function createClient() {
  const cookieStore = await cookies()
  return createServerClient(URL, KEY, { cookies: { /* getAll / setAll */ } })
}
```

### 5. Contain the blast radius of a leak that already happened

Sign the affected users out everywhere. The Supabase guide notes that `signOut()` without a scope "defaults to `scope: 'global'`, which revokes the refresh token for every session that user has". An already-issued access token stays valid until it expires, so any route that must notice a revoked session has to call `getUser()`: "`getClaims()` verifies the token's signature and expiry [...] but an unexpired token stays valid even when the session behind it was revoked."

## Prevention

- **CI gate on headers:** a smoke test against the preview deployment that sends an expired session cookie to one protected route and fails if the response has `Set-Cookie` without `no-store` (Check 1 as an assertion).
- **Lint for the old `setAll` shape:** fail the build when the proxy/middleware `setAll` declares a single parameter.
- **Route-config review:** any `export const revalidate` in a segment that imports the Supabase server client needs an explicit reviewer sign-off.
- **Pin `@supabase/ssr` ≥ 0.10.0** and re-read the SSR guide on every upgrade of it.
- **Code-review checklist line:** "Is the Supabase server client created inside the request, and does every response carrying `Set-Cookie` also carry `no-store`?"

## References

- Supabase — Advanced SSR guide, "Can I use server-side rendering with a CDN or cache?" (ISR, CDN/reverse proxy, CloudFront, Vercel Fluid compute): https://supabase.com/docs/guides/auth/server-side/advanced-guide#can-i-use-server-side-rendering-with-a-cdn-or-cache
- Supabase — Creating a Supabase client for SSR (`setAll` receives the cache headers; copying cookies and headers onto a new response): https://supabase.com/docs/guides/auth/server-side/creating-a-client
- Supabase official Next.js example — `examples/auth/nextjs/lib/supabase/proxy.ts`: https://github.com/supabase/supabase/blob/master/examples/auth/nextjs/lib/supabase/proxy.ts
- `@supabase/ssr` v0.10.0 release notes ("pass cache headers to setAll to prevent CDN caching of auth responses"): https://github.com/supabase/ssr/releases/tag/v0.10.0
- supabase/ssr#176 — the change, its rationale, and the silent-miss warning for single-parameter `setAll`: https://github.com/supabase/ssr/pull/176
- supabase/supabase-js#1682 — "Got user session from different user?" (Nuxt and Next.js on AWS Amplify reports; diagnosis by the author of supabase/ssr#176): https://github.com/supabase/supabase-js/issues/1682
- supabase/ssr#94 — "User token transfered to different user." (reporter suspects ISR; a commenter points to a shared server variable; unresolved): https://github.com/supabase/ssr/issues/94
