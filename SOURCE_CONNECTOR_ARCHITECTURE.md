# Lodestar Tender — Source Connector Architecture

This document defines the contract source integrations satisfy, so that adding a source never requires touching qualification logic, company logic, the database core, or the UI.

**Status (Phase 8–9):** the interface, the ingestion pipeline, and a generic OCDS connector are implemented.
- `backend/sources/types.ts` — the `TenderSourceConnector` interface.
- `backend/sources/ocds/` — the shared Open Contracting (OCDS) layer: `map.ts` (release → `NormalizedTender`), `sector.ts` (CPV division → sector tag), `cpv.ts` (opt-in energy/O&G filter — connectors keep all sectors by default), and **`connector.ts` — `OcdsConnector`, a config-driven connector for any portal that publishes OCDS release packages over a paginated GET API.** Adding such a source is a config entry, not new code.
- `backend/sources/connectors/ted/` — EU TED, a bespoke connector (not OCDS): keyless search API, all sectors, listing only (documents are behind a bot challenge). `map.ts` is the pure notice → `NormalizedTender` mapper; `backend/sources/countries.ts` turns TED's ISO alpha-3 into the English country names scouting profiles match on.
- `backend/sources/connectors/find-a-tender/` and `.../contracts-finder/` — the two live connectors (UK Find a Tender, UK Contracts Finder), each a ~15-line config of `OcdsConnector`, both verified against their real APIs.
- `backend/sources/registry.ts` — the key→connector lookup.
- `backend/lib/tender-ingestion.ts` — the pipeline. `backend/lib/source-health.ts` + `/admin/sources` — operator health view.

Run manually via `npm run ingest -- <sourceKey>` or `/admin/sources` until scheduling lands (Phase 10).

## 1. The Connector Interface

`src/sources/types.ts` defines `TenderSourceConnector`:

```ts
interface TenderSourceConnector {
  readonly sourceId: string;
  readonly sourceName: string;

  fetchOpportunities(since?: string): Promise<RawSourceItem[]>;
  fetchTenderDetail(externalId: string): Promise<RawSourceItem>;
  fetchTenderDocuments(externalId: string): Promise<TenderDocument[]>;
  normalize(item: RawSourceItem): Promise<Tender>;
  checkHealth(): Promise<SourceHealth>;
}
```

Each future source — international aggregators (GlobalTenders, TendersInfo, TendersOnTime, GlobalsBid, Bidding Source), international organizations (UNGM, World Bank, African Development Bank, ...), and national/official portals (NipeX, government procurement portals, sector-specific sources) — is implemented as one module under `src/sources/connectors/<source-key>/` satisfying this interface. Nothing else in the codebase imports a source-specific module directly; everything downstream depends only on `Tender` (`src/domain/tenders/types.ts`).

## 2. Ingestion Pipeline

```
SOURCE → INGEST → NORMALIZE → VALIDATE → DEDUPLICATE → STORE →
DOCUMENT PROCESSING → REQUIREMENT EXTRACTION → QUALIFICATION
```

| Step | Responsibility | Where |
|---|---|---|
| **Ingest** | Call `fetchOpportunities()`/`fetchTenderDetail()` on the connector; record a `source_runs` row for the invocation and a `source_items` row per raw item fetched (kept for audit regardless of outcome). | Ingestion job (future: scheduled/queued; Phase 0 defines the shape only) |
| **Normalize** | Call `connector.normalize(item)` to produce a `Tender` in the platform's common shape. Source-specific field mapping lives entirely inside the connector. | Connector |
| **Validate** | Structural validation (required fields present, dates parseable, currency codes valid) before anything is written. | Ingestion job, using shared validators |
| **Deduplicate** | Look up an existing `tenders` row by `(source_id, source_identifier)`; update in place if found, insert if not. This is the single source of truth for "have we seen this tender before" — no separate dedup table. | Ingestion job, against `tenders` |
| **Store** | Persist the `Tender` and its `tender_documents` (fetched via `fetchTenderDocuments()`). | Ingestion job |
| **Document processing** | Hand off newly stored `tender_documents` to Document Intelligence for classification/extraction (of the notice/spec content, not company evidence — see `ARCHITECTURE.md` §6). | `src/ai/document-intelligence` (Phase 3+) |
| **Requirement extraction** | Tender Intelligence reads the tender's documents (authoritative) — not just an aggregator's short summary — and produces `tender_requirements`/`tender_evaluation_criteria`. | `src/ai/tender-intelligence` (Phase 4+) |
| **Qualification** | Unrelated to source at this point — the qualification engine consumes `Tender` + `TenderRequirement`, never source-specific data. | `src/domain/qualification` (Phase 5+) |

## 3. Normalization

`normalize()` is the only place source-specific knowledge is allowed to exist. It must map the source's native fields onto the full `Tender` shape, including the structured eligibility fields (`foreignBidderPolicy`, `localPartnerRequired`, `consortiumRulesDescription`) — even if that requires the connector to call Tender Intelligence itself on the source's raw eligibility text during normalization, rather than leaving those fields `UNKNOWN` when better information is available in the raw payload.

**No connector, and no code outside a connector, may special-case a country or jurisdiction's requirements.** A connector for a Nigerian source is not permitted to assume Nigerian requirements apply to everything it returns; requirement content always comes from the tender's own text via Tender Intelligence, never from which connector fetched it. See `PROJECT_SPEC.md` §"Non-goals".

## 4. Deduplication

Deduplication key: `(tenders.source_id, tenders.source_identifier)`, enforced by a unique constraint (`DATABASE_DESIGN.md` §6). A connector must supply a stable `source_identifier` for the same opportunity across runs — if the source itself has no stable id, the connector is responsible for deriving one deterministically (e.g. a hash of buyer + title + publication date) and documenting how, since an unstable identifier would silently create duplicate tenders on every run.

Cross-source duplicates (the same real-world tender appearing on two aggregators) are explicitly **out of scope** for Phase 0–era connectors; if it becomes a problem once multiple sources are live, it is solved as a separate matching step downstream of per-source dedup, not by weakening the per-source key.

## 5. Source Health

`checkHealth()` reports whether a source is currently reachable and behaving as expected (e.g. a smoke-test fetch succeeds, response shape matches expectations). Combined with `source_runs.status`/`error_message`, this gives an operational view of source reliability without requiring a separate monitoring system in Phase 0. A future "Monitoring + Notifications" phase (`IMPLEMENTATION_PLAN.md` Phase 10) can alert on repeated health-check failures or stale `source_runs`.

## 6. Source-Specific Metadata

Anything a source provides that doesn't map onto the common `Tender` shape is preserved, not discarded: `source_items.raw_payload` (JSON) always holds the untouched original fetch, so a future normalization improvement can be replayed against history without re-fetching from the source. Connector-specific configuration (API keys, base URLs, rate limits) lives in that connector's own module and its own environment variables — never in shared code.

## 7. Adding a New Source — Checklist

1. Create `src/sources/connectors/<source-key>/`.
2. Implement `TenderSourceConnector`.
3. Add a `tender_sources` row (`source_key`, `name`, `base_url`).
4. Confirm `normalize()` populates the structured eligibility fields whenever the raw payload contains enough information to do so.
5. Add integration tests asserting: normalization produces a valid `Tender`, dedup key is stable across two fetches of the same opportunity, and `checkHealth()` correctly reports an unreachable source as unhealthy.
6. No change to `src/domain/*`, `src/ai/*`, or `src/app/*` should be required.
