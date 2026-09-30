# INC-023: Server code authorizes with `getSession()` — the decision rests on an unverified cookie

Last verified: 2026-09-30
Pinned to: `@supabase/ssr` + `supabase-js` v2 in Next.js App Router server code (proxy/middleware, Server Components, Server Actions, Route Handlers). `getClaims()` is the documented replacement; its behavior differs between projects on asymmetric JWT signing keys and projects still on a symmetric secret (see Cause 2).

> Related: [INC-018 ORM bypassing RLS](./orm-bypassing-rls-postmortem.md) is where this bug turns into a data leak: an identity read from an unverified cookie, passed to a query that does not re-check the token.

## Symptom

There is usually no visible failure, and that is the problem. What teams see:

- The server log repeats this warning, emitted by `auth-js` the first time the `user` object returned by `getSession()` is accessed:

  ```
  Using the user object as returned from supabase.auth.getSession() or from some supabase.auth.onAuthStateChange() events could be insecure! This value comes directly from the storage medium (usually cookies on the server) and may not be authentic. Use supabase.auth.getUser() instead which authenticates the data by contacting the Supabase Auth server.
  ```

- Protected pages and Server Actions are gated like this, and every test passes:

  ```ts
  const { data: { session } } = await supabase.auth.getSession()
  if (!session) redirect('/login')
  const userId = session.user.id   // used for authorization decisions below
  ```

- A security review, not a user, reports that a page renders for a request whose session cookie was edited by hand.

## Impact

The Supabase SSR guide states it in a danger admonition: "Anyone can forge the session cookie, so trusting it without verification lets an attacker render another user's page." The page guard is bypassed, and so is every decision made from `session.user` (role checks, ownership checks, which tenant to load).

How far it goes depends on what the server does next with that identity:

- **Worst case:** the forged `session.user.id` feeds a query that does not re-verify the token: a `service_role` client, a Prisma/Drizzle connection ([INC-018](./orm-bypassing-rls-postmortem.md)), or a call to a third-party API. Then the attacker reads or writes another user's data.
- **Everything rendered from the session itself** (name, email, plan, role claims) is attacker-controlled on that request.

This incident makes no claim about queries that forward the forged token to the Supabase Data API under RLS; none of the sources below describe that path. Do not rely on it as a mitigation: the guard is already broken, and anything downstream that trusts `session.user` breaks with it.

## Root cause

### Cause 1 — `getSession()` returns what the cookie says, without verifying it

From the Supabase SSR guide: "*Never* trust `supabase.auth.getSession()` inside server code such as Proxy. It reads the session out of the cookie without revalidating it."

In the browser this is fine: the SDK wrote that storage itself. On the server the "storage" is the request's cookie header, which the client controls. The `auth-js` source attaches the warning above to the `user` object that `getSession()` returns, to flag exactly this ([`packages/core/auth-js/src/lib/helpers.ts`](https://github.com/supabase/supabase-js/blob/master/packages/core/auth-js/src/lib/helpers.ts)). The request to verify the JWT inside `getSession()` itself, [supabase/supabase-js#1706](https://github.com/supabase/supabase-js/issues/1706) ("`getSession` should validate the session with the JWT_SECRET"), has been open since 2024-05-13; the documented answer is to call a different method.

Repro — confirm the guard does not verify:

```ts
// app/api/whoami/route.ts — WRONG
import { createClient } from '@/lib/supabase/server'

export async function GET() {
  const supabase = await createClient()
  const { data: { session } } = await supabase.auth.getSession()
  if (!session) return new Response('unauthorized', { status: 401 })
  return Response.json({ id: session.user.id, email: session.user.email })
}
```

The Supabase auth-methods summary names the exact weakness: with `getSession()`, "The session is loaded directly from local storage and isn't re-validated against the Auth server, so the embedded user object shouldn't be trusted on its own when storage is shared with the client (cookies, request headers)." To test it, take a valid `sb-<project-ref>-auth-token` cookie, decode the stored session, change the embedded `user` object (for example `user.id`), re-encode it the same way, and send it to this route. Against the `getClaims()` version below the edit has no effect on identity, because "The returned claims always come from decoding the JWT, not from a user lookup" — `claims.sub` is still the real user.

### Cause 2 — Assuming `getClaims()` is always a local check (the fix, misread)

`getClaims()` is the documented replacement, but it is not "no network call" on every project. The `getClaims` reference: "If the project is not using an asymmetric JWT signing key (like ECC or RSA) it always sends a request to the Auth server (similar to GoTrueClient.getUser) to verify the JWT." The SSR guide says the same: on projects with asymmetric signing keys ("the default for new projects") it verifies locally against a cached copy of the project's public keys; "On projects still using a symmetric secret, it calls the Auth server instead."

