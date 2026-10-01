# Agarikon index — published data

The data behind [agarikon-index.com](https://agarikon-index.com). Rebuilt daily from
public sources; this repository is a mirror of what the site serves, kept separate from
the application code so the citable record contains data and nothing else.

**Cite this dataset**

> Mayne, Erich. *Agarikon: A Screened Index of Psychedelic Medicine*. Zenodo.
> https://doi.org/10.5281/zenodo.23090788

That DOI always resolves to the latest version. Each quarterly release also mints its own
version DOI, frozen to that snapshot.

## What is here

`data/` is the JSON the site loads. `csv/` is the same content in the analysis-facing
form, plus the underlying tables. `EXCLUSION_LOG.md` and `exclusion_log.csv` record what
was removed and under which rule — read those before citing a count. `SOURCES.md` is the
data contract: every field, cadence, gate and licence.

## Current snapshot

Generated 2026-10-01T12:57:52-07:00

| dataset | rows |
|---|---|
| clinics | 2,259 |
| industry | 167 |
| papers | 45,619 |
| policy | 1,532 |
| research | 4,549 |
| researchers | 1,618 |
| retreats | 214 |
| studies | 1,631 |

## Licensing is mixed, per source

Clinical trials, CMS payment and claims data, DEA quotas and Federal Register material are
U.S. Government work in the public domain. Publications and author metrics come from
OpenAlex and PubMed under CC0 or permissive terms. **State legislation comes from LegiScan
under CC BY 4.0 and attribution is required** — "via LegiScan" plus the bill URL, shipped in
the `attribution` column and to be preserved in any reuse. Clinic and retreat listings are
screened aggregations of public directories and state licensing feeds; they are not
independently verified field data and directory-sourced listings are not relicensed here.

The compilation — the selection, screening and arrangement — is © Erich Mayne / Agarikon,
released CC BY 4.0. Individual records remain under their source licences.

## Caveats worth reading before you cite

A legal-status classification is present on every facility row; the enabling statute is
cited where one exists, which covers retreat and legal-access rows with a mappable
jurisdiction and is largely absent for ordinary clinics. Trial-location rows the registry
published without a facility name are flagged `unnamed` and are not distinct sites. The
policy set is a controlled-substance corpus with a psychedelic-specific subset flagged
`psy`, not a curated psychedelic bill tracker. `exclusions.json` states the anesthesia
residual that remains after filtering.

Corrections: open an issue, or write to data@agarikon-index.com. Fixes land in the next
daily build.
