# SOURCES.md — the data contract
_Every dataset the pipeline produces: where it comes from, what it means, its schema, cadence, gates, and how the site should envelop it. The site agent builds against THIS file; prep_data.py should never have to guess._

## Family A — facilities & entities (existing, unchanged)
| File | Grain | Key | Cadence |
|---|---|---|---|
| `facilities_master.csv` | one facility | name\|city\|country | daily (refresh.py) |
| `admin_sites.csv`, `healingmaps_sites.csv`, `oregon_licenses.csv` | source layers behind master | — | weekly |
| `industry_entities.csv` | one org, `layer` faceted | name | monthly (/agarikon-sweep) |

## Family B — research (existing + one upgrade)
| File | Grain | Key | Cadence |
|---|---|---|---|
| `studies.csv` | one trial (`relevance` filtered site-side) | nct_id | 6h |
| `research_locations.csv` | one trial-site | nct_id+facility | 6h |
| `researchers.csv` | one author (PubMed, ≥3 papers) | author | monthly |
| **`researchers_openalex.csv`** (new) | one author with citation metrics + ORCID + institution | openalex_id | monthly |
| **`coauthor_edges.csv`** (new) | one collaboration edge (author-pair, weight = co-papers) | a\|b | monthly |
| **`publications.csv`** (new 2026-09-04) | one scholarly work: title, year, citations, first/senior author, OpenAlex link | openalex_id | monthly |

## Publications re-scoped to psychedelic medicine, 2026-10-02
- **Corpus 45,628 -> 33,166.** The gate splits on where the subject appears. A psychedelic in the TITLE is admitted unless the title frames it as humanities, ethnography, taxonomy or cultivation. A psychedelic mentioned ONLY in the abstract needs a clinical title. Every piece of junk in the precision audit was abstract-only; the title-subject papers were never the problem. Drops are recorded in EXCLUSION_LOG.md as three "Publications: scope" stages.
- The first fix demanded a clinical word in every title and deleted Griffiths 2006 (1,718 citations) — 85% of what it dropped had no abstract in OpenAlex. Caught mid-run; both the junk set and the landmark set are now regression tests.
- `--rescope` starts the harvest from an empty corpus (a re-walk keeps held works, so a stricter gate would never reach them) and is the only way past the collapse guard. It is a deliberate, logged human decision.
- **New per-work fields:** pmid, pmcid, date, journal, language, fwci, oa_status, oa_pdf (free full text where it exists), oa_url, oa_license, landing_page, retracted, funders (`id|name`), corresponding_author_ids, admitted_by (which query let it in), oa_topic, oa_subfield. OpenAlex charges per request, not per field, so these cost nothing.
- **`paper_authors.csv`** — one row per (work, author): OpenAlex author id, ORCID, position, is_corresponding, institution, institution id, ROR, country, email. The foundation of the entity graph. Previously all of this was fetched and discarded at emit.
- **`abstracts.jsonl`** — reconstructed abstracts, append-only sidecar. For detail shards only; never the list payload.
- **`paper_author_emails.csv`** (`harvest_pubmed_emails.py`) — corresponding-author emails from PubMed, which keeps them inside a specific author's affiliation where OpenAlex strips them. Matched to the OpenAlex author on the same paper by surname then position. `name_match=CHECK` flags addresses whose local part does not plausibly belong to that person (initials-style addresses such as jqs@ for Jane Q. Smith pass; generic info@ or editor@ do not). Refreshes on the 30-day OpenAlex window.
- **Emails are internal.** paper_authors.csv and paper_author_emails.csv are deliberately NOT in sync_data_repo.py's allowlist. Publication emails feed outreach; they are not republished in bulk.

## Quarantine fixed, 2026-10-02
`quarantine.csv` was append-only with a header written once at creation. Every refresh re-added the same rows (35 distinct organizations appeared ~14 times each), and when `basis_source` was added to facilities_master on 2026-09-11 every later row became one field wider than the header, shifting study titles into `qc_reason`. The published count read 1,332 and grew daily. It is now rewritten each run as the CURRENT quarantine, atomically, carrying each row's first-seen date; the 36 distinct historical rows are archived in `quarantine_archive_2026-10-02.csv`. Validation currently quarantines nothing because non-facility organizations are now caught earlier, at merge — which EXCLUSION_LOG.md now records as its own "Facilities: merge" stages.

