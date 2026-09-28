# Lodestar Tender — Project Specification

Status: Phase 0 (architecture & foundation). See `IMPLEMENTATION_PLAN.md` for what comes next.

## 1. Product Vision

Lodestar Tender is an AI-powered tender intelligence and bid-readiness platform. It helps companies discover procurement opportunities and — more importantly — determine whether those opportunities are realistically worth pursuing, given the company's actual capabilities and evidence.

The guiding principle:

> Do not simply show companies more tenders. Help them understand which opportunities are actually worth pursuing.

The platform turns thousands of opportunities into a small number of relevant, actionable ones, with an explainable reason for every qualification decision.

## 2. Target Users

- **Bid managers / business development leads** at companies (initially SME-to-mid-market contractors and service providers) who currently screen tenders manually and want to stop wasting effort on opportunities they can't realistically win.
- **Company administrators** who maintain the company's capability evidence (certifications, financials, project history, personnel, equipment) so it stays current for every future qualification run.
- **Bid teams** who, once an opportunity is confirmed worth pursuing, need a workspace to track preparation tasks and deadlines.

Lodestar Tender is built as a multi-tenant SaaS product from day one: many organizations, each with fully isolated data, sharing one platform and one catalog of discovered tenders.

## 3. Core Problem

Procurement opportunities are numerous, poorly summarized by aggregators, and expensive to evaluate by hand. Two failure modes dominate today:

1. **Wasted effort** — teams spend hours reading a tender notice and its documents only to discover a hard eligibility blocker (e.g. a required local registration the company doesn't hold) on page 40.
2. **Missed opportunity or false confidence** — without a structured comparison against the company's actual, current evidence, teams either skip viable tenders or pursue ones they can't substantiate at submission time.

Lodestar Tender addresses both by building a structured, evidence-backed model of what the company can prove, and comparing it — deterministically wherever possible — against what a specific tender actually requires.

## 4. Core Workflow

```
DISCOVER → UNDERSTAND → QUALIFY → IDENTIFY GAPS → PREPARE → MONITOR
```

1. **Discover** — opportunities are ingested from one or more tender sources (Phase 0 defines the connector abstraction; no live source is connected yet).
2. **Understand** — tender notices and documents are read; requirements, eligibility conditions, and evaluation criteria are extracted into a structured, source-traceable form.
3. **Qualify** — the company's Capability Passport is compared against the tender's requirements, requirement by requirement, producing an explainable outcome for each.
4. **Identify gaps** — missing or uncertain requirements, and any critical eligibility blockers, are surfaced clearly, with the specific gap quantified where possible (e.g. a turnover shortfall).
5. **Prepare** — for tenders worth pursuing, a Bid Workspace tracks the requirements checklist, missing documents, tasks, and deadlines.
6. **Monitor** — (future) sources are watched continuously and the company is notified of new or updated relevant opportunities.

## 5. V1 Scope

V1 is scoped around proving the core value proposition end to end with synthetic data before any real external source is connected. In priority order:

- Multi-tenant authentication and organization management.
- Company Capability Passport: structured, flexible capture of a company's identity, services, products, certifications, experience, personnel, equipment, financials, and legal/regulatory records.
- Document intelligence: upload, store, classify, and extract structured evidence from company documents, with human review for low-confidence extractions.
- Tender intelligence: structured representation of a tender and its requirements, sourced from tender documents (not just an aggregator summary).
- Qualification engine: deterministic, explainable comparison of company evidence against tender requirements, producing per-requirement outcomes, an overall match score, a separate confidence figure, and a recommendation.
- A complete demo environment (one synthetic company, five synthetic tenders) that exercises every qualification outcome the engine supports — see `DEMO_DATA_SPEC.md`.
- Bid Workspace scaffolding (data model and basic tracking) for tenders marked worth pursuing.

## 6. Future Scope (explicitly deferred)

These are part of the long-term vision and inform the architecture (nothing below should require rearchitecting V1), but are **not** built until their designated phase:

- Live connectors to real tender sources (NipeX, UNGM, World Bank, GlobalTenders, national procurement portals, etc.).
- Continuous source monitoring and proactive notifications.
- AI-assisted proposal drafting and requirement-to-response mapping (Bid Intelligence).
- Background job processing at scale (queue workers, scheduled ingestion).
- Advanced analytics across an organization's opportunity pipeline.

## 7. Non-Goals

- **No hard-coded country or jurisdiction requirements, anywhere.** Nigerian document types (CAC, FIRS, PENCOM, ITF, NSITF, NUPRC, NCDMB, etc.) are examples used during design and demo-data construction — never logic. The system must derive requirements from the tender's own text and represent company legal/regulatory evidence as an open-ended list, not a fixed schema. See `ARCHITECTURE.md` §"No Hard-Coded Jurisdiction Logic" and `DATABASE_DESIGN.md`.
- **No fixed company document checklist.** Different companies hold different evidence; the platform does not require any specific document to exist before a company profile is usable.
- **No live scraping in Phase 0 or Phase 1.** The source connector interface is designed for it; no implementation is built until explicitly scheduled (`IMPLEMENTATION_PLAN.md` Phase 8+).
- **No LLM-invented qualification scores.** Match score and confidence are computed by deterministic logic from structured factors; AI assists extraction and interpretation, never the arithmetic. See `AI_ARCHITECTURE.md`.
- **No blind trust of AI output.** Every AI response is schema-validated before it is stored or acted on.
- **No treating uploaded/tender documents as trusted instructions.** They are data to extract from, never prompts to follow. See `AI_ARCHITECTURE.md` §"Prompt Injection Protection".
- **Not a general procurement marketplace or e-tendering system.** Lodestar Tender does not run tenders or handle bid submission/evaluation on behalf of buyers — it is a bidder-side intelligence tool.
