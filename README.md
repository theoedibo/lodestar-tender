# Lodestar Tender

**Lodestar Tender is an AI-powered tender automation platform.** It scouts public tender
notices across multiple government and OCDS-standard procurement portals, matches them
against a company's real capabilities and evidence, tells a bid team exactly what they'd
need to qualify, and assembles the paperwork for the ones worth pursuing — closing the
loop from *"a tender was published"* to *"a bid decision-maker has what they need to act."*

This repository holds the **project's documentation and architecture** — product spec,
system design, data model, and the AI/automation design that the platform is built on.
The application's source code is maintained in a private repository.

## What it does

Lodestar Tender runs an end-to-end pipeline:

1. **Scout** — connectors continuously pull tender notices from live public sources
   (UK Find a Tender, UK Contracts Finder, EU TED — all OCDS/eForms-based), tag them by
   sector and geography, and normalize them into one schema.
2. **Qualify** — a deterministic qualification engine checks each tender's stated
   requirements against a company's "Capability Passport" (accreditations, financials,
   past performance, evidence documents), with an AI layer that reads unstructured
   requirement text and turns it into structured, checkable criteria.
3. **Route** — a scouting-profile and autonomy-gate system decides, per organization,
   which matches go straight to a review queue, which get promoted automatically, and
   which are dispatched without a human in the loop, based on match confidence and
   the org's configured autonomy level.
4. **Pack** — for tenders worth pursuing, the system assembles a bid pack: cover sheet,
   matched evidence documents, statutory declarations, and a manifest, zipped and ready
   to hand to a bid team.
5. **Dispatch & track** — bid packs are routed to the relevant business development
   manager (via a signed magic-link "bid room"), and the whole thing is tracked through
   a bid workspace — checklist, deadline reminders, submission status, and win/loss —
   feeding a pipeline analytics view (value, win rate, time saved, source performance).

Everything after "a tender was published" up to "a decision-maker has a ready bid pack
in their hands" is designed to run with as little manual triage as the organization's
own autonomy settings allow.

## Documentation

| Doc | What's in it |
|---|---|
| [`PROJECT_SPEC.md`](./PROJECT_SPEC.md) | Product vision, target users, core workflow, scope and non-goals |
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | System architecture: layers, domain/AI boundaries, multi-tenancy, security |
| [`DATABASE_DESIGN.md`](./DATABASE_DESIGN.md) | Entities, relationships, tenancy strategy, enums |
| [`AI_ARCHITECTURE.md`](./AI_ARCHITECTURE.md) | Where AI is used vs. deterministic logic, structured outputs, prompt-injection protection, confidence handling |
| [`AUTOMATION_ARCHITECTURE.md`](./AUTOMATION_ARCHITECTURE.md) | The autonomous scout → qualify → pack → dispatch pipeline, autonomy gating, and the phased build-out |
| [`SOURCE_CONNECTOR_ARCHITECTURE.md`](./SOURCE_CONNECTOR_ARCHITECTURE.md) | How tender source connectors plug into the ingestion pipeline |
| [`SOURCE_REGISTRY.md`](./SOURCE_REGISTRY.md) | Catalog of tender sources (live, planned, and evaluated-but-unavailable), by country/sector |
| [`DEMO_DATA_SPEC.md`](./DEMO_DATA_SPEC.md) | The synthetic company and demo tenders used to prove the qualification engine end to end |
| [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) | The phased build plan the project has followed |

## How it's built

- **Frontend:** Next.js (App Router) + TypeScript + Tailwind CSS, with a distinct
  gold-and-black brand theme.
- **Backend:** a dedicated `backend/` module (actions, domain logic, AI integration,
  source connectors) kept separate from the Next.js app shell — see `ARCHITECTURE.md`.
- **Data:** PostgreSQL via Supabase (Auth + Storage), Prisma as the schema/query layer,
  row-level security enforcing multi-tenant isolation.
- **AI:** Claude, used specifically for the parts that need language understanding —
  document classification/extraction, unstructured tender-requirement interpretation,
  qualification-gap explanation — behind Zod-validated structured outputs and a
  prompt-injection boundary. Everything that can be deterministic (the qualification
  engine itself, the autonomy gate, scheduling) is deterministic and unit-tested.
- **Source connectors:** a common `TenderSourceConnector` interface with an OCDS-based
  implementation shared by the UK and EU sources, so adding a new procurement portal is
  mostly a mapping exercise, not a new pipeline.
- **Billing:** multi-provider (Stripe, Paystack, Flutterwave) with a trial-by-default
  model and tier gating on autonomy level and source access.
- **Testing:** unit, integration, and end-to-end suites (Vitest + Playwright) — including
  integration tests that run the qualification engine against a live database and
  reproduce the demo-data spec's worked example.

## Status

The platform has moved through the full pipeline described above — sourcing, AI
document/tender intelligence, qualification, bid workspace, autonomous fan-out, bid
pack assembly, BDM dispatch, submission/outcome tracking, billing, and analytics — and
into production hardening (RLS audit, rate limiting, multi-tenant isolation tests).
See `AUTOMATION_ARCHITECTURE.md` and `IMPLEMENTATION_PLAN.md` for the detailed,
phase-by-phase account of what's been built and what's still ahead.
