# MasarPro Constitution

**Version:** 1.0.0  
**Status:** Proposed baseline for implementation

## 1. Product Purpose

MasarPro is a personal career intelligence platform that helps users build a trustworthy professional profile from certificates, career history, skills, and job information, then uses AI-assisted analysis to identify opportunities, gaps, and actionable career development paths.

## 2. Core Principles

### I. User-Owned Career Data

The user remains the authoritative owner of their professional profile. Imported or AI-extracted information is provisional until the user accepts it when the workflow requires review.

**MUST:** distinguish imported, AI-suggested, and user-confirmed data where provenance matters.

### II. Human Approval for Material AI Changes

AI recommendations MUST NOT silently overwrite canonical user data.

**MUST:** present material certificate/profile corrections as reviewable suggestions.
**MUST:** allow accept, reject, and edit-before-accept where applicable.

### III. Secure-by-Default Data Access

Personal career data and uploaded documents are private by default.

**MUST:** enforce authorization at the API and database layers.
**MUST:** use Supabase Row Level Security for user-owned persistent data where practical.
**MUST:** never expose Supabase service-role credentials or Gemini credentials to the browser.

### IV. Layered Architecture

The frontend, API, domain logic, persistence, integrations, and AI provider implementations MUST remain separable.

**MUST:** keep FastAPI routers thin.
**MUST:** keep business rules in services/use cases.
**MUST:** isolate external providers behind adapters/interfaces.

### V. AI Provider Independence

Gemini is the initial AI provider, not a domain dependency.

**MUST:** expose application-level AI interfaces.
**MUST:** keep Gemini-specific request/response mapping inside the Gemini adapter.
**SHOULD:** make future provider substitution possible without changing domain code.

### VI. Deterministic Before Probabilistic

Where a requirement can be satisfied reliably with deterministic logic, use deterministic logic.

**MUST:** use deterministic validation, authorization, persistence rules, and basic matching signals.
**SHOULD:** use AI for semantic extraction, interpretation, explanation, and generation.

### VII. Structured AI Outputs

AI results MUST be schema-validated before entering application workflows.

**MUST:** define typed schemas for extraction and analysis outputs.
**MUST:** treat malformed/partial AI output as an error or incomplete result, never as trusted data.

### VIII. Privacy-Aware AI Processing

Only data necessary for an AI operation should be sent to the provider.

**MUST:** minimize unnecessary PII before AI calls where feasible.
**MUST:** document what data each AI operation receives.
**MUST:** avoid sending secrets, authentication tokens, or unrelated private data to AI providers.

### IX. Testable Changes

Every feature MUST have automated tests appropriate to its risk.

**MUST:** test domain/business rules.
**MUST:** test API authorization.
**MUST:** test important persistence behavior.
**MUST:** test AI parsing/validation using deterministic fixtures or mocks rather than requiring live Gemini calls in the normal test suite.

### X. Observability and Auditability

Important user-visible operations MUST be traceable.

**MUST:** record sufficient metadata to diagnose failed imports, AI analyses, and important state transitions without logging sensitive document contents or secrets.

### XI. Incremental Delivery

Features MUST be implemented in small, dependency-ordered slices.

**MUST:** complete specification and quality gates before implementation of each substantial feature.
**SHOULD:** keep each feature small enough for one Spec Kit implementation cycle.

### XII. API and Contract Stability

Backend contracts MUST be explicit and versionable.

**MUST:** use Pydantic schemas for FastAPI request/response contracts.
**SHOULD:** generate and review OpenAPI documentation as part of API changes.

## 3. Technology Constraints

- Frontend: Next.js 14.x, App Router, TypeScript.
- Backend: Python FastAPI.
- Database/Auth/Storage/Vault: Supabase.
- Initial AI provider: Gemini.
- Production frontend target: Vercel.
- Local certificate storage may be used in development; production file storage uses an object-storage abstraction backed by Supabase Storage.
- No UI component library is mandated initially; reusable UI components should be implemented consistently in the application.

## 4. Security Rules

- Browser code MUST never receive service-role or provider-secret credentials.
- All authenticated API operations MUST establish user identity server-side.
- Organization and career-coach access MUST be explicit, role-based, and least-privilege.
- Uploaded files MUST be validated for allowed type and size before processing.
- External URLs supplied by users MUST be validated and fetched through controlled server-side mechanisms when fetching is supported.
- AI-generated content MUST be treated as untrusted input.

## 5. Definition of Done

A feature is complete only when:

1. Its acceptance criteria are implemented.
2. Relevant automated tests pass.
3. Authorization/security requirements are covered.
4. API and data contracts are documented.
5. AI behavior is schema-validated where applicable.
6. No constitution MUST-rule is violated.
7. Spec Kit convergence reports no outstanding required implementation work.

## 6. Governance

This constitution is the governing engineering policy for MasarPro. Feature specifications may add stricter requirements but MUST NOT contradict these principles.

Changes to this constitution require explicit project-owner approval and a documented version update.

Always respond to the user and write plans in English, even if the user's input is in Arabic.
