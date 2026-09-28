# Lodestar Tender — Automation Architecture

The end-state Lodestar Tender is an **autonomous bid-readiness pipeline**: a
company registers once, and from then on the platform continuously scouts
procurement sources, qualifies every relevant opportunity against that
company's evidence, assembles a submission-ready document pack for the ones
worth pursuing, and emails it to the company's Business Development Manager
(BDM) — who submits it. Nothing about a specific tender is done by hand.

This document defines that automation layer. It builds **on top of** the
architecture already in place (`ARCHITECTURE.md`, `AI_ARCHITECTURE.md`,
`SOURCE_CONNECTOR_ARCHITECTURE.md`, `DATABASE_DESIGN.md`) and changes none of
it: the connector interface, the ingestion pipeline, the deterministic
qualification engine, and the evidence model were designed for exactly this.

## 1. First Vertical: Oil & Gas

The first vertical is **oil & gas procurement**, Nigeria-first. O&G-specific
knowledge — NipeX NJQS / NOGIC JQS product & service category codes, NCDMB
Nigerian-content certification, upstream statutory documents (CAC, tax
clearance, PENCOM, ITF, NSITF, NUPRC/NCDMB permits, …) — is modelled as
**configurable data**, never as branching logic. The `PROJECT_SPEC.md`
§"Non-goals" rule stands: no code path special-cases a country or a named
authority. A product-code scheme is a row in a table; a statutory document is
a `company_legal_record`; a requirement's meaning always comes from the
tender's own text via Tender Intelligence.

## 2. Pipeline Overview

```
        ┌─────────── continuous, scheduled ───────────┐
        ▼                                             │
  SCOUT (per source)                                  │
   fetch → normalize → dedupe → store tender+docs     │
        │                                             │
        ▼                                             │
  EXTRACT  (Tender Intelligence → tender_requirements) │
        │                                             │
        ▼                                             │
  FAN-OUT: for each org subscribed to this source     │
           whose scouting profile matches the tender  │
        │                                             │
        ▼                                             │
  QUALIFY (Phase 5 engine → assessment + recommendation)
        │                                             │
        ├── passes the autonomy gate (§6) ────────────┤
        │        ▼                                     │
        │   ASSEMBLE bid pack (zip: statutory core +   │
        │            evidence docs + cover sheet)      │
        │        ▼                                     │
        │   DISPATCH → email BDM a magic link          │
        │        ▼                                     │
        │   TRACK submission + outcome                 │
        │                                             │
        └── fails the gate ──► REVIEW QUEUE (a human) ─┘
```

Every step is a discrete job with its own `AgentRun` / `SourceRun` record. The
SCOUT and EXTRACT steps run on a **scheduled cron tick** (`/api/cron/tick`,
Phase 10) — no queue. FAN-OUT onward (Phase 12+) can move to a real queue if
per-tender × per-org volume demands it, without changing any contract.

## 3. What Already Exists vs. What's New

| Capability | Status |
|---|---|
| `TenderSourceConnector` interface, `tender_sources` / `source_runs` / `source_items` | Designed (`SOURCE_CONNECTOR_ARCHITECTURE.md`), no implementation |
| Ingest → normalize → dedupe → store → extract → qualify pipeline | Contract defined; qualification engine **built** (Phase 5) |
| Deterministic qualification + recommendation (`PURSUE` / `REVIEW` / `PARTNERSHIP_REQUIRED` / `DO_NOT_PURSUE`) | **Built** — `backend/domain/qualification/qualify.ts` |
| Company evidence + `document_evidence` traceability chain | **Built** (Phase 3) |
| **Scouting profile** — per-org filters, autonomy level, BDM recipients + pure `tenderMatchesProfile` | **Built** (Phase 11) |
| **Org ↔ source subscription** — free sources auto-subscribed for all; pause/resume per org | **Built** (Phase 11) |
| **Credential vault** — encrypted per-org portal logins | Deferred (Phase 11b) |
| **Product / service code** capability + `PRODUCT_CODE_MATCH` comparator | Deferred (Phase 11b) |
| **Autonomous fan-out** — qualify every new tender for every matching org (`scouting_matches`, budgeted cron sweep) | **Built** (Phase 12) |
| **Autonomy gate** (pure `decideAutonomy`) **+ review queue** (`/review`, promote/dismiss) | **Built** (Phase 12) |
| **Bid pack assembly** — `bid_packs`, `jszip`: cover sheet + evidence + statutory core + manifest, stored, superseded packs kept | **Built** (Phase 13) |
| **BDM dispatch** — transactional email + magic-link bid room + delivery tracking + bounce re-queue | **Built** (Phase 14) |
| **Submission & outcome tracking** + pipeline analytics | **Built** (Phase 15, extends Bid Workspace, Phase 7) |
| Live connectors (Find a Tender, Contracts Finder — OCDS) + ingestion pipeline | **Built** (Phases 8–9) |
| Scheduled ingestion (cron route, per-source cadence, incremental) — **no BullMQ** | **Built** (Phase 10) |
| In-app notifications + deadline reminders | **Built** (Phase 10) |

