# Agarikon — Exclusion & Screening Log
_Generated 2026-10-06 from live pipeline state. Every count is recomputed from the datasets on each refresh, except the one row marked historical (a stage that no longer runs)._

## Why this file exists
Agarikon is a screened index, not a scrape. Sources present records that look in-scope but are not: a directory that also lists weight-loss clinics, a trial registry where one search term spans two medical fields, organizations that are not treatment facilities. This log is the record of what was removed and under what rule — the difference between a methodology and a list.

## Screening flow (identified → excluded → included)

| Stage | Exclusion rule | Records removed | Enforced by |
|---|---|---|---|
| Clinics: source ingest | HealingMaps listings NOT tagged as a psychedelic/psychedelic-therapy category (GLP-1, peptide, generic wellness clinics the source also lists). Historical: the 2026-09-02 harvest; the source has refused us since | 980 | psychedelic-only keep-rule in harvest_healingmaps.py |
| Trials: query expansion | ClinicalTrials.gov records matching a search term but carrying NO psychedelic intervention (93 when the rule began on 2026-09-02; 0 since, because the harvester now queries only the intervention field) | 0 | no-drug-evidence artifact drop in harvest_ctgov.py (count from its own log) |
| Trials: relevance | Studies whose indication is anesthesia/analgesia/procedural-sedation, sharing only a molecule (ketamine) with the therapeutic field | 837 | relevance classifier (psychiatric/anesthesia_analgesia/other); anesthesia excluded from index |
| Trial sites: relevance | Trial-site rows belonging to anesthesia/analgesia studies above | 1,030 | same classifier, propagated to site rows |
| Publications: scope | Psychedelic term appears only in the abstract and the title is not clinical — a passing mention (how a COVID-19 meta-analysis and a Japan-Hungary diplomatic history entered the corpus before 2026-10-02) | 13,931 | is_relevant() in scope.py; scope = psychedelic medicine |
| Publications: scope | Returned by the search but no psychedelic term in title or abstract | 6,453 | is_relevant() in scope.py |
| Publications: scope | Psychedelic is the title subject but the framing is humanities, ethnography, taxonomy or cultivation, not medicine | 1,174 | is_relevant() in scope.py |
| Publications: scope | 'Microdosing' in the title without any psychedelic named: pharmacology's phase-0 microdose (oncology, fertility, veterinary anaesthesia) | 1,551 | microdose_only() in scope.py |
| Publications: scope | Phencyclidine (PCP) or MK-801 is the title's only drug: pharmacology's psychosis-model NMDA antagonists, in on a ketamine mention in the abstract | 117 | nmda_tool_only() in scope.py |
| Facilities: merge | Non-treatment organizations (law firms, funders, regulators, advocacy groups, drug developers) reclassified out of the clinics layer — they remain in the industry layer | 144 | industry cross-reference in merge_master.py |
| Facilities: merge | Duplicate operator rows collapsed (same name and country, city missing or contained) | 81 | second-pass dedupe in merge_master.py; every merge logged to dedupe_merges.csv |
| Facilities: merge | One licensee listed twice (legal name and dba) folded on its state licence and website | 9 | licence fold in merge_master.py; logged to dedupe_merges.csv |
| Facilities: merge | Contact values stripped as source-template leaks (e.g. an NIMH inquiry address printed on hundreds of unrelated clinic listings) | 1,159 | shared-contact guard in merge_master.py; values logged to shared_contacts_stripped.csv |
| Facilities: validation | Rows quarantined at merge: non-facility organizations, wellness residue, unidentifiable rows | 36 | validate.py gate; quarantine.csv with written reasons |

## Locked figures (this snapshot)
- Studies excluded as anesthesia/analgesia: **837 of 2,437 = 34.3%**
- Trial-site rows excluded on the same basis: **1,030 of 5,663 = 18.2%**
- Facility rows quarantined at validation: **36** (36 non-facility org in facilities master)
- Wellness/non-psychedelic clinic listings dropped at source ingest: **980**, historical (the HealingMaps harvest of 2026-09-02; the source has refused us since)
- Query-expansion artifact studies dropped this run: **0** (93 when the rule began; the intervention-field query no longer produces them)
- **Retained on purpose:** 147 studies whose title or conditions name an anesthesia or procedural term stay in the index. Many are psychiatric trials that merely mention a procedural agent. The removal count above is what was removed, not a claim the category is empty.

## Doctrine
Classify relevance at ingest, validate at merge, quarantine with a written reason — never silently drop. Excluded records are retained in `quarantine.csv` with their reason, not deleted, so any exclusion is auditable and reversible.

## Known provenance limits (stated plainly)
- The clinics dataset is ~84% sourced from the HealingMaps directory; it is aggregated and screened, not independently verified field data.
- A legal-status classification is on every facility row; the enabling statute is cited where one exists (all retreat/legal-access rows where jurisdiction is mappable; near-absent for ordinary clinics, which operate under generally-applicable medical law rather than a psychedelic-specific statute).