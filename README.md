# Case Studies

Technical case studies for projects I've designed, built, and shipped end-to-end. The source code for these products is private, so these writeups document the problems, architecture, technical decisions, and trade-offs behind each one.

Built by [@charan333777](https://github.com/charan333777).

## Projects

### [CVMindAI](./cvmindai) — recruiter-grade AI CV optimiser

Live at [cvmindai.com](https://cvmindai.com)

Takes a UK job seeker's CV plus a target job description and rewrites it into an ATS-friendly, keyword-aligned CV — with an honest before/after ATS score and concrete improvement tips. Built around a hard constraint: it never fabricates content. Notable engineering includes a deterministic (non-LLM) ATS scorer, a multi-tier model pipeline behind a central registry, JSON-mode + Zod-validated AI calls, and Stripe-webhook-derived entitlements.

**Stack:** Next.js, React, TypeScript, Node.js/Express, OpenAI, Supabase (Auth + Postgres + RLS), Stripe, Upstash Redis, Resend.

[Read the full case study →](./cvmindai)

### [Banddle](./banddle) — job-application tracker

Live at [banddle.com](https://www.banddle.com)

Turns a scattered job search into a secure, organized pipeline of applications, statuses, job descriptions, and notes, with a kanban workflow and full-text search/filtering. Built Supabase-first with Row-Level Security as the real data-isolation boundary, plus a privileged account-deletion Edge Function.

**Stack:** Next.js, React, TypeScript, Tailwind CSS, Supabase (Auth + Postgres + RLS + Edge Functions), Vercel.

[Read the full case study →](./banddle)

## A note on metrics

These case studies intentionally stick to what's verifiable in each codebase. Where reliable usage or performance metrics weren't recorded, they're left out rather than invented.