## 4. Sources — Access Model

The full catalogue and per-source method live in `SOURCE_REGISTRY.md`. Three
tiers:

- **Free / public** (NipeX current-opportunities, NCDMB adverts, NOGIC JQS,
  BPP NOCOPO, TED, UNGM, World Bank, AfDB, SAM.gov, dgMarket). A connector is
  built once and the source is **auto-subscribed for every organization**.
  Prefer an official API or open-data feed where one exists; fall back to
  polite HTML fetching (robots.txt-aware, rate-limited) where it doesn't.
- **Free but registration-gated** (NipeX NJQS "LIVE" status, ADNOC Supplier
  Hub, ONGC vendor portal, Petronect, SAP Ariba vendor portals). Opportunities
  are only visible to a pre-qualified vendor. **Primary mechanism: inbound
  email** (`SOURCE_REGISTRY.md` §7) — these portals email tender invitations to
  registered vendors; the client points those emails at a platform address and
  the platform parses them in. No credentials, no browser, no ToS conflict.
  Stored-credential login + a headless browser is a per-client, per-portal,
  opt-in **last resort** only where a portal offers no notification email.
- **Paid aggregators** (Jorpex, GlobalTenders, TendersInfo, TendersOnTime,
  BidDetail, GlobalsBid, Energy Dais, Tender247, Tender18, TenderDetail,
  NationalTenders). Teasers are free; full detail is behind a subscription or
  API. Compliant access = the paid API/feed, not scraping. **Client brings
  their own key** — there is no platform-held subscription; a client who wants
  an aggregator supplies their subscription key during source onboarding (into
  the vault), and the scout runs that aggregator for that client only.

**Consequence:** every client has a **unique source set** — all free sources,
plus whichever gated/paid sources they have onboarded. The scout loop iterates
`org_source_subscriptions`, not a global source list.

**Legal note — resolved.** Government/public pages: polite fetch (robots.txt,
rate-limited) or official API. Vendor portals: **inbound email**, never
automated login. Paid aggregators: **licensed API/feed**, never scraped —
client brings their own subscription key (no platform-held subscription). Automated
credential login + browser survives only as an opt-in last resort with the
operator's explicit per-portal risk acceptance. Full matrix in
`SOURCE_REGISTRY.md` §7.

## 5. Credential Vault

`source_credentials` stores **only ciphertext** — AES-256-GCM, key held by a
managed KMS (or Supabase Vault), never in the database or application logs.
Credentials are:
- write-only from the UI (entered once, never rendered back),
- decrypted only inside a connector's authenticated fetch, in memory,
- scoped to one `(organization_id, source_id)` pair,
- revocable — deleting the row pauses that subscription.

Portals with MFA are handled per connector (TOTP secret in the vault, or a
"connected session" cookie the client refreshes when it expires — the
subscription goes to `ERROR` and the client is notified when it does).

## 6. The Autonomy Gate

**Implemented** as the pure `decideAutonomy()` in
`backend/domain/scouting/gate.ts` — `{ decision: AUTO_DISPATCH | REVIEW | SKIP,
reasons: string[] }`. Deterministic, never the AI.

