# Lodestar Tender — Source Registry

The catalogue of procurement sources the scout draws from, with the access
model and integration method for each. See `AUTOMATION_ARCHITECTURE.md` §4 for
the tiering rules and `SOURCE_CONNECTOR_ARCHITECTURE.md` for the connector
contract every entry here must satisfy.

**Tiers**
- **Free / public** — connector built once, auto-subscribed for every
  organization.
- **Gated (free)** — visible only to a pre-qualified vendor; onboarded
  per client (client registers, reaches required status, stores credentials).
- **Paid** — teasers free, full data behind a subscription/API; onboarded
  per client (client's key, or a platform-held subscription).

**Method**
- **API** — official REST/JSON or open-data endpoint. Preferred.
- **Feed** — bulk/RSS/CSV export.
- **HTML** — polite fetch + parse of public pages (robots.txt-aware,
  rate-limited).
- **Browser** — headless Chromium needed (JS-rendered, CAPTCHA, or
  authenticated session).

**Status** is the operator's integration decision, not a technical fact —
`planned`, `approved`, `deferred`, or `declined` (with reason).

---

## 1. Nigeria — Core Oil & Gas

| Source | URL | Tier | Method | Notes |
|---|---|---|---|---|
| NipeX — current opportunities | `nipexmain2.nipex-ng.com/supplier-notice/current-opportunities/` | Free/public (teaser) | HTML / Browser | The upstream mandatory portal. Public list of open adverts is browsable; full documents + bidding require NJQS "LIVE" vendor status. |
| NipeX NJQS — vendor status / gated tender detail | `vendors.nipex-ng.com` | Gated (free) | Browser | Per-client onboarding: client must be NJQS-registered and LIVE; credentials → vault. |
| NCDMB — adverts | `ncdmb.gov.ng/adverts/` | Free/public | HTML | Tenders, notices, training offers. Simple page. |
| NOGIC JQS | `nogicjqs.gov.ng` | Gated (free) | Browser | NCDMB Nigerian-content certification + contractor competency search. Used for **product/service code verification**, not just tenders. |
| NLNG supplier portal (eVIMS) | `supplierportal.nlng.com` | Gated (free) | Browser | Registered NLNG suppliers only. |
| NNPC Limited | — | — | — | No standalone tender site; routes through NipeX. Covered by the NipeX connector. |
| Shell / SNEPCo | via SAP Ariba | Gated (free) | Browser | Cross-posted to NipeX; the Ariba route is per-client. |
| TotalEnergies / TEPNG | SAP Business Network ("Sequana") | Gated (free) | Browser | As above. |
| Chevron / ExxonMobil / NAOC / Renaissance / Seplat | — | — | — | Advertise through NipeX; no separate connector. |

## 2. Nigeria — General Government Procurement

| Source | URL | Tier | Method | Notes |
|---|---|---|---|---|
| BPP — Bureau of Public Procurement | `bpp.gov.ng` | Free/public | HTML | Parent site. |
| NOCOPO — Nigeria Open Contracting Portal | `nocopo.bpp.gov.ng` | Free/public | Browser / data request | 700+ MDAs. Named "open contracting" but **no public OCDS API or data endpoint is exposed** (confirmed 2026-09-10) — the site is a portal UI. Needs a browser connector or a formal data-sharing request to BPP. |
| Federal Contractors Portal | `federalcontractors.bpp.gov.ng` | Gated (free) | Browser | Contractor registration; relevant for "registered to bid" readiness. |

## 3. Nigeria — Private Aggregators (include an O&G category)

| Source | URL | Tier | Method | Notes |
|---|---|---|---|---|
| Jorpex | `jorpex.com` | Paid (from $49/mo) | API / Feed | Aggregates NOCOPO, NNPC, state boards, multilateral. Slack/email delivery. Per-client subscription. |
| eTenders Nigeria | `etenders.com.ng` | Paid | HTML / API | |
| eBid Nigeria | `ebid.com.ng` | Paid | HTML | |
| NigerianTenders.com | `nigeriantenders.com` | Paid | HTML | |
| PublicProcurement.ng | `publicprocurement.ng` | Paid | HTML | |
| NGO Procurement (ngojobsite) | `procurement.ngojobsite.com` | Free/public | HTML | NGO/development tenders. |
| Naijabusiness.com.ng | `naijabusiness.com.ng/topic/oil-gas-tenders` | Free/public | HTML | |

## 4. International — Global O&G Aggregators

| Source | URL | Tier | Method | Notes |
|---|---|---|---|---|
| GlobalTenders | `globaltenders.com/global-tenders-oil-gas` | Paid | API | |
| TendersInfo | `tendersinfo.com/global-oil-and-gas-tenders.php` | Paid | API | Broad global O&G coverage. |
| TendersOnTime | `tendersontime.com/oil-and-gas-tenders` | Paid | API | |
| BidDetail | `biddetail.com/global-tenders/oil-and-gas-tenders` | Paid | HTML/API | |
| GlobalsBid | `globalsbid.com/oil-and-gas-tenders-and-procurement-opportunities` | Paid | HTML | |
| Energy Dais | `energydais.com/global-oil-and-gas-tenders` | Paid | API | ADNOC, Aramco, ONGC, IOCL archives + full documents. Strong O&G fit. |
| dgMarket | `dgmarket.com` | Free/public | HTML/Feed | Donor/NGO-oriented, mostly open. Partial exception to the paid pattern. |
| Tender247 / Tender18 / TenderDetail / NationalTenders | — | Paid | HTML/API | India/Asia-heavy, include O&G. Lower priority. |

## 5. Multilateral / Development Finance

| Source | URL | Tier | Method | Notes |
|---|---|---|---|---|
| UNGM — UN Global Marketplace | `ungm.org` | Gated (free) | API | Tender notices API for registered vendors. UN Secretariat solicitations also via SAP Ariba (`service.ariba.com/Supplier.aw`). |
| World Bank | `projects.worldbank.org` | Free/public | API | Projects & Operations API + procurement notices. |
| African Development Bank | `afdb.org` | Free/public | HTML/Feed | Procurement opportunities section. |
| IFC | `ifc.org` | Free/public | HTML | Corporate procurement. |
| TED — Tenders Electronic Daily | `ted.europa.eu` | Free/public | API / Feed | Official EU-wide portal, energy-sector notices. Bulk + API. |
| SAM.gov | `sam.gov` | Free/public | API | US federal; DOE / military fuel & energy. API key required (free). |

## 6. NOC / IOC Vendor Portals (per-client, best-effort)

All **gated (free)**, all **Browser**, all onboarded per client. Tenders are
shown only to pre-qualified vendors; automating the authenticated session may
conflict with the portal's terms — flag for the operator's decision per
portal.

| Source | Portal |
|---|---|
| ADNOC (UAE) | `supplierhub.adnoc.ae` (SAP Ariba) |
| Saudi Aramco | e-Marketplace / SAP Ariba |
| QatarEnergy | own "Business/Suppliers" section |
| Petrobras / Transpetro (Brazil) | Petronect (`petronect.com.br`), Canal Fornecedor, `transparencia.petrobras.com.br/licitacoes` |
| ONGC (India) | `vendor.ongc.co.in` — enlisted vendors only |
| GAIL / IOCL / HPCL / Oil India | company e-procurement sites (often `eprocure.gov.in`-linked) |
| Equinor / bp / Chevron / ExxonMobil / Petronas | SAP Ariba / SAP Business Network supplier pages |
| MEED Projects, Zawya | Paid Gulf project/tender intelligence — API, per-client subscription |

---

## 7. Resolved Acquisition Strategy

Decided 2026-09-09, so Phase 8+ starts without open questions.

| Tier | Method | Credentials | Browser | ToS risk |
|---|---|---|---|---|
| Free — has an API (NOCOPO, TED, UNGM, World Bank, SAM.gov) | Official API/feed | none (some need a free API key, platform-held) | no | none |
| Free — HTML only (NipeX adverts, NCDMB, AfDB, dgMarket, NGO/Naijabusiness) | Polite fetch — robots.txt, ≥3s between requests, real User-Agent + contact, hard cache, public list + public PDFs only | none | only if JS-rendered (NipeX — Phase 8 spike) | low |
| **Vendor portals** (Ariba, ADNOC Hub, ONGC, NLNG, Shell, Total, Petronect, Aramco, QatarEnergy) | **Inbound email.** Each org gets a dedicated address (`<org-slug>@inbound.<domain>`); the client adds it as a notification recipient on every portal they're registered on. The platform parses invitation emails (title, buyer, deadline, links, attachments) into the pipeline. | none | no | none |
| Paid aggregators (Jorpex, TendersInfo, Energy Dais, GlobalTenders, TendersOnTime, BidDetail, GlobalsBid, MEED, Zawya, Tender247…) | **Paid API/feed only — never scrape. Client brings their own key** — no platform-held subscription. A client who wants an aggregator adds their subscription key during source onboarding; it goes in the vault and the scout runs that aggregator for that client only. | client's own aggregator API key (vault) | no | none (licensed) |
| Stored-credential portal login | Encrypted vault + Playwright worker | password/TOTP in vault | yes | grey — **opt-in, per client, per portal, with explicit risk acceptance.** Last resort only, where a portal has no notification-email option. |

**Inbound email** is the primary mechanism for everything vendor-portal-shaped:
it needs no credentials, no browser, and does not touch any portal's terms. A
new connector kind `InboundEmailConnector` parses `{ from, subject, body,
attachments }` into `RawSourceItem`s; the same normalize → dedupe → store path
follows. Requires an inbound-email provider (`INBOUND_EMAIL_DOMAIN`, e.g. via
the same Resend/SES account used for dispatch, or Cloudflare Email Routing).

**Headless browser** (Playwright worker pool — Browserless or self-hosted on the
Phase-10 queue) is used only where a free HTML source turns out to be
JS-rendered, or for an explicitly opted-in credential-login portal. A connector
declares `usesBrowser: true`; failure is isolated to that connector and shown by
`checkHealth()`.

## 8. Connector Order

1. **UK Find a Tender (OCDS)** — **BUILT (Phase 8).** `find-a-tender`. Live-verified.
2. **UK Contracts Finder (OCDS)** — **BUILT (Phase 9).** `contracts-finder`.
   Live-verified; a config of the shared `OcdsConnector` — future OCDS portals
   are config entries, not code.
3. **NOCOPO** (`nocopo.bpp.gov.ng`) — **no public API confirmed** (site is a
   portal UI only). Needs a browser connector or a data-sharing request.
3. **TED** (`api.ted.europa.eu/v3`) — **BUILT.** `ted`. All sectors, EU/EEA,
   keyless POST search API, live-verified 2026-09-25. **Listing only**: the
   notice PDF/XML pages on `ted.europa.eu` sit behind an AWS WAF human-verification
   challenge (HTTP 405 to any non-browser client) and the API has no document
   endpoint, so tenders arrive with title / description / buyer / value / deadline
   / CPV sector and a link, but no downloadable documents — requirement extraction
   has nothing to read until a person opens the notice. Not worked around.
4. **UNGM** / **World Bank** / **AfDB** — official APIs / OCDS, Phase 9.
5. **NipeX current-opportunities** — the Nigeria O&G anchor; HTML/JS, needs a
   browser worker. Phase 9.
6. **NCDMB adverts** — small, public, O&G-specific HTML. Phase 9.
7. **Inbound-email connector** — one build covers every vendor portal
   (Ariba/ADNOC/ONGC/NLNG/Shell/Total…). Phase 9.
8. **NOGIC JQS** — for product/service **code verification** feeding
   `company_product_codes` (Phase 11), as much as for tenders.
9. **Paid aggregators** — client-BYO key only (Jorpex / Energy Dais /
   TendersInfo / …); onboarding UI accepts the client's key. Phase 11+.
10. **Stored-credential portal login** — last, per client, per operator risk
    sign-off, only where no notification email exists.

---

## 9. All-Sector Sources (added 2026-09-25)

The product is sector-agnostic: a caterer, a school, a printer and a pipeline
contractor draw from the same pool. Sections 1–8 lead with oil & gas because
that was the first vertical; nothing in the pipeline is O&G-specific any more
(connectors keep every sector and tag it via `backend/sources/ocds/sector.ts`;
`matchesEnergyCpv` is an opt-in filter). What relevance means for a given client
is decided by their scouting profile, not by the source.

Access claims below come from a third-party source list and are **unverified**
unless marked. Verify each endpoint before a connector spike — the earlier
NOCOPO "API" turned out not to exist.

### Nigeria — states

Every state has its own procurement body, but portal quality varies widely and
the per-state working/not-working ratings we were handed are unverified, so they
are deliberately **not** recorded here. Approach: one config-driven HTML/browser
connector per state that has a machine-readable public notice list; states
without one fall back to NOCOPO (which federates FG MDAs only) or a paid
aggregator. Confirmed-by-name so far, all unverified for access: Lagos
(`lagosppa.gov.ng`), Kaduna (`kadppa.kdsg.gov.ng`), Ekiti, Kogi, Ondo, Osun.
Newspaper notice pages (Guardian, Punch, ThisDay) are a **last resort** — copyright
and ToS review needed first.

### Global — official feeds, no scraping

| Source | Method | Notes |
|---|---|---|
| TED (EU) | API | **Built** (`ted`) — see §8. Listing only; documents are bot-walled. |
| World Bank | API | Procurement notices API. |
| UNGM | API (registered vendors) | As §5. |
| SAM.gov (US) | API | Free key, platform-held. |
| AusTender (`tenders.gov.au`) | OCDS API | Believed to publish OCDS — likely a config of `OcdsConnector`; verify. |
| South Africa eTenders (`etenders.gov.za`) | OCDS API | Believed to publish OCDS; verify. |
| CanadaBuys | Open-data feeds | Bulk notice files; verify format. |
| Kenya (`tenders.go.ke`), Ghana | HTML | No known API. |
| India GeM / eprocure | Browser | CAPTCHA-gated; ToS risk — defer. |
| UAE, Saudi Etimad | Browser / gated | Registration-gated; per-client at best. |

### Not adopting

- **Bypassing portal logins or CAPTCHAs** (offered in the source list). Conflicts
  with §7: credentialed automation is opt-in, per client, with explicit risk
  acceptance, and never circumvents access controls.
- **Scraping paid aggregators** (GlobalTenders, TendersInfo, BidDetail,
  Tender247…). §7 stands: client-BYO paid API key only.
- **Separate workers per source group.** The Phase 10 tick already runs each
  source on its own cadence and feeds one tender pool; parallel worker pools,
  Celery/Redis, MongoDB and a vector DB add cost for no gain at this stage
  (pre-launch: no paid infra).

**Suggested next connectors, in order:** ~~TED~~ (built) → World Bank →
AusTender / SA eTenders (if OCDS is confirmed) → UNGM.