## Scope: how complete is this index? (measured 2026-10-01)
Full survey in `research/scope-survey-2026-10.md`. Headlines: publications ~95%+ complete
after the rebuild (45,619 works, 86.1% with a DOI); trials ~78% (classic psychedelics ~86%,
psychiatric ketamine ~76%, the gap concentrated in ChiCTR which blocks all probing).
The English-language share is real, not an artifact — Brazilian psychedelic researchers
publish in English, and every accessible regional source combined adds 3-5%. The one real
publication blind spot is Japan, via JaLC-not-Crossref DOI registration.

**Contamination rule that keeps our trial data clean, do not change it:** `harvest_ctgov.py`
queries `query.intr` (the intervention field), never full text. Full-text term matching
over-reports several-fold — "DMT" matches diabetes mellitus type 2, "LSD" matches Least
Significant Difference, and exclusion-criteria boilerplate makes one esketamine trial match
MDMA, LSD and hallucinogen at once. Any future registry ingestion must filter on
title/intervention/drug fields.

**Registries that forbid republication** (do not ingest): WHO ICTRP, PACTR, jRCT, Scopus,
Web of Science, Dimensions, ProQuest, CNKI, Wanfang, eLibrary.ru. Clean: ISRCTN (CC0 on
post-2019 metadata), CTIS/EUCTR (EMA, commercial OK with acknowledgement), ANZCTR
(attribution + currency), theses.fr (Etalab v2), OpenNeuro/Dryad/DOAJ/Crossref (CC0).

## OpenAlex budget and pacing, 2026-10-01
- **Authenticated limit, measured from response headers: 10,000 credits/day = $1/day, at a flat 10 credits ($0.001) per request — so 1,000 requests/day.** Anonymous access is ~100/day shared per IP, which is why the key is required. Headers to watch: `x-ratelimit-remaining`, `x-ratelimit-remaining-usd`, `x-ratelimit-reset`.
- **Cost does not scale with page size** (verified: per-page 1, 100 and 200 all cost 10 credits). `per-page` is therefore 200, the maximum, which halves the request count for free.
- **The `ambiguous` query filters server-side.** Bare `"LSD" OR "DMT"` is 59,995 works, ~92% agronomy (Least Significant Difference) and MS research (disease-modifying therapy). Pairing it with a psychedelic term in the query itself cuts it to 4,889 before anything is fetched. The local relevance gate still runs on what returns; this just stops us paying to download what the gate would discard.
- **A full rebuild is ~336 requests (3,360 credits), 34% of one day.** Before these changes it was 1,219 requests / 12,190 credits, which exceeded the daily budget and could not have completed in a single day.
- **A completed run used to make the next one a no-op.** The per-query `complete` flags persist in `oa_state.json`, so the 30-day scheduled run would load state, skip all five queries and exit 0 with the corpus unchanged. `harvest()` now clears those flags when it finds a completed state and re-walks; works already held are skipped by id, so a re-walk pays only for paging and the corpus stays additive. The 30-day schedule is unchanged and comfortably affordable.

## Guardrails, 2026-10-01
- **Every CSV-writing harvester now writes through `harvest_guard.safe_write()`** (9 of 9; previously 2). It replaces `open(path, "w", newline="")` and adds two protections. Rows go to a `.tmp` and only replace the target at the end, so a crash or a kill mid-write cannot truncate a good file. And a write producing under 50% of the data rows already on disk raises `HarvestCollapse` and discards the `.tmp`, because a harvest that collapses is a broken run, not a shrinking field. Override per call with `safe_write(path, threshold=...)`; `allow_empty=True` only where emptiness is a real state, never for a scrape.
- **Why it is per-write and not per-source:** on 2026-09-26 the laptop was offline, every LegiScan request failed with a DNS error, the retry loop gave up quietly, and a header-only CSV was written over 1,892 bills with exit 0. A DNS failure means *we* are offline, and any harvester is free to misread that as "the source returned nothing". The guard sits at the only place every source has in common.
- **Test coverage:** `test_pipeline.py` fails if any CSV-writing harvester lacks the guard, and exercises collapse-refusal, file-intactness and tmp-cleanup directly. 29 assertions total.
- **Schedules: nothing fires more often than daily.** CT.gov (`com.emayne.psychedelic-harvest`) moved from a 6-hour `StartInterval` to 86400s, and its staleness window in refresh.py moved from 0.25d to 1d to match — the window and the schedule have to move together or the alarm cries constantly. `healingmaps-harvest` and `pubmed-harvest` are `RunAtLoad` one-shots with no interval and were already effectively daily via refresh.py. Windows above 24h (openalex 30d, openpayments/partb/quotas 90d, legiscan 7d) are unchanged.

