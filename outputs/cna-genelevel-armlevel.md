# Cohort Copy-Number Files: `cna_genelevel.txt` & `cna_armlevel.txt`

> **Quick answer:** These two cohort-level aggregate files give per-gene and per-chromosome-arm copy-number calls across all samples. Tempo builds them by concatenating per-sample facets-suite outputs and **dropping DIPLOID segments** (gene-level also keeps only `filter == PASS` or `RESCUE`). The column names come from facets-suite, not from Tempo, so the exact schema is set by the facets-suite version your cohort was run with.

## How they are built (verified — `modules/process/Aggregate/SomaticAggregateFacets.nf`)

Both files aggregate the per-sample `*gene_level.txt` / `*arm_level.txt` produced by `run-facets-wrapper.R --everything` (facets-suite) during `DoFacets`.

**Gene-level** keeps the header, then filters by column:
```bash
cat facets_tmp/*gene_level.txt | head -n 1 > cna_genelevel.txt
awk -v FS='\t' '{ if ($24 != "DIPLOID" && ($25 == "PASS" || $25 == "RESCUE")) print $0 }' \
    facets_tmp/*gene_level.txt >> cna_genelevel.txt
```
- `$24` = the copy-number **state/call** column (`cn_state`)
- `$25` = facets-suite's **filter** column — only `PASS` and `RESCUE` rows are kept
- `DIPLOID` genes are excluded → the file contains only **altered** genes that passed QC.

**Arm-level** filters by string match:
```bash
cat facets_tmp/*arm_level.txt | head -n 1 > cna_armlevel.txt
cat facets_tmp/*arm_level.txt | grep -v "DIPLOID" | grep -v "Tumor_Sample_Barcode" >> cna_armlevel.txt
```
- Any row containing `DIPLOID` is dropped (so only altered arms remain)
- Repeated header rows (matching `Tumor_Sample_Barcode`) from subsequent samples are removed.

## Columns

The column names are **inherited verbatim from facets-suite** (`run-facets-wrapper.R --everything`) and are not enumerated in the Tempo repo. They have changed across facets-suite versions — **always read the header from your own file**.

**Gene-level columns observed in a recent delivery (facets-suite 0.5.14, 25 columns):**

```
sample, gene, chrom, gene_start, gene_end, tsg, seg,
median_cnlr_seg, segclust, seg_start, seg_end,
cf.em, tcn.em, lcn.em, cf, tcn, lcn, seg_length, mcn,
genes_on_seg, gene_snps, gene_het_snps, spans_segs,
cn_state, filter
```

- `sample` is the `{tumor}__{normal}` pair barcode (**not** `Tumor_Sample_Barcode`).
- `gene` is the gene symbol (**not** `Hugo_Symbol`).
- `cn_state` (col 24) and `filter` (col 25) are the columns the aggregation awk filters on.

**Arm-level columns observed (8 columns):**

```
sample, arm, tcn, lcn, cn_length, arm_length, frac_of_arm, cn_state
```

> **Watch-out:** the aggregator's `grep -v "Tumor_Sample_Barcode"` step is meant to strip duplicated header lines, but the real header begins with `sample` — so **duplicated `sample\tarm\t…` header rows remain in `cna_armlevel.txt`** (one per sample's `*arm_level.txt`). Filter them out yourself before analysis: `awk 'NR==1 || $1!="sample"'` or `df[df['sample'] != 'sample']`.

**`cn_state` values observed in a recent WES delivery:** `AMP`, `AMP (BALANCED)`, `AMP (LOH)`, `AMP (many states)`, `CNLOH`, `CNLOH & GAIN`, `CNLOH AFTER`, `CNLOH BEFORE`, `CNLOH BEFORE & GAIN`, `CNLOH BEFORE & LOSS`, `DIPLOID or CNLOH`, `DOUBLE LOSS AFTER`, `GAIN`, `GAIN (many states)`, `HETLOSS`, `HOMDEL`, `INDETERMINATE`, `LOSS (many states)`, `LOSS & GAIN`, `LOSS AFTER`, `LOSS BEFORE`, `LOSS BEFORE & AFTER`, `LOSS BEFORE or DOUBLE LOSS AFTER`, `TETRAPLOID`, `TETRAPLOID or CNLOH BEFORE`. (`DIPLOID` itself is the value excluded by the aggregator.) The set is much broader than the canonical `AMP / GAIN / HETLOSS / HOMDEL / CNLOH` — it encodes timing relative to WGD and uncertainty modifiers, and the exact set depends on the facets-suite version. For coarse filtering use `str.contains("AMP")`, `str.contains("HOMDEL")`, `str.contains("LOSS")`, etc.

## Loading and filtering (Python)

```python
import pandas as pd
gl = pd.read_csv("cohort_level/<cohort>/cna_genelevel.txt", sep="\t")

# already non-DIPLOID + PASS/RESCUE; e.g. high-level amplifications and deep deletions:
amp = gl[gl["cn_state"].str.contains("AMP", na=False)]
homdel = gl[gl["cn_state"].str.contains("HOMDEL", na=False)]

# recurrence of a gene's alteration across the cohort
recurrent = gl.groupby(["gene", "cn_state"]).size().sort_values(ascending=False)

# arm-level: filter out the duplicated header rows the aggregator leaves behind
al = pd.read_csv("cohort_level/<cohort>/cna_armlevel.txt", sep="\t")
al = al[al["sample"] != "sample"].copy()
```

(Adjust column names to match your file's header — facets-suite has changed names across versions.)

## Caveats

- The files contain **only altered** loci (DIPLOID removed). Absence of a gene means "diploid or filtered," not "not assayed."
- Gene-level is QC-filtered (`PASS`/`RESCUE`); arm-level is not filtered beyond removing DIPLOID.
- Calls depend on the FACETS purity/ploidy fit — if a sample's purity is the **0.3 fallback** ([../tools/facets-interpreting.md](../tools/facets-interpreting.md)), treat its CN calls with caution.

## See Also

- [cohort-aggregates.md](cohort-aggregates.md) — all cohort-level files
- [somatic-facets.md](somatic-facets.md) — per-pair FACETS outputs
- [../tools/facets-interpreting.md](../tools/facets-interpreting.md) — CN call reliability, purity fallback
- [../recipes/resolve-purity-ploidy-wgd.md](../recipes/resolve-purity-ploidy-wgd.md) — when ploidy/WGD looks wrong
