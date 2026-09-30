# INC-024: `FATAL: Tenant or user not found` after switching `DATABASE_URL` to the Supabase pooler

Last verified: 2026-09-30
Pinned to: Supabase shared pooler (Supavisor; session mode on port `5432`, transaction mode on port `6543`) used from a Next.js server runtime through Prisma, Drizzle, `postgres`, or `pg`. Does not apply to the direct connection (`db.<project-ref>.supabase.co`) or the dedicated pooler, which use a different username format.

> Related: [INC-017 connection pool exhaustion](./connection-pool-exhaustion-postmortem.md) is why teams move runtime traffic to the pooler in the first place. That move is when this error appears. Despite the wording, this error is **not** a saturated pool: pool exhaustion surfaces as `Max client connections reached` (see INC-017).

## Symptom

A deploy that switches `DATABASE_URL` from the direct connection to the pooler string (usually to fix INC-017) fails on its first query:

```
Invalid `prisma.cover.findMany()` invocation:
Error in connector: Error querying the database: FATAL: Tenant or user not found
```

(verbatim from [prisma/orm#22824](https://github.com/prisma/orm/issues/22824)). Supabase notes that some drivers word it differently: `FATAL: (ENOTFOUND) tenant/user postgres.<project-ref> not found`.

Every database route returns 500. The same password works over the direct connection, so the natural next step is to reset the password — which changes nothing.

## Impact

A full outage of every data route on the deploy that changes the connection string, often shipped as the fix for another outage. Time is lost because the message reads like a credentials problem. Supabase: "The error reads like a credentials problem, so it often sends people to reset a password that was never wrong. The cause is almost always the host or the username, not the password."

## Root cause — two documented causes

Supabase defines the error as: "the shared pooler couldn't match your host and username to a project."

### Cause 1 — The username has no project ref

Through the shared pooler the username is `postgres.<project-ref>`, not `postgres`. Supabase: "Shared pooler connections use `postgres.[PROJECT-REF]`, not `postgres`. Direct connections and the dedicated pooler use `postgres`." For a custom role it is `<role>.<project-ref>`; "Supplying only the role name, with no project ref, produces this error rather than an authentication failure."

This is what goes wrong when a direct string is edited into a pooler string by changing only host and port:

```env
# WRONG — direct-style username on the shared pooler host
DATABASE_URL="postgresql://postgres:<password>@aws-<N>-<region>.pooler.supabase.com:6543/postgres?pgbouncer=true"
```

The reverse edit breaks too: `postgres.<project-ref>` is only for the shared pooler. The direct host and the dedicated pooler (`db.<project-ref>.supabase.co`) expect plain `postgres`.

### Cause 2 — The pooler host was typed, not copied

Supabase: "Shared pooler hosts look like `aws-1-us-east-2.pooler.supabase.com`. The number is a pooler cluster index, not part of the region name, and a region can have more than one. `aws-0` is not a safe default, and you can't work the host out from your region."

An example that hard-codes `aws-0-<region>` (this repo's INC-017 postmortem did until 2026-09-30) is wrong for any project on another cluster, and the pooler answers with this error.

```env
# WRONG if the project's pooler is not on cluster 0
DATABASE_URL="postgresql://postgres.<project-ref>:<password>@aws-0-us-east-1.pooler.supabase.com:6543/postgres?pgbouncer=true"
```

In prisma/orm#22824 (the repository was formerly `prisma/prisma`), a Prisma contributor replied that the message "does not look like a Prisma related error message, but something that comes directly from the database you are talking to", and asked whether the reporter was using the correct Supabase connection strings. A later commenter got it working by copying the ORM connection strings from the dashboard's Connect dialog.

## Detection (run these now)

### Check 1 — Parse the string you actually deployed

```bash
# Print user and host of the runtime connection string, without the password
node -e 'const u = new URL(process.env.DATABASE_URL); console.log({ user: u.username, host: u.hostname, port: u.port })'
```

- **Healthy (shared pooler):** `user` is `postgres.<project-ref>` (or `<role>.<project-ref>`) and `host` is exactly the one shown in the Connect dialog.
- **Incident:** `user` is `postgres` on a `*.pooler.supabase.com` host (Cause 1), or `host` differs from the dashboard's (Cause 2).
- **Opposite mistake:** `user` is `postgres.<project-ref>` on `db.<project-ref>.supabase.co` (direct or dedicated pooler), which expect plain `postgres`.

### Check 2 — Compare against the dashboard, not your memory

Open the Connect dialog, choose **Session pooler** or **Transaction pooler**, and diff host, port, and username against every environment where the variable is set (Production, Preview, local `.env`, CI secrets).

### Check 3 — Tell it apart from INC-017

| Message | Meaning | Incident |
| --- | --- | --- |
| `Tenant or user not found` | host/username do not match a project | INC-024 (this) |
| `Max client connections reached` | pooler or Postgres refused more connections | [INC-017](./connection-pool-exhaustion-postmortem.md) |
| `remaining connection slots are reserved…` | Postgres `max_connections` reached | [INC-017](./connection-pool-exhaustion-postmortem.md) |
| `password authentication failed for user …` | a different failure | neither — see [Supabase: Password authentication failed](https://supabase.com/docs/guides/troubleshooting/fatal-password-authentication-failed) |

## Fix

Supabase's fix, verbatim:

1. Open the Connect dialog in the Supabase Dashboard.
2. Choose **Session pooler** or **Transaction pooler**.
3. Copy the whole string, and replace only the password placeholder.

For a Next.js app on serverless functions, the runtime string is the transaction pooler (port `6543`), and migrations keep the direct string, as described in [INC-017](./connection-pool-exhaustion-postmortem.md#1-use-the-right-connection-string-for-the-right-job):

```env
# Runtime — transaction pooler, copied from the Connect dialog
DATABASE_URL="postgresql://postgres.<project-ref>:<password>@aws-<N>-<region>.pooler.supabase.com:6543/postgres?pgbouncer=true"

# Migrations — direct connection, plain postgres user
DIRECT_URL="postgresql://postgres:<password>@db.<project-ref>.supabase.co:5432/postgres"
```

Do not reset the database password to fix this error. If you do reset it, every environment holding the old password breaks as well.

## Prevention

- **Boot-time assertion:** at server start, parse `DATABASE_URL` and fail fast (with a clear message, not a Postgres error) when the host ends in `.pooler.supabase.com` and the username has no `.` suffix, or when the host starts with `db.` and the username does.
- **No composed hosts in templates:** `.env.example` should carry `<copy from Supabase Connect dialog>`, never an `aws-0-…` host.
- **Preview-first rollout:** change the connection string on a preview deployment and hit one database route before promoting it.
- **Code-review checklist line:** "Was the connection string copied whole from the Connect dialog, with only the password replaced?"

## References

- Supabase — Troubleshooting: "Tenant or user not found when connecting through the shared pooler" (both causes and the fix): https://supabase.com/docs/guides/troubleshooting/tenant-or-user-not-found
- Supabase — Connect to your database (hosts, ports, and usernames for direct, shared pooler, and dedicated pooler): https://supabase.com/docs/guides/database/connecting-to-postgres
- Supabase — Prisma troubleshooting (`Max client connections reached`, for telling this apart from INC-017): https://supabase.com/docs/guides/database/prisma/prisma-troubleshooting
- prisma/orm#22824 — "Cannot query data from supabase" (verbatim error; contributor comment that it comes from the database): https://github.com/prisma/orm/issues/22824
