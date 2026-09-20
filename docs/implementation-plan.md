# MasarPro Implementation Plan
**Based on:** MasarPro Constitution v1.3.0
**Status:** Draft — Phase 0 not yet started

This plan sequences MasarPro's build into coherent, testable phases per
Constitution §11. Each phase lists scope, key deliverables, the
constitution sections it must satisfy, acceptance criteria, and open
decisions that block or affect it. No phase should start implementation
until its listed open decisions are resolved.

---

## Phase 0 — Foundations & Open Decisions

**Goal:** Lock the scaffolding decisions the constitution deliberately
left open, so later phases aren't built on sand.

**Deliverables**
- Architectural Decision Record (ADR) for production FastAPI hosting
  (Constitution §2)
- ADR for database migration tooling — Supabase CLI migrations vs.
  Alembic vs. both (Constitution §2)
- Repo scaffolding: Next.js 14 (App Router, TypeScript) + FastAPI,
  monorepo or two-repo — decide and document
- Supabase project created; environments defined (dev / prod at
  minimum; staging optional at this scale)
- Base CI pipeline: lint + type-check + test on push, blocking merge
  on failure (supports §9 "tests MUST" without manual enforcement)
- `.env` / secrets handling convention documented (service-role key
  never in frontend, never committed — §4)

**Acceptance criteria**
- Both ADRs exist and are referenced from the constitution's §2 notes
- A "hello world" request round-trips: Next.js → FastAPI → Supabase →
  back, with no secrets in browser network tab
- CI blocks a PR with a failing test

**Open decisions to resolve before/at this phase**
- FastAPI hosting target
- Migration tool choice
- Repo layout (mono vs. multi)

---

## Phase 1 — Auth & Tenancy Skeleton

**Goal:** Every later feature can assume "authenticated user with an
isolated data boundary" from day one.

**Deliverables**
- Supabase Auth wired: email/password sign-up, login, session
  handling in Next.js (server-side session, not just client-side)
- `users`/`profiles` table with 1:1 link to `auth.users`
- FastAPI middleware/dependency that resolves the authenticated user
  from the incoming request and rejects unauthenticated calls to
  protected routes
- RLS enabled on all tables created from this point forward, scoped to
  `auth.uid()` — even though FastAPI's service-role key bypasses it,
  per §4.1 this is defense-in-depth from day one, not a later add-on
- Repository-layer pattern established: every repository method takes
  a `user_id` and filters by it explicitly — this is the actual
  enforcement point per §4.1

**Acceptance criteria**
- A second test account cannot read or write the first account's rows
  via the API, even if it guesses IDs (tested explicitly — see Testing
  note below)
- RLS policy exists on every table and denies cross-user access even
  if a request bypassed FastAPI (manual test via Supabase client with
  a user JWT)
