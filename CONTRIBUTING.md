# Contributing

This repository is a symptom-first index of Next.js + Supabase production incidents, with evidence, reference assets and runnable examples. It is maintained by one person, with an AI agent helping draft changes. Every merge is reviewed by the maintainer, and links are checked automatically in CI.

## Three ways to contribute

1. **Report an incident.** Use the [incident form](https://github.com/mahdibrr/awesome-nextjs-supabase/issues/new?template=incident_report.yml). You do not need to write the postmortem. A precise symptom, versions and evidence are enough to start.
2. **Add evidence or a fix to an existing incident.** Open a pull request against its row in [`reference/incident-index/README.md`](reference/incident-index/README.md) or its postmortem in [`reference/playbooks/`](reference/playbooks/). Useful additions include a second reproduction, an upstream issue, a version where the behaviour changed, a detection query, or a test in [`examples/`](examples/README.md). This is usually the easiest place to start.
3. **Suggest a resource.** Use the [resource form](https://github.com/mahdibrr/awesome-nextjs-supabase/issues/new?template=resource_request.yml), or open a pull request with the entry.

To fix something wrong, use the [correction form](https://github.com/mahdibrr/awesome-nextjs-supabase/issues/new?template=correction.yml) or send a pull request. For questions, use [Discussions](https://github.com/mahdibrr/awesome-nextjs-supabase/discussions).

## What makes an incident acceptable

- **It happened in production or in a production-like deployment**, not only in local development.
- **It is reproducible or evidenced.** It needs one of: a reproduction (steps or a failing test), logs or traces from the incident, or an upstream issue or official doc that describes the same failure.
- **The root cause and fix cite primary sources.** That means official documentation, the upstream issue tracker, changelogs or release notes. A blog post can add context but cannot be the only source.
- **Versions are stated.** Next.js, `@supabase/supabase-js`, `@supabase/ssr` and anything else involved. Many incidents only exist in certain versions.
- **It is not already covered.** If it is close to an existing INC-0xx entry, add to that entry instead.

An incident with an unknown root cause is still welcome as a report. It is added to the index once the cause is confirmed.

Do not attribute a fix to a release unless a source names that release. "Fixed in PR #123" can be checked. "Shipped in version X" is a separate claim and needs its own source.

## What makes a resource acceptable

A resource is listed when it helps someone diagnose, fix or prevent a production problem on this stack. Please avoid:

- generic listicles or thin AI-generated content;
- duplicates, unless they solve a different production problem;
- tutorial-only material with no production applicability;
- abandoned repositories, unless the entry carries a clear warning;
- anything unrelated to Next.js, Supabase, PostgreSQL, SaaS or production engineering;
- keyword-stuffed or promotional descriptions;
- broken links, redirects to spam, or pages behind a login or payment wall.

Prefer official docs and maintained repositories over personal articles when both cover the same point.

### Paid products and affiliation

- **Paid and freemium products** go in **Tools and Services**, not in the Curated Resources sections that list docs and open-source material. Their entry carries a pricing marker: `(freemium)` for a free tier with paid plans, `(paid)` for paid only.
- **Disclose any affiliation** in the issue or pull request: author, employee, founder, or paid or sponsored in any way. Disclosure does not count against a suggestion. Undisclosed affiliation is a reason to close it.
- **Affiliate and referral links are not accepted.**
- These rules apply to the maintainer's own content as well.

## Formats

Curated Resources bullet:

```md
- [Resource Name](https://example.com) - Short, neutral description of the production problem it helps with.
```

Tools and Services row:

```md
| [Tool Name](https://example.com) (freemium) | What it solves, in one line. |
```

Incident index row: follow the existing columns in [`reference/incident-index/README.md`](reference/incident-index/README.md). A postmortem in `reference/playbooks/` uses the sections of the existing ones: symptom, impact, root cause, detection, fix, prevention, references and a "Last verified" date.

Use `Next.js`, `Supabase`, `PostgreSQL`, `Vercel`, `RLS`, `Auth` and `App Router` consistently. Small, focused pull requests are the easiest to review. The link checker runs on every pull request and must pass.

## Response times

- **First reply to a new issue or pull request: within 7 days.** It may be a question, a request for evidence, or a decision.
- **A decision (merge, change requested or close with a reason) within 14 days** of the pull request being ready.
- If you have heard nothing after 7 days, comment on the thread to bring it back up. That is welcome, not rude.

These are targets set by one person working on this in spare time, not a service agreement.

## Credit

- **Release notes.** Each release lists the contributors whose work it includes, by GitHub handle.
- **Incident files.** A contributed incident, or evidence added to one, gets a credit line in the incident entry or postmortem, e.g. `Reported by @handle` or `Evidence: @handle`.
- **Commits.** Pull requests are merged as they are where possible. If the maintainer has to re-land your change in a new pull request, your commit keeps a `Co-authored-by` trailer so it still counts towards your GitHub contributions.

You can ask for no credit in the form or the pull request.
