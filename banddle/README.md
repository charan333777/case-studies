——# Banddle — Technical Case Study

**Live at [banddle.com](https://www.banddle.com)**

## One-line pitch

Banddle is a job-application tracker that turns a scattered job search into a secure, organized pipeline of applications, statuses, job descriptions, and notes.

## Core problem

Job seekers often track applications across spreadsheets, emails, bookmarks, and notes. Banddle gives that process one focused home so users can see where every opportunity stands and what needs attention next.

## Key features

- **Application dashboard:** create, edit, delete, search, filter, sort, and star job applications.
- **Kanban workflow:** group applications by status — Applied, Interview, Offer, Rejected.
- **Application details:** store job descriptions, notes, applied date, status, and priority.
- **Search and filtering:** filter by status, starred applications, company, role, and sort order.
- **Authentication:** Supabase email/password auth, OTP verification, password reset, profile updates.
- **Account lifecycle:** authenticated account deletion through a Supabase Edge Function.
- **Public pages:** landing, about, contact, privacy, and terms pages.

> Dedicated follow-up/reminder scheduling is not a separate data model yet; follow-up context is currently captured in notes and is a strong roadmap item.

## Architecture overview

```
User / Browser
   |
      v
      Vercel-hosted Next.js App Router app
         |-> Supabase Auth  (session / JWT in browser)
            |-> Supabase JS client
               |      \-> public.applications table
                  |             \-> PostgreSQL + Row-Level Security
                     \-> Supabase Edge Function: delete-account
                               |-> validates user JWT
                                         \-> deletes that user's application rows and auth account
                                         ```

                                         The frontend is a client-heavy Next.js app. Auth state is managed by `AuthProvider`, protected pages use `ProtectedRoute`, and job data is fetched/mutated through `jobsApi` using the authenticated Supabase session.

                                         ## Tech stack

                                         | Technology | Role |
                                         | --- | --- |
                                         | Next.js 14 (App Router) | Routing, metadata, public pages, protected dashboard pages. |
                                         | React 18 | Interactive dashboard, modals, forms, cards, kanban board. |
                                         | TypeScript | Typed job application model, auth flows, API helpers. |
                                         | Tailwind CSS | Responsive UI, dark/light/system theme styling, status colors. |
                                         | Supabase Auth | Signup, login, OTP verification, password reset, email/password changes. |
                                         | Supabase PostgreSQL | Stores job application records. |
                                         | Supabase Row-Level Security | Database-level user isolation. |
                                         | Supabase Edge Functions | Privileged account deletion flow. |
                                         | Vercel | Production hosting, HTTPS, apex-to-www redirect, managed caching. |

                                         ## Data model overview

                                         ```
                                         auth.users
                                            id
                                               email
                                                  user_metadata.full_name

                                                  public.applications
                                                     id
                                                        user_id -> auth.users.id
                                                           company
                                                              role
                                                                 status
                                                                    applied_at
                                                                       notes
                                                                          starred
                                                                             job_description
                                                                                updated_at
                                                                                ```

                                                                                **Relationship:** one authenticated user owns many applications. Applications are grouped into kanban columns by status. Notes and job descriptions are stored directly on the application row; there is no separate notes or reminders table in the current codebase.

                                                                                ## Security model

                                                                                Banddle uses Supabase Auth for identity and Supabase RLS for data isolation. The intended RLS approach is: only authenticated users can access rows where `applications.user_id = auth.uid()`.

                                                                                The client also scopes reads/writes by the current user ID, but RLS is the real security boundary. A local `verify_rls.ts` script checks isolation by creating two synthetic users and confirming one user cannot read another user's application row. Privileged service-role access is kept out of the frontend and only used in local scripts or the `delete-account` Edge Function after validating the caller's JWT.

                                                                                ## Technical decisions and trade-offs

                                                                                1. **Supabase-first backend.** Reduced custom backend code by using Supabase Auth, Postgres, REST, and RLS directly from the frontend.
                                                                                2. **RLS instead of only backend checks.** User isolation lives at the database layer, which is safer for a client-connected app.
                                                                                3. **Simple status-based kanban.** Stages are stored as status strings, which keeps the MVP lightweight. The trade-off is that richer custom stage ordering would need more schema later.
                                                                                4. **Client-side job state.** `useJobs` keeps local state, memoized filtering/sorting, and immediate UI updates after mutations. This keeps the dashboard fast without adding server cache complexity.
                                                                                5. **Vercel + Supabase deployment.** Vercel handles hosting, HTTPS, redirects, and frontend delivery while Supabase owns auth/data. This keeps operations simple for an early product.

                                                                                ## Challenges solved / UX highlights

                                                                                - Protected routes show loading states while auth is checked.
                                                                                - Empty states, error banners, and success toasts make CRUD flows clearer.
                                                                                - Dashboard density controls help users scan many applications.
                                                                                - Kanban and list views share the same underlying application model.
                                                                                - Modal details keep users in context instead of forcing a separate page.
                                                                                - Light/dark/system theme support is built into the app shell.
                                                                                - Production is live with Vercel HTTPS and banddle.com redirecting to www.banddle.com.

                                                                                ## Outcomes / metrics

                                                                                - Live production app: [https://www.banddle.com](https://www.banddle.com)
                                                                                - Verified architecture from codebase and live deployment headers.

                                                                                > No trustworthy user/usage metrics were found in the repo or public app, so public case-study metrics should be added only from analytics or production data.

                                                                                ## Roadmap

                                                                                - Dedicated follow-up/reminder dates and notifications.
                                                                                - Separate reminders or activity/history table.
                                                                                - Custom kanban stages and ordering.
                                                                                - Import/export from CSV or spreadsheets.
                                                                                - Calendar/email integrations for interviews and follow-ups.
                                                                                - Optional analytics around application volume and response rates.
                                                                                
