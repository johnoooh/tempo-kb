# Extracting Cancer Cell Fraction and Clonality from Tempo MAF

> **Quick answer:** The FACETS-annotated final MAF contains cancer cell fraction (CCF) columns that estimate the fraction of cancer cells carrying each mutation. Mutations with CCF near 1.0 are clonal (present in all cancer cells), while those with CCF below approximately 0.85 are subclonal.

## What Is Cancer Cell Fraction

Cancer Cell Fraction (CCF) estimates the proportion of cancer cells in a sample that carry a given somatic mutation. Unlike VAF, CCF corrects for tumor purity and local allele-specific copy number, providing a biologically meaningful estimate of whether a mutation is present in all tumor cells (clonal) or only a subset (subclonal).

CCF is computed during Tempo's FACETS annotation step, which integrates variant allele counts from the MAF with copy number and purity estimates from FACETS.

## Key MAF Columns

The final Tempo MAF includes these CCF-related columns:

| Column | Description |
|--------|-------------|
| `ccf_expected_copies` | Point estimate of CCF |
| `ccf_expected_copies_lower` | Lower bound of CCF 95% confidence interval |
| `ccf_expected_copies_upper` | Upper bound of CCF 95% confidence interval |
| `clonality` | Pre-computed clonality call (if available) |

These columns are only populated in the final MAF (`*.somatic.final.maf`) because CCF computation depends on FACETS purity and copy number results.

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/combined_mutations/{idTumor}__{idNormal}.somatic.final.maf
```

CCF columns are absent from unfiltered or pre-FACETS MAF files.

## Example: Loading CCF Data

```python
import pandas as pd

maf_path = "outDir/somatic/TUMOR__NORMAL/combined_mutations/TUMOR__NORMAL.somatic.final.maf"
maf = pd.read_csv(maf_path, sep='\t', comment='#', low_memory=False)  # comment='#' is safe even though Tempo MAFs have no comment lines

ccf_cols = [
    'Hugo_Symbol', 'HGVSp_Short', 'Variant_Classification',
    'Tumor_Sample_Barcode', 't_alt_count', 't_depth',
    'ccf_expected_copies', 'ccf_expected_copies_lower',
    'ccf_expected_copies_upper'
]

# Select rows with valid CCF values
maf_ccf = maf[ccf_cols].dropna(subset=['ccf_expected_copies']).copy()
print(f"Mutations with CCF estimates: {len(maf_ccf)} out of {len(maf)}")
print(maf_ccf['ccf_expected_copies'].describe())
```

## Classifying Clonal vs Subclonal Mutations

The conventional threshold for clonality varies across studies. A common approach:

- **Clonal**: CCF >= 0.85 (mutation present in most or all cancer cells)
- **Subclonal**: CCF < 0.85 (mutation present in a subset of cancer cells)

Some studies use more conservative thresholds (e.g., CCF >= 0.90) or incorporate the confidence interval.

```python
CCF_THRESHOLD = 0.85

maf_ccf['clonality_category'] = maf_ccf['ccf_expected_copies'].apply(
    lambda x: 'clonal' if x >= CCF_THRESHOLD else 'subclonal'
)

print(maf_ccf['clonality_category'].value_counts())
```

### Using the Confidence Interval

A more conservative classification uses the lower bound of the confidence interval:

```python
def classify_clonality(row, threshold=0.85):
    """Classify using confidence interval for robustness."""
    ccf = row['ccf_expected_copies']
    ccf_lower = row['ccf_expected_copies_lower']
    ccf_upper = row['ccf_expected_copies_upper']

    if pd.isna(ccf):
        return 'unknown'
    elif ccf_lower >= threshold:
        return 'clonal'        # confidently clonal
    elif ccf_upper < threshold:
        return 'subclonal'     # confidently subclonal
    else:
        return 'indeterminate' # CI spans the threshold

maf_ccf['clonality_strict'] = maf_ccf.apply(classify_clonality, axis=1)
print(maf_ccf['clonality_strict'].value_counts())
```

## Summary Statistics

```python
# Fraction of clonal mutations per sample
clonal_frac = (
    maf_ccf
    .groupby('Tumor_Sample_Barcode')
    .apply(lambda x: (x['ccf_expected_copies'] >= CCF_THRESHOLD).mean())
    .reset_index(name='clonal_fraction')
)
print(clonal_frac)

# Commonly clonal genes
clonal_genes = (
    maf_ccf[maf_ccf['clonality_category'] == 'clonal']
    ['Hugo_Symbol'].value_counts().head(20)
)
print("Most frequently clonal genes:")
print(clonal_genes)
```

## Important Notes

- CCF values greater than 1.0 can occur due to uncertainty in the copy number model. These are typically capped or treated as clonal.
- CCF estimates are unreliable when FACETS purity is below 0.2. Check FACETS QC before interpreting CCF.
- Not all mutations receive a CCF value. Mutations in regions where FACETS could not determine copy number will have missing CCF.

## See Also

- [calculate-vaf.md](calculate-vaf.md) -- VAF calculation (CCF adjusts VAF for purity/CN)
- [facets-qc-r.md](facets-qc-r.md) -- verifying FACETS quality before trusting CCF
- [load-maf-python.md](load-maf-python.md) -- loading MAF files in Python