Nothing breaks, but the latency budget someone planned around a local check is wrong on older projects. Migrating to asymmetric signing keys is what makes it local.

### Cause 3 — Assuming a verified token means a live session

`getClaims()` checks signature and expiry. The Supabase guide: "an unexpired token stays valid even when the session behind it was revoked. Call `getUser()` where that gap matters." Sign-out-everywhere, a password change, or a ban does not stop a token that has not expired yet from passing `getClaims()`.

## Detection (run these now)

### Check 1 — Find server-side `getSession()` calls

```bash
grep -rn "auth.getSession()" --include=*.ts --include=*.tsx app/ lib/ src/ proxy.ts middleware.ts 2>/dev/null
```

Any hit in a file without `'use client'` (proxy/middleware, Server Components, Server Actions, Route Handlers, `lib/` helpers they import) is a finding. Hits inside Client Components are not.

### Check 2 — Search the server logs for the auth-js warning

Search production function logs for `could be insecure! This value comes directly from the storage medium`. Every hit is a server-side read of `session.user`.

### Check 3 — Know which `getClaims()` path you are on

In the Supabase dashboard, check whether the project signs JWTs with an asymmetric key (ECC/RSA) or a legacy symmetric secret. On a symmetric secret, every `getClaims()` call is a request to the Auth server (Cause 2).

### Check 4 — Negative test

Send the tampered-cookie request from the Cause 1 repro to each protected route. Expected: 401 or a redirect to login. A 200 that answers with the edited `user.id` is this incident.

## Fix

### 1. Replace `getSession()` with `getClaims()` in every server-side authorization check

```ts
// Server Component / Server Action / Route Handler
const supabase = await createClient()
const { data, error } = await supabase.auth.getClaims()
if (error || !data?.claims) redirect('/login')

const userId = data.claims.sub
```

The official Supabase Next.js proxy calls `getClaims()` right after creating the client, with this warning in the example: "Do not run code between createServerClient and supabase.auth.getClaims(). A simple mistake could make it very hard to debug issues with users being randomly logged out."

### 2. Use `getUser()` where revocation must be noticed

For destructive or high-value operations (deleting an account, changing email or password, payouts, admin actions), call `getUser()`, which asks the Auth server for the current user record, so a revoked session is rejected (Cause 3).

### 3. Keep `getSession()` for what it is for

Supabase documents `getSession` for "when you need the raw session (the access token, refresh token, and expiry). For example to forward the access token to another service." Reading the raw token is fine. Deciding who the user is from its `user` object, on the server, is not: "don't rely on the user object it returns for authorization decisions."

### 4. Never pass an unverified identity to a query that skips RLS

If a server path uses `service_role` or an ORM connection, the user id it filters on must come from `getClaims()`/`getUser()`, never from `getSession()`. See [INC-018](./orm-bypassing-rls-postmortem.md) for making such paths honor RLS at all.

## Prevention

- **Lint rule:** ban `auth.getSession()` in any file without `'use client'` (an ESLint `no-restricted-syntax` rule on the member call, scoped with an override for client directories).
- **Log alert:** alert on the auth-js "could be insecure!" warning in production logs; it should never appear after the fix.
- **E2E negative test:** the tampered-cookie request from Check 4 runs in CI against each protected route and must not return 200.
- **Code-review checklist line:** "Does every server-side identity come from `getClaims()` (or `getUser()` for revocation-sensitive actions), never from `getSession()`?"

## References

- Supabase — Creating a Supabase client for SSR (danger admonition: forged session cookie, "Never trust `supabase.auth.getSession()` inside server code", `getClaims()` local vs Auth-server verification): https://supabase.com/docs/guides/auth/server-side/creating-a-client
- Supabase — Server-side auth for Next.js, "Choosing an auth method" (`getClaims` / `getUser` / `getSession`; the embedded user object from `getSession` is not to be trusted): https://supabase.com/docs/guides/auth/server-side/nextjs
- Supabase — Advanced SSR guide (revoked sessions: `getClaims()` vs `getUser()`): https://supabase.com/docs/guides/auth/server-side/advanced-guide
- Supabase JS reference — `auth.getClaims()` (symmetric-key projects always call the Auth server): https://supabase.com/docs/reference/javascript/auth-getclaims
- Supabase official Next.js example — `examples/auth/nextjs/lib/supabase/proxy.ts`: https://github.com/supabase/supabase/blob/master/examples/auth/nextjs/lib/supabase/proxy.ts
- `auth-js` source of the "could be insecure!" warning: https://github.com/supabase/supabase-js/blob/master/packages/core/auth-js/src/lib/helpers.ts
- supabase/supabase-js#1706 — "`getSession` should validate the session with the JWT_SECRET" (open): https://github.com/supabase/supabase-js/issues/1706
