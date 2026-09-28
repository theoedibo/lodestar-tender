# Lodestar Tender — Architecture

See also: `DATABASE_DESIGN.md`, `AI_ARCHITECTURE.md`, `SOURCE_CONNECTOR_ARCHITECTURE.md`, `AUTOMATION_ARCHITECTURE.md` (the autonomous scout → qualify → pack → dispatch layer, Phases 8–16), `SOURCE_REGISTRY.md`.

## 1. System Overview

```
                         ┌───────────────────────────────────────────┐
                         │              Next.js Application            │
                         │  (UI + server-side route/action handlers)   │
                         └───────────────────────────────────────────┘
                                   │                       │
                    ┌──────────────┘                       └──────────────┐
                    ▼                                                     ▼
        ┌─────────────────────┐                              ┌─────────────────────┐
        │   Domain Services     │                              │    AI Modules          │
        │  (deterministic)      │◄────structured, validated───►│  (Claude API + Zod)   │
        │  company / tenders /  │        requests/responses     │  document / tender /  │
        │  qualification / bids │                              │  qualification / bid  │
        └─────────────────────┘                              └─────────────────────┘
                    │
                    ▼
        ┌─────────────────────┐        ┌─────────────────────┐
        │  Source Connectors    │  ──►   │   Ingestion Pipeline   │
        │  (interface only in   │        │  normalize/dedupe/     │
        │   Phase 0)             │        │  store                │
        └─────────────────────┘        └─────────────────────┘
                    │                               │
                    └───────────────┬───────────────┘
                                     ▼
                        ┌─────────────────────────┐
                        │   PostgreSQL (Supabase)   │
                        │  + Row Level Security      │
                        │  + Supabase Storage         │
                        └─────────────────────────┘
```

## 2. Application Layers

| Layer | Location | Responsibility |
|---|---|---|
| UI / routes | `frontend/app` | Next.js App Router pages, layouts, server actions/route handlers. Thin — delegates to domain services. |
| Domain logic | `backend/domain/{company,tenders,qualification,bids}` | Deterministic business rules and types: capability passport shape, requirement matching, scoring, workspace lifecycle. No AI calls, no framework/database code. |
| Documents | `backend/domain/documents`, future `src/documents/*` | Document/evidence data shapes and the (future) processing pipeline orchestration. |
| AI | `backend/ai/{document-intelligence,tender-intelligence,qualification-intelligence,bid-intelligence,shared}` | All Claude API calls. Structured input/output only, validated against Zod schemas. No direct database access — returns data to the calling domain service. |
| Source connectors | `backend/sources` | `TenderSourceConnector` interface and (future) per-source implementations. Produces normalized `Tender` objects only. |
| Data access | `backend/lib/prisma.ts`, `backend/prisma/schema.prisma` | Prisma client and schema — the single source of truth for persisted shape. |
| Auth/tenancy | `backend/lib/supabase/*` | Supabase Auth session handling (browser + server clients) and the admin client used only by trusted server-side jobs. |
| Tests | `backend/tests/{unit,integration,e2e}` | See `ARCHITECTURE.md` §"Testing Strategy" below. |

**Dependency direction:** UI → domain → (AI | data access | source connectors). Domain modules never import from `frontend/app`. AI modules never import Prisma directly — they receive plain data and return plain, validated data; the calling domain service persists it. This keeps the qualification engine testable without a live database or a live LLM call.

## 3. Domain Boundaries

Four domain modules, matching the four AI modules but independently testable without AI:

- **Company** (`backend/domain/company`) — the Capability Passport: identity, services, products, certifications, experience, personnel, equipment, financials, legal records. Owns no document bytes — only structured facts plus `EvidenceRef`s pointing at documents.
- **Documents** (`backend/domain/documents`) — the evidence lifecycle: upload → extraction → evidence link → review. Documents are organization-owned; `TenderDocument`s (tender notices/specs) are a separate, platform-shared concept — see `DATABASE_DESIGN.md` §"Tenancy Strategy".
- **Tenders** (`backend/domain/tenders`) — the normalized tender shape every source connector must produce, plus extracted requirements and evaluation criteria.
- **Qualification** (`backend/domain/qualification`) — compares a Capability Passport against a Tender's requirements and produces `QualificationResult`s, `QualificationGap`s, and an overall `QualificationAssessment`.
- **Bids** (`backend/domain/bids`) — the workspace created for a tender an organization decides to pursue (design-only in Phase 0; see `IMPLEMENTATION_PLAN.md` Phase 7).

Each domain module owns its own TypeScript types (`types.ts`) independent of the Prisma schema's exact shape, so the persistence layer can evolve without changing what the qualification engine reasons about.

## 4. AI Boundaries

AI is scoped to **extraction and interpretation**, never to **arithmetic or final decisions**. See `AI_ARCHITECTURE.md` for the full breakdown. In short:

- AI classifies documents, extracts structured fields, extracts tender requirements from unstructured text, and helps interpret ambiguous language.
- Deterministic code performs every comparison that has an objective answer (turnover ≥ threshold, years ≥ threshold, certification present/absent/expired, jurisdiction match) and computes the match score and confidence from structured factors.
- Every AI response is validated against a Zod schema before it is trusted; invalid output is rejected, logged, and retried or escalated to human review — never silently accepted or silently dropped.
- Documents (uploaded or tender-sourced) are untrusted content. Prompts always separate trusted system/task instructions from untrusted document text (`backend/ai/shared/prompt-boundaries.ts`).

