# Debugging Playbook

Use this playbook when a Next.js + Supabase app works locally but breaks in preview, production, or authenticated flows.

## Supabase Auth Redirect Issues

Common symptoms:

- OAuth returns to the wrong URL.
- Magic links work locally but fail in production.
- User is logged in but redirected back to login.
- Callback route receives a code but no session is created.

Check:

- Supabase Dashboard -> Authentication -> URL Configuration.
- Site URL is the production domain.
- Redirect allowlist includes local, preview, and production callback URLs.
- The callback route exchanges the code for a session.
- Middleware is not redirecting the callback route before the session is set.
- Browser cookies are present after the callback completes.

Useful resources:

- [Supabase Auth Redirect Not Working in Next.js App Router](https://www.iloveblogs.blog/post/supabase-auth-redirect-fix)
- [Supabase + Google OAuth on Next.js 15](https://www.iloveblogs.blog/guides/supabase-google-oauth-nextjs-15-complete-guide)

## RLS Errors

Common symptoms:

- Queries return empty arrays instead of errors.
- Insert works with service role but fails for real users.
- Updates silently affect zero rows.
- Admin screens cannot read data after RLS is enabled.

Check:

- RLS is enabled intentionally.
- Policies exist for the exact operation: select, insert, update, delete.
- `using` and `with check` are both correct where needed.
- `auth.uid()` matches the row ownership column.
- Team access checks the membership table correctly.
- Policy columns are indexed.
- You are not testing with the service role key by accident.

Useful resources:

- [Supabase RLS Policy Not Working: Debug Checklist](https://www.iloveblogs.blog/guides/supabase-rls-policy-not-working-debug-checklist)

## Next.js Hydration Errors

Common symptoms:

- `Text content does not match server-rendered HTML`.
- Components flicker after page load.
- Auth-dependent UI renders differently on server and client.
- Dates, random values, or browser-only values cause mismatches.

Check:

- Avoid `Date.now()`, `Math.random()`, and browser APIs during server render.
- Move browser-only logic into `useEffect`.
- Render stable placeholders until client-only state is ready.
- Do not render different auth UI on server and client without a stable loading state.
- Validate HTML nesting.

Useful resource:

- [Next.js Hydration Mismatch Error](https://www.iloveblogs.blog/post/nextjs-hydration-mismatch-fix)

## API Connection Issues

Common symptoms:

- Supabase requests fail only on Vercel.
- API routes work locally but time out in production.
- Database connections spike under traffic.
- Environment variables are undefined during build or runtime.

Check:

- `NEXT_PUBLIC_SUPABASE_URL` and the public key (`NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, or the legacy `NEXT_PUBLIC_SUPABASE_ANON_KEY`, which Supabase is deprecating by the end of 2026) are set in the deployment platform.
- Server-only secrets do not use the `NEXT_PUBLIC_` prefix.
- Connection pooling is configured for serverless environments.
- Route handlers do not create unnecessary long-lived connections.
- Supabase project is not paused.
- CORS and allowed redirect URLs match the deployed domain.

Useful resources:

- [Supabase Connection Pooling with PgBouncer on Vercel](https://www.iloveblogs.blog/guides/supabase-connection-pooling-vercel)
- [Deploy Next.js 15 to Vercel Without Environment Variable Errors](https://www.iloveblogs.blog/post/nextjs-vercel-env-variables-fix)

## General Debugging Order

1. Reproduce the issue locally with production-like environment variables.
2. Check browser console and network tab.
3. Check Vercel function logs or deployment logs.
4. Check Supabase Auth logs and database logs.
5. Confirm whether the failing request uses anon, authenticated, or service role access.
6. Test the underlying SQL in Supabase SQL editor with an equivalent role.
7. Add one small fix, redeploy, and verify the original failing path.

## External Debugging Resources

Use these official references and tools when debugging production issues.

| Resource | Area | Why It Helps |
|---|---|---|
| [Next.js Hydration Error Guide](https://nextjs.org/docs/messages/react-hydration-error) | Next.js | Official explanation of hydration mismatch causes and fixes. |
| [Next.js Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route) | API debugging | Confirms route handler behavior, caching, and supported HTTP methods. |
| [Next.js Proxy / Middleware](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) | Auth debugging | Reference for request interception, redirects, and auth protection. |
| [Supabase Troubleshooting](https://supabase.com/docs/guides/troubleshooting) | Supabase | Official troubleshooting entry point for platform and project issues. |
| [Supabase Server-Side Auth](https://supabase.com/docs/guides/auth/server-side) | Auth debugging | Cookie/session reference for SSR auth problems. |
| [Supabase Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security) | RLS debugging | Official policy behavior reference for empty data and blocked writes. |
| [Vercel Runtime Logs](https://vercel.com/docs/logs/runtime) | Deployment debugging | Inspect production route handler and server logs. |
| [Sentry for Next.js](https://docs.sentry.io/platforms/javascript/guides/nextjs/) | Error monitoring | Capture server and client exceptions with release context. |
| [Playwright Trace Viewer](https://playwright.dev/docs/trace-viewer) | Browser debugging | Replay failing browser flows and inspect network, DOM, and console output. |
| [Chrome DevTools Network Panel](https://developer.chrome.com/docs/devtools/network/) | Browser debugging | Inspect failed requests, redirects, cookies, and response headers. |
| [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) | Query debugging | Diagnose slow Supabase queries and missing indexes. |
| [Supabase Status](https://status.supabase.com/) | Status page | Check platform incidents before debugging your own code. |
| [Vercel Status](https://www.vercel-status.com/) | Status page | Check hosting incidents and regional outages. |
| [Stripe Status](https://status.stripe.com/) | Status page | Check billing and webhook incidents. |
## Production Cross-Links

- Incident Index row: [INC-004 OAuth callback loops](../../reference/incident-index/README.md#inc-004-oauth-callback-loops-to-login-in-app-router)
- Reference asset: [Server Actions debugging matrix](../../reference/playbooks/server-actions-debugging-matrix.md)
- Hub page: [Documentation Hub](../README.md)
