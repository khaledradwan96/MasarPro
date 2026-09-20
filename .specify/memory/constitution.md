# MasarPro Constitution
**Version:** 1.3.0
**Last Update:** 2026-09-20
**Status:** Active

## Project

MasarPro is a Personal Career Assistant that manages career profiles,
experience, certificates, skills, jobs, and AI-powered career
intelligence.

---

## 1. Product and Tenancy

MasarPro MUST be built as a multi-user product with per-account data
isolation from day one, even if only one person uses it at first.
MasarPro is being built for solo use initially, with a planned future
public release to other users; the security, privacy, and data-ownership
principles in this constitution MUST be treated as binding from day one
rather than deferred until other users are onboarded.

Authentication MUST be email and password via Supabase Auth.

The product UI MUST support Arabic and English. The Arabic locale MUST
use a right-to-left (RTL) layout. User-entered career content MUST be
stored in the language the user typed; the system MUST NOT require
translation of stored content.

Jobs in MasarPro MUST include:

* A personal job-application tracker (saved jobs, status, notes)
* AI matching and recommendations against the user profile

Job matching MUST use rule-based filtering (e.g., location, role type,
seniority) combined with LLM-based scoring against job descriptions.
A vector-embedding/similarity-search architecture is NOT required
unless a future architectural decision states otherwise.

---

## 2. Technology Stack

The project MUST use:

* **Frontend:** Next.js 14.x, TypeScript, App Router
* **Backend:** Python, FastAPI
* **Database/Auth/Storage/Vault:** Supabase
* **AI:** Gemini through an AI provider abstraction
* **Frontend deployment:** Vercel

Python FastAPI MUST remain the primary backend. Supabase Edge Functions
MUST NOT be used as the main API.

The production FastAPI host is an explicit later decision and MUST be
recorded in an architectural decision when chosen.

Database migration tooling (e.g., Supabase CLI migrations, Alembic) is
likewise an explicit later decision and MUST be recorded in an
architectural decision when chosen.

Next.js MUST serve pages and SSR. FastAPI MUST own AI and other heavy
APIs. The frontend MUST NOT become the place for AI inference or
business-critical heavy processing.

No replacement of the core stack without an explicit architectural
decision.

---

## 3. User Ownership

User career data is user-owned.

The system MUST NOT silently overwrite or delete user data, including
during import or AI merge of jobs, skills, and experiences.

Users MUST be able to export their owned data and to fully delete their
account and owned data (PostgreSQL rows, stored files, and AI
artifacts).

Account deletion MUST remove the user's PostgreSQL rows, stored files,
and AI artifacts from primary storage immediately. Backups containing
deleted user data MAY persist for up to one month under a fixed backup
retention window before permanent purge. This retention window MUST be
disclosed to users as part of the account-deletion flow.

Because MasarPro is planned to open to other users in the future, the
system MUST provide a clear privacy notice, published before any
non-owner account can register, describing what data is collected, how
it is used (including AI processing via Gemini), and how long it is
retained. Applicable data protection obligations (e.g., Egypt's
Personal Data Protection Law) MUST be reviewed and addressed before
public launch.

Invoking an AI feature MAY send the user's full career content to
Gemini. That invocation is consent to send. Canonical writes from AI
still require review as below.

AI-generated changes to canonical user data MUST require user review
and allow:

* Accept
* Edit
* Reject

User data, imported data, and AI-generated data MUST remain
distinguishable where relevant.

---

## 4. Security

Security MUST be enforced server-side.

* Authentication is required for protected resources.
* Authorization MUST verify resource ownership.
* Supabase RLS MUST protect user-owned data where applicable.
* Service-role keys and other secrets MUST never reach the browser.
* Secrets MUST NOT be committed to source control.
* Uploaded files and external URLs MUST be validated.
* Least-privilege access MUST be used.

### 4.1 Authorization Enforcement Model

FastAPI MUST be the sole authenticated gateway to PostgreSQL,
connecting via a service-role key. Because a service-role key bypasses
Supabase RLS, resource-ownership authorization MUST be enforced
explicitly in the FastAPI Repository/Service layer on every query and
mutation — this is the **primary** enforcement mechanism, not RLS.

Supabase RLS MUST still be enabled on all user-owned tables as
defense-in-depth, protecting against a leaked service-role key, direct
Supabase Studio/dashboard access, or any future path where the frontend
queries Supabase directly. If direct frontend-to-Supabase access is
introduced later, RLS becomes the primary control for that path and
MUST be re-verified via an explicit architectural decision.

### 4.2 AI Rate Limiting

AI-invoking endpoints MUST enforce a per-user rate limit to prevent
runaway Gemini cost from retries, bugs, or abuse. Specific thresholds
are an implementation detail left to individual feature specs, but the
presence of a limit is a MUST for every AI-invoking endpoint.