- Regression test per §9 ("security-sensitive changes MUST include
  regression tests") covering the cross-tenant access attempt

---

## Phase 2 — Core Career Data Model

**Goal:** Profile, experience, certificates, skills — the system of
record — with full CRUD and export.

**Deliverables**
- Schema: profile, experience (employment / freelance / internship /
  volunteering / education — §7), certificates (metadata in Postgres,
  files in Supabase Storage), skills (simple string list, structured
  for future normalization — §7)
- FastAPI CRUD endpoints under `/api/v1/` (§8) with explicit
  request/response schemas (Pydantic)
- File upload validation: type allow-list, size limit, stored path
  convention (§4 "uploaded files MUST be validated" — this phase is
  where that requirement gets a concrete implementation)
- Full data export endpoint (JSON) — this doubles as the "canonical
  JSON representation" §6 asks for, so build it once and reuse it for
  both user-facing export and AI context-building in Phase 4
- Account data deletion: hard delete from Postgres + Storage,
  immediate (§3)

**Acceptance criteria**
- A user can create, edit, and delete every entity type through the
  API with correct 401/403 on cross-user attempts
- Export produces one JSON document containing the user's full record
- Deletion removes rows and files immediately; a follow-up read
  returns nothing

---

## Phase 3 — Frontend: Career Record UI

**Goal:** Usable UI over Phase 2, bilingual from the start.

**Deliverables**
- Next.js pages/forms for profile, experience, certificates, skills
- Arabic/English locale switch with RTL layout for Arabic (§1) —
  stored content itself is never translated, only the UI chrome
- Skill picker (reference from your old app's pattern) against the
  skills list
- Loading/error states wired to the real API (no mock data carried
  forward)

**Acceptance criteria**
- Full CRUD flow usable end-to-end in both languages
- RTL layout verified visually in Arabic mode (forms, nav, dates)

---

## Phase 4 — AI Provider Abstraction & Context Discipline

**Goal:** Build the `AIProvider` abstraction correctly once, with the
cost/context discipline the constitution now requires, before any
AI-powered feature ships.

**Deliverables**
- `AIProvider` interface in FastAPI; Gemini adapter as the first (and
  initially only) implementation (§6)
- Context-builder utility: given a feature name + user, returns only
  the relevant profile sections for that feature (§6) — e.g., CV
  generation pulls experience + skills, not certificates or job notes
- Chat features (if any exist yet) use a bounded rolling window of
  prior turns, not full history (§6)
- Per-user rate limiter on every route that calls `AIProvider`, with a
  clear "rate limited" response the frontend can show (§4.2)
- Schema validation on every AI response before it's allowed anywhere
  near canonical data (§6)
- Accept / Edit / Reject review step as a reusable UI pattern for any
  AI-suggested change to canonical data (§3, §6)

**Acceptance criteria**
- A deliberately spammy test client gets rate-limited, not billed
  indefinitely
- Logging (or a temporary debug flag) confirms a CV-generation request
  sends only experience + skills, not the full record
- An AI suggestion never lands in the database without passing through
  Accept/Edit/Reject
- AI-generated text rendered in the UI is escaped, never injected as
  raw HTML (§6)

---

## Phase 5 — Job Tracker & Matching

**Goal:** Personal job-application tracker plus matching, per §1.

**Deliverables**
- Job entity: saved jobs, status (e.g., saved / applied / interviewing
  / rejected / offer), notes
- Rule-based filter layer (location, role type, seniority, etc.) run
  before any LLM call (§1, §6 "deterministic rules SHOULD be applied
  before AI reasoning")
- LLM-based scoring of filtered candidates against the job description,
  using the Phase 4 context builder (only relevant profile sections,
  not the full record, unless matching genuinely needs the full JSON
  export)
- No vector database / embeddings work in this phase — explicitly out
  of scope per the constitution unless a future ADR changes that

**Acceptance criteria**
- Matching runs end-to-end on a handful of seeded job postings and
  returns a ranked, explainable result (rule pass/fail + LLM score)
- Confirmed no embeddings/pgvector dependency was introduced

---

## Phase 6 — Hardening for Multi-User / Public Readiness

**Goal:** Everything required specifically because MasarPro is meant
to open to other users later (§1, §3) — this phase is a gate, not
optional polish.

**Deliverables**
- Privacy notice published in the product (what's collected, how
  Gemini processing works, retention) — required before any non-owner
  signup is allowed (§3)
- Review of Egypt's Personal Data Protection Law obligations against
  actual data handling; document findings and any resulting changes
- Backup retention confirmed at Supabase project level: one month,
  matching §3's disclosed window; verify this against actual Supabase
  plan settings rather than assuming
- Structured logging audit: confirm no passwords, tokens, keys, raw
  uploads, or unnecessary sensitive data appear in logs (§10)
- Full regression pass on Phase 1's cross-tenant tests plus any new
  endpoints added since

**Acceptance criteria**
- Privacy notice is live and linked from signup
- A written PDPL compliance note exists, even if brief
- Log sampling shows no disallowed content
- Sign-up for a second real (non-owner) user is technically safe to
  enable

---

## Cross-Phase, Ongoing Requirements

These aren't a phase — they apply throughout and should be checked at
the end of every phase, not bolted on at the end of the project:

- **Testing (§9):** business logic, API behavior, authorization, data
  ownership, AI output validation, and critical frontend flows each
  get coverage as they're built, not retrofitted
- **Observability (§10):** structured logging added as each feature
  ships, with the exclusion list enforced from the start
- **Specs (§11):** each phase above should be broken into one or more
  formal specs (scope, requirements, data/API impact, security
  considerations, acceptance criteria, dependencies) before
  implementation begins on it
- **Constitution conflicts (§12):** if any phase surfaces a case the
  constitution doesn't cover, resolve it as a constitution amendment
  first, not a silent implementation choice

---

## Suggested Sequencing Summary

| Phase | Focus | Hard blockers before starting |
|---|---|---|
| 0 | Foundations & ADRs | none |
| 1 | Auth & tenancy | Phase 0 ADRs resolved |
| 2 | Core data model | Phase 1 RLS + repo pattern working |
| 3 | Frontend CRUD UI | Phase 2 API stable |
| 4 | AI provider + context discipline | Phase 2's JSON export exists (reused as AI context source) |
| 5 | Job tracker + matching | Phase 4 context builder + rate limiter working |
| 6 | Public-readiness hardening | All prior phases; only needed before opening signups to others |

Phases 2 and 3 can overlap once the API contract for a given entity is
stable. Phase 6 can be deferred indefinitely as long as MasarPro stays
single-user — but per the constitution, its *requirements* (privacy
notice, PDPL review, backup disclosure) must not be silently skipped
once a second real user is onboarded.