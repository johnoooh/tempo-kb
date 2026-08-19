# Batch Processing Multiple Tempo Output Samples

> **Quick answer:** Use glob patterns to iterate over Tempo output directories and collect results across all tumor-normal pairs. Extract sample IDs from directory names by splitting on the double-underscore (`__`) separator.

## Why Batch Process

Tempo produces outputs organized per tumor-normal pair under `outDir/somatic/`. When you have tens or hundreds of pairs, you need systematic approaches to collect MAFs, FACETS results, QC metrics, and other outputs into unified tables for cohort-level analysis.

## Output Directory Structure

Each tumor-normal pair has its own subdirectory. **FACETS adds three levels of nesting**: `facets/{pair}/facets{version}c{cval}pc{purity_cval}/`. The versioned subdirectory name changes with FACETS version and parameters, so always use a glob.

```
outDir/somatic/
  TUMOR_01__NORMAL_01/
    combined_mutations/
      TUMOR_01__NORMAL_01.somatic.final.maf
      TUMOR_01__NORMAL_01.somatic.unfiltered.maf
    combined_svs/
      TUMOR_01__NORMAL_01.final.bedpe
    facets/
      TUMOR_01__NORMAL_01/
        TUMOR_01__NORMAL_01.facets_qc.txt        # best-fit summary, MetaDataParser reads this
        TUMOR_01__NORMAL_01_OUT.txt              # TSV: Sample/Purity/Ploidy/dipLogR rows
        facets0.5.14c100pc500/                   # facets{version}c{cval}pc{purity_cval}
          TUMOR_01__NORMAL_01_hisens.Rdata
          TUMOR_01__NORMAL_01_purity.Rdata
          TUMOR_01__NORMAL_01_hisens.out         # COMMENT block (# Purity = …), not a TSV
          TUMOR_01__NORMAL_01_purity.out
          TUMOR_01__NORMAL_01_hisens.cncf.txt    # TSV: 18 segment cols incl. ID
          TUMOR_01__NORMAL_01_purity.cncf.txt
          TUMOR_01__NORMAL_01.gene_level.txt
          TUMOR_01__NORMAL_01.arm_level.txt
          TUMOR_01__NORMAL_01.qc.txt             # per-fit facetsPreview QC
  TUMOR_02__NORMAL_02/
    ...
```

## File Locations (Glob Patterns)

| Output | Glob Pattern |
|--------|-------------|
| Final MAF | `outDir/somatic/*/combined_mutations/*.somatic.final.maf` |
| Unfiltered MAF | `outDir/somatic/*/combined_mutations/*.somatic.unfiltered.maf` |
| Final SV BEDPE | `outDir/somatic/*/combined_svs/*.final.bedpe` |
| FACETS `_OUT.txt` (Purity/Ploidy TSV) | `outDir/somatic/*/facets/*/*_OUT.txt` |
| FACETS hisens Rdata | `outDir/somatic/*/facets/*/facets*/*_hisens.Rdata` |
| FACETS purity Rdata | `outDir/somatic/*/facets/*/facets*/*_purity.Rdata` |
| FACETS hisens cncf (TSV) | `outDir/somatic/*/facets/*/facets*/*_hisens.cncf.txt` |
| FACETS best-fit QC | `outDir/somatic/*/facets/*/*.facets_qc.txt` |

## Example: Bash Approach

### Collect All MAFs into One File

```bash
OUT_DIR="outDir/somatic"

# Get header from first MAF (skip comment lines)
first_maf=$(ls ${OUT_DIR}/*/combined_mutations/*.somatic.final.maf | head -1)
grep -v '^#' "$first_maf" | head -1 > combined_cohort.maf

# Append data rows from all MAFs
for maf in ${OUT_DIR}/*/combined_mutations/*.somatic.final.maf; do
    grep -v '^#' "$maf" | tail -n +2 >> combined_cohort.maf
done

echo "Combined $(wc -l < combined_cohort.maf) lines into combined_cohort.maf"
```

### Extract Purity from All Samples

Use `{pair}_OUT.txt` (a TSV with `Sample/Facets/.../Purity/Ploidy/dipLogR` columns and one row each for `_hisens` and `_purity` fits) — **not** `_hisens.out` / `_purity.out`, which are comment-style key=value blocks (`# Purity = 0.53`, `# Ploidy = 4.8`) and don't parse as TSV.

