# Lodestar Tender — Implementation Plan

Each phase must leave the project in a runnable state. No phase is started until the previous one is explicitly signed off — see `PROJECT_SPEC.md` and the "Important Development Rule" in the source brief: do not build functionality merely because it appears in the long-term vision.

## Phase 0 — Architecture (this phase)

Repository inspection, `PROJECT_SPEC.md`, `ARCHITECTURE.md`, `DATABASE_DESIGN.md`, `AI_ARCHITECTURE.md`, `SOURCE_CONNECTOR_ARCHITECTURE.md`, `DEMO_DATA_SPEC.md`, `IMPLEMENTATION_PLAN.md` (this file), initial Next.js/TypeScript/Tailwind project structure, Prisma schema, lean dependency installation, `.env.example`, `.gitignore`. No application features implemented. **Status: complete.**

## Phase 1 — Application Foundation + Authentication + Organizations

- Supabase Auth wiring: sign up, sign in, session handling (`src/lib/supabase/*` already scaffolded).
- Organization creation and OWNER membership on creation (`organizations`, `organization_members`).
- Row Level Security policies applied via Supabase SQL migrations for every organization-owned table already defined in `prisma/schema.prisma` (policy shape documented in `DATABASE_DESIGN.md` §2).
- Basic app shell: authenticated layout, protected routes, role-gated navigation.
- **Exit criteria:** a user can register, create an organization, and see an empty authenticated dashboard; a second user cannot see the first organization's (still-empty) data. **Status: complete** — verified end-to-end against a live Supabase project (real signup/login/logout, migration + RLS applied, e2e suite passing).
- **Deferred to a later phase:** an invite/join flow for a *second* member of an existing organization. Only the creating user (as OWNER) is provisioned right now; `hasRequiredRole()` and the `ADMIN`/`MEMBER`/`VIEWER` roles exist and are unit-tested, but nothing assigns them yet.

## Phase 2 — Company Capability Passport

- CRUD UI and server actions for `company_profiles` and all child tables (`company_services` … `company_legal_records`).
- Form/validation layer using Zod schemas mirroring `src/domain/company/types.ts`.
- No document upload yet — capability records are entered directly, evidence links come in Phase 3.
- **Exit criteria:** an organization can fully populate a Capability Passport by hand, matching every field in `DATABASE_DESIGN.md` §4. **Status: complete** — `/company` covers identity plus all 8 capability domains with add/remove, organization-scoped, 33/33 tests passing against the live database.

## Phase 3 — Document Intelligence

- Upload flow to Supabase Storage; `documents` row creation. **Done.**
- Document Intelligence AI module (`src/server/ai/document-intelligence`): classification + structured extraction, validated against Zod schemas, following the prompt-boundary pattern (`src/server/ai/shared/prompt-boundaries.ts`). **Done** — single bounded, schema-validated call with one corrective retry (`src/server/ai/shared/client.ts`); PDFs/images go to the model as native content blocks, DOCX via mammoth, XLSX deferred.
- `document_extractions`, `document_reviews` — including the human-review UI for low-confidence extractions. **Done** — inline processing on upload (`src/server/lib/document-processing.ts`), status transitions to a terminal state, `/company/documents/[id]` review page (confirm/reject, reprocess).
- `document_evidence` — linking an extraction to the capability record it supports. **Done** — `backend/domain/documents/evidence.ts` (capability-type allow-list) + `backend/lib/document-evidence.ts` (polymorphic-reference resolution with org-scoping per `DATABASE_DESIGN.md` §5); link / verify / unlink actions on `/company/documents/[id]`; linked-document counts surfaced on `/company`. Integration-tested against the live database.
- **Exit criteria:** uploading one of the demo company's documents (`DEMO_DATA_SPEC.md` §1) produces a correctly classified, structured extraction linked as evidence to the relevant capability record, with confidence and source page shown. **Status: substantially complete** — full pipeline, review UI, and evidence-linking built and tested; the one remaining step is a live `analyzeDocument` run (needs `ANTHROPIC_API_KEY`; `isAIConfigured()` is the gate and uploads stay UPLOADED until it is set).

## Phase 4 — Tender Intelligence

- `tenders`, `tender_documents`, `tender_requirements`, `tender_evaluation_criteria` CRUD (manually created/uploaded tenders — no live source yet). **Done** — `/tenders` list + `/tenders/[id]` detail (inline-editable fields, document upload, requirement add/edit/delete, evaluation criteria). Manual tenders are attributed to a built-in `manual` `TenderSource` (`backend/lib/tenders.ts`); tenders are platform-shared, any authenticated member may create/edit them.
- Tender Intelligence AI module (`backend/ai/tender-intelligence`): requirement extraction, evaluation-criteria extraction, eligibility interpretation (`foreign_bidder_policy`, `local_partner_required`) from tender document text. **Done** — one bounded, schema-validated call (`backend/ai/shared/client.ts`, reused); `prepareDocumentContent` lifted to `backend/ai/shared/text-extraction.ts`. Runs inline on upload via `backend/lib/tender-processing.ts`, with processing state + idempotency tracked on an `AgentRun` row (`tender_documents` has no status column).
- **Exit criteria:** uploading one of the five demo tenders (`DEMO_DATA_SPEC.md` §2) produces a correct, structured set of `tender_requirements` with the right `category`, `mandatory`, and jurisdiction fields. **Status: in progress** — module + UI built and unit-tested (`tests/unit/tender-intelligence.test.ts`, `tenders-validation.test.ts`); live `analyzeTender` run pending an `ANTHROPIC_API_KEY` (same gate as Phase 3).

## Phase 5 — Qualification Engine

- Deterministic comparison logic (`backend/domain/qualification/qualify.ts` — pure, no AI/DB): dispatches on a `RequirementComparator` (`FINANCIAL_MIN`, `EXPERIENCE_YEARS_MIN`, `CERTIFICATION_PRESENT`, `JURISDICTION_REGISTRATION`, `LOCAL_PARTNER`, `GENERIC_EVIDENCE`). **Done.**
- Qualification Intelligence AI module (`backend/ai/qualification-intelligence`) — one bounded schema-validated call turning each free-text requirement into a comparator + candidate evidence refs from the org's capability catalogue. Never decides an outcome. **Done.**
- Scoring: match score + confidence (kept separate) + eligibility-blocker override (a mandatory `ELIGIBILITY_BLOCKER` forces `recommendation != PURSUE`) + recommendation derivation, all in `qualify.ts`. **Done.**
- `qualification_assessments` / `qualification_results` / `qualification_gaps` / `qualification_evidence` persistence via `backend/lib/qualification.ts` (loads the passport with `backend/lib/company-passport.ts`, runs the AI interpreter, runs the pure engine, writes rows in one transaction, records an `AgentRun`). Report UI at `/tenders/[id]/qualification` (recommendation banner, per-requirement outcome + explanation + evidence chain to the source document, gaps). **Done.**
- **Exit criteria (the project's primary success criterion):** running qualification for Atlas Energy against each of the five demo tenders produces exactly the outcomes in `DEMO_DATA_SPEC.md` §3. **Status: met at the unit level** — `tests/unit/qualification-acceptance.test.ts` reproduces the full §3 table with hand-written interpretations (what `DEMO_DATA_SPEC.md` §4 asks for); the live end-to-end run through the AI interpreter needs `ANTHROPIC_API_KEY` (same gate as Phases 3–4), and the fixture documents are Phase 6.

## Phase 6 — Demo End-to-End Workflow

- Fixture files under `demo-data/` (`company.ts`, `documents/*.md` sources + manifest, `tenders.ts`, `expected-outcomes.ts`) — all marked `DEMO DATA — FICTIONAL`. **Done.**
- `backend/lib/demo-seed.ts` + `npm run seed:demo` — creates a signed-in-able demo organisation (Atlas Energy, login `demo@lodestar.test`) with its passport, 11 verified evidence documents (rendered to real PDFs from the Markdown sources via `backend/lib/demo-pdf.ts` + `pdf-lib`, uploaded to Storage), the five demo tenders, **and a persisted qualification assessment per tender** — so the dashboard, `/company`, `/tenders`, and the qualification reports all render real results with no API key. Re-runnable. **Done.**
- `tests/integration/qualification.test.ts` — seeds the dataset into the real database and asserts the full `DEMO_DATA_SPEC.md` §3 outcome/recommendation table on the persisted assessment (through the real `runQualification` persistence path, with the AI interpreter bypassed by the pre-authored interpretations — one additive optional `interpretations?` param on `runQualification`). **Done.** (`vitest.config.ts` now sets `fileParallelism: false` — the integration suites share one Supabase pooler and were flaky in parallel.)
- `tests/e2e/demo-workflow.spec.ts` — the browser workflow (register → seed → run qualification → verify Pursue / eligibility-blocker / gap). Authored; `test.skip` until `ANTHROPIC_API_KEY` is set.
- **Exit criteria:** `npm run test` passes against the seeded dataset and a new team member can `npm run seed:demo` and click through the whole workflow. **Status: met (6a).** Pending `ANTHROPIC_API_KEY` (6b): the live upload → extract path (the PDFs are ready for it), `npm run test:e2e` green, and confirming the live interpreter's output matches the authored interpretations.

## Platform Admin Console (cross-cutting, added between Phase 6 and 7)

A platform-operator view over every tenant. **Not a phase** — an operational
capability requested once clients could self-serve.

- **Identity:** env allow-list `PLATFORM_ADMIN_EMAILS` (never self-service). An
  allow-listed email signs up / signs in normally and lands on `/admin`
  instead of a client dashboard. `backend/lib/platform-admin.ts`
  (`isPlatformAdmin`, `requirePlatformAdmin`, `logAdminAction`).
- **Tenancy exception (documented):** `/admin/*` reads across organizations via
  the Prisma owner connection (already bypasses RLS — `ARCHITECTURE.md` §8).
  Isolated route tree, gated on the allow-list, every mutation logged to
  `admin_actions`.
- **Overview** (`/admin`): platform totals, recommendation breakdown, activity
  feed, org table (passport % · docs · qualified · pursue · suspended).
- **Per-org** (`/admin/organizations/[id]`): read-only mirror (members,
  passport, documents + downloads, qualified tenders + reports) plus operator
  actions — **suspend / reactivate** (`organizations.suspended_at`; a
  suspended org's members are blocked at `requireOrganizationContext()` →
  `/suspended`), **re-run a qualification**, **internal notes**, **re-seed the
  demo org**.
- Schema: migration `admin_console` — `organizations.suspended_at` +
  `admin_actions` + `admin_org_notes` (RLS enabled, no `authenticated` policy —
  `supabase/migrations/*_admin_console_rls.sql`).
- `<QualificationReport>` (`frontend/components/qualification-report.tsx`)
  extracted so the client report page and the admin report page share one body.

## Phase 7 — Bid Workspace

- `bid_workspaces`, `bid_tasks`, `bid_deadlines` CRUD, created from a qualification assessment with `recommendation` in `{PURSUE, REVIEW, PARTNERSHIP_REQUIRED}`. **Done** — `backend/lib/bid-workspace.ts` (`createBidWorkspace` guards recommendation, idempotent on `@@unique([organizationId, tenderId])`, seeds tasks + the submission deadline in one transaction, pinned to the assessment it was created from); `backend/actions/bids.ts`; `/bids` list + `/bids/[id]` workspace.
- Requirements checklist and gaps checklist derived from the qualification assessment (no duplicated business logic — `backend/domain/bids/checklist.ts` `deriveBidChecklist` is a pure projection of `qualification_results` / `qualification_gaps`). **Done.**
- No migration (tables + enums + RLS existed since Phase 0/1). No AI dependency — works fully without an `ANTHROPIC_API_KEY`.
- Entry points on the tender page + qualification report; nav item, dashboard tile + step, admin per-org "Bid workspaces" section.
- **Exit criteria:** from a `PURSUE` demo qualification, a user can create a workspace, see a pre-populated checklist, track tasks/deadlines. **Met** — `tests/integration/bids.test.ts` reproduces create-from-PURSUE (with seeded deadline + tasks), the DO_NOT_PURSUE guard, idempotency, and a task-status update against the live DB.
- **Follow-up:** "sync workspace to latest qualification" when a tender is re-qualified (the workspace currently stays pinned to its original assessment).

---

## Automation layer (Phases 8–16)

Phases 8+ build the **autonomous scout → qualify → pack → dispatch pipeline**
described in `AUTOMATION_ARCHITECTURE.md`. First vertical: **oil & gas**,
Nigeria-first. Source catalogue and access model: `SOURCE_REGISTRY.md`. None of
this changes Phases 1–7 code — it layers on the connector interface, the
ingestion pipeline, and the Phase 5 qualification engine, all of which were
designed for it.

**Decisions taken (2026-09-09):**
- **Dispatch is auto, gated** — the platform emails the BDM automatically only
  when qualification passes the deterministic autonomy gate
  (`AUTOMATION_ARCHITECTURE.md` §6: `PURSUE` + high confidence + no
  `NEEDS_REVIEW`/`UNKNOWN` + deadline far enough out + org onboarded on the
  source). Everything else goes to a human review queue.
- **All free/public sources are auto-subscribed for every organization.**
  Registration-gated and paid sources are onboarded **per client** — the
  client registers on the portal / supplies a subscription key, credentials
  go in an encrypted vault, and only then does the scout run that source for
  that client. Every client therefore has a **unique source set**.
- **O&G is the first vertical, not the whole product.** NJQS / NOGIC product
  & service codes, NCDMB certification, and Nigerian statutory documents are
  modelled as **configurable data**, never branching logic — the
  `PROJECT_SPEC.md` §"Non-goals" rule stands.
- **BDM = configurable per organization** — one or more email addresses (not
  necessarily platform users); a magic link opens a read-only bid room.

### Phase 8 — First Live Tender Source — **DONE**

- `TenderSourceConnector` interface + registry (`backend/sources/types.ts`,
  `backend/sources/registry.ts`); a shared **OCDS** mapper
  (`backend/sources/ocds/` — `map.ts`, `cpv.ts`) reused across every
  OCDS-publishing portal; first connector **UK Find a Tender**
  (`backend/sources/connectors/find-a-tender/`) — NOCOPO exposes no public API
  (confirmed), so an OCDS source was chosen for data-access feasibility and
  reuse. Energy / O&G filtered by CPV prefix.
- Ingestion pipeline `backend/lib/tender-ingestion.ts` (`ingestSource`): fetch →
  `SourceItem` (raw audit) → `normalize` → Zod validate → upsert `Tender` by
  `@@unique([sourceId, sourceIdentifier])` → download documents (processable
  MIME only, size-capped) → `processTenderDocument` inline. Never throws; writes
  a `SourceRun`. Trigger: `npm run ingest -- find-a-tender` or the `/admin`
  Sources section (`adminRunIngestion`).
- **Exit criteria met:** 17 real UK energy tenders ingested through
  ingest → normalize → dedupe → store, verified live
  (`tests/integration/tender-ingestion.test.ts`, `find-a-tender-live.test.ts`),
  and an ingested tender qualified end to end. Document processing +
  requirement extraction run inline where docs + an `ANTHROPIC_API_KEY` are
  present (OCDS releases often carry no doc URLs — handled gracefully). Zero
  change to `backend/domain` / `backend/ai` / DB core.
- No migration (`TenderSource`/`SourceRun`/`SourceItem` + RLS existed).

### Phase 9 — Source fleet (lean) + source health — **DONE**

- `OcdsConnector` (`backend/sources/ocds/connector.ts`) — config-driven
  connector for any OCDS-publishing portal (fetch/paginate/filter/normalise).
  `find-a-tender` refactored to a config of it (behaviour-preserving); second
  connector **UK Contracts Finder** (`contracts-finder`) added as a config,
  live-verified, real ingest of 13 energy tenders. **Two live connectors; the
  next OCDS source is a config entry.**
- Source health: `backend/lib/source-health.ts` (`isStale`, `sourceHealthRows`)
  + `/admin/sources` page (per-connector badge, last run, tender count,
  **Run now** + **Check health** buttons; `adminCheckSourceHealth` action).
- No migration, no AI dependency. `backend/domain` / `backend/ai` untouched.
- **Phase 9b backlog** (self-contained connector modules, added later without
  touching anything else): TED (`api.ted.europa.eu/v3`, POST, own mapper),
  UNGM / World Bank / AfDB, the **NipeX** headless-browser connector, and the
  **inbound-email** connector for vendor portals (`SOURCE_REGISTRY.md` §7).
- Source-health dashboard (source list, last run, error rate) over
  `source_runs` / `checkHealth()`.
- **Exit criteria:** ≥4 free sources ingest on demand; the connector
  abstraction holds under real diversity of payloads; health view is live.

### Phase 10 — Scheduled Ingestion + Notifications — **DONE**

- **No BullMQ/Redis** — a queue at two connectors is infra for its own sake.
  Mechanism: a secured cron route `frontend/app/api/cron/tick/route.ts`
  (`Authorization: Bearer $CRON_SECRET`; unset → 503) driven by `vercel.json`
  crons, GitHub Actions, a system cron, or `npm run tick`. A real queue is
  revisited at the Phase 12 fan-out only if volume demands it.
- `backend/lib/scheduled-ingestion.ts` — `runDueIngestions()`: per-source
  cadence (`sourceCadenceHours`, default 6h), incremental `since` from the last
  successful `SourceRun` (−1h overlap), sequential with a ~55s budget. Failed
  runs → `notifyPlatformAdmins`.
- `backend/lib/notifications.ts` — `createNotification` (content-deduped, no
  schema change), `notifyPlatformAdmins`, `listNotifications`, `unreadCount`.
  `/notifications` page + header "Alerts" pill (`NotificationBell`).
  `backend/actions/notifications.ts` (mark read / all).
- `backend/lib/deadline-reminders.ts` — `sendDeadlineReminders()`: open bid
  workspaces, deadlines in 7/3/1-day buckets → `DEADLINE` notifications.
- `backend/lib/platform-admin-emails.ts` split out so job code doesn't pull
  `next/navigation`.
- `/admin/sources` shows cadence + "next due".
- **Exit criteria met:** `npm run tick` runs due sources unattended and skips
  fresh ones; verified live (`tests/integration/scheduled-ingestion.test.ts` —
  due-detection, incremental `since`, dedup, cron auth). No migration; no
  `backend/domain` / `backend/ai` change.

### Phase 11 — Scouting Profiles + Source Subscriptions — **DONE** (lean cut)

- Migration `scouting` (additive): `scouting_profiles` (per-org country /
  keyword / value filters, `active`, `autonomy_level`, `min_response_days`),
  `scouting_bdm_recipients`, `org_source_subscriptions`
  (`@@unique([organizationId, sourceId])`, `status`, `onboarding_notes`).
  Standard `is_organization_member` RLS
  (`supabase/migrations/20260910175500_scouting_rls.sql`).
- `backend/domain/scouting/match.ts` — pure `tenderMatchesProfile(tender,
  profile, now)`; excludes only on *positive* mismatch evidence (missing
  value / deadline never drops a tender). Returns every failing check in
  `reasons` for the Phase 12 review queue.
- `backend/lib/scouting.ts` — `getScoutingProfile` (upsert-on-read),
  `updateScoutingProfile`, add/remove BDM recipient,
  `ensureFreeSubscriptions` (every `connectorKeys()` source → `ACTIVE`
  subscription, idempotent), `backfillFreeSubscriptions` (all orgs; runs on
  the cron tick), `setSubscriptionStatus`, `listSubscriptions`,
  `previewMatches` (runs the matcher over already-ingested tenders).
- `/settings/scouting` — policy card (toggle, autonomy, filters), bid-pack
  recipients, sources (pause/resume), "what this scout would match" preview.
  "Settings" nav item; dashboard "Autonomous scouting" stat; read-only view
  on the admin org page.
- Auto-subscribe wired into `createOrganization`; demo seed gets an active
  `AUTO` profile + a BDM recipient + a subscription.
- **Exit criteria met:** an org sets its filters and BDM recipients, is
  auto-subscribed to every free source, and the match preview reflects the
  policy. `tests/unit/scouting-match.test.ts`,
  `tests/integration/scouting.test.ts`.

**Phase 11b (deferred, non-blocking):** `company_product_codes` +
`PRODUCT_CODE_MATCH` comparator; encrypted `source_credentials` vault +
gated-source onboarding — lands with the first gated/paid connector.

### Phase 12 — Autonomous Qualification Fan-out + Review Queue — **DONE**

- Migration `scouting_fanout` (additive): `scouting_matches` — one row per
  (organization, tender) the scout considered. `status`
  (`QUALIFYING`/`QUEUED`/`APPROVED`/`DISPATCHED`/`DISMISSED`/`SKIPPED`/`ERROR`),
  `decision` (`AUTO_DISPATCH`/`REVIEW`/`SKIP`), `decisionReasons[]`,
  `assessmentId`, `recommendation`, `matchScore`, `confidence`,
  `reviewedById`/`reviewedAt`, `dispatchedAt`, `error`.
  `@@unique([organizationId, tenderId])`. Standard org-member RLS.
- `backend/domain/scouting/gate.ts` — pure `decideAutonomy()`. `DO_NOT_PURSUE`
  → SKIP; `REVIEW`/`PARTNERSHIP_REQUIRED` → REVIEW; `PURSUE` → AUTO_DISPATCH
  only under `AUTO` autonomy, confidence ≥ `AUTO_DISPATCH_MIN_CONFIDENCE` (75),
  no `NEEDS_REVIEW` results, no unmet mandatory — else REVIEW, listing every
  blocker.
- `backend/lib/scouting-fanout.ts` — `evaluateMatch` (match → qualify (reuse an
  existing assessment, else run fresh; `QualificationError` → `ERROR` status +
  a notification) → gate → upsert + notify), `fanOutTender(tenderId)` (one
  tender across every ACTIVE subscriber with an active profile),
  `sweepScoutingFanout({ budgetMs })` (every not-yet-evaluated
  subscription × tender, restartable). A human-decided row
  (APPROVED/DISMISSED/DISPATCHED) is never re-touched.
- Wired: `/api/cron/tick` (`runDueIngestions` 35s → `sweepScoutingFanout` 15s),
  `npm run tick`, `npm run ingest`, `adminRunIngestion`; demo seed records a
  match per demo tender.
- `backend/lib/scouting-review.ts` + `backend/actions/review.ts` +
  `/review` — the queue (with the gate's reasons), promote / dismiss,
  a status roll-up, and a recent-activity audit list. "Review queue" nav item;
  dashboard stat; admin org page shows the per-status counts.
- **Exit criteria met:** the demo's clean-PURSUE tender is auto-approved; the
  `NEEDS_REVIEW` and `PARTNERSHIP_REQUIRED` tenders queue with the reason;
  `DO_NOT_PURSUE` is recorded `SKIPPED`, never dropped.
  `tests/unit/scouting-gate.test.ts`, `tests/integration/scouting-fanout.test.ts`.

**Not yet:** a real job queue (still the budgeted cron sweep — revisit when
tender volume needs it); automatic re-fan on tender *update* (only new tenders
are swept today).

### Phase 13 — Bid Pack Assembly — **DONE**

- Migration `bid_packs` (additive): `BidPack` — `status` (`CURRENT` /
  `SUPERSEDED`), `storagePath`, `fileName`, `sizeBytes`, `fileCount`,
  `manifest Json`, `assessmentId`, `scoutingMatchId?`.
  `@@index([organizationId])`, `@@index([tenderId])`. Standard org-member RLS.
- `jszip` added. `backend/domain/bid-pack/manifest.ts` — pure manifest types +
  `safeEntryName` / `bidPackFileName` / `outcomeLabel`.
- `backend/lib/bid-pack-cover.ts` — `renderCoverSheet(manifest)` → a pdf-lib
  PDF (tender header, qualification summary, per-requirement outcome table,
  open gaps, contents list). WinAnsi `ascii()` sanitiser as in `demo-pdf.ts`.
- `backend/lib/bid-pack.ts` — `assembleBidPack({ organizationId, tenderId })`:
  refuses `DO_NOT_PURSUE`; collects the documents behind each
  `qualification_evidence` row (with what each backs) + the statutory core
  (documents linked to a `company_legal_record` / `company_certification`);
  downloads them from Storage (a missing file is logged in
  `missingDocuments`, not fatal); builds `cover-sheet.pdf` + `evidence/…` +
  `statutory/…` + `manifest.json`; zips (DEFLATE); uploads to the private
  bucket under `<org>/bid-packs/`; marks a prior `CURRENT` pack `SUPERSEDED`
  in the same transaction. Plus `getCurrentBidPack`, `listBidPacks`,
  `bidPackDownloadUrl`, `sweepBidPacks({ budgetMs })`.
- Wired: `/api/cron/tick` (`sweepBidPacks` 12s after the fan-out sweep),
  `npm run tick`, `npm run ingest`, `adminRunIngestion`; the demo seed
  assembles a pack for every APPROVED demo tender.
- `backend/actions/bid-pack.ts` `assembleBidPackAction` +
  `/tenders/[id]/bid-pack/download` (fresh signed URL at click time). Buttons
  on `/review` (an "Approved — bid packs" section) and the qualification
  report. Admin org page shows the pack count.
- **Exit criteria met:** the demo's DEMO-TENDER-1 produces a ZIP whose
  `manifest.json` maps every `evidence/*` file to the requirement(s) it backs
  and names its source document; re-assembly supersedes; `DO_NOT_PURSUE` is
  refused. `tests/unit/bid-pack-manifest.test.ts`,
  `tests/integration/bid-pack.test.ts`.

### Phase 14 — BDM Dispatch — **DONE**

- Migration `bid_pack_dispatches` (additive): one row per (bid pack,
  recipient) send — `status` (`QUEUED`/`SENT`/`DELIVERED`/`BOUNCED`/`FAILED`),
  `providerMessageId`, `bidRoomExpiresAt`, `sentAt`/`deliveredAt`/`bouncedAt`,
  `errorMessage`. Standard org-member RLS.
- Provider: **Resend**, called via plain `fetch` (`backend/lib/email.ts`) —
  no SDK dependency, same approach as the Anthropic/Supabase clients.
  `isEmailConfigured()` (`RESEND_API_KEY`) gates every send.
- `backend/domain/dispatch/magic-link.ts` — pure, stateless HMAC-signed token
  (`dispatchId.expiresAt.signature`, base64url); `backend/domain/dispatch/
  email-template.ts` — pure HTML+text builder.
- `backend/lib/bid-pack-dispatch.ts` — `dispatchBidPack()` (assembles the pack
  if missing, one `BidPackDispatch` + signed bid-room link per recipient, sets
  `ScoutingMatch` → `DISPATCHED`); `sweepDispatches()` (cron — only
  `decision = AUTO_DISPATCH` matches; a manually-promoted match always needs
  the person's own "Send to BD manager" click); `getBidRoomByToken` /
  `bidRoomPackStoragePath` / `bidRoomDocumentStoragePath` (token re-verified
  at every call, never trusted from a prior render);
  `recordDispatchDelivered` / `recordDispatchBounced` (a bounce re-queues the
  match to `QUEUED` with the reason + notifies the org).
- `/bid-room/[token]` — no-login page (`middleware.ts`'s protected-prefix list
  doesn't include it): tender + the Phase 5 report UI + pack download; an
  invalid/expired/tampered token renders the ordinary 404, leaking nothing.
- `frontend/app/api/webhooks/resend/route.ts` — Resend's Svix-format
  bounce/delivery webhook, verified with `node:crypto` (no `svix` package).
- Wired: cron tick (`sweepDispatches` after the bid-pack sweep), `npm run
  tick`/`ingest`; manual "Send to BD manager" on `/review` and the
  qualification report (works for a person-promoted match, which the sweep
  never touches).
- **Exit criteria met:** an `AUTO_DISPATCH` match is emailed automatically;
  its link opens the bid room and downloads the pack with no login; a
  simulated bounce re-queues the match. `tests/unit/magic-link.test.ts`,
  `tests/unit/dispatch-email-template.test.ts`,
  `tests/integration/bid-pack-dispatch.test.ts`.

### Phase 15 — Submission & Outcome Tracking — **DONE**

- Migration `bid_workspace_submission` (additive columns on `bid_workspaces`):
  `submittedAt`, `submittedVia` ("BID_ROOM"|"MANUAL"), `submittedByEmail`,
  `submissionReference`, `receiptStoragePath`/`receiptFileName`,
  `notSubmittedAt`, `notSubmittedReason`. No RLS change needed — RLS is
  row-level, already covers the table from Phase 1.
- `backend/lib/bid-workspace.ts` gains `markSubmitted()` / `markNotSubmitted()`
  — both idempotently ensure the workspace exists (`createBidWorkspace`, so
  the BDM can be the first person to ever touch this tender) then set
  `status` (`SUBMITTED` / `WITHDRAWN`) and the fields above. A later
  `markSubmitted` clears a prior not-submitted state.
- Bid-room actions (`backend/actions/bid-room.ts`, no session — the verified
  token is the only credential): **mark submitted** (optional reference +
  receipt upload, stored via `uploadSubmissionReceiptFile`) and **mark not
  submitted** (required reason). Rendered on `/bid-room/[token]`, alongside
  the existing report + pack download.
- `/bids/[id]` shows the submission (who, when, via which channel, reference,
  receipt download) or the not-submitted reason. `WON`/`LOST` were already
  reachable there via the existing `updateWorkspaceStatus` action (Phase 7) —
  no new UI needed for that part.
- `backend/lib/pipeline-analytics.ts` — `getPipelineStats()`: scouted →
  qualified → approved → dispatched → submitted → won, derived entirely from
  rows Phases 12/14/15 already own. A "Pipeline" card on `/dashboard`.
- **Exit criteria met:** a BDM can mark submitted/not-submitted from the
  emailed link with no login; the outcome shows on `/bids/[id]` and the
  dashboard funnel. `tests/unit/bid-room-validation.test.ts`,
  `tests/integration/bid-workspace-submission.test.ts`.

### Phase 16 — Production Hardening — **in progress**

- **RLS audit — done.** Queried the live database's `pg_tables`/`pg_policies`
  directly (not just the migration files) for every one of the 40 application
  tables: every table has RLS enabled; every organization-owned table has the
  uniform member policy; the platform-shared tender tables have the
  authenticated-read policy; the service-role-only tables (`admin_actions`,
  `admin_org_notes`, `source_runs`, `source_items`) correctly have RLS
  enabled with **no** policy for `authenticated`. No gaps found.
- **Rate limiting — done.** `backend/lib/rate-limit.ts` — Postgres-backed
  (counts recent rows, no Redis): `assertAiRateLimit` (10 AI calls/org/min,
  counted against `agent_runs` — now also logged by
  `document-processing.ts`, which previously didn't), `assertDispatchRateLimit`
  (20 dispatches/org/hour), `assertBidPackRateLimit` (10 assemblies/org/min).
  Wired into `runQualification`, `processUploadedDocument`, `dispatchBidPack`,
  `assembleBidPack`; a `RateLimitError` surfaces as a clear message in every
  action that can trigger one. Ingestion and the cron tick are deliberately
  left un-rate-limited — `requirePlatformAdmin()` / `CRON_SECRET` are a
  stronger control than a counter for those.
- **Auth edge cases reviewed** — `requireOrganizationContext` / suspended-org
  redirect, `requirePlatformAdmin`'s env allow-list (case-insensitive,
  trimmed), middleware's narrow protected-prefix list + `getUser()` (not
  `getSession()`) revalidation — all as designed, no changes needed.
- **Timing-safe secret comparison — fixed a real gap.** The cron route
  compared its bearer token with plain `!==` (a timing side-channel against a
  long-lived secret); the magic-link and webhook-signature checks already
  used `timingSafeEqual`. Added `backend/lib/secure-compare.ts` and applied
  it to the cron route.
- File-upload validation (mime allow-list + 25MB cap, sanitized storage
  paths) and the magic-link token review were already covered building
  Phases 3 and 14 respectively — re-checked, no gaps found.
- **Security headers — done.** `frontend/next.config.ts` now sends
  `X-Frame-Options: DENY` + `Content-Security-Policy: frame-ancestors 'none'`
  (clickjacking — the bid room is the one page a third party might try to
  frame), `X-Content-Type-Options: nosniff`, `Referrer-Policy:
  strict-origin-when-cross-origin`, and a `Permissions-Policy` disabling
  camera/mic/geolocation (unused). Deliberately conservative: a full
  script-src CSP needs a nonce wired through the App Router and real
  in-browser verification, so it's left for a follow-up rather than shipped
  unverified.
- **Found and fixed a real capacity ceiling (added 2026-09-11, Phase 18
  work).** The Supabase session pooler's project-wide `pool_size` is **15
  concurrent clients, shared by everything** — the app, every script, every
  test file. With no `connection_limit` set on `DATABASE_URL`, a single
  `PrismaClient` instance's own default pool (~2×CPU cores+1) could alone
  approach or exceed that under a query with several concurrent reads (e.g.
  one `Promise.all` of counts) — hit for real as `EMAXCONNSESSION`, both in a
  live dev-server request and in a test. Fixed by pinning
  `connection_limit=3` on `DATABASE_URL`/`DIRECT_URL` in `.env` and
  `frontend/.env.local` — safe for local dev and small-scale production, but
  **revisit before higher concurrency**: either request a larger `pool_size`
  from Supabase, or move to the transaction pooler (6543) if the earlier
  Prisma-reachability issue with it turns out to be resolved.
- **Not yet:** the credential vault (Phase 11b — no gated source exists to
  need it yet), a performance/index pass against real usage, `agent_runs`/
  `source_runs` cost & latency dashboards, a full script-src CSP, deployment
  hardening beyond the default Vercel + Supabase stack.

### Phase 17 — Billing & Subscriptions — **DONE**

Not part of the original automation roadmap — added after a product/pricing
discussion: three tiers (Starter/Growth/Enterprise), a 14-day full-access
trial, and multi-provider checkout so the platform can actually charge for
itself once onboarding starts.

- Migrations `billing` + `billing_trial_default` (additive): `organizations`
  gains `planTier`, `billingStatus`, `trialEndsAt` (**defaults at the
  database level** to `now() + 14 days` — not just in application code — so
  every org, however created, including pre-Phase-17 test/demo fixtures that
  don't know to set it, starts with a real active trial instead of reading as
  already-expired), `billingProvider`/`billingCustomerId`/
  `billingSubscriptionId`/`currentPeriodEnd`. New table `billing_events` — an
  idempotent (`@@unique([provider, externalId])`) audit trail of every
  webhook delivery. Standard org-member RLS (read-only — only the Prisma
  owner connection, i.e. the webhook routes, writes).
- `backend/lib/billing.ts` — the gate logic. `isBillingActive()` (trialing-
  and-unexpired, or `ACTIVE`, or `PAST_DUE` within a 7-day grace) and tier
  gates `canUseAutoDispatch()` / `canUseGatedSources()` (both wide open while
  trialing — the trial is deliberately full-access — then keyed to
  `planTier` once `ACTIVE`: Starter = review-first + free sources only;
  Growth = full auto-dispatch; Enterprise = adds gated/paid sources, Phase
  11b). `applyBillingEvent()` is the only writer of billing state outside the
  initial trial start — idempotent, resolves the org from the event's
  metadata or, failing that, `billing_customer_id`, and never throws (a
  webhook route always wants its 200 back once the event is durably
  recorded).
- **Three provider adapters**, one interface
  (`backend/lib/billing/types.ts`), picked per checkout by the client:
  - `stripe.ts` — the official SDK (unlike everywhere else in this codebase,
    which prefers `fetch` — Stripe's webhook signature scheme is genuinely
    worth not reimplementing).
  - `paystack.ts` — plain `fetch`; webhook verified via HMAC-SHA512 over the
    raw body using the secret key.
  - `flutterwave.ts` — plain `fetch`; webhook verified via a shared
    `verif-hash` string (not a computed signature — compared with
    `secure-compare.ts`, same helper the cron route uses).
  - All three written against documented provider behavior; **not yet
    exercised against a live account** — verify field names against a real
    webhook once each provider's keys are set (see memory:
    `phase17-billing.md`).
- Gated: `runQualification`'s real-AI path, `processUploadedDocument`,
  `assembleBidPack`, `dispatchBidPack`, and the scouting fan-out sweep (skips
  a billing-inactive org's subscriptions before it ever reaches
  `evaluateMatch`, rather than erroring out per-tender). `BillingError` is a
  known, user-facing error type in every action that can throw one — same
  pattern as `QualificationError`/`RateLimitError`/`BidPackError`.
- `/settings/billing` — plan/trial status, upgrade cards per configured
  provider, a Stripe "Manage billing" portal link, recent billing activity.
  Webhook routes `/api/webhooks/{stripe,paystack,flutterwave}`. Dashboard
  gained a Plan stat + a paused-scouting banner; the admin org page shows
  plan/status/provider/trial-end.
- **Exit criteria met:** a new org gets a real, working 14-day trial with no
  card; every autonomous/AI action refuses cleanly once billing lapses;
  `canUseAutoDispatch`/`canUseGatedSources` correctly gate by tier once paid.
  `tests/unit/billing.test.ts`, `tests/unit/billing-{stripe,paystack,
  flutterwave}.test.ts` (webhook signature verification + event
  normalization, no real provider account needed), `tests/integration/
  billing.test.ts`.
- **Not yet:** actually charging a real customer (no live provider keys set),
  a Stripe/Paystack/Flutterwave live-webhook smoke test, an invite-a-teammate
  flow (seats aren't part of the tier gate, so nothing blocks on this, but
  "multiple users" in the Growth pitch has no UI yet), currency/region-aware
  provider ordering on the pricing page (all configured providers show for
  every plan today). (The one-time onboarding fee and cross-org payments
  monitoring below were the two pieces of this list the user asked for next
  — see Phase 19.)

### Phase 18 — Insights (Analytics) — **DONE**

Also from the pricing discussion — the pipeline funnel (Phase 15) counts
opportunities; this answers what they're *worth*, how often a pursuit wins,
and how much of the loop needed no one's time, which is the renewal-argument
the pricing rests on. No schema change — every number is derived from rows
Phases 12/14/15 already own.

- `backend/domain/analytics/aggregate.ts` — pure, unit-tested shaping:
  `percent()`, `valueByCurrency()` (grouped, never converted across
  currencies — a tender with no stated value or currency is excluded, never
  guessed at), `rankGapCategories()`, `averageHoursBetween()`,
  `rankSourcePerformance()`.
- `backend/lib/analytics.ts` `getAnalyticsSnapshot()` — fetches the rows,
  calls the pure functions. Reuses `getPipelineStats()` (Phase 15) for the
  funnel.
- `/insights` (new nav item) — opportunity value by currency (scouted /
  pursued / won), win rate (won÷dispatched, won÷submitted), time saved
  (`decision = AUTO_DISPATCH` **or** `SKIP` both count as needing no human
  time — only `REVIEW` did; average hours to a decision), the most common
  gap categories (a to-do list for the Capability Passport, CSS-bar ranked —
  no new charting dependency), and per-source match/win counts (the
  evidence for whether a paid aggregator is worth onboarding). Linked from
  the dashboard's Pipeline card.
- **Also found and fixed while building this**: `getAnalyticsSnapshot`'s
  multi-query fan-out is what first surfaced the Supabase connection-pool
  ceiling described above (`EMAXCONNSESSION`) — see that note; fixed at the
  `DATABASE_URL` level, not by reducing this module's query count.
- **Exit criteria met:** `tests/unit/analytics-aggregate.test.ts` (13, pure
  functions), `tests/integration/analytics.test.ts` (6 — value funnel, win
  rate, time-saved split, gap surfacing, source attribution, funnel parity
  with `getPipelineStats`).
- **Not yet:** cross-org/admin-level analytics, currency conversion/
  normalization (deliberately out — never invent an exchange rate), a
  historical trend view (today's snapshot is a point-in-time read, not a
  time series).

### Phase 19 — First-payment fee + admin payments monitoring — **DONE**

Requested directly: a one-time first-payment charge on top of the recurring
monthly fee, a "Billing" entry in the client-facing nav (it was reachable
only via Settings before), and a platform-admin view of every payment made
by every client.

- `billing_events` migration `billing_payment_amounts` (additive):
  `amountMinorUnits`/`currency`, set only when a provider's payload actually
  states a charge amount — null for pure status-transition events.
- `NormalizedBillingEvent` gained a `"payment.succeeded"` type — a charge
  with no status change attached (the first payment, if not already covered
  by `"subscription.activated"`, or any later renewal). This is what makes
  the admin feed show one row per real charge instead of one "activated" row
  that never repeats:
  - Stripe: new `invoice.paid` handling (fires on every successful invoice,
    first and every renewal after).
  - Paystack: split the two events that used to both map to
    `subscription.activated` — `subscription.create` (fires once, still the
    status-flip signal) now stays `subscription.activated`; `charge.success`
    (fires on every charge) now maps to `payment.succeeded` with amount +
    currency.
  - Flutterwave: no equivalent second event exists in its webhook model —
    documented limitation, every successful `charge.completed` still reports
    as `subscription.activated` (amount/currency are still captured
    correctly; ×100 to convert Flutterwave's major-unit `amount` to minor
    units).
- `backend/lib/billing/onboarding-fee.ts` — the one-time-fee env lookup,
  one function per provider. Stripe supports a genuine second one-time
  `Price` as a checkout line item natively, so `createCheckoutSession` just
  adds it — no duplication needed. Paystack/Flutterwave have no such concept:
  the fee is folded into the FIRST charge's total, which means each tier's
  own recurring amount has to be duplicated into env
  (`PAYSTACK_PLAN_AMOUNT_KOBO_*` / `FLUTTERWAVE_PLAN_AMOUNT_*`) alongside the
  flat fee — a real cost of those two providers' APIs, not a workaround
  avoidable in code. Every field defaults to "no fee". Named and priced at
  the user's direction (2026-09-12): the recurring charge is labeled
  "Subscription" (never "maintenance fee" — deliberately avoided, reads as
  "this breaks every month"), the one-off charge is the "Onboarding fee".
  `/settings/billing`'s `PLANS` array carries the advertised figures for
  both — Starter $499, Growth $999, Enterprise "Contact us" — shown
  regardless of whether a provider is connected yet (`hasOnboardingFeeConfigured(tier)`
  gates whether checkout can actually *charge* it, not whether the price is
  *displayed*; the two must be kept in sync once a real provider is wired
  up).
- `applyBillingEvent`'s `applied` return value was redefined: it used to mean
  "attributed to an org AND changed billing status," which under-reported a
  successfully-recorded `payment.succeeded` renewal (no status change, but
  correctly attributed). Now means "attributed to a real organization."
- `/admin/billing` (new, linked from the admin top bar) — platform-wide
  payments view: org counts by billing status, event counts by provider,
  revenue this month summed **by currency** (never combined across
  currencies — same rule as Phase 18's `valueByCurrency`), an unattributed-
  events count (a sign a provider's metadata/customer-id resolution is
  broken), and the last 50 billing events with org, provider, event type,
  amount, and timestamp. Aggregation logic split into a pure, unit-tested
  `backend/domain/billing/admin-summary.ts` (`summarizeRevenueByCurrency`)
  behind the Prisma-fetching `backend/lib/admin-billing.ts`, same split as
  Phase 18's analytics.
- Client-facing nav (`app-header.tsx`) gained a "Plan & Billing" entry — it
  was reachable only from within Settings before.
- **Exit criteria met:** `npx tsc --noEmit` and `npm run lint` clean.
  `tests/unit/billing-onboarding-fee.test.ts` (4), `tests/unit/
  billing-admin-summary.test.ts` (4), plus new cases added to the existing
  `billing-{stripe,paystack,flutterwave}.test.ts` for the new event type and
  amount extraction.
- **Not yet:** the figures are decided ($499 Starter / $999 Growth
  onboarding fee, alongside the existing $399/$1,499 subscriptions) and
  displayed on `/settings/billing`, but the actual per-provider env values
  that make checkout *charge* them (Stripe Price ids, Paystack/Flutterwave
  amounts) are still unset — the user is connecting the payment gateways
  themselves; those values need to be entered to match once each provider's
  Price/Plan objects exist. Also still open: a Flutterwave-side "first
  payment vs renewal" distinction (see above — no clean provider event
  exists for it today); pagination on the admin feed (currently the last 50
  events, no "load more").

---

**Sequencing note:** Phase 6 (demo end-to-end with automated tests) sits
*before* Phase 7 (Bid Workspace) and well before any live source — proving the
qualification engine against the demo dataset is the project's primary success
criterion (`PROJECT_SPEC.md` §"V1 Scope") and must be locked in first. The
automation layer (8–16) is deliberately staged so that each phase leaves a
runnable system: sources before the queue, the queue before the fan-out, the
fan-out before packing, packing before dispatch.
