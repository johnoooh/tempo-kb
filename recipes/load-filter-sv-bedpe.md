# Loading and Filtering the Cohort SV BEDPE (`sv_somatic.bedpe`)

> **Quick answer:** The cohort `sv_somatic.bedpe` concatenates each pair's PASS-filtered somatic SV BEDPE. Every row is already a multi-caller PASS call (the `FILTER` column = `PASS`). Columns are the svtools BEDPE columns plus Tempo `TUMOR_ID`/`NORMAL_ID`, a cDNA-contamination flag, iAnnotateSV gene annotations, and (WGS only) ClusterSV columns. Caller support is carried inside the `INFO_A`/`INFO_B` fields, not as top-level columns.

> **Older deliveries ship `sv_somatic.vcf.gz` instead of `sv_somatic.bedpe`.** If your cohort_level directory contains a VCF, it was produced before Tempo switched its SV aggregator to BEDPE — the columns below do not apply. Decompress with `bgzip -d` (or `gunzip`) and parse it as a standard multi-sample VCF (`#CHROM POS ID REF ALT QUAL FILTER INFO FORMAT <samples…>`); `FILTER == PASS` and `INFO=SVTYPE=...` carry the same information as the BEDPE's `FILTER` and `TYPE`. Re-run on a newer pipeline version if you need the BEDPE schema.

## How it is built (verified — `modules/process/`)

1. Per-pair SV VCFs → BEDPE via **svtools `vcftobedpe`** (`SomaticSVVcf2Bedpe.nf`), with `TUMOR_ID`/`NORMAL_ID` appended.
2. `detect_cdna.py` appends `POTENTIAL_CDNA_CONTAMINATION`.
3. **iAnnotateSV** (`run_iannotatesv.py`) merges gene/site annotation columns.
4. WGS only: **ClusterSV** appends clustering columns.
5. Each pair is filtered to `FILTER == "PASS"` (`$12 == "PASS"`) → `*.final.bedpe`.
6. `SomaticAggregateSv.nf` concatenates all pairs, de-duplicating header lines.

So **every row in the cohort file is already a PASS call** — no further FILTER step is needed.

## Header preamble (read this first)

Both per-pair and cohort BEDPEs start with a **VCF-style preamble** — `##fileformat=BEDPEVCFv4.2`, `##FILTER=…`, `##contig=…` lines — before the tab-delimited column header. `pd.read_csv(sep='\t')` alone will fail because pandas sees the `##` lines as 1-column rows. Skip them with `comment='#'`, or grab the `#CHROM_A` header explicitly (recipe below). The cohort aggregator (`SomaticAggregateSv.nf`) keeps the first pair's preamble, then drops subsequent preambles, so the cohort file has the same structure.

## Columns (verified order — per-pair WES `.final.bedpe`)

Columns 1–25 are the svtools/Tempo block:

```
#CHROM_A START_A END_A CHROM_B START_B END_B ID QUAL STRAND_A STRAND_B
TYPE FILTER NAME_A REF_A ALT_A NAME_B REF_B ALT_B INFO_A INFO_B
FORMAT TUMOR NORMAL TUMOR_ID NORMAL_ID
```
- Breakpoint A: `CHROM_A/START_A/END_A/STRAND_A`; breakpoint B: the `_B` columns.
- `TYPE` = SV type (DEL/DUP/INV/BND/…); `FILTER` = `PASS` for all kept rows (col 12).
- `INFO_A`/`INFO_B` carry the merged-caller INFO. Caller support is in `INFO_A` as `Callers=manta,delly`, `NumCallers=2`, `NumCallersPass=1` — parse these out of the `;`-delimited string.

Then `POTENTIAL_CDNA_CONTAMINATION` (col 26, from `detect_cdna.py`), followed by **iAnnotateSV** columns 27–43 in this exact order (verified):

```
gene1 transcript1 site1
gene2 transcript2 site2
fusion Cosmic_Fusion_Counts
repName-repClass-repFamily:-site1 repName-repClass-repFamily:-site2
CC_Chr_Band CC_Tumour_Types(Somatic) CC_Cancer_Syndrome
CC_Mutation_Type CC_Translocation_Partner
DGv_Name-DGv_VarType-site1 DGv_Name-DGv_VarType-site2
```

- The `repName-repClass-repFamily:-site1/2` and `DGv_Name-DGv_VarType-site1/2` column names contain hyphens, colons, and parentheses — quote them when subsetting (`df["repName-repClass-repFamily:-site1"]`).
- Column count for WES = **43**. WGS adds **ClusterSV** columns at the end: `cluster_id`, `cluster_total_count`, `footprint_id_low`, `footprint_id_high`, `coord_footprint_id_low`, `coord_footprint_id_high`, `clustersv_pval`. (Not present in WES — verified.)

## Loading (Python)

```python
import pandas as pd

path = "cohort_level/<cohort>/sv_somatic.bedpe"  # or somatic/{T}__{N}/combined_svs/*.final.bedpe

# Pull the column header line yourself so the leading '#' is stripped cleanly.
with open(path) as f:
    for line in f:
        if line.startswith("#CHROM_A"):
            cols = line.lstrip("#").rstrip("\n").split("\t")
            break

bedpe = pd.read_csv(path, sep="\t", comment="#", names=cols, header=None)
# `comment='#'` skips the ##fileformat / ##FILTER / ##contig preamble and the
# duplicated #CHROM_A line (the aggregator keeps only the first pair's header,
# but comment='#' handles either way).

# All rows are PASS already. Example: gene fusions annotated by iAnnotateSV
fusions = bedpe[bedpe["fusion"].astype(str).str.len() > 0]

# Per-sample SV burden
burden = bedpe.groupby("TUMOR_ID").size().sort_values(ascending=False)

# Pull caller support out of INFO_A (format is KEY=VAL;KEY=VAL; flags have no '=')
def info_val(info, key):
    for kv in str(info).split(";"):
        if kv.startswith(key + "="):
            return kv.split("=", 1)[1]
    return None
bedpe["NumCallersPass"] = bedpe["INFO_A"].map(lambda s: info_val(s, "NumCallersPass"))
bedpe["Callers"]        = bedpe["INFO_A"].map(lambda s: info_val(s, "Callers"))
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
