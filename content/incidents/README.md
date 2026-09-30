# Symptom Reference: Next.js + Supabase Debugging

A symptom-organized debugging reference for Next.js and Supabase apps, grouped by area (auth, RLS, deployment, caching, billing, database, realtime), with links to the official docs that fix each one. Use this when the symptom matters more than the framework category.

> Looking for the numbered incident catalog that maps each failure to a reusable asset in this repo (audit SQL, templates, playbooks)? See the [Production Incident Index](../../reference/incident-index/README.md).

Search phrases covered naturally here include: "Supabase returns empty array", "Next.js auth session lost after refresh", "RLS silently fails", and "middleware redirect loop".

## How To Use

1. Match the production symptom.
2. Check the likely root cause.
3. Apply the fix.
4. Run the verification step before shipping.

## Authentication

| Symptom                                                                     | Root Cause                                                                                        | Fix                                                                                                                                                                                              |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Next.js auth session lost after refresh.                                    | Browser client has a session, but the server client is not reading or refreshing cookies.         | Use Supabase SSR Auth and verify cookies on a server-rendered route. See [Supabase Server-Side Auth](https://supabase.com/docs/guides/auth/server-side).                                         |
| OAuth login succeeds, then redirects back to login.                         | Callback URL is not allowlisted, or middleware redirects the callback route before code exchange. | Add every local, preview, and production callback URL in Supabase Auth settings. See [Supabase Auth Redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls).                         |
| Magic link works locally but fails in production.                           | `SITE_URL`, redirect allowlist, or email template URL points to a local or stale domain.          | Set the production Site URL and use explicit `redirectTo` values. See [Supabase Auth Email Templates](https://supabase.com/docs/guides/auth/auth-email-templates).                               |
| User appears logged in in the navbar but server route returns unauthorized. | UI trusts client state while route handlers call `auth.getUser()` and find no valid cookie.       | Treat server auth as source of truth and require a server user check inside protected route handlers. See [Supabase Auth for Next.js](https://supabase.com/docs/guides/auth/server-side/nextjs). |
| Middleware redirect loop between `/login` and `/dashboard`.                 | Middleware protects login/callback routes or refreshes cookies on every redirect path.            | Exclude auth routes from the matcher and return the same response object after cookie writes. See [Next.js Proxy docs](https://nextjs.org/docs/app/api-reference/file-conventions/proxy).        |
| User opens the app signed in as a different account.                       | A session refresh's `Set-Cookie` response was cached (ISR or CDN) and served to another user, or a Supabase client is shared at module scope. | No ISR on auth routes, apply the `setAll` cache headers from `@supabase/ssr` ≥ 0.10.0, and create the client per request. See [INC-022 postmortem](../../reference/playbooks/session-leak-cached-set-cookie-postmortem.md) and [Supabase SSR caching](https://supabase.com/docs/guides/auth/server-side/advanced-guide#can-i-use-server-side-rendering-with-a-cdn-or-cache). |
| Logs warn `Using the user object as returned from supabase.auth.getSession() ... could be insecure!` | Server code decides identity from `getSession()`, which reads the cookie without revalidating it. | Use `getClaims()` (or `getUser()` for revocation-sensitive actions). See [INC-023 postmortem](../../reference/playbooks/getsession-server-trust-postmortem.md). |

## RLS

| Symptom                                                | Root Cause                                                                                         | Fix                                                                                                                                                                                                                    |
| ------------------------------------------------------ | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Supabase returns empty array for rows that exist.      | RLS is enabled and no `select` policy matches the authenticated role.                              | Add a policy for `authenticated` and test as the real user, not the service role. See [Supabase Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security).                            |
| RLS silently fails on update.                          | `using` allows row visibility, but `with check` blocks the new row state.                          | Define both `using` and `with check` for update policies. See [RLS policies](https://supabase.com/docs/guides/database/postgres/row-level-security#policies).                                                          |
| Insert succeeds with service role but fails for users. | App is accidentally tested with service role locally, but production uses anon/authenticated keys. | Test RLS with anon and authenticated clients. Never expose the service role key. See [Supabase API keys](https://supabase.com/docs/guides/api/api-keys).                                                               |
| Multi-tenant users can see no team rows.               | Policy checks `owner_id = auth.uid()` but data is owned by team or workspace.                      | Use an `exists` check against a membership table and index `team_id` plus `user_id`. See [RLS performance best practices](https://supabase.com/docs/guides/troubleshooting/rls-performance-and-best-practices-Z5Jjwv). |
| RLS policy works but query becomes slow in production. | Policy subqueries scan membership or tenant tables without indexes.                                | Add indexes for every column used in policy predicates. Verify with PostgreSQL `EXPLAIN`. See [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html).                                        |

## Deployment

| Symptom                                                         | Root Cause                                                                                                        | Fix                                                                                                                                                                            |
| --------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Vercel deployment cannot connect to Supabase.                   | Production environment variables are missing or set in the wrong Vercel environment.                              | Check Production, Preview, and Development env scopes. See [Vercel environment variables](https://vercel.com/docs/projects/environment-variables).                             |
| App works locally but production callback uses localhost.       | Supabase Site URL or redirect allowlist still contains local-only URLs.                                           | Add exact production and preview callback URLs. See [Supabase redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls).                                             |
| Server Action works locally but fails at runtime.               | The action uses Node-only APIs while deployed to an Edge runtime, or the route inherited an incompatible runtime. | Pin runtime to Node.js for code that needs Node APIs. See [Next.js route segment config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config).     |
| Build passes but runtime says env var is undefined.             | Server-only env vars are accessed in client components or public vars are missing `NEXT_PUBLIC_`.                 | Keep secrets server-only and expose only safe public values. See [Next.js environment variables](https://nextjs.org/docs/app/guides/environment-variables).                    |
| Production logs show intermittent database connection failures. | Serverless functions open too many direct PostgreSQL connections.                                                 | Use Supabase connection pooling for serverless traffic. See [Supabase connection pooling](https://supabase.com/docs/guides/database/connecting-to-postgres#connection-pooler). |
| `FATAL: Tenant or user not found` right after switching to the pooler string. | Shared-pooler username is missing `.<project-ref>`, or the pooler host was typed instead of copied. | Copy the full string from the Connect dialog and replace only the password. See [INC-024 postmortem](../../reference/playbooks/pooler-tenant-not-found-postmortem.md) and [Supabase troubleshooting](https://supabase.com/docs/guides/troubleshooting/tenant-or-user-not-found). |

## Caching

| Symptom                                                   | Root Cause                                                              | Fix                                                                                                                                                                   |
| --------------------------------------------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| User-specific dashboard shows another user's stale state. | Authenticated data was fetched through a cached path or static segment. | Mark user-specific data as dynamic and avoid sharing cache across users. See [Next.js caching](https://nextjs.org/docs/app/deep-dive/caching).                        |
| Updated Supabase data does not appear after mutation.     | Mutation succeeds, but the path/tag cache is not invalidated.           | Call `revalidatePath` or `revalidateTag` after Server Actions. See [Next.js revalidating data](https://nextjs.org/docs/app/getting-started/caching-and-revalidating). |
| Public page remains stale after deploy.                   | Static generation or CDN cache serves older content.                    | Verify cache headers and revalidation settings in production. See [Vercel cache docs](https://vercel.com/docs/edge-cache).                                            |
| Middleware changes appear ignored.                        | Browser or deployment cache hides old redirects or headers.             | Test with `curl -I`, a fresh deployment, and Vercel runtime logs. See [Vercel runtime logs](https://vercel.com/docs/logs/runtime).                                    |

## Billing

| Symptom                                                  | Root Cause                                                                                  | Fix                                                                                                                                                                     |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Stripe webhook succeeds in test but fails in production. | Wrong signing secret, raw body not preserved, or endpoint URL differs between environments. | Verify the webhook signature with the production secret. See [Stripe webhooks](https://docs.stripe.com/webhooks).                                                       |
| Paid user still sees free plan after checkout.           | Checkout completed, but webhook sync did not update Supabase subscription state.            | Store customer and subscription IDs, then update state from webhook events. See [Vercel subscription payments](https://github.com/vercel/nextjs-subscription-payments). |
| Duplicate billing events create duplicate rows.          | Webhook handler is not idempotent and processes retried Stripe events multiple times.       | Store processed event IDs or upsert by Stripe object ID. See [Stripe webhook best practices](https://docs.stripe.com/webhooks#handle-duplicate-events).                 |
| Customer portal works locally but fails in production.   | Return URL or Stripe customer ID is missing for the production user.                        | Persist `stripe_customer_id` on account creation and validate portal return URLs. See [Stripe customer portal](https://docs.stripe.com/customer-management).            |

## Database

| Symptom                                                     | Root Cause                                                                               | Fix                                                                                                                                                                    |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Query gets slower as tenants grow.                          | Tenant filter, RLS predicate, or foreign-key lookup lacks an index.                      | Add compound indexes matching tenant and membership predicates. Verify with `EXPLAIN`. See [PostgreSQL indexes](https://www.postgresql.org/docs/current/indexes.html). |
| Database migration works locally but breaks production.     | Migration assumes empty tables, missing extensions, or unseeded data.                    | Make migrations idempotent and test against a production-like branch. See [Supabase migrations](https://supabase.com/docs/guides/local-development/overview).          |
| Function or trigger works in SQL editor but fails from app. | Function security mode, search path, or RLS interaction differs by caller role.          | Set explicit `security definer` only when needed and define `search_path`. See [Supabase database functions](https://supabase.com/docs/guides/database/functions).     |
| API returns too much data.                                  | Query misses tenant/user filter or relies on client-side filtering after a broad select. | Put authorization in RLS and keep API queries scoped. See [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/).                      |

## Realtime

| Symptom                                             | Root Cause                                                                                     | Fix                                                                                                                                                                                  |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Realtime subscription receives no database changes. | Replication is not enabled for the table or RLS blocks the listening user.                     | Enable Realtime for the table and test with the same authenticated role. See [Supabase database changes](https://supabase.com/docs/guides/realtime/subscribing-to-database-changes). |
| Messages appear twice in chat.                      | Optimistic UI inserts a local message, then Realtime inserts the same row again.               | De-duplicate by message ID and reconcile optimistic rows after insert confirmation. See [Supabase Realtime](https://supabase.com/docs/guides/realtime).                              |
| Presence state grows after navigation.              | Client subscribes repeatedly and does not unsubscribe on unmount.                              | Remove channels on cleanup and verify subscriptions in browser devtools. See [Supabase Realtime channels](https://supabase.com/docs/guides/realtime/concepts).                       |
| Realtime works for admins but not normal users.     | Admin uses service role or broad RLS policy, while normal users are blocked by row visibility. | Test with a normal authenticated user and align Realtime access with RLS policies. See [Realtime authorization](https://supabase.com/docs/guides/realtime/authorization).            |

## Quick Debugging Snippets

Check whether the server sees the same user as the browser:

```ts
const {
  data: { user },
  error,
} = await supabase.auth.getUser();

console.log({
  userId: user?.id,
  authError: error?.message,
});
```

Verify a route is not statically caching user data:

```ts
export const dynamic = "force-dynamic";
```

Test cache and redirect behavior from production:

```bash
curl -I https://your-domain.com/dashboard
curl -I https://your-domain.com/auth/callback
```

Check whether RLS is blocking a query:

```sql
select auth.uid();
select * from your_table limit 5;
```

If the SQL editor works but the app returns empty rows, retest as the real `authenticated` role and inspect policies.

## Verification Checklist

- Confirm the failing request uses the same user, role, domain, and environment as production.
- Check browser network tab for redirects, cookies, cache headers, and status codes.
- Check Vercel runtime logs and Supabase Auth/database logs before changing code.
- Test RLS with anon and authenticated clients, not only the service role.
- Verify Stripe webhooks with production signing secrets.
- Reproduce once, patch once, then re-run the original failing path.

## Common Mistakes

- Trusting client auth state for server authorization.
- Using the service role key in local tests and assuming RLS works.
- Protecting `/auth/callback` with middleware.
- Caching user-specific data in a shared route.
- Forgetting `with check` on update policies.
- Processing Stripe webhook retries without idempotency.
- Creating Realtime subscriptions without cleanup.
## Production Cross-Links

- Incident Index row: [INC-002 Supabase returns empty array](../../reference/incident-index/README.md#inc-002-supabase-returns-empty-array-for-existing-rows)
- Reference asset: [RLS audit SQL](../../reference/sql/rls-audit.sql)
- Hub page: [Documentation Hub](../README.md)
