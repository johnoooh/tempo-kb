# Batch Processing Multiple Tempo Output Samples

> **Quick answer:** Use glob patterns to iterate over Tempo output directories and collect results across all tumor-normal pairs. Extract sample IDs from directory names by splitting on the double-underscore (`__`) separator.

## Why Batch Process

Tempo produces outputs organized per tumor-normal pair under `outDir/somatic/`. When you have tens or hundreds of pairs, you need systematic approaches to collect MAFs, FACETS results, QC metrics, and other outputs into unified tables for cohort-level analysis.

## Output Directory Structure

Each tumor-normal pair has its own subdirectory:

```
outDir/somatic/
  TUMOR_01__NORMAL_01/
    combined_mutations/
      TUMOR_01__NORMAL_01.somatic.final.maf
    facets/
      TUMOR_01__NORMAL_01_hisens.Rdata
      TUMOR_01__NORMAL_01_purity.out
      facets_qc.txt
  TUMOR_02__NORMAL_02/
    combined_mutations/
      TUMOR_02__NORMAL_02.somatic.final.maf
    facets/
      ...
```

## File Locations (Glob Patterns)

| Output | Glob Pattern |
|--------|-------------|
| Final MAF | `outDir/somatic/*/combined_mutations/*.somatic.final.maf` |
| FACETS purity file | `outDir/somatic/*/facets/*_purity.out` |
| FACETS hisens Rdata | `outDir/somatic/*/facets/*_hisens.Rdata` |
| FACETS QC | `outDir/somatic/*/facets/facets_qc.txt` |

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

```bash
echo -e "tumor_id\tnormal_id\tpurity\tploidy" > purity_summary.tsv

for purity_file in ${OUT_DIR}/*/facets/*_purity.out; do
    pair_dir=$(basename $(dirname $(dirname "$purity_file")))
    tumor_id=$(echo "$pair_dir" | cut -d'_' -f1-2 --output-delimiter='_')
    normal_id=$(echo "$pair_dir" | awk -F'__' '{print $2}')
    tumor_id=$(echo "$pair_dir" | awk -F'__' '{print $1}')

    # Parse purity and ploidy from the purity.out file
    purity=$(awk 'NR==2{print $1}' "$purity_file")
    ploidy=$(awk 'NR==2{print $2}' "$purity_file")

    echo -e "${tumor_id}\t${normal_id}\t${purity}\t${ploidy}" >> purity_summary.tsv
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

```python
for maf_file in maf_files:
    pair_name = maf_file.parts[-3]  # e.g., "TUMOR_01__NORMAL_01"
    tumor_id, normal_id = pair_name.split("__")
    print(f"Tumor: {tumor_id}, Normal: {normal_id}")
```

### Collect FACETS Purity and Ploidy

```python
purity_files = sorted(out_dir.glob("*/facets/*_purity.out"))

records = []
for f in purity_files:
    pair_name = f.parts[-3]
    tumor_id, normal_id = pair_name.split("__")
    purity_df = pd.read_csv(f, sep='\t')
    records.append({
        'tumor_id': tumor_id,
        'normal_id': normal_id,
        'purity': purity_df.iloc[0].get('Purity', None),
        'ploidy': purity_df.iloc[0].get('Ploidy', None),
    })

purity_summary = pd.DataFrame(records)
print(purity_summary)
purity_summary.to_csv("purity_summary.csv", index=False)
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