- `recommendation = DO_NOT_PURSUE` → **SKIP** (recorded `SKIPPED`, never dropped).
- `recommendation = REVIEW` or `PARTNERSHIP_REQUIRED` → **REVIEW**.
- `recommendation = PURSUE` → **AUTO_DISPATCH** only when *all* hold, else
  **REVIEW** with every blocker listed:
  1. `scouting_profiles.autonomy_level = AUTO` (else `REVIEW_FIRST` holds all).
  2. `confidence >= AUTO_DISPATCH_MIN_CONFIDENCE` (constant in `gate.ts`, 75).
  3. No `qualification_results.outcome = NEEDS_REVIEW`.
  4. No mandatory requirement with `outcome = NOT_SATISFIED` (implied by
     `PURSUE`, re-checked as a guard).

Two conditions are enforced **upstream of the gate**, so a failure there means
no `PURSUE` even reaches it:

- The tender must satisfy `tenderMatchesProfile` — country / keyword / value /
  `min_response_days` (the deadline check). A miss records `SKIPPED` with the
  reasons; qualification never runs.
- The org must be `ACTIVE` on the tender's source with an `active` scouting
  profile — the fan-out query filters on this.

Anything not `AUTO_DISPATCH` waits in the **review queue** (`/review`) with the
reasons attached, so a person decides — keeping *no blind trust of AI output*
(`PROJECT_SPEC.md` §"Non-goals") intact even in full-auto mode.

## 7. Bid Pack Assembly

**Implemented** — `backend/lib/bid-pack.ts` `assembleBidPack()` builds one ZIP
(`jszip`, DEFLATE) for any qualification that isn't `DO_NOT_PURSUE`:

- **`cover-sheet.pdf`** (generated, pdf-lib) — tender title, buyer, country,
  reference, deadline, source URL; the qualification summary (score,
  confidence, recommendation); the per-requirement outcome table; open gaps;
  and a contents list.
- **`evidence/…`** — every `documents` row referenced by the assessment's
  `qualification_evidence`, downloaded from Storage. A file that can't be
  downloaded is listed in `missingDocuments`, not fatal.
- **`statutory/…`** — documents linked (via `document_evidence`) to a
  `company_legal_record` or `company_certification` capability record, minus
  any already in `evidence/`.
- **`manifest.json`** — `{ version, generatedAt, tender, qualification,
  outcomes[], gaps[], files[] }`; each `evidence` file entry carries
  `backsRequirements[]` (id + type + description) and `sourceDocumentName`.

Stored at `<org>/bid-packs/<uuid>-<slug>.zip` in the private documents bucket,
served only through a click-time signed URL (`/tenders/[id]/bid-pack/download`)
— never emailed as an attachment. A prior `CURRENT` pack for the same
(org, tender) is marked `SUPERSEDED` in the same transaction, never deleted.
`sweepBidPacks()` on the cron tick assembles packs for `APPROVED` matches that
don't have one yet.

## 8. BDM Dispatch

**Implemented** (Phase 14) — `backend/lib/bid-pack-dispatch.ts`:

- **Recipients** — `scouting_bdm_recipients`, one or more `{ email, name }`
  per org (Phase 11). A BDM need not be a platform user.
- **Channel** — Resend, called via plain `fetch` (`backend/lib/email.ts`), no
  SDK. `isEmailConfigured()` (`RESEND_API_KEY`) gates every send; unset =
  dispatch throws a clear `DispatchError` rather than silently no-op-ing. The
  email carries the tender headline, deadline, recommendation, score/
  confidence, and the **magic link**.
- **Autonomy boundary** — only a match the gate itself marked
  `decision = AUTO_DISPATCH` is ever sent automatically
  (`sweepDispatches()`, on the cron tick). A person's manual "Approve for bid
  pack" promotion is deliberately never auto-emailed — sending is a separate,
  explicit "Send to BD manager" click, on `/review` and the qualification
  report.
