# Learning Paths

Use this roadmap to move from a basic Next.js + Supabase app to production-ready SaaS patterns.

## 7-Day Next.js + Supabase Plan

| Day | Focus                        | Outcome                                                                                              |
| --- | ---------------------------- | ---------------------------------------------------------------------------------------------------- |
| 1   | App Router basics            | Understand layouts, pages, route handlers, Server Components, and Client Components.                 |
| 2   | Supabase project setup       | Create a Supabase project, connect environment variables, and initialize the browser/server clients. |
| 3   | Authentication               | Add sign up, sign in, sign out, session refresh, and protected routes.                               |
| 4   | Database and RLS             | Create tables, enable RLS, write policies, and test authenticated reads/writes.                      |
| 5   | CRUD and file uploads        | Build real forms, mutations, storage uploads, and server-side validation.                            |
| 6   | Realtime and background work | Add realtime subscriptions, webhooks, scheduled work, or async job patterns.                         |
| 7   | Production launch            | Audit env vars, deployment, security, performance, backups, observability, and billing readiness.    |

## Beginner Roadmap

Start here if you are still learning how the stack fits together.

- Learn Next.js App Router routing, layouts, and loading states.
- Learn when to use Server Components versus Client Components.
- Create a Supabase project and understand the project URL, the publishable and secret keys, and the legacy anon and service role keys they replace ([being deprecated by the end of 2026](https://supabase.com/docs/guides/api/api-keys)).
- Build a small CRUD app with one table.
- Enable Row Level Security before adding real user data.
- Deploy a simple version to Vercel.

Recommended resources:

- [Next.js App Router](https://nextjs.org/docs/app)
- [Use Supabase with Next.js](https://supabase.com/docs/guides/getting-started/quickstarts/nextjs)
- [Vercel with-supabase Example](https://github.com/vercel/next.js/tree/canary/examples/with-supabase)
- [Build a Full-Stack App with Next.js and Supabase](https://blog.logrocket.com/build-full-stack-app-next-js-supabase/)

## Intermediate Roadmap

Move here once you can build a basic authenticated app.

- Add SSR-safe session handling.
- Protect pages with middleware and server-side checks.
- Write RLS policies for user-owned data and team-owned data.
- Add Zod validation around mutations.
- Add file uploads with Supabase Storage.
- Add realtime updates for the parts of the app that benefit from live sync.
- Add basic tests for auth, RLS, and important database flows.

Recommended resources:

- [Supabase Server-Side Auth](https://supabase.com/docs/guides/auth/server-side)
- [Supabase Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Next.js Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route)
- [Supabase Storage](https://supabase.com/docs/guides/storage)
- [Supabase Auth and Middleware Session Management](https://www.iloveblogs.blog/guides/supabase-auth-complete-session-middleware-guide)

## Advanced Roadmap

Use this path for SaaS, scale, security, and production engineering.

- Design multi-tenant data models with strict tenant isolation.
- Use connection pooling for serverless deployments.
- Add Stripe subscriptions and webhook idempotency.
- Add structured logging, monitoring, and error reporting.
- Add CI/CD with preview deployments and migration checks.
- Add caching, invalidation, and performance budgets.
- Add AI search with embeddings and pgvector when it solves a real product problem.

Recommended resources:

- [Supabase Going Into Production](https://supabase.com/docs/guides/deployment/going-into-prod)
- [Vercel Production Checklist](https://vercel.com/docs/production-checklist)
- [Stripe Webhooks](https://docs.stripe.com/webhooks)
- [Sentry for Next.js](https://docs.sentry.io/platforms/javascript/guides/nextjs/)
- [Playwright](https://playwright.dev/docs/intro)
- [Multi-Tenant SaaS Architecture with Next.js and Supabase](https://www.iloveblogs.blog/guides/nextjs-supabase-multi-tenant-saas-architecture)
- [Supabase Connection Pooling with PgBouncer on Vercel](https://www.iloveblogs.blog/guides/supabase-connection-pooling-vercel)
- [Production RAG with Supabase pgvector and Next.js](https://www.iloveblogs.blog/guides/production-rag-supabase-pgvector-nextjs)

## External Production Resources

Use these external references to learn from primary sources and production-grade examples.

| Resource                                                                                               | Type               | Why It Helps                                                                                               |
| ------------------------------------------------------------------------------------------------------ | ------------------ | ---------------------------------------------------------------------------------------------------------- |
| [Next.js Documentation](https://nextjs.org/docs)                                                       | Official docs      | Core framework reference for App Router, rendering, caching, and deployment behavior.                      |
| [Next.js Mutating Data](https://nextjs.org/docs/app/getting-started/mutating-data)                     | Official docs      | Current guidance for forms, mutations, and Server Actions.                                                 |
| [Supabase Documentation](https://supabase.com/docs)                                                    | Official docs      | Primary reference for Supabase Auth, PostgreSQL, Storage, Realtime, Edge Functions, and local development. |
| [Supabase CLI Getting Started](https://supabase.com/docs/guides/local-development/cli/getting-started) | Official tool docs | Learn local Supabase development, migrations, seed data, and environment parity.                           |
| [Vercel Next.js Deployment](https://vercel.com/docs/frameworks/nextjs)                                 | Official docs      | Production deployment reference for Next.js on Vercel.                                                     |
| [Stripe Billing](https://docs.stripe.com/billing)                                                      | Official docs      | Billing model reference for subscriptions, invoices, customers, and pricing.                               |
| [GitHub Actions](https://docs.github.com/en/actions)                                                   | Official docs      | CI/CD foundation for testing, deployments, and scheduled automation.                                       |
| [supabase/supabase](https://github.com/supabase/supabase)                                              | GitHub repo        | Main Supabase open-source repository and issue history for platform behavior.                              |
| [Drizzle with Supabase](https://supabase.com/docs/guides/database/drizzle)                             | Official docs      | Type-safe SQL workflow for teams that prefer schema-first database access.                                 |
| [Prisma with Next.js](https://www.prisma.io/docs/guides/nextjs)                                        | Official docs      | Alternative ORM workflow for teams using Prisma Client and Prisma Migrate.                                 |
| [shadcn/ui](https://ui.shadcn.com/docs)                                                                | UI toolkit         | Practical component baseline for dashboards, forms, and SaaS settings screens.                             |
## Production Cross-Links

- Incident Index row: [INC-005 Session disappears after refresh](../../reference/incident-index/README.md#inc-005-session-disappears-after-refresh-in-ssr)
- Reference asset: [Zero-downtime rollout checklist](../../reference/checklists/zero-downtime-rollout-checklist.md)
- Hub page: [Documentation Hub](../README.md)
