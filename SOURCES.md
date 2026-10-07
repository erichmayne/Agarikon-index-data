# The Agarikon data contract

Every file the pipeline publishes, what one row means, where it comes from and how often it changes. The site is built from these files and nothing else, so if a column isn't described here, don't build on it.

Two machine files sit beside this one. `sources.json` is the curated registry of every source we pull, have measured, or have ruled out, with its licence, what we may republish, how often it refreshes and what blocks it. `sources_status.json` is rewritten on every refresh with each source's live health, the last run's outcome, and how much of the real world each slice of the index holds. History and the reasons behind the rules are in `CHANGELOG.md`. The readable version is `research/SOURCES_STATUS.md`, regenerated with them. It opens with how each part of the site refreshes (scheduled, frozen, manual or hard-coded) and which routes a manual pull can reach, from the `refresh` block in sources.json.

## Places: clinics, retreats and trial sites

| File | One row is | Key | Refreshed |
|---|---|---|---|
| `facilities_master.csv` | a facility, merged across every source | name, city and `cc` | every refresh |
| `admin_sites.csv` | a row from an agent sweep of public pages | name, city, country | with each sweep |
| `healingmaps_sites.csv` | a HealingMaps listing, frozen at 2026-09-02 | name, city, region | never (the source refuses us) |
| `oregon_licenses.csv` | an Oregon psilocybin licensee who joined OHA's public directory | name, licence type | weekly |
| `colorado_licenses.csv` | a Colorado natural-medicine facility licence, any status | licence number | weekly |

facilities_master merges on the ISO country code, so "USA" and "United States" are one place. `country` is the display name, `cc` the ISO code and `state` the USPS code for US rows. A chain listed in two or more cities keeps one row per branch, and a HealingMaps neighbourhood stays in the city ("New York – Chelsea").

`legal_status` says what the law allows, and it needs evidence. `legal_retreat` appears only with a `legal_basis`. A directory listing without one is `directory_claim`. A Colorado row whose register status isn't Approved reads `licence_pending`, `licence_withdrawn`, `licence_voluntary_surrender` and so on, and never `gov_approved_treatment`. `basis_source` records where the basis came from: `license_record` (a state register), `operator_stated` (the operator's own words) or `jurisdiction_template` (a country-and-drug rule shared by many rows, so don't quote it as anyone's claim). A template exists only where `entities/jurisdictions.csv` has a primary-source line for the same place and drug. Today that's three: Dutch truffles, Mexican ibogaine and US ketamine. The licence columns (`licence_no`, `licence_type`, `licence_status`, `licence_fetched`) come from the registers, never from sweep text.

`drugs` holds the shared drug labels in `vocab.py` (psilocybin, MDMA, 5-MeO-DMT and the rest). TMS is a device, so it lives in `modalities`.

## Research: trials, papers and the people who write them

| File | One row is | Key | Refreshed |
|---|---|---|---|
| `studies.csv` | a registered trial | nct_id | daily |
| `research_locations.csv` | a trial at one site | nct_id, facility, city, country | daily |
| `publications.csv` | a paper in scope | openalex_id | monthly |
| `paper_authors.csv` | one author on one paper | work_id, author_id | monthly |
| `researchers_openalex.csv` | an author with 3+ papers in scope, with citation counts | openalex_id | monthly |
| `coauthor_edges.csv` | two authors who share 2+ papers | author_a, author_b | monthly |
| `researchers.csv` | a PubMed author with 3+ therapy papers since 2015 | author | monthly |
| `nih_grants.csv` | an NIH project, all fiscal years rolled up | core_project_num | weekly |
| `nih_grant_pubs.csv` | a paper NIH links to a project | core_project_num, pmid | weekly |

Trials are found by searching the intervention field only, never full text. Full-text matching over-reports several times over: "DMT" also means diabetes mellitus type 2, "LSD" means Least Significant Difference, and one esketamine trial's exclusion criteria name MDMA, LSD and hallucinogens. Any registry we add later has to follow the same rule. `relevance` sorts trials into psychiatric, anesthesia_analgesia and other, and the site leaves out the anaesthesia ones. `studies.csv` also carries `summary` (the registry's brief summary), `sponsor_class`, `collaborators`, `secondary_ids` and the PMIDs the registry links to the trial (`result_pmids`, `derived_pmids`).

