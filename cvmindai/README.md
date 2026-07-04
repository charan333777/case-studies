# CVMindAI — Technical Case Study

**Live at [cvmindai.com](https://cvmindai.com)**

## One-line pitch

CVMindAI is a recruiter-grade AI tool that takes a UK job seeker's existing CV plus a target job description and rewrites it into an ATS-friendly, keyword-aligned CV — with an honest before/after ATS score and concrete improvement tips.

## The core problem

Most CVs are first read by Applicant Tracking Systems (ATS), not humans. Good candidates get filtered out because their CV doesn't mirror the language of the job description, lacks standard structure, or buries achievements. Generic "AI CV writers" solve this by inventing impressive-sounding content — which backfires in interviews and is dishonest. CVMindAI takes the opposite stance: it never fabricates. It optimises what the candidate actually provides (uploaded CV or guided form) against a specific UK role, and tells them honestly what to add.

## Key features (in order of importance)

1. **CV parsing (truth layer)** — Extracts an uploaded PDF/DOCX into structured, typed CV data. This is the "source of truth" that every later stage is checked against.
2. **Deterministic ATS scoring** — A pure-TypeScript scorer that grades the CV against the job on five sub-scores (keyword match, structure, completeness, achievements, conciseness). Reproducible, noise-free, and never penalises a genuine improvement.
3. **AI CV rewrite / optimisation** — Recruiter-grade rewrite of summary, bullets and skills to align with the target role, guarded against hallucination.
4. **Before/After score comparison** — Re-parses and re-scores the optimised CV so the improvement shown is real, not mechanically inferred.
5. **Recruiter review + quality/safety audit** — A polish pass and a deferred hallucination/over-optimisation audit.
6. **Keyword-gap flow** — Surfaces missing role keywords; if the user attests they genuinely have a skill, drafts one conservative bullet (no invented specifics) and re-scores for free.
7. **PDF & DOCX export** — In-memory, ATS-readable exports with matched keywords emphasised, capped to two pages.
8. **Role intelligence** — 24 curated UK role profiles seed the pipeline with realistic recruiter expectations and ATS keywords.
9. **Monetisation** — One-time "Job Hunt Pack" via Stripe; free usage gate with an email capture step.

## Architecture overview

Two fully independent, separately deployable services: a **Next.js frontend** (all UI, never touches the AI provider) and a **Node.js + Express backend** (parsing, prompts, AI calls, document generation, payments). The frontend never sees the OpenAI key, prompt format, or model names — keys stay server-side, prompts can be versioned without a frontend release, and rate-limiting/cost-control live in one place.

### Request flow

```
Browser (React)
   |  HTTPS
      v
      Next.js frontend (lib/api.ts)  -- Supabase Auth (JWT) -->  Supabase (Auth + Postgres + RLS)
         |  REST / JSON
            v
            Express API (Node / TS)
               |-> OpenAI (multi-tier pipeline)
                  |-> Upstash Redis (usage gate + rate limits)
                     |-> Resend (email delivery)

                     Stripe Checkout --> webhook --> purchases row "paid" --> export_access entitlement granted
                     ```

                     The optimisation pipeline inside the API (orchestrated in `cv.service.ts`):

                     ```
                     parseCv ----\  (parallel, both required)
                     analyseJob --/
                           |
                                 v  build GenerationContext (parsed CV + job intelligence + role profile)
                                       |-- deterministic ATS score  (user-facing source of truth)
                                             |-- analyseAts               (qualitative recruiter intelligence)
                                                   \-- generateCv               (the rewrite)
                                                         |
                                                               v
                                                               recruiterReview (polish) --> validationIntelligence (deterministic checks)
                                                                     |
                                                                           v
                                                                           evaluateCv (deferred safety audit) --> runHallucinationChecks --> telemetry log
                                                                           ```

                                                                           ## Tech stack and the role of each piece

                                                                           | Technology | Role |
                                                                           | --- | --- |
                                                                           | Next.js 16 (App Router) | Frontend: all pages, routing, SEO route clusters (programmatic `{role} CV UK` landing pages), client-side previews and downloads. |
                                                                           | React 18 + TypeScript + Tailwind CSS | UI components, typed contracts, styling. |
                                                                           | Node.js + Express + TypeScript | REST API: file parsing, prompt construction, OpenAI calls, document generation, payments. Controllers (HTTP shape) to Services (logic) to Utils (pure helpers). |
                                                                           | OpenAI (Chat Completions, JSON mode) | Multi-tier AI pipeline (extraction / deep-reasoning / writing tiers) behind a central model registry. |
                                                                           | Zod | Schema validation of every AI JSON response before it's trusted. |
                                                                           | Supabase | Auth (JWT bearer tokens verified server-side) + Postgres with Row-Level Security for workspaces, purchases, entitlements, usage events. |
                                                                           | Stripe | One-time Job Hunt Pack checkout + webhooks as the source of truth for entitlement grants; native promotion codes. |
                                                                           | Upstash Redis + @upstash/ratelimit | Free-usage gate counters and rate limiting (with in-process fallback). |
                                                                           | pdf-parse + mammoth | Extract raw text from uploaded PDF / DOCX. |
                                                                           | pdfkit + docx | Generate ATS-readable PDF and real Word exports in memory (no headless browser). |
                                                                           | Resend | Transactional email (verification code, purchase delivery). |
                                                                           | Helmet + strict CSP | Security headers; CSP locks connect-src to own origin, API, Supabase, Stripe only. |

                                                                           ## How the AI parts work (high level)

                                                                           **Multi-tier model routing.** A single `MODEL_REGISTRY` maps each pipeline stage to one of three tiers, so models can be swapped/A-B-tested in one file:

                                                                           - **Extraction tier** (`parseCv`, temperature 0) — deterministic, machine-like parsing; same CV in produces the same structured data out.
                                                                           - **Deep-reasoning tier** (`analyseJob`, `analyseAts`, `evaluateCv`) — recruiter judgement: required vs preferred skills, hidden signals, evidence-sensitive matching, hallucination audit.
                                                                           - **Writing tier** (`generateCv`, `recruiterReview`, `suggestEdit`, `draftSkillBullet`) — recruiter-realistic wording.

                                                                           **Every call is JSON-mode and schema-validated.** `runChat()` wraps each call, forces `response_format: json_object`, and emits one structured telemetry line per call (stage, model, tier, duration, token counts) for cost/latency dashboards. Each response is parsed and run through a Zod `safeParse`; a malformed shape throws `AIResponseError` rather than leaking bad data downstream. The generation stage additionally passes through a defensive normaliser so the frontend never sees `undefined`.

                                                                           **CV parsing is a pure extraction prompt** — no rewriting, no inference — producing a typed `ParsedCV`. This becomes the "truth" that the safety audit later diffs the rewrite against, to catch drift, fabricated metrics, or invented skills.

                                                                           **ATS scoring is deliberately NOT an LLM call.** It's pure, reproducible TypeScript (`ats-scoring/`). Given the same CV + job, it always yields the same number — no plus/minus 5-7 point model noise — and adding a genuinely-matched keyword can never lower the score. It grades five sub-scores with a fixed, tunable weighting (keywordMatch 0.55, structure 0.15, completeness 0.12, achievements 0.10, conciseness 0.08); inside keyword match, required skills dominate tools/keywords/title. Matching uses an alias index (so "JS" is treated as "JavaScript") over both the structured skill set and a word-boundary text blob. Weights are isolated as the single source of truth so calibration against market scorers (e.g. Jobscan) is a config change, not a logic change.

                                                                           **Reliability / honesty guardrails:**

                                                                           - Hallucination avoidance is baked into the prompts (never invent facts, metrics, dates, skills) and enforced afterward by deterministic `runHallucinationChecks` + validation-intelligence rules.
                                                                           - The before/after score re-parses the optimised CV independently rather than mechanically mapping — so the improvement shown is honest.
                                                                           - The keyword-gap "draft a bullet" stage caps output tokens and forbids invented specifics — the user's own words are the only permitted source.

                                                                           ## Interesting technical decisions & trade-offs

                                                                           1. **Replaced the LLM ATS scorer with a deterministic one.** The original scorer was an o3 model call. An internal audit found the optimiser only moved the LLM score ~+0.7 on average, while scorer noise was plus/minus 5-7 — the noise dwarfed the signal, and users could see their score drop after a genuine improvement. Rewriting it as pure TypeScript made scores reproducible, monotonic in the user's favour, free to re-run, and calibratable via constants. Trade-off: a static dictionary can miss exotic synonyms (mitigated by an alias index, with an optional LLM gap-fill deferred).
                                                                           2. **Entitlements are server-derived from Stripe webhooks, never from frontend state.** `checkout.session.completed` marks a purchases row `paid` and grants an `export_access` entitlement; protected exports unlock only when the server confirms both workspace ownership and an active entitlement. "Frontend state is never proof of purchase" is an explicit principle. The one-time payment model (no subscriptions/RBAC) keeps the surface small, with clean future seams (new product_keys, export credits, refund-driven revocation).
                                                                           3. **Deferred the expensive quality audit out of the request path.** `evaluateCv` added ~10-20s before the user saw their CV. It now runs after the response is sent; the client gets an `evaluationId` and polls for it. Combined with an async job flow (`/api/optimisation-jobs/*` returning a job id to poll) plus a fast "decision" callback, the user sees progress and results quickly rather than staring at a spinner through the full pipeline.
                                                                           4. **Central model registry + three-tier routing.** Rather than hard-coding model names at call sites, one file maps stage to tier to concrete model + sampling. This makes model swaps, A/B tests, per-plan premium routing, and fallback chains one-line changes — and pairs with per-call token telemetry for cost control. (Note: premium reasoning models reject custom temperature on Chat Completions, so those stages deliberately omit the param instead of sending an unsupported one.)
                                                                           5. **Curated role intelligence over raw JD text.** 24 structured UK role profiles (recruiter expectations + ATS keyword sets) seed both JD synthesis (title-only path) and generation, giving realistic output even when the user pastes no JD. Authoring structured profiles — rather than dumping example JDs — is what makes the keyword matching honest and role-aware.

                                                                           ## Challenges solved / things I'm proud of

                                                                           - **Honesty as an engineering constraint.** The hardest part wasn't generating impressive CVs — it was generating truthful ones. Layered defence: truth-layer parse, prompt-level "never invent" rules, deterministic hallucination + validation checks, and a diff-based safety audit against the original. The product's honesty cap is enforced in code, not just copy.
                                                                           - **Killing score noise.** Turning a noisy, expensive LLM score into a free, deterministic, monotonic one is the change I'm proudest of — it made the before/after feature trustworthy.
                                                                           - **No headless browser for exports.** PDF via pdfkit and DOCX via the docx package, both fully in-memory — cheaper, faster, more reliable to host, and the PDFs are text-based / ATS-readable rather than rasterised. A "never-trim" 2-page fit ladder shrinks formatting first and only trims the lowest-value bullets last.
                                                                           - **Cost-aware pipeline.** JSON-mode everywhere, per-call token telemetry, in-process JD synthesis cache, tight token caps on micro-generations (single-bullet drafts capped at 200 tokens), and the ATS scorer costing zero per run.
                                                                           - **Solo build, idea to ship to revenue.** Designed, built, and launched end-to-end by one person.

                                                                           ## Outcomes / metrics

                                                                           - Live and monetised in production with real paying customers.
                                                                           - Deterministic, reproducible ATS scoring — replaced a scorer whose noise (plus/minus 5-7) exceeded the signal.
                                                                           - 24 curated UK role profiles driving role-aware optimisation (up from an initial 5).
                                                                           - Programmatic SEO cluster of `{role} CV UK` landing pages (6 shipped as a test batch, expandable to all 24).

                                                                           > Note: kept to what's verifiable in the codebase and internal notes — no invented user counts, uptime figures, or average score-lift numbers, since those aren't reliably recorded in the repo.

                                                                           ## Roadmap / what's next

                                                                           - LLM synonym gap-fill as an opt-in layer over the deterministic scorer for keywords the static dictionary misses.
                                                                           - Expand role profiles to full coverage and grow the SEO landing-page cluster if Search Console shows traction.
                                                                           - Form-intake ("create from scratch") path — currently the one gap; the from-scratch generate path is a stub by design (product principle: optimise, don't invent).
                                                                           - Premium-tier reasoning routing for Job Hunt Pack users (the registry already has the seam).
                                                                           - Additional one-time bundles / export credits via new entitlement product_keys.
                                                                           
