# Loading and Filtering the Cohort SV BEDPE (`sv_somatic.bedpe`)

> **Quick answer:** The cohort `sv_somatic.bedpe` concatenates each pair's PASS-filtered somatic SV BEDPE. Every row is already a multi-caller PASS call (the `FILTER` column = `PASS`). Columns are the svtools BEDPE columns plus Tempo `TUMOR_ID`/`NORMAL_ID`, a cDNA-contamination flag, iAnnotateSV gene annotations, and (WGS only) ClusterSV columns. Caller support is carried inside the `INFO_A`/`INFO_B` fields, not as top-level columns.

## How it is built (verified — `modules/process/`)

1. Per-pair SV VCFs → BEDPE via **svtools `vcftobedpe`** (`SomaticSVVcf2Bedpe.nf`), with `TUMOR_ID`/`NORMAL_ID` appended.
2. `detect_cdna.py` appends `POTENTIAL_CDNA_CONTAMINATION`.
3. **iAnnotateSV** (`run_iannotatesv.py`) merges gene/site annotation columns.
4. WGS only: **ClusterSV** appends clustering columns.
5. Each pair is filtered to `FILTER == "PASS"` (`$12 == "PASS"`) → `*.final.bedpe`.
6. `SomaticAggregateSv.nf` concatenates all pairs, de-duplicating header lines.

So **every row in the cohort file is already a PASS call** — no further FILTER step is needed.

## Columns (verified order from `SomaticSVVcf2Bedpe.nf`)

The svtools/Tempo block (columns 1–25):

```
#CHROM_A START_A END_A CHROM_B START_B END_B ID QUAL STRAND_A STRAND_B
TYPE FILTER NAME_A REF_A ALT_A NAME_B REF_B ALT_B INFO_A INFO_B
FORMAT TUMOR NORMAL TUMOR_ID NORMAL_ID
```
- Breakpoint A: `CHROM_A/START_A/END_A/STRAND_A`; breakpoint B: the `_B` columns.
- `TYPE` = SV type (DEL/DUP/INV/BND/…); `FILTER` = `PASS` for all kept rows (col 12).
- `INFO_A`/`INFO_B` carry the merged-caller INFO, including **caller support** (`NumCallers`, `NumCallersPass`, `Callers`) — parse these strings if you need per-caller evidence.

Then appended:
- `POTENTIAL_CDNA_CONTAMINATION` (from `detect_cdna.py`)
- **iAnnotateSV** annotation columns — typically `gene1`, `site1`, `gene2`, `site2`, `fusion`/`description` (exact set defined by iAnnotateSV, **not enumerated in the Tempo repo** — read your header).
- **WGS only** (ClusterSV): `cluster_id`, `cluster_total_count`, `footprint_id_low`, `footprint_id_high`, `coord_footprint_id_low`, `coord_footprint_id_high`, `clustersv_pval`.

## Loading (Python)

```python
import pandas as pd
bedpe = pd.read_csv("cohort_level/<cohort>/sv_somatic.bedpe", sep="\t")
bedpe = bedpe.rename(columns={bedpe.columns[0]: "CHROM_A"})  # strip leading '#'

# All rows are PASS already. Example: gene fusions annotated by iAnnotateSV
fusions = bedpe[bedpe.get("fusion", "").astype(str).str.len() > 0]

# Per-sample SV burden
burden = bedpe.groupby("TUMOR_ID").size().sort_values(ascending=False)

# Pull caller support out of INFO (format is KEY=VAL;KEY=VAL)
def info_val(info, key):
    for kv in str(info).split(";"):
        if kv.startswith(key + "="):
            return kv.split("=", 1)[1]
    return None
bedpe["NumCallersPass"] = bedpe["INFO_A"].map(lambda s: info_val(s, "NumCallersPass"))
```

## Filtering notes

- The PASS rule applied upstream is **≥2 callers when >2 callers ran, else ≥1** (verified, `filter-sv-vcf.py`); near-duplicate breakpoints (both ends ≤1 bp apart) were dropped. See [../tools/sv-callers.md](../tools/sv-callers.md).
- To prioritize highest-confidence events, sort by `NumCallersPass` (parsed from INFO) and, for WGS, lower `clustersv_pval`.
- `POTENTIAL_CDNA_CONTAMINATION`-flagged rows may be processed-pseudogene artifacts — inspect before trusting.

## See Also

- [../tools/sv-format.md](../tools/sv-format.md) — BEDPE format basics
- [../tools/sv-callers.md](../tools/sv-callers.md) — callers and the merge/PASS rule
- [../outputs/cohort-aggregates.md](../outputs/cohort-aggregates.md) — all cohort-level files
- [../outputs/somatic-sv-vcf.md](../outputs/somatic-sv-vcf.md) — per-pair SV outputs