---

## 5. Architecture

The backend MUST maintain clear separation between:

```text
API → Service → Repository → Database
              ↓
        Integrations
```

Routers MUST remain thin.

External providers MUST be isolated behind adapters/interfaces.

The frontend MUST NOT contain business-critical authorization logic.

---

## 6. AI

Gemini MUST be accessed only on the server through an `AIProvider`
abstraction so another provider can be added later. Gemini credentials
and raw model calls MUST NOT run in the browser.

AI output MUST be treated as untrusted data, including when rendered in
the UI: AI-generated text MUST NOT be injected as raw HTML.

Using an AI feature constitutes consent to send needed career content
to Gemini. That consent does NOT approve writing results into canonical
user data.

AI requests to Gemini MUST send only the profile sections relevant to
the specific feature being invoked (e.g., only work-experience entries
for a CV-tailoring request) rather than the full career record by
default. Chat-style AI features MUST cap the conversation history sent
per request to a bounded rolling window rather than replaying full
history indefinitely.

The system SHOULD maintain a canonical JSON representation of a user's
full career record, generated on demand, so that AI features which
genuinely need broader context (e.g., full-profile job matching) can
consume a single structured document rather than assembling context
from multiple queries.

AI results MUST:

1. Be structured where applicable
2. Be schema-validated
3. Respect domain rules
4. Clearly distinguish facts from inference
5. Require user approval for material changes

Deterministic business rules SHOULD be applied before AI reasoning when
possible.

---

## 7. Data

PostgreSQL is the source of truth for persistent application data.

User-owned entities MUST have clear ownership.

Career experiences MUST support:

* Employment
* Freelance
* Internship
* Volunteering
* Education

Stored career text (profile, experience, skills, certificates, job
notes) is user-authored. The architecture MUST NOT force localization
of that stored text.

Skills initially use simple strings but the architecture SHOULD allow
future normalization.

Certificate metadata belongs in PostgreSQL; certificate files belong in
object storage.

---

## 8. API Contracts

FastAPI APIs SHOULD use `/api/v1/`.

Request and response schemas MUST be explicit and validated.

Next.js MAY call FastAPI for AI and heavy operations. Changes to API
contracts MUST consider existing frontend and backend consumers.

---

## 9. Testing

Every significant feature MUST include appropriate tests.

Testing SHOULD cover:

* Business logic
* API behavior
* Authorization
* Data ownership
* AI output validation
* Critical frontend flows

Security-sensitive changes MUST include regression tests.

---

## 10. Observability

Important operations SHOULD use structured logging.

Logs MUST NOT contain:

* Passwords
* Access tokens
* API keys
* Service-role keys
* Unnecessary sensitive user data
* Raw uploaded documents

---

## 11. Specification and Delivery

MasarPro MUST be developed incrementally using multiple coherent
specifications.

Specifications SHOULD cover meaningful product capabilities and MAY
span database, backend, and frontend.

Every significant feature MUST define:

* Scope
* Requirements
* Data/API impact
* Security considerations
* Acceptance criteria
* Dependencies

A feature is complete only when implementation, security, validation,
and relevant tests are complete.

---

## 12. Constitution Authority

This constitution is the highest-level engineering authority for
MasarPro.

Specifications, plans, tasks, and implementation decisions MUST comply
with it.

Any conflict MUST be resolved before implementation. After an
amendment, in-flight specs, plans, and tasks MUST be re-checked for
conflicts before further implementation.

The project owner amends this document through a versioned edit.

Versioning:

* **MAJOR:** Incompatible principle change or removal
* **MINOR:** New principle or materially expanded guidance
* **PATCH:** Clarifications, wording, or non-semantic fixes

---

## Changelog

**1.3.0 (2026-09-20)**
* Clarified authorization model: FastAPI Repository/Service layer is
  the primary enforcement mechanism (service-role key bypasses RLS);
  RLS remains enabled as defense-in-depth (Section 4.1)
* Added AI rate-limiting requirement for all AI-invoking endpoints
  (Section 4.2)
* Added AI context-minimization principles: send only relevant profile
  sections, cap chat history window, and maintain an on-demand JSON
  export of the full career record for features that need broad
  context (Section 6)
* Specified job matching as rule-based filtering + LLM scoring, no
  vector-embedding requirement (Section 1)
* Added account-deletion backup retention window (one month) and
  disclosure requirement (Section 3)
* Added privacy notice / data protection review requirement ahead of
  public launch (Section 3)
* Noted database migration tooling as an explicit later architectural
  decision, alongside the FastAPI hosting decision (Section 2)
* Added AI-output HTML-injection safeguard (Section 6)