## Guardrails, 2026-09-12
- **`test_pipeline.py` gates publication.** 25 regression assertions over the merge rules and the served JSON. refresh.py runs it after validate and **returns without publishing if it fails**, writing TESTS_FAILING.txt. Every assertion exists because something broke: NIMH template leak stripped but Janssen/Cybin sponsor contacts kept, no funder/advocacy/regulator/legal/developer org served as a clinic, duplicate surplus bounded, no template string labelled operator_stated, no province in the country column, anesthesia rows absent from studies.json, the unnamed/psy flags present, meta counts matching served rows. Run `python3 test_pipeline.py -v` after touching merge_master, validate, prep_data or a harvester.
- **Three alarms, one report.** A failed publish, a failed test run, and a dataset past twice its refresh window each write a marker file, and ticker.py prints all of them at the top of `research/MAINTENANCE_DUE.md`. They exist because a job that runs and silently fails is indistinguishable from one that works: the publish step failed for two days in exactly that way.
- **Guard scoping matters.** The shared-contact resource-domain rule applies to every row; the spread heuristic applies only to non-research rows, because on a trial site the facility is the hospital and the contact is the sponsor, so "unrelated domain across many names" is the correct shape there, not a leak.

## Data integrity pass, 2026-09-11 (changed the contract)
- **`basis_source` column** on facilities_master: `operator_stated` | `jurisdiction_template` | `license_record` | empty. Flows to clinics/retreats JSON as `basis_src`. Copy may claim a quote ONLY where it reads `operator_stated`; the template values come from the merge-time (country, drug-class) lookup, so dozens of rows share one string.
- **Non-treatment industry orgs are no longer clinics.** merge_master cross-references industry_entities; a pure `admin` row whose name matches a non-treatment layer (legal, funder, advocacy, association, regulator, tech, developer, supplier, training) with no drugs and no address/phone becomes `kind=industry`, and prep_data drops those from clinics.json. Clinic total fell 2,564 → 2,413.
- **Second-pass dedupe.** After the strict `name|city|country` key, rows sharing name+country merge when one city is blank or contained in the other. Never merges two different real cities. Every merge logged to `dedupe_merges.csv` (65 on the first run, 0 of them blank-on-both-sides).
- **Shared-contact guard.** A contact value is stripped only on treatment rows, only when its domain is a known resource domain or it spans ≥8 distinct operators whose names share no token with it. Sponsor central contacts (Janssen 53 sites, Cybin 69) and chain domains survive by design. Stripped values logged to `shared_contacts_stripped.csv`. First run removed the NIMH inquiry address and page from 602 clinic rows.
- **Country sanity.** A country string over 28 chars or containing a dash/ampersand/slash is rejected to empty and flagged in notes.
- **`exclusions.json`** ships next to EXCLUSION_LOG.md and is copied to `public/data/`. Site copy MUST read purge numbers from it rather than hardcoding; it also carries `residual_anesthesia_served` (165), because the removal count is not a claim the category is empty.
- **meta.json additions:** `countries_definition` (one definition, ends the 68/75/76 disagreement), `distinct_operators` (clinics 1,823 of 2,413 rows; retreats 205 of 280), `legal_basis` breakdown by provenance.
- **Papers `compounds`** (9 classes) alongside `topics`. Title-derived today (54%); the harvester now classifies from title+abstract, so coverage rises at the next monthly run without a site change.

**Provenance note:** `publications.csv` carries `topics` (up to 3, from title+abstract at harvest, ~79% coverage) and `type` (article/review/preprint). prep_data.py READS these columns — it must not recompute topics (doing so drops coverage to title-only ~48%). One source of truth per field.
**Site adoption (papers well):** `publications.csv` powers a "Papers" index — default sort citations desc, facet by year; join first/senior author names to researcher cards ("4 papers · 1 registered trial" is the display pattern; papers ≠ trials, label both). Link out to OpenAlex per row. Per-paper pages stay ON HOLD.
**Site adoption:** join `researchers_openalex.csv` to `researchers.csv` on normalized name (both carry it); prefer OpenAlex citation counts for display, PubMed counts for the ranking they already ship. `coauthor_edges.csv` powers a network view when wanted — nodes are researchers, weight is shared papers.

## Family C — market signals (ALL NEW: the federal-data layer)
These are **metric time-series and reference tables**, not facilities. They feed dashboard tiles, Census charts, and enrichment joins — never the facility index itself.

