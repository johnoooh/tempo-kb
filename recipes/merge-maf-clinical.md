# Merging Tempo MAF Files with Clinical Metadata in Python

> **Quick answer:** Use pandas to merge MAF data with clinical metadata by joining on `Tumor_Sample_Barcode`. Combine multiple MAFs with `pd.concat()` and then merge with a clinical DataFrame to add diagnosis, stage, treatment, or cohort information.

## Why Merge MAF with Clinical Data

Tempo MAF files contain mutation-level information but no clinical annotations. To perform analyses stratified by diagnosis, treatment arm, tumor stage, or any other clinical variable, you need to merge the MAF with an external clinical metadata table. This is a standard step before cohort-level analyses like survival-associated mutation discovery or subgroup oncoplots.

## File Locations

Individual MAF files:
```
outDir/somatic/{idTumor}__{idNormal}/combined_mutations/{idTumor}__{idNormal}.somatic.final.maf
```

Your clinical metadata file should be a CSV or TSV with at least one column that matches `Tumor_Sample_Barcode` in the MAF.

## Example: Reading Multiple MAFs

```python
import pandas as pd
from pathlib import Path

# Find all final MAF files
maf_dir = Path("outDir/somatic/")
maf_files = list(maf_dir.glob("*/combined_mutations/*.somatic.final.maf"))
print(f"Found {len(maf_files)} MAF files")

# Read and combine
maf_list = []
for f in maf_files:
    df = pd.read_csv(f, sep='\t', comment='#', low_memory=False)
    maf_list.append(df)

maf_all = pd.concat(maf_list, ignore_index=True)
print(f"Combined MAF: {len(maf_all)} mutations across "
      f"{maf_all['Tumor_Sample_Barcode'].nunique()} samples")
```

## Example: Merging with Clinical Data

```python
# Load clinical metadata
clinical = pd.read_csv("clinical_metadata.csv")

# Preview the join key
print("Clinical columns:", clinical.columns.tolist())
print("Sample IDs in clinical:", clinical['sample_id'].head())
print("Sample barcodes in MAF:", maf_all['Tumor_Sample_Barcode'].unique()[:5])
```

If the clinical table uses a different column name for the sample identifier, rename it before merging:

```python
# Rename if needed
clinical = clinical.rename(columns={'sample_id': 'Tumor_Sample_Barcode'})

# Merge: left join to keep all mutations
maf_clinical = maf_all.merge(clinical, on='Tumor_Sample_Barcode', how='left')

# Check for unmatched samples
unmatched = maf_clinical[maf_clinical['diagnosis'].isna()]['Tumor_Sample_Barcode'].unique()
if len(unmatched) > 0:
    print(f"Warning: {len(unmatched)} samples had no clinical match:")
    print(unmatched)
```

## Common Operations After Merging

### Filter by Cohort or Diagnosis

```python
# Keep only a specific cancer type
lung_maf = maf_clinical[maf_clinical['diagnosis'] == 'Lung Adenocarcinoma'].copy()
print(f"Lung adenocarcinoma: {len(lung_maf)} mutations, "
      f"{lung_maf['Tumor_Sample_Barcode'].nunique()} samples")
```

### Add Treatment Information

```python
# Flag treated vs untreated
maf_clinical['treatment_group'] = maf_clinical['treatment'].apply(
    lambda x: 'Treated' if pd.notna(x) and x != 'None' else 'Untreated'
)
```

### Per-Group Mutation Counts

```python
# Count mutations per sample, grouped by diagnosis
mutation_counts = (
    maf_clinical
    .groupby(['diagnosis', 'Tumor_Sample_Barcode'])
    .size()
    .reset_index(name='mutation_count')
)

# Summary statistics per diagnosis
summary = mutation_counts.groupby('diagnosis')['mutation_count'].describe()
print(summary)
```

### Gene Mutation Frequency by Group

```python
# Fraction of samples with TP53 mutation, by diagnosis
tp53 = maf_clinical[maf_clinical['Hugo_Symbol'] == 'TP53']
tp53_freq = (
    tp53.groupby('diagnosis')['Tumor_Sample_Barcode']
    .nunique()
    .div(
        maf_clinical.groupby('diagnosis')['Tumor_Sample_Barcode'].nunique()
    )
    .fillna(0)
    .sort_values(ascending=False)
)
print("TP53 mutation frequency by diagnosis:")
print(tp53_freq)
```

## Tips

- Always verify that `Tumor_Sample_Barcode` values in the MAF match your clinical table's sample identifiers exactly. Tempo uses the `idTumor` value from the pairing file as the barcode.
- Use `how='inner'` in the merge if you want only samples present in both the MAF and clinical data.
- When combining many MAF files, memory may become an issue. Select only the columns you need before concatenating.
- If Tempo was run with `--aggregate`, check `outDir/cohort_level/` for a pre-merged cohort MAF.

## See Also

- [load-maf-python.md](load-maf-python.md) -- loading individual MAF files
- [batch-process-samples.md](batch-process-samples.md) -- batch-collecting outputs
- [oncoplot-maftools.md](oncoplot-maftools.md) -- visualizing merged cohort data
