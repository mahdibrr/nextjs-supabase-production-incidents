# INC-008: revalidatePath runs but stale data persists

Last verified: 2026-07-19
Pinned to: Next.js 15 App Router (default caching semantics — `fetch` is **not** cached by default in 15; you must opt in via `cache: 'force-cache'`, `next: { revalidate }`, or `next: { tags }` for any of this to matter).

> **Next.js 16 note:** the single-argument `revalidateTag('tag')` used below is deprecated in 16 and produces a TypeScript error — pass a `cacheLife` profile (`revalidateTag('posts', 'max')`, stale-while-revalidate), or use the new Server-Actions-only `updateTag('posts')` when the user must read their own write immediately. Source: [Next.js 16 upgrade guide — Caching APIs](https://nextjs.org/docs/app/guides/upgrading/version-16#caching-apis).

## Symptom

A Server Action mutates the database, calls `revalidatePath('/dashboard')`, returns, the user reloads — and the UI still shows the pre-mutation state. No error thrown. The data in Postgres is correct (so it's not a write bug). The bug is in the cache invalidation, not the mutation.

## Root cause — three distinct causes, each with a minimal repro

### Cause 1 — Wrong path argument

`revalidatePath` matches against the **route file structure**, not the browser URL. The path must include a leading slash and must reflect the actual rendered route, including dynamic segments and route groups. Common misses:

- `revalidatePath('dashboard')` — no leading slash; does not match `/dashboard`.
- `revalidatePath('/posts')` when the rendered route is `/posts/[id]` — invalidates the index only, not the dynamic child.
- `revalidatePath('/dashboard')` when the route lives in `app/(app)/dashboard/page.tsx` — for a literal page this is fine, but if you target a dynamic pattern you must include the group: `revalidatePath('/(app)/dashboard/[id]', 'page')`.

Repro:

```ts
// app/posts/[id]/page.tsx  — rendered route is /posts/123
import { revalidatePath } from 'next/cache'

export async function editPost(id: string) {
  'use server'
  await db.update(posts).set({ title }).where(eq(posts.id, id))
  revalidatePath('/posts') // wrong — invalidates /posts index, not /posts/123
}
```

Fix — pass the exact rendered route (literal path, no `type` needed):

```ts
revalidatePath(`/posts/${id}`)          // literal path → no type required
// or, to invalidate every /posts/[id]:
revalidatePath('/posts/[id]', 'page')    // dynamic pattern → type is required
```

Per the official API: a literal path like `/product/1` takes no `type`; a dynamic pattern like `/product/[slug]` **requires** `type` as `'page'` or `'layout'`. Do not append `/page` or `/layout` — use the `type` parameter. With rewrites, pass the **destination** path (route file location), not the source URL.

### Cause 2 — Path vs tag mismatch

Data was fetched and tagged with `next: { tags: ['posts'] }`, but the action calls `revalidatePath`, or vice-versa. `revalidatePath` invalidates a **specific page or layout path**; `revalidateTag` invalidates **all cached data tagged with that tag across every page**. They operate on different cache layers — invalidating one does not invalidate the other.

Repro:

```ts
// app/blog/page.tsx
const posts = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] },        // data is cached under tag 'posts'
  cache: 'force-cache',
})

// app/actions.ts
export async function publishPost() {
  'use server'
  await createPost()
  revalidatePath('/blog')           // wrong mechanism — tag cache is untouched
}
```

After `revalidatePath('/blog')`, page `/blog` re-renders, but the `fetch(..., { next: { tags: ['posts'] } })` Data Cache entry is **still cached** (Next.js sees the tag as fresh) and the page renders from stale data. Fix — invalidate with the same mechanism the fetch opted into:

```ts
import { revalidateTag } from 'next/cache'
await createPost()
revalidateTag('posts')              // matches the tag the fetch declared
```

The reverse is also a bug: data fetched without tags (cached by URL) cannot be invalidated by `revalidateTag` — there's no tag bound to it. Use `revalidatePath` of the route, or add tags to the fetch.

### Cause 3 — Static/dynamic boundary

The route is **statically prerendered at build time** (`○` in `next build` output), so `revalidatePath` at request time has nothing to revalidate — the route is already a static HTML file with no request-time cache entry. This happens when a route uses no request-time APIs (`cookies()`, `headers()`, `searchParams`) and all its `fetch` calls opt into caching (or the route sets `export const dynamic = 'force-static'` / `export const revalidate = false`). The page is served as a static asset; `revalidatePath` only marks a path for revalidation **on the next visit**, but a fully static route has no Data Cache entry to refresh.

Repro:

```ts
// app/dashboard/page.tsx
export const dynamic = 'force-static'   // prerendered at build → ○ in build output

export default async function Page() {
  const data = await fetch('https://api.example.com/stats', {
    cache: 'force-cache',
  })
  return <Dashboard data={await data.json()} />
}

// app/actions.ts
export async function refreshStats() {
  'use server'
  await mutateStats()
  revalidatePath('/dashboard')   // nothing to revalidate — page is static HTML
}
```

Diagnostic — run `next build` and read the route markers next to each route:

```
Route (app)                              Size     First Load JS
┌ ○ /dashboard                           1.2 kB        87 kB     ← static (cause #3)
├ ƒ /posts/[id]                          2.1 kB        89 kB     ← dynamic server
└ ● /blog                                 3.0 kB        92 kB     ← prerendered + cached
```

Legend: `ƒ` = dynamic (server-rendered on demand), `○` = static, `●` = prerendered (static + cached data). If the route showing stale data is `○`, cause #3 is the culprit.

Fix — pick one based on the data's nature:

```ts
// If the data is user-scoped/per-request (dashboard, account, cart):
export const dynamic = 'force-dynamic'   // route becomes ƒ — fresh every request

// If the data is global but changes on mutation, keep it cached and tag it:
export default async function Page() {
  const data = await fetch('https://api.example.com/stats', {
    next: { tags: ['stats'] },
    cache: 'force-cache',
  })
  return <Dashboard data={await data.json()} />
}
// then in the action:
revalidateTag('stats')                   // invalidates the Data Cache entry
```

## Decision matrix

| Situation | Use |
| --- | --- |
| Page rendered from `fetch()` with `next: { tags }` | `revalidateTag('tag')` — matches the tag the fetch declared |
| Page rendered from `fetch()` without tags, cached by URL (`cache: 'force-cache'`) | `revalidatePath('/route')` of the route that displays the data |
| Server Action mutates then the same route renders | `revalidatePath('/route')` (literal) or `revalidatePath('/route/[id]', 'page')` (pattern) |
| Route is static (`○`) but needs fresh data per request | `export const dynamic = 'force-dynamic'` OR move fetch to tagged cache + `revalidateTag` |
| Cross-route invalidation (one mutation affects many pages) | `revalidateTag('domain')` — decouples invalidation from route paths |
| Time-based freshness, no on-demand mutation | `export const revalidate = 60` (seconds) on the segment, or `fetch(url, { next: { revalidate: 60 } })` |
| Cache a non-`fetch` function (ORM/DB query) | `unstable_cache(fn, [key], { tags: ['x'], revalidate: 3600 })` → invalidate with `revalidateTag('x')` |
| Override default fetch cache behavior for the whole segment | `export const fetchCache = 'force-no-store'` (dynamic) or `'force-cache'` (static) |
| Force a segment to opt out of static rendering | `export const dynamic = 'force-dynamic'` |
| Invalidate everything (nuclear) | `revalidatePath('/', 'layout')` — purges client cache + all cached data |

## Detection

1. Run `next build` and locate the route showing stale data in the route table. The marker tells you the layer:
   - `○` → cause #3 (static boundary). Fix the rendering mode or tag the fetch.
   - `ƒ` or `●` → cause #1 or #2 (path/tag mismatch).
2. Log the exact path/tag passed to revalidate inside the Server Action to catch #1/#2:

```ts
import { revalidatePath, revalidateTag } from 'next/cache'

export async function editPost(id: string) {
  'use server'
  await db.update(posts).set({ title }).where(eq(posts.id, id))

  if (process.env.NODE_ENV !== 'production') {
    console.log('[revalidate]', { path: `/posts/${id}`, tag: 'posts' })
  }
  revalidatePath(`/posts/${id}`)
  revalidateTag('posts')
}
```

3. Confirm the fetch's cache opt-in matches the invalidation. Grep the data-fetching layer:

```bash
# tags-based fetches:
rg "next:\s*\{\s*tags:" app/
# force-cache fetches:
rg "cache:\s*'force-cache'" app/
# segment configs:
rg "export const (dynamic|fetchCache|revalidate)" app/
```

If a `tags: ['posts']` fetch exists but the action calls only `revalidatePath`, that's cause #2.

## Fix summary

1. **Confirm the route's build marker** — `next build` → find the route → `○` means cause #3; `ƒ`/`●` means #1 or #2.
2. **Match the revalidate call to the fetch's cache mechanism** — tagged fetch → `revalidateTag`; URL-cached fetch → `revalidatePath` of the route; `unstable_cache` with `tags` → `revalidateTag`.
3. **Pass the exact rendered route path** — leading slash, include dynamic segments (`/posts/123` literal, or `/posts/[id]` with `type: 'page'`); use the **destination** path if rewrites are in play.
4. **Verify with a hard reload** after the action — `Ctrl+Shift+R` (bypass browser cache); confirm fresh data, then test a soft reload.

## Prevention

- **Tag fetches by domain** (`next: { tags: ['posts'] }`) and invalidate by tag (`revalidateTag('posts')`). Tags survive route renames, dynamic-segment changes, and refactor of path strings; paths do not.
- **A dev-only revalidation helper** that logs what was invalidated:

  ```ts
  // lib/revalidate.ts
  import { revalidatePath, revalidateTag } from 'next/cache'

  export function revalidate(opts: { path?: string; tag?: string }) {
    if (process.env.NODE_ENV !== 'production') {
      console.log('[revalidate]', opts)
    }
    if (opts.path) revalidatePath(opts.path)
    if (opts.tag) revalidateTag(opts.tag)
  }
  ```
- **CI/preview smoke test** — mutate → reload → assert fresh. A Playwright step that clicks the action, waits for navigation, and asserts the updated value catches regression before deploy.
- **Pin cache opt-ins explicitly** — in Next.js 15 `fetch` is uncached by default, so revalidation is a no-op unless the fetch declared `cache: 'force-cache'`, `next: { revalidate }`, or `next: { tags }`. Audit that the fetch you expect to be invalidated is actually cached.

## References

- Next.js Caching and Revalidating — https://nextjs.org/docs/app/guides/caching-without-cache-components
- `revalidatePath` API reference — https://nextjs.org/docs/app/api-reference/functions/revalidatePath
- `revalidateTag` API reference — https://nextjs.org/docs/app/api-reference/functions/revalidateTag
- `unstable_cache` API reference — https://nextjs.org/docs/app/api-reference/functions/unstable_cache
- Route Segment Config (`dynamic`, `fetchCache`, `revalidate`) — https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config
- [Server Actions Debugging Matrix](../playbooks/server-actions-debugging-matrix.md)