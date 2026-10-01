# Agarikon — Exclusion & Screening Log
_Generated 2026-10-01 from live pipeline state. Every count recomputed from the datasets, not hand-entered. Regenerated on each refresh._

## Why this file exists
Agarikon is a screened index, not a scrape. Sources present records that look in-scope but are not: a directory that also lists weight-loss clinics, a trial registry where one search term spans two medical fields, organizations that are not treatment facilities. This log is the record of what was removed and under what rule — the difference between a methodology and a list.

## Screening flow (identified → excluded → included)

| Stage | Exclusion rule | Records removed | Enforced by |
|---|---|---|---|
| Clinics: source ingest | HealingMaps listings NOT tagged as a psychedelic/psychedelic-therapy category (GLP-1, peptide, generic wellness clinics the source also lists) | 980 | psychedelic-only keep-rule in harvest_healingmaps.py |
| Trials: query expansion | ClinicalTrials.gov records matching a search term but carrying NO psychedelic intervention (e.g. 'DMT' = disease-modifying therapy in multiple-sclerosis trials) | 93 | no-drug-evidence artifact drop in harvest_ctgov.py |
| Trials: relevance | Studies whose indication is anesthesia/analgesia/procedural-sedation, sharing only a molecule (ketamine) with the therapeutic field | 804 | relevance classifier (psychiatric/anesthesia_analgesia/other); anesthesia excluded from index |
| Trial sites: relevance | Trial-site rows belonging to anesthesia/analgesia studies above | 997 | same classifier, propagated to site rows |
| Facilities: validation | Rows quarantined at merge: non-facility organizations, wellness residue, unidentifiable rows | 1,296 | validate.py gate; quarantine.csv with written reasons |

## Locked figures (this snapshot)
- Studies excluded as anesthesia/analgesia: **804 of 2,434 = 33.0%**
- Trial-site rows excluded on the same basis: **997 of 5,655 = 17.6%**
- Facility rows quarantined at validation: **1,296** (504 non-facility org in facilities master, 66 , 44 Lysergic Acid Diethylamide (LSD) in Palliative Care)
- Wellness/non-psychedelic clinic listings dropped at source ingest: **~980** (HealingMaps keep-rule)
- Query-expansion artifact studies dropped: **~93** (bare-DMT → multiple-sclerosis)
- **Retained on purpose:** 167 studies whose title or conditions name an anesthesia or procedural term stay in the index. Many are psychiatric trials that merely mention a procedural agent. The removal count above is what was removed, not a claim the category is empty.

## Doctrine
Classify relevance at ingest, validate at merge, quarantine with a written reason — never silently drop. Excluded records are retained in `quarantine.csv` with their reason, not deleted, so any exclusion is auditable and reversible.

## Known provenance limits (stated plainly)
- The clinics dataset is ~84% sourced from the HealingMaps directory; it is aggregated and screened, not independently verified field data.
- A legal-status classification is on every facility row; the enabling statute is cited where one exists (all retreat/legal-access rows where jurisdiction is mappable; near-absent for ordinary clinics, which operate under generally-applicable medical law rather than a psychedelic-specific statute).