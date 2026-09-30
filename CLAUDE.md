# CLAUDE.md — awesome-nextjs-supabase

## What This Repo Is

A symptom-first index of Next.js + Supabase **production incidents** (reference/incident-index/, postmortems in reference/playbooks/), with a curated list of production-focused resources and a small runnable `examples/` workspace. It is no longer formatted as an awesome list (repositioned 2026-09-30); `examples/` adds minimal broken-vs-fixed code you can run and verify in CI.

---

## Repo Structure

```
README.md                  # Master list — the main entry point
AGENTS.md                  # Drop-in AI-agent guardrails (Cursor, Copilot, Claude Code)
CLAUDE.md                  # This file — curation rules and repo orientation for AI helpers
CONTRIBUTING.md            # Contribution rules
.lychee.toml               # Link-checker config (CI)
.github/
  workflows/
    link-check.yml         # lychee broken-link check (required on main)
    examples-ci.yml        # Vitest + pgTAP for examples/ (not required)
  ISSUE_TEMPLATE/
  PULL_REQUEST_TEMPLATE.md
content/                   # Curated links, organized by topic
  README.md                # Hub index
  learning-paths/          # 7-day roadmap, beginner → advanced
  starter-kits/            # SaaS, auth, Stripe, realtime starters
  open-source-examples/    # Real GitHub projects using Next.js + Supabase
  production-checklists/   # Auth, RLS, deployment, Stripe, perf, security
  snippets/                # Code snippets: auth, middleware, RLS, API helpers
  debugging-playbook/      # Auth, RLS, hydration, API connection fixes
reference/                 # In-repo production assets (not curated links)
  incident-index/          # 21 incidents: symptom → root cause → fix → asset
  playbooks/               # Postmortems and step-by-step recovery guides
  diagrams/                # Mermaid state-machine / flow diagrams
  sql/                     # RLS audit and diagnostic SQL
  templates/               # Stripe webhook idempotency, pgvector benchmark, etc.
  checklists/              # Zero-downtime rollout and release gates
examples/                  # Runnable broken-vs-fixed workspace (Tier 1)
  rls-pgtap/               # pgTAP tests for RLS incidents (INC-002/003/015/018/021)
  stripe-webhook-idempotency/  # Vitest for webhook idempotency (INC-006/007/012/016)
  nextjs15-cache-and-params/   # Vitest for cache + async params (INC-008/011/020)
```

---

## Content Scope

Resources must be specific to one or more of:

- Next.js (App Router, Server Components, middleware, caching, Server Actions)
- Supabase (auth, RLS, Postgres, realtime, pgvector, Edge Functions)
- SaaS architecture (multi-tenancy, billing, webhooks, background jobs)
- Stripe subscriptions
- Production debugging, CI/CD, deployment (Vercel), observability

**Reject:** generic listicles, thin AI filler, duplicates, unrelated topics.

---

## How Resources Are Added

1. Add a bullet to the correct section in `README.md` (or the relevant `content/*/README.md`).
2. Format: `- [Title](url) - Short description of the production problem it solves.`
3. Keep descriptions concrete — what specific problem does this solve?
4. Run link check before opening a PR.

---

## CI

- **Link checker** (`.github/workflows/link-check.yml`): required on `main`. Uses [lychee](https://github.com/lycheeverse/lychee) with config in `.lychee.toml`. Scans `README.md`, `CONTRIBUTING.md`, `AGENTS.md`, `CLAUDE.md`, `content/`, `reference/`, and `.github/`. Accepts HTTP 200/204/206/301/302/429 (429 avoids false positives from rate limits). 2 retries, 20s timeout. The `reference/**`, `AGENTS.md`, and `CLAUDE.md` paths (plus the `your-domain.com` placeholder exclusion) were added so meta-doc/reference-only PRs trigger the check — it is a required gate.
- **examples CI** (`.github/workflows/examples-ci.yml`): runs on `examples/**` changes. Vitest job (stripe + nextjs15 examples, no services) and pgTAP job (postgres:17 container, fixed RLS suite). Not a required gate.

---

## Stats (as of July 2026)

| Type | Count |
|--------|------:|
| Third-party resources (official docs, repos, tools) | ~176 |
| iloveblogs.blog guides + posts | 12 |
| Total unique external links | ~188 |
| Production incidents catalogued | 21 |
| Runnable examples | 3 |

Counts are unique external URLs across `README.md`, `content/**`, `reference/**`, and `examples/**` (badge and placeholder hosts excluded). The `reference/` postmortems cite official docs as evidence, which raises the third-party share to roughly 15:1 over the blog; the curated core (`README.md` + `content/**`) alone is closer to 11:1.

---

## What Claude Should Help With

- **Adding resources:** find the right section in README.md or a content sub-file, format the bullet correctly.
- **Editing descriptions:** keep them practical and specific, no keyword stuffing.
- **Maintaining structure:** the README "Curated Resources" sections mirror `content/` sub-directories; the "Reference Assets" table points into `reference/` and `examples/`.
- **Incidents and postmortems:** new incidents get a row in `reference/incident-index/README.md` (symptom → root cause → fix → asset links) plus an anchor-map entry, and a postmortem under `reference/playbooks/` with verified-evidence doc links.
- **Runnable examples:** `examples/` ships broken-vs-fixed pairs with pgTAP/Vitest verification and a CI job in `examples-ci.yml`. Each example links back to its incident(s); the incident-index row links forward to the example.
- **Link hygiene:** flag duplicates, dead links, or off-topic entries. Run link check before opening a PR.
- **README improvements:** tables, navigation, section ordering.

## What Claude Should NOT Do

- Generate fake resource links or invent blog post URLs.
- Add resources that don't exist or aren't real, vetted external links.
- Refactor the repo structure without explicit instruction — the current layout is intentional.