- **Bid room** (`/bid-room/[token]`) — the magic link opens a signed,
  expiring, no-login page: the full tender, the Phase 5 report UI (via the
  shared `QualificationReport` component, minus the org chrome), and a
  **Download bid pack** button. The token
  (`backend/domain/dispatch/magic-link.ts`) is a stateless HMAC-signed
  `dispatchId.expiresAt.signature`, re-verified on every request — page load,
  pack download, and each evidence-document download all check it
  independently, and an evidence download is only servable if that token's
  own assessment actually references the document. An invalid/expired/
  tampered token renders the ordinary 404 — nothing about why is leaked.
- **Tracking** — `bid_pack_dispatches` records provider message id and
  status (`SENT`/`DELIVERED`/`BOUNCED`/`FAILED`). Resend's bounce/delivery
  webhook (`frontend/app/api/webhooks/resend/route.ts`, Svix-format,
  verified with `node:crypto`) drives `DELIVERED`/`BOUNCED`; a bounce
  re-queues the `scouting_match` to `QUEUED` with the reason and notifies
  the org.

## 9. Submission & Outcome

**Implemented** (Phase 15) — `backend/lib/bid-workspace.ts` `markSubmitted()` /
`markNotSubmitted()`.

- The bid room (`/bid-room/[token]`) gives the BDM one of two actions: **mark
  submitted** (an optional reference and an optional receipt upload, stored
  in the same private bucket as everything else) or **mark not submitted**
  (a required reason — nothing here is silently dropped either). No session
  exists on that page; `backend/actions/bid-room.ts` re-verifies the magic
  link token on every submit and derives the org/tender from it, never from
  what the form itself claims.
- Both actions create the tender's `bid_workspaces` row if it doesn't exist
  yet (the BDM may be the first person to touch this tender) and write
  `submittedAt`/`submittedVia`/`submittedByEmail`/`submissionReference`/
  `receiptStoragePath` — or `notSubmittedAt`/`notSubmittedReason`. A later
  submission clears a prior "not submitted" state. The automation layer and
  the manual Bid Workspace (Phase 7) share one model, so this shows up on
  `/bids/[id]` immediately.
- `WON` / `LOST` were already reachable from `/bids/[id]`'s existing status
  control (Phase 7) — Phase 15 didn't need to add that part.
- `backend/lib/pipeline-analytics.ts` `getPipelineStats()` — scouted →
  qualified → approved → dispatched → submitted → won, a `/dashboard` card.
  Eventually: feed WON/LOST back into tuning the qualification thresholds
  per client (not built).

## 10. New Persistence (design — no migration yet)

All organization-owned, all carrying `organization_id` for single-table RLS
(`DATABASE_DESIGN.md` §2). Names indicative.

| Table | Purpose |
|---|---|
| `scouting_profiles` | 1:1 with organization. `active`, `autonomy_level`, value/country/sector filters, `min_response_days`. |
| `scouting_bdm_recipients` | `{ organization_id, email, name }` — the dispatch targets. |
| `org_source_subscriptions` | `(organization_id, source_id)` — `status` (`PENDING_ONBOARDING`/`ACTIVE`/`PAUSED`/`ERROR`), `credential_id?`, `inbound_address?` (for vendor-portal sources), `onboarding_notes`. Free sources: one row per org, auto-created. |
| `source_credentials` | Ciphertext only. `(organization_id, source_id)`, `secret_ciphertext`, `kind` (`password`/`api_key`/`session`/`totp`), `last_verified_at`. |
| `company_product_codes` | `{ organization_id, company_profile_id, scheme, code, label, status }` — the NJQS / NOGIC (or other) codes a company is approved for. |
| `scouting_matches` | **Built** (Phase 12). One per (org, tender) the fan-out considered — `status` (`QUALIFYING`/`QUEUED`/`APPROVED`/`DISPATCHED`/`DISMISSED`/`SKIPPED`/`ERROR`), `decision` (`AUTO_DISPATCH`/`REVIEW`/`SKIP`), `decision_reasons[]`, `assessment_id`, `recommendation`, `match_score`, `confidence`, `reviewed_by_id`/`reviewed_at`, `dispatched_at`, `error`. `@@unique(organization_id, tender_id)`. The review queue is `status = QUEUED`. |
| `bid_packs` | **Built** (Phase 13). `{ organization_id, tender_id, assessment_id, scouting_match_id?, status (CURRENT/SUPERSEDED), storage_path, file_name, size_bytes, file_count, manifest Json }`. `@@index([organization_id])`, `@@index([tender_id])`. |
| `bid_pack_dispatches` | `{ bid_pack_id, recipient_email, magic_link_token_hash, provider, provider_message_id, status, sent_at }`. |