Papers are in scope when they're psychedelic medicine, and `scope.py` is the only place that rule is written. A psychedelic in the title gets a paper in unless the title frames it as history, art, taxonomy or cultivation. One named only in the abstract needs a clinical title. A title that says "microdosing" has to name a psychedelic, because most of those are phase-0 drug-development microdoses. Grants pass the same rule, and a ketamine grant needs a psychiatric indication.

`paper_author_emails.csv` and the `email` column of `paper_authors.csv` are internal. They're never part of the public deposit.

## Money, law and supply

| File | One row is | Refreshed | Display rule |
|---|---|---|---|
| `spravato_providers.csv` | a provider billing Medicare for Spravato observation (HCPCS G2082/G2083) | quarterly | Medicare fee-for-service only, and CMS hides providers with under 11 beneficiaries: a verification badge, never a count of centres |
| `openpayments_spravato.csv` | Spravato payments to one clinician in one year, by kind | quarterly | aggregates or per-recipient totals, with the program named |
| `dea_quotas.csv` | DEA's production quota for one substance in one year | weekly | show only `superseded=no` and `extraction=parsed`, with `doc_url` |
| `legiscan_bills.csv` | a state or federal bill naming a psychedelic | weekly | "via LegiScan" and the bill link on every bill (CC BY 4.0) |
| `sec_issuers.csv` | a public company whose filings discuss psychedelics | weekly | `role=core` are the psychedelic companies |
| `dea_registrants.csv` | a company licensed or applying to make or import Schedule I psychedelics | weekly | `catalogue_supplier=yes` sell lab standards, not medicine |

## The entity graph

`entities/` holds the graph the site's panels walk. `eid_registry.csv` is the id authority: it ties each source row to a stable id, keeps a renamed row's id, and turns a merged row's id into an alias of the survivor. A published URL never just disappears. `people.csv` has one row per person, `orgs.csv` one per organisation, and `links.csv` one per relationship. `org_aliases.csv` is curated by hand and covers renames and parent companies no source states (Janssen's seven sponsor names are one company, owned by Johnson & Johnson).

`jurisdictions.csv` has one row per place and drug: the law in one line (`status`), the page it comes from (`source_url`), the access program where there is one (`program`, `program_source`) and the date it was checked (`as_of`). Every line comes from a statute, a regulator or an official gazette. A place with no verified line is left out rather than guessed, which is why Jamaica, Costa Rica and Peru are missing for now.

Every link carries its `basis`, and the site renders three kinds. `id` means both ends share a hard identifier: an OpenAlex author id, an NCT number, a PMID or an NIH profile. `exact` means the same normalised name, unique on both sides. `two_signal` means two independent things agree, such as a name and an institution. Anything weaker goes to `links_review.csv`, which the site never reads.

| rel | from → to |
|---|---|
| authored | person → paper |
| affiliated | person → institution (latest paper) |
| site_of | trial site → trial |
| investigator, contact | person → trial |
| sponsors, collaborates_on | organisation → trial |
| reports, mentions | paper → trial (a results paper, or one the registry or abstract cites) |
| funded_by | paper or trial → NIH grant |
| pi_of | person → NIH grant |
| awarded_to | NIH grant → organisation |
| retracts | retraction notice → the paper it retracts |

## Rules every file follows

A harvester never writes over good data with a broken run. Every write goes to a temp file first, and one that would drop more than half the rows on disk is refused. A source that refuses us stays refused: we don't use proxies, rotated fingerprints or headless browsers. HealingMaps blocked us in September and those 2,080 rows are frozen and labelled as such. Oregon contact details are exactly what OHA's public directory shows, and a field a licensee didn't consent to arrives empty from OHA itself. Rows that fail validation go to `quarantine.csv` with a written reason instead of vanishing. What each stage removed is counted in `EXCLUSION_LOG.md` and `exclusions.json`, so the site never prints a purge number by hand.