```bash
echo -e "tumor_id\tnormal_id\tfit\tpurity\tploidy\tdipLogR" > purity_summary.tsv

for out_file in ${OUT_DIR}/*/facets/*/*_OUT.txt; do
    pair_dir=$(basename "$(dirname "$out_file")")
    tumor_id=${pair_dir%%__*}
    normal_id=${pair_dir#*__}

    # Columns: Sample(1) Facets(2) snp.nbhd(3) ndepth(4) purity_cval(5) cval(6)
    #         min.nhet(7) genome(8) Purity(9) Ploidy(10) dipLogR(11) loglik(12) flags(13)
    awk -v t="$tumor_id" -v n="$normal_id" -F'\t' 'NR>1 {
        fit = ($1 ~ /_hisens$/) ? "hisens" : "purity"
        print t "\t" n "\t" fit "\t" $9 "\t" $10 "\t" $11
    }' "$out_file" >> purity_summary.tsv
done
```

## Example: Python Approach

### Collect All MAFs

```python
import pandas as pd
from pathlib import Path

out_dir = Path("outDir/somatic")
maf_files = sorted(out_dir.glob("*/combined_mutations/*.somatic.final.maf"))
print(f"Found {len(maf_files)} MAF files")

maf_list = []
for f in maf_files:
    df = pd.read_csv(f, sep='\t', comment='#', low_memory=False)
    maf_list.append(df)

maf_all = pd.concat(maf_list, ignore_index=True)
print(f"Total mutations: {len(maf_all)}")
print(f"Samples: {maf_all['Tumor_Sample_Barcode'].nunique()}")
```

### Extract Sample IDs from Directory Names

The pair name is `{idTumor}__{idNormal}`. Its position in `Path.parts` depends on the file:

| File | Path index for pair name |
|---|---|
| `somatic/{pair}/combined_mutations/*.somatic.final.maf` | `parts[-3]` |
| `somatic/{pair}/combined_svs/*.final.bedpe` | `parts[-3]` |
| `somatic/{pair}/facets/{pair}/*_OUT.txt` or `*.facets_qc.txt` | `parts[-4]` |
| `somatic/{pair}/facets/{pair}/facets{ver}/*_hisens.Rdata` etc. | `parts[-5]` |

```python
for maf_file in maf_files:
    pair_name = maf_file.parts[-3]  # e.g., "TUMOR_01__NORMAL_01"
    tumor_id, normal_id = pair_name.split("__")
    print(f"Tumor: {tumor_id}, Normal: {normal_id}")
```

### Collect FACETS Purity and Ploidy

Use `*_OUT.txt` (TSV), **not** `*_purity.out` / `*_hisens.out` (comment-style text). Each `_OUT.txt` has two rows: one for the hisens fit (`Sample` ends in `_hisens`) and one for the purity fit (`_purity`).

```python
out_files = sorted(out_dir.glob("*/facets/*/*_OUT.txt"))

records = []
for f in out_files:
    pair_name = f.parts[-4]                     # somatic/<pair>/facets/<pair>/<pair>_OUT.txt
    tumor_id, normal_id = pair_name.split("__")
    df = pd.read_csv(f, sep='\t')               # cols: Sample, Facets, ..., Purity, Ploidy, dipLogR, ...
    for _, row in df.iterrows():
        fit = "hisens" if row['Sample'].endswith('_hisens') else "purity"
        records.append({
            'tumor_id': tumor_id,
            'normal_id': normal_id,
            'fit': fit,
            'purity': row['Purity'],
            'ploidy': row['Ploidy'],
            'dipLogR': row['dipLogR'],
        })

purity_summary = pd.DataFrame(records)
print(purity_summary)
purity_summary.to_csv("purity_summary.csv", index=False)
```

### Read the FACETS Best-Fit QC for Each Pair

```python
qc_files = sorted(out_dir.glob("*/facets/*/*.facets_qc.txt"))   # NOT *.qc.txt
qc_all = pd.concat(
    (pd.read_csv(f, sep='\t') for f in qc_files),
    ignore_index=True
)
# Single row per pair. Key columns: purity, ploidy, dipLogR, wgd,
# facets_qc (overall TRUE/FALSE), and the *_filter_pass booleans.
print(qc_all[['tumor_sample_id', 'purity', 'ploidy', 'wgd', 'facets_qc']].head())
```

## Cohort-Level Aggregated Files

If Tempo was run with the `--aggregate` flag, pre-combined cohort-level outputs are available in:

```
outDir/cohort_level/
```

This directory contains merged MAFs, aggregated QC metrics, and combined signature results. Use these files instead of manually merging per-pair outputs when available.

## See Also

- [load-maf-python.md](load-maf-python.md) -- loading individual MAF files
- [merge-maf-clinical.md](merge-maf-clinical.md) -- merging with clinical metadata
- [facets-qc-r.md](facets-qc-r.md) -- quality-checking FACETS results