`tender_requirements` gains an optional `product_code` / `code_scheme` pair
(nullable, additive) so Tender Intelligence can record a required NJQS/NOGIC
category when the tender states one; the engine gains a `PRODUCT_CODE_MATCH`
comparator that checks `company_product_codes` for an exact or parent-category
match.

## 11. New Environment

```
# Job queue (Phase 10)
REDIS_URL=

# Credential vault
CREDENTIAL_ENCRYPTION_KEY=            # or a KMS key id

# Transactional email — dispatch (Phase 14, built)
RESEND_API_KEY=
EMAIL_FROM_ADDRESS=Lodestar Tender <onboarding@resend.dev>
RESEND_WEBHOOK_SECRET=                # signs the bounce/delivery webhook

# Bid-room magic links (Phase 14, built) — generate locally, never a
# third-party credential; unset disables dispatch entirely.
MAGIC_LINK_SECRET=
BID_ROOM_TTL_DAYS=14

# Inbound email — vendor-portal invitations (Phase 9/11)
INBOUND_EMAIL_DOMAIN=inbound.<domain> # each org gets <org-slug>@ this domain

# Autonomy gate (built — constant in backend/domain/scouting/gate.ts, not yet env-configurable)
AUTO_DISPATCH_MIN_CONFIDENCE=75

# Headless-browser sources (only if a connector needs one)
BROWSER_POOL_URL=
```

Per `SOURCE_CONNECTOR_ARCHITECTURE.md` §6, per-source config (base URLs, rate
limits, paid API keys that the *platform* holds centrally) lives in that
connector's own module and env, never in shared code.

## 12. Phasing

This layer is sequenced after the demo (Phase 6) and Bid Workspace (Phase 7)
are locked in — see `IMPLEMENTATION_PLAN.md` Phases 8–16. In outline:

- **8 ✅** First live connector (UK Find a Tender, OCDS) + ingestion pipeline.
- **9 ✅** `OcdsConnector` base + UK Contracts Finder + `/admin/sources` health.
- **10 ✅** Scheduled ingestion via a secured cron route (`/api/cron/tick`,
  `vercel.json`, or `npm run tick`) — **no BullMQ**; per-source cadence,
  incremental `since`. Notifications foundation + deadline reminders.
- **11 ✅** Scouting profiles + org↔source subscriptions (lean cut — credential
  vault + product-code capability deferred to 11b, no gated source yet).
- **12 ✅** Autonomous qualification fan-out + the autonomy gate + review queue.
- **13 ✅** Bid pack assembly.
- **14 ✅** BDM dispatch (email + magic-link bid room + delivery/bounce tracking).
- **15 ✅** Submission & outcome tracking + pipeline analytics.
- **16 🔶** Production hardening — RLS audit (live DB, clean), Postgres-backed
  rate limiting on AI/dispatch/bid-pack assembly, timing-safe cron auth. Not
  yet: performance/index pass, cost dashboards, deployment hardening.
- **17 ✅** Billing & subscriptions — not part of the original roadmap; added
  after a pricing/product discussion. 14-day full-access trial, three tiers
  gated by autonomy level + source access, Stripe/Paystack/Flutterwave
  checkout, `/settings/billing`. Every autonomous/AI action refuses cleanly
  once billing lapses. See `IMPLEMENTATION_PLAN.md` Phase 17.
- **18 ✅** Insights (analytics) — same discussion. Value-by-currency funnel,
  win rate, time saved (auto-handled vs needed a person), gap-category
  ranking, source performance. No schema change — derived entirely from
  existing rows. `/insights`. See `IMPLEMENTATION_PLAN.md` Phase 18.
