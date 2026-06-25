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

The column names are **inherited verbatim from facets-suite** and are not enumerated in the Tempo repo. For a typical facets-suite `gene_level` output expect (confirm against your file's header line): `Tumor_Sample_Barcode`, `Hugo_Symbol`, `chr`, `seg.start`, `seg.end`, `tcn`, `lcn`, `cn_state`, `filter`, plus per-gene focality/CI fields. Arm-level outputs carry the sample barcode, chromosome arm, the arm's copy state, and fraction-of-arm-altered fields.

> **Read the header from your own file** rather than assuming positions — facets-suite has changed column order across versions. The Tempo aggregation only guarantees the `cn_state` (col 24) and `filter` (col 25) positions used by the awk above for the version it was run with.

**`cn_state` values:** the code only names `DIPLOID` (the excluded value). The other categories come from facets-suite; for facets-suite ~1.6.x these are typically `AMP`, `GAIN`, `DIPLOID`, `HETLOSS`, `HOMDEL` (and CN-LOH variants). This list is **from facets-suite, not confirmed by the Tempo repo** — verify against your data and the facets-suite version.

## Loading and filtering (Python)

```python
import pandas as pd
gl = pd.read_csv("cohort_level/<cohort>/cna_genelevel.txt", sep="\t")

# already non-DIPLOID + PASS/RESCUE; e.g. high-level amplifications and deep deletions:
amp = gl[gl["cn_state"].str.contains("AMP", na=False)]
homdel = gl[gl["cn_state"].str.contains("HOMDEL", na=False)]

# recurrence of a gene's alteration across the cohort
recurrent = gl.groupby(["Hugo_Symbol","cn_state"]).size().sort_values(ascending=False)
```

(Adjust column names to match your header.)

## Caveats

- The files contain **only altered** loci (DIPLOID removed). Absence of a gene means "diploid or filtered," not "not assayed."
- Gene-level is QC-filtered (`PASS`/`RESCUE`); arm-level is not filtered beyond removing DIPLOID.
- Calls depend on the FACETS purity/ploidy fit — if a sample's purity is the **0.3 fallback** ([../tools/facets-interpreting.md](../tools/facets-interpreting.md)), treat its CN calls with caution.

## See Also

- [cohort-aggregates.md](cohort-aggregates.md) — all cohort-level files
- [somatic-facets.md](somatic-facets.md) — per-pair FACETS outputs
- [../tools/facets-interpreting.md](../tools/facets-interpreting.md) — CN call reliability, purity fallback
- [../recipes/resolve-purity-ploidy-wgd.md](../recipes/resolve-purity-ploidy-wgd.md) — when ploidy/WGD looks wrong