| File | What it is | Grain | Cadence | Source |
|---|---|---|---|---|
| `spravato_providers.csv` | every provider who billed Medicare for Spravato observation (HCPCS G2082/G2083): NPI, name, address, services count, avg payment | provider×code×year | quarterly | data.cms.gov Part B by Provider & Service |
| `openpayments_spravato.csv` | industry payments to physicians tied to SPRAVATO: recipient, amount, nature (speaker/consulting/food), payer | payment-year aggregate per recipient | quarterly | CMS Open Payments API |
| `dea_quotas.csv` | DEA aggregate production quotas for psilocybin/psilocyn/MDMA/ibogaine etc. by year, with Federal Register doc link | substance×year | quarterly watch | federalregister.gov API |
| `legiscan_bills.csv` | every state psychedelic bill: state, bill, title, status, last action, sponsors, url | bill | weekly (needs free API key — see below) | LegiScan API |

**Site adoption contracts:**
- `spravato_providers.csv` ↔ facilities: fuzzy-join on (name, city, state) to badge clinic rows "Medicare-verified Spravato provider (N services, YYYY)". Never invent a facility from it; unmatched providers can seed a "claims-verified, unlisted" review queue. **Scope caveat (display it):** this is Medicare fee-for-service claims only — commercially insured Spravato volume does not appear, so it's a positive verification signal, never a completeness claim (~76 NPIs vs 7,000+ REMS-certified centers).
- `openpayments_spravato.csv` is people-grade data from a federal transparency program; display only aggregates or per-recipient totals with the program cited as source. Powers a "top Spravato KOLs" table in the Census.
- `dea_quotas.csv` is a headline chart: quota trend per substance per year. One row per substance-year; `doc_url` is the provenance link.
- `legiscan_bills.csv` powers the legal map's PENDING tier + changes-feed entries ("HB123 passed committee"). `status` uses LegiScan's numeric status; `status_text` is the display string.

**Keys & joins summary:** NPI is the canonical provider key where present. Facility joins are always fuzzy (name+city+state) and must be flagged `join_confidence` if materialized.

## Gates for Family C (same doctrine: classify at ingest, quarantine with reason)
- Part B: HCPCS whitelist only (G2082, G2083); rows with other codes never enter.
- Open Payments: product-name filter server-side (SPRAVATO variants); drop rows whose product field doesn't contain it after fetch (belt+suspenders).
- Quotas: regex extraction of substance amounts is best-effort — rows carry `extraction` = parsed|manual_needed; never publish a parsed number without its doc_url.
- LegiScan: keyword query (psilocybin, psychedelic, ibogaine, MDMA, entheogen); bills matching only "ketamine" excluded (too noisy — anesthesia formulary bills). **Relevance gate**: full-text matches below LegiScan relevance 50 dropped (kills appropriations acts and hemp bills that mention a keyword once); every row carries `relevance` + `match_quality` (title | strong ≥75 | contextual 50-74) — site should default to title+strong, offer contextual behind a toggle.
- Quotas: sequential mid-year revisions are normal — `superseded` column flags earlier docs per (substance, year, kind); **display only `superseded=no` AND `extraction=parsed`**, always with `doc_url`.

## OpenAlex: needs a free API key as of 2026-10-01 (anonymous access is now budgeted)

`harvest_openalex.py` builds `publications.csv` (the papers well), `researchers_openalex.csv`
and `coauthor_edges.csv`. **It cannot run until `OPENALEX_API_KEY=` is added to
`~/psych/product/.keys`** and exits 3 with that message until then, leaving every existing
file untouched.

OpenAlex moved to a credit budget during 2026. The 429 response spells it out:

    x-ratelimit-limit: 1000          (credits)
    x-ratelimit-limit-usd: 0.1       (per day, FREE/anonymous)
    x-ratelimit-cost-required-usd: 0.001   (per works query)
    retry-after: 29314               (resets midnight UTC)

That is ~100 anonymous requests per day **shared by every client on the IP address**, and a
full corpus pull is ~300 pages. This is not a pace you can engineer around; the `mailto`
polite pool no longer buys headroom. Keys are free, carry their own budget, and are the
documented fix: <https://help.openalex.org/api/authentication/>.

### The corpus rewrite this blocks (built, tested, waiting on the key)

The old query was ten "-assisted" phrases from 2015 onward → 9,757 works. Measured against
OpenAlex that is **49% of the modern literature and 33% of all of it**. Two causes: the term
list missed `microdosing`, bare `psychedelics`, `psilocin`, `mescaline` and the full chemical
names; and the 2015 floor deleted the 1950s-70s research wave plus most of the MDMA
literature (10,919 papers all-time vs 5,162 since 2015).

Rewritten to five queries at two trust levels, with no date floor:

| Query | Trusted? | Why |
|---|---|---|
| `compounds` | yes | a compound name is self-evidently on topic |
| `assisted` | yes | the old query, kept whole so nothing found before is lost |
| `ketamine_psych` | yes | esketamine/arketamine/HNK only — bare `ketamine` would import the whole anesthesia literature |
| `field` | **gated** | `psychedelic` also describes rock music and visual art |
| `ambiguous` | **gated hard** | `LSD` is Fisher's Least Significant Difference; a raw search returns 24,727 works, mostly agronomy |

Gated queries must match both `PSYCH_CONTEXT` and `MED_CONTEXT`; the ambiguous pair must
additionally name a real compound. Drop counts and reasons are logged per run.

Expected on first keyed run: **~19,700 works at the 2015 window, ~29,900 with history** —
roughly triple the current corpus. Both CSV writes carry the collapse guard, so a bad run
cannot replace a good corpus.

## HealingMaps: harvesting stopped 2026-09-30 (a licensing question, not a bug)

`harvest_healingmaps.py` had pulled `healingmaps.com/wp-json/wp/v2/listing` since the
project began. It contributes ~1,991 of the clinic rows — by far the largest single source
in the index. **It is now stopped, deliberately, and should not be restarted as-is.**

Two independent signals say the publisher does not want automated access to that endpoint:

1. `healingmaps.com/robots.txt` disallows `/wp-json/` — listed twice, once by their managed
   host (BigScoots) and once in the site's own block.
2. Since 2026-09-12 the endpoint returns `HTTP 403` with `cf-mitigated: challenge` and
   `server: cloudflare`. That is Cloudflare bot management actively challenging us, not a
   transient outage. The harvester failed silently for 18 days; the file behind a fifth of
   the index froze on 2026-09-02.

The harvester now checks `robots.txt` with `urllib.robotparser` before any fetch and exits
**3 = NOT PERMITTED**, leaving the existing CSV untouched.

**Why we are not routing around it.** A headless browser, a residential proxy or a rotated
TLS fingerprint would all restore the data tomorrow. None of them are available to us. This
index publishes its own source mix on `/data`, tells visitors every record is traceable to
a public source, and asks to be trusted as a reference layer. Harvesting a publisher that
has explicitly blocked us would make that claim hollow, and bypassing a deployed access
control carries real legal exposure besides. The block is an answer. We take it.

**What to do instead**, in order of value:

- **Ask.** HealingMaps is a directory; we credit and deep-link every record we carry. A
  licensed feed or an API key is a normal commercial conversation, and the pitch is
  genuinely mutual. This is Erich's call to make.
- **Reduce the dependency.** One directory behind ~1,991 clinic rows is a single point of
  failure for the whole product, and this outage is what that risk looks like in practice.
  The durable sources are the ones that cannot block us and do not want to: state licence
  registers (Oregon OHA, Colorado NMHC), CMS NPI/Part B, state medical boards. They are
  public-domain, authoritative, and better provenance than a directory listing.
- **Disclose while it lasts.** `/data` now publishes when each source was last read
  successfully and marks anything past twice its window as stale. Those records remain
  accurate as of their harvest date; what changed is that we no longer imply otherwise.

## LegiScan compliance (their terms, our obligations — noted 2026-09-03)
- **Hash-driven work loop implemented**: getSearchRaw → compare cached `change_hash` per bill (`ls_state.json`) → getBill only for new/changed. First run ~240 queries; steady-state weekly runs ~10-30 queries against the 30,000/month limit.
- Every response's `status` field checked for OK/ERROR; errors logged and that loop aborted.
- All JSON cached locally (`ls_state.json`) — no re-fetching unchanged data.
- One public key only, never scrape legiscan.com's front end, CC BY 4.0 attribution on every displayed bill ("via LegiScan" + bill URL — the `attribution` column ships in the CSV).
- Key lives in `.keys` (gitignored). If we ever need texts/votes or >30k queries, that's their paid Pull tier — revisit then.

## Operational
- All Family C harvesters run via `refresh.py` staleness windows (see table in README); each is checkpointed and idempotent like the rest.
- `harvest_legiscan.py` requires `LEGISCAN_API_KEY` in `~/psych/product/.keys` (format: `LEGISCAN_API_KEY=xxx`). Free at legiscan.com/legiscan (2-minute signup). Harvester logs a skip-notice until the key exists — nothing breaks.
- Rate etiquette: OpenAlex polite-pool (mailto param), CMS ≤1 req/s, all harvesters sleep between pages.