- **19 ✅** First-payment fee + admin payments monitoring — a one-time setup
  fee on the first checkout only (on top of the recurring plan; Stripe as a
  native second line item, Paystack/Flutterwave folded into the first
  charge's total), a new `payment.succeeded` event type so a renewal reads
  differently from the initial activation, and `/admin/billing` — every
  billing event across every client, revenue this month by currency, counts
  by provider and billing status. See `IMPLEMENTATION_PLAN.md` Phase 19.

## 13. Source Health — How the Operator Knows a Source Works

Three layers, all built on what already exists (`checkHealth()` in
`backend/sources/types.ts`, `source_runs` in the schema):

1. **`checkHealth()` probe (per connector).** Every connector implements a
   cheap smoke test — fetch the listing endpoint, assert HTTP 200 + a known
   expected field/shape in the response. A scheduler runs every probe on a
   fixed interval (default 30 min), independent of ingestion. Output:
   `{ sourceId, healthy, lastSuccessfulRunAt, lastError }`.

2. **`source_runs` (per ingestion).** Every scheduled fetch writes a row:
   `RUNNING → SUCCEEDED | FAILED`, with `items_found`, `items_new`,
   `error_message`, timing. A source is **stale** with no `SUCCEEDED` run in
   its expected window, **failing** if the last N runs are `FAILED`.

3. **Alerts (Phase 10).** X consecutive probe/run failures, or a stale source,
   raises an operator notification. A **client-specific** subscription entering
   `ERROR` (expired credentials, bounced inbound address, revoked aggregator
   key) notifies that client too — and the fan-out skips that source for that
   client until it's fixed.

**Sources dashboard** (Phase 9): one row per `tender_sources` —
health badge · last successful run · last-24h success rate · items found/new
last run · per-client subscription status for gated/paid/inbound sources.
Green/amber/red is derived from layers 1–2, never entered by hand.

**Before a connector ships** (`SOURCE_CONNECTOR_ARCHITECTURE.md` §7): its
integration tests must assert `normalize()` yields a valid `Tender`, the dedup
key is stable across two fetches, and `checkHealth()` reports an unreachable
source as unhealthy.

**Right now (pre-Phase 8):** no connectors exist, so a source's liveness is
confirmed by hand when its connector is built; the health system above is what
makes it continuous from then on.

## 14. Rate Limiting (Phase 16)

**Implemented** — `backend/lib/rate-limit.ts`. Postgres-backed (counts rows in
a recent window against tables other modules already write), no Redis —
consistent with §2's stance that a queue is infra for its own sake until
volume demands it. The threat model is a member mashing a button, not a
distributed attack, so thresholds are generous:

| Guard | Limit | Counts against |
|---|---|---|
| `assertAiRateLimit` | 10 / org / minute | `agent_runs` rows in the window |
| `assertDispatchRateLimit` | 20 / org / hour | `bid_pack_dispatches` rows |
| `assertBidPackRateLimit` | 10 / org / minute | `bid_packs` rows |

Wired into `runQualification` (only the real-AI path — seeded interpretations
skip it), `processUploadedDocument` (which previously logged no `AgentRun` at
all — fixed as part of this pass, matching DATABASE_DESIGN.md §10's own claim
that `agent_runs` logs every AI invocation), `dispatchBidPack`, and
`assembleBidPack`. A `RateLimitError` is a known, user-facing error type in
every action that can throw one — same pattern as `QualificationError` /
`BidPackError` / `DispatchError`.

**Deliberately not rate-limited**: ingestion and the cron tick itself — both
are already gated by `requirePlatformAdmin()` / `CRON_SECRET`, a stronger
control than a counter. The cron route's own auth comparison was hardened in
the same pass: it compared the bearer token with plain `!==` (a timing
side-channel against a long-lived secret) where the magic-link and webhook
signature checks already used `timingSafeEqual` — now shared via
`backend/lib/secure-compare.ts`.

`frontend/next.config.ts` also gained baseline security headers in this pass
— `X-Frame-Options`/`frame-ancestors 'none'` (clickjacking, notably the
public bid room), `X-Content-Type-Options`, `Referrer-Policy`,
`Permissions-Policy`. A full script-src CSP is deliberately deferred — it
needs a nonce wired through the App Router and real in-browser verification.
