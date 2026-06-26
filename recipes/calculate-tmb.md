# Calculating Tumor Mutational Burden (TMB) from Tempo MAF Files

> **Quick answer:** TMB is the count of non-silent somatic mutations per megabase of coding sequence. Count qualifying variants in the filtered Tempo MAF and divide by the size of the targeted region.

## What Is TMB

Tumor Mutational Burden (TMB) quantifies the total number of somatic coding mutations in a tumor sample, normalized to the size of the sequenced region. TMB is a biomarker used in immunotherapy response prediction. Higher TMB is associated with increased neoantigen load and potentially better response to immune checkpoint inhibitors.

## Formula

```
TMB = (number of non-silent somatic mutations) / (size of coding region in megabases)
```

## Non-Silent Variant Classifications

Tempo's `MetaDataParser` (`create_metadata_file.py`) counts mutations with these `Variant_Classification` values (verified against the develop branch):

- `Missense_Mutation`
- `Nonsense_Mutation`
- `Nonstop_Mutation`
- `Frame_Shift_Ins`
- `Frame_Shift_Del`
- `In_Frame_Del`
- `In_Frame_Ins`
- `Translation_Start_Site`
- `Splice_Site`
- `Splice_Region`

Silent mutations (e.g., `Silent`, `Intron`, `3'UTR`, `5'UTR`, `IGR`) are excluded from the TMB calculation. Tempo also excludes any row with `Mutation_Status == "GERMLINE"` before counting.

## Coding Region Size (Tempo's exact denominators)

Tempo computes TMB in `MetaDataParser` (`create_metadata_file.py`) using a **coding-sequence (CDS) size in Mb that depends on the bait set**, not the full capture footprint. Use the exact values Tempo uses so your TMB matches the pipeline's `metadata` output:

| Assay / bait set | CDS denominator (Mb) |
|------------------|----------------------|
| Agilent Exon 51MB v3 | **30.89918** |
| IDT Exome v1 FP | **36.00458** |
| WGS (genome) | **45.57229** |

> **Important:** because the denominator differs by bait set, raw TMB values are **not directly comparable across assay types** unless you know which CDS size was used. The WGS value (45.57 Mb) is the coding region Tempo scores against, not the whole 3-Gb genome — Tempo's WGS TMB is still a coding-mutation density.
>
> **Pipeline TMB also intersects mutations with the assay coding-baits BED** before counting (`pybedtools.intersect`), so a naive `len(maf_nonsilent) / CODING_MB` will overcount slightly relative to the pipeline's `TMB` in `sample_data.txt`. To exactly reproduce the pipeline value, intersect with `ensGene.all_CODING_exons.reference.bed` for your bait set; the simpler `len()/Mb` recipe below is a close-but-not-identical approximation.

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
    'Missense_Mutation', 'Nonsense_Mutation', 'Nonstop_Mutation',
    'Frame_Shift_Ins', 'Frame_Shift_Del', 'In_Frame_Del', 'In_Frame_Ins',
    'Translation_Start_Site', 'Splice_Site', 'Splice_Region'
]

maf_nonsilent = maf[
    maf['Variant_Classification'].isin(non_silent)
    & (maf['Mutation_Status'].astype(str) != 'GERMLINE')
]

# Tempo CDS denominator (Mb) — match your assay: Agilent 30.89918, IDT 36.00458, WGS 45.57229
CODING_REGION_MB = 30.89918  # Agilent Exon 51MB v3
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

maf <- fread("outDir/somatic/TUMOR__NORMAL/combined_mutations/TUMOR__NORMAL.somatic.final.maf")
# Tempo MAFs start directly with the column header. If your MAF has a leading
# `#version` comment, add `skip = "Hugo_Symbol"` to fread.

non_silent <- c(
  "Missense_Mutation", "Nonsense_Mutation", "Nonstop_Mutation",
  "Frame_Shift_Ins", "Frame_Shift_Del", "In_Frame_Del", "In_Frame_Ins",
  "Translation_Start_Site", "Splice_Site", "Splice_Region"
)

maf_ns <- maf[Variant_Classification %in% non_silent &
              (is.na(Mutation_Status) | Mutation_Status != "GERMLINE")]

CODING_REGION_MB <- 30.89918  # Agilent Exon 51MB v3 — match your assay (IDT 36.00458, WGS 45.57229)
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
