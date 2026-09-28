# Lodestar Tender — Demo Data Specification

Status: specification only. No fixture files exist yet — they are created in Phase 6 (`IMPLEMENTATION_PLAN.md`), once the document, tender, and qualification models this spec depends on are implemented (Phases 3–5). This document is what those fixtures must satisfy, and doubles as the acceptance test for the qualification engine: if the engine produces every outcome below from this data, the primary success criterion (`PROJECT_SPEC.md` §28 of the source brief) is met.

All data described here is **entirely fictional**, created for testing. No real company or tender information is used. Every fixture file, once created, must be clearly marked as demo data (e.g. a `DEMO DATA — FICTIONAL` header/watermark where the format allows it, and placement under `backend/demo-data/` only).

## 1. Demo Company — Atlas Energy Services Ltd

| Field | Value |
|---|---|
| Legal name | Atlas Energy Services Ltd |
| Country of registration | Nigeria |
| Operating countries | Nigeria, Ghana |
| Industry | Energy / Industrial Services |
| Years in operation | 9 |
| Annual turnover | $7.2M USD (most recent financial year) |

Capabilities represented in its Capability Passport:

- **Certifications:** ISO 9001 (Quality Management), ISO 45001 (Occupational Health & Safety) — both `ACTIVE`.
- **Experience:** pipeline maintenance projects (Nigeria), industrial maintenance projects (Nigeria, Ghana) — at least one project with an African country outside Nigeria, to support Demo Tender 2's "similar project experience" check.
- **Financial:** $7.2M annual turnover, audited statements available, insurance available.
- **Legal records:** Nigerian certificate of incorporation; Nigerian tax clearance. Deliberately **no** Kenyan registration and **no** registered local partner in any other jurisdiction — these absences are load-bearing for Demo Tenders 3 and 4.
- **Personnel / equipment:** present but minimal — enough to exist as evidence, not a focus of the demo scenarios.

### Demo Company Documents

Each is a fictional fixture file, referenced by the company's evidence and clearly marked as demo data:

1. Certificate of incorporation (Nigeria)
2. Tax clearance documentation (Nigeria)
3. ISO 9001 certificate
4. ISO 45001 certificate
5. Insurance certificate
6. Audited financial statement (most recent year, showing $7.2M turnover)
7. Company profile document (narrative: identity, history, capabilities overview)
8. Personnel CV (at least one, supporting the personnel capability domain)
9. Previous project evidence (supporting the pipeline/industrial maintenance experience records)
10. Equipment list
11. HSE policy document

The gaps are intentional: no Kenyan registration document, no local-partner agreement, no evidence of turnover anywhere near $20M. A demo built with a company that satisfies every possible requirement would not test the engine's negative paths, which are as important as its positive ones.

## 2. Demo Tenders

Five tenders, each targeting a distinct qualification outcome so the full `RequirementOutcome`/`Recommendation` space (`DATABASE_DESIGN.md` §8) is exercised at least once.

### Demo Tender 1 — Nigerian Opportunity

- Nigeria-specific requirements matching evidence Atlas Energy actually holds (Nigerian incorporation, Nigerian tax clearance, ISO 9001/45001, relevant experience).
- **Expected:** every requirement resolves `SATISFIED`; `recommendation = PURSUE`; high match score and high confidence (extractions drawn from clean, complete demo documents).

### Demo Tender 2 — International Opportunity

- Foreign bidders explicitly allowed (`foreign_bidder_policy = OPEN_TO_FOREIGN`).
- Requirements: minimum 5 years' experience, minimum $5M annual turnover, ISO 9001, similar project experience, bid security.
- **Expected:** 9 years ≥ 5 (`SATISFIED`), $7.2M ≥ $5M (`SATISFIED`), ISO 9001 present and active (`SATISFIED`), similar project experience present (`SATISFIED`, drawing on the African project experience evidence), bid security requirement acknowledged (`NEEDS_REVIEW` or `SATISFIED` depending on whether bid-security evidence is modeled as a capability or a bid-preparation task — see note below). Overall: high match score, `recommendation = PURSUE`.

### Demo Tender 3 — Kenya Registration Requirement

- Requires the bidder to be registered in Kenya.
- Atlas Energy is Nigerian-registered only; no Kenyan registration evidence exists.
- **Expected:** the Kenyan-registration requirement resolves `ELIGIBILITY_BLOCKER`. Per the eligibility-blocker override rule (`AI_ARCHITECTURE.md` §5), `recommendation` cannot be `PURSUE` regardless of how well other requirements score — expected `recommendation = DO_NOT_PURSUE`. The result's `explanation` must state plainly: tender requires Kenyan registration; company holds Nigerian registration only; no evidence of Kenyan registration exists.

### Demo Tender 4 — Local Partner Requirement

- Foreign companies may participate only through an eligible local partner (`foreign_bidder_policy = LOCAL_PARTNER_REQUIRED`).
- Atlas Energy has no registered local partner in the tender's jurisdiction.
- **Expected:** the partnership requirement resolves `PARTNERSHIP_REQUIRED`; overall `recommendation = PARTNERSHIP_REQUIRED`. The explanation must identify the specific required action (secure and evidence an eligible local partner) rather than a generic "not satisfied".

### Demo Tender 5 — Financial Threshold Failure

- Requires minimum annual turnover of $20M.
- Atlas Energy has $7.2M.
- **Expected:** the turnover requirement resolves `NOT_SATISFIED`, and a `qualification_gaps` row is created with exactly:
  - Tender requirement: $20M
  - Company evidence: $7.2M
  - Gap: $12.8M
- Since this requirement is mandatory but not an eligibility blocker in the same sense as jurisdiction (a financial shortfall is a capability gap, not a legal bar), expected `recommendation = DO_NOT_PURSUE` (or `REVIEW` if other requirements are otherwise strong and the shortfall is judged worth a human look — the fixture should make the intended recommendation unambiguous by keeping every other requirement clearly `SATISFIED`, so the financial gap alone drives the outcome).

## 3. Coverage Check

| Outcome / Recommendation | Exercised by |
|---|---|
| `SATISFIED` | Tenders 1, 2 |
| `NOT_SATISFIED` + `qualification_gaps` | Tender 5 |
| `ELIGIBILITY_BLOCKER` | Tender 3 |
| `PARTNERSHIP_REQUIRED` | Tender 4 |
| `PURSUE` | Tenders 1, 2 |
| `DO_NOT_PURSUE` | Tenders 3, 5 |
| `PARTNERSHIP_REQUIRED` (recommendation) | Tender 4 |
| `REVIEW` | Not forced by a dedicated tender — expected to arise naturally once low-confidence extractions are part of the fixture set (e.g. a slightly ambiguous personnel qualification match); tracked as a Phase 6 fixture-authoring note, not a gap in this spec. |

## 4. Use in Automated Tests

Per `ARCHITECTURE.md` §"Testing Strategy" (source brief §27), this demo dataset becomes the fixture basis for:

- **Unit tests** — individual comparisons (turnover vs. threshold, years vs. threshold, certification validity, jurisdiction match) using the exact numbers above.
- **Integration tests** — the full qualification workflow for each of the five tenders, asserting the expected outcome/recommendation table in §3.
- **End-to-end tests** — document upload → company setup → tender analysis → qualification report, using Atlas Energy and Demo Tender 1 (the clean-pass case) as the primary happy-path fixture, and Tenders 3–5 for negative-path UI coverage (blocker banner, partnership-required call-to-action, gap display).