## 5. Source Connector Architecture (summary)

See `SOURCE_CONNECTOR_ARCHITECTURE.md` for the full design. The qualification engine, company domain, database core, and UI depend only on the normalized `Tender` type (`backend/domain/tenders/types.ts`) and never on which source produced it. Adding a source means implementing `TenderSourceConnector` (`backend/sources/types.ts`) — no other layer changes.

## 6. Document Processing Architecture (summary)

```
UPLOAD → STORE ORIGINAL → EXTRACT CONTENT → CLASSIFY DOCUMENT →
EXTRACT STRUCTURED DATA → VALIDATE → STORE EVIDENCE → HUMAN REVIEW IF NECESSARY
```

The original file is stored in Supabase Storage and is never overwritten or deleted by later processing steps. Every derived fact (`DocumentExtraction`) carries a confidence score and, where applicable, a source page. `DocumentEvidence` links an extraction to the specific company-capability record it supports, so any qualification result can be traced back to the original file. Implemented starting `IMPLEMENTATION_PLAN.md` Phase 3.

## 7. Qualification Engine (summary)

Requirement-level outcomes: `SATISFIED`, `NOT_SATISFIED`, `UNKNOWN`, `NEEDS_REVIEW`, `NOT_APPLICABLE`, `ELIGIBILITY_BLOCKER`, `PARTNERSHIP_REQUIRED`.

The engine is deterministic wherever a requirement has an objective, structured comparison (financial thresholds, years of experience, certification presence/validity, jurisdiction/registration match). AI is used only to (a) extract the requirement's structured meaning from free text and (b) locate and interpret relevant company evidence when the match is not a simple field comparison. A mandatory `ELIGIBILITY_BLOCKER` overrides an otherwise-high match score — see `AI_ARCHITECTURE.md` §"Deterministic vs. AI Responsibilities" and `DATABASE_DESIGN.md`'s `qualification_*` tables. Implemented starting `IMPLEMENTATION_PLAN.md` Phase 5.

## 8. Multi-Tenancy

- Every organization-owned table carries a denormalized `organization_id` column (see `backend/prisma/schema.prisma`), even where it is also reachable through a parent join — this keeps Row Level Security policies single-table and index-friendly.
- Tenders, tender sources, and tender documents are **platform-shared reference data**, not organization-owned: many organizations may legitimately view the same discovered tender. Organization-scoped data begins at the qualification assessment (an organization's *interpretation* of a tender), not the tender record itself.
- Row Level Security is enforced in Postgres (via Supabase) as the backstop for any direct Supabase client access (a browser using the anon key; future PostgREST/Realtime usage). See `DATABASE_DESIGN.md` §"Tenancy Strategy" and `supabase/migrations/*_phase1_rls.sql` for policy shape.
- **Prisma-based server code does not go through RLS.** The role Prisma connects as (via `DATABASE_URL`) owns these tables, and table owners bypass RLS by default in Postgres regardless of policy — this was confirmed while implementing Phase 1. RLS therefore protects direct Supabase client access; it is not a backstop for a bug in Prisma-based route/action code. Every Prisma query that touches organization-scoped data MUST derive its own `organization_id` from a verified session (`backend/lib/organization-context.ts`) and MUST NOT trust a client-supplied id. Treat this as the authorization boundary that actually matters for server-side code.
- The Supabase service-role key (which also bypasses RLS) is used only by trusted server-side code with its own explicit authorization checks (e.g. source ingestion jobs) — never in any client-reachable code path. See `backend/lib/supabase/server.ts`.
- **Platform-admin exception.** The operator console at `/admin` reads across every tenant. This is deliberate and isolated: a separate route tree, gated on an env allow-list (`PLATFORM_ADMIN_EMAILS`, never self-service) via `requirePlatformAdmin()`, using the Prisma owner connection (which already bypasses RLS). It does not weaken the client-facing rule above — no client code path can read another tenant's data. Every admin mutation is written to `admin_actions`. See `backend/lib/platform-admin.ts`.

## 9. Security

- Secrets (Supabase service-role key, Claude API key, database URL) are server-only environment variables, validated at startup via `backend/lib/env.ts`, and never referenced from client components. `.env.example` documents every variable name with no values.
- Uploaded files and tender documents are treated as untrusted input: validated by type/size before storage, never executed or parsed in a way that trusts embedded instructions (see AI boundaries above).
- Authorization is checked at the organization level on every data access — RLS is the backstop, but route/action handlers also verify the acting user's `OrganizationMember` role before mutating state.
- No secrets are committed; `.gitignore` excludes `.env*` (except `.env.example`), build output, and logs.

## 10. Future Scaling

The architecture is deliberately layered so scaling steps don't require rewrites:

- **More tender sources**: add a connector implementing `TenderSourceConnector`; no change to qualification, company, database core, or UI.
- **Background processing**: ingestion and document processing are already modeled as discrete steps (`SourceRun`, `AgentRun`); they can move from request-time execution to a queue (Redis/BullMQ on Railway/Render, per `PROJECT_SPEC.md`'s stated stack) without changing their input/output contracts.
- **Read scaling**: platform-shared tender data (high read, low per-tenant variance) and organization-owned data (tenant-isolated) are already separated, which allows different caching/indexing strategies for each without a schema change.
- **Additional AI modules**: Bid Intelligence (Phase 7+) slots into the existing `backend/ai/*` pattern without touching the other three modules.
