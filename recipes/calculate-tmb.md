# Calculating Tumor Mutational Burden (TMB) from Tempo MAF Files

> **Quick answer:** TMB is the count of non-silent somatic mutations per megabase of coding sequence. Count qualifying variants in the filtered Tempo MAF and divide by the size of the targeted region.

## What Is TMB

Tumor Mutational Burden (TMB) quantifies the total number of somatic coding mutations in a tumor sample, normalized to the size of the sequenced region. TMB is a biomarker used in immunotherapy response prediction. Higher TMB is associated with increased neoantigen load and potentially better response to immune checkpoint inhibitors.

## Formula

```
TMB = (number of non-silent somatic mutations) / (size of coding region in megabases)
```

## Non-Silent Variant Classifications

Count only mutations with these `Variant_Classification` values:

- `Missense_Mutation`
- `Nonsense_Mutation`
- `Frame_Shift_Del`
- `Frame_Shift_Ins`
- `In_Frame_Del`
- `In_Frame_Ins`
- `Splice_Site`
- `Translation_Start_Site`
- `Nonstop_Mutation`

Silent mutations (e.g., `Silent`, `Intron`, `3'UTR`, `5'UTR`, `IGR`) are excluded from the TMB calculation.

## Coding Region Size

<!-- TODO: VERIFY WITH USER -- exact values depend on bait set -->

The denominator depends on your assay type:

| Assay | Approximate Coding Region | Notes |
|-------|--------------------------|-------|
| WES (exome) | ~30 Mb | Varies by bait set (IDT vs Agilent) |
| WGS (genome) | ~2800 Mb | Whole genome callable region |

The exact value for exome depends on the specific bait set used in your Tempo run. Check with your sequencing facility for the precise capture region size.

## File Locations

Use the final filtered MAF file:

```
outDir/somatic/{idTumor}__{idNormal}/combined_mutations/{idTumor}__{idNormal}.somatic.final.maf
```

Do not use the unfiltered MAF, as it contains artifacts and low-confidence calls that inflate TMB.

## Example: Python

```python
import pandas as pd

maf_path = "outDir/somatic/TUMOR__NORMAL/combined_mutations/TUMOR__NORMAL.somatic.final.maf"
maf = pd.read_csv(maf_path, sep='\t', comment='#', low_memory=False)

non_silent = [
    'Missense_Mutation', 'Nonsense_Mutation', 'Frame_Shift_Del',
    'Frame_Shift_Ins', 'In_Frame_Del', 'In_Frame_Ins',
    'Splice_Site', 'Translation_Start_Site', 'Nonstop_Mutation'
]

maf_nonsilent = maf[maf['Variant_Classification'].isin(non_silent)]

# For exome (~30 Mb)
CODING_REGION_MB = 30  # Adjust for your bait set
tmb = len(maf_nonsilent) / CODING_REGION_MB
print(f"Non-silent mutations: {len(maf_nonsilent)}")
print(f"TMB: {tmb:.2f} mutations/Mb")
```

For a cohort of samples:

```python
tmb_per_sample = (
    maf[maf['Variant_Classification'].isin(non_silent)]
    .groupby('Tumor_Sample_Barcode')
    .size()
    .div(CODING_REGION_MB)
    .reset_index(name='TMB')
)
print(tmb_per_sample.sort_values('TMB', ascending=False))
```

## Example: R

```r
library(data.table)

maf <- fread("outDir/somatic/TUMOR__NORMAL/combined_mutations/TUMOR__NORMAL.somatic.final.maf",
             skip = "#version")  # skip comment lines

non_silent <- c(
  "Missense_Mutation", "Nonsense_Mutation", "Frame_Shift_Del",
  "Frame_Shift_Ins", "In_Frame_Del", "In_Frame_Ins",
  "Splice_Site", "Translation_Start_Site", "Nonstop_Mutation"
)

maf_ns <- maf[Variant_Classification %in% non_silent]

CODING_REGION_MB <- 30  # Adjust for your bait set
tmb <- nrow(maf_ns) / CODING_REGION_MB
cat(sprintf("Non-silent mutations: %d\nTMB: %.2f mutations/Mb\n", nrow(maf_ns), tmb))
```

## Interpretation Guidelines

| TMB Range (mut/Mb) | Category | Notes |
|---------------------|----------|-------|
| < 5 | Low | Below average |
| 5 - 10 | Intermediate | Near average for many cancer types |
| 10 - 20 | High | May indicate checkpoint inhibitor benefit |
| > 20 | Very high | Strong immunotherapy candidacy (e.g., melanoma, MSI-H) |

These thresholds are general guidelines. The FDA-approved TMB cutoff for pembrolizumab is 10 mut/Mb (using the FoundationOne CDx assay), but thresholds vary by context.

## See Also

- [load-maf-python.md](load-maf-python.md) -- loading and exploring MAF files
- [calculate-vaf.md](calculate-vaf.md) -- computing variant allele frequency
- [genome-vs-exome.md](genome-vs-exome.md) -- exome vs genome region sizes
