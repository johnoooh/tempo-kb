# HRDetect: Homologous Recombination Deficiency Detection in Tempo (Genome Only)

> **Quick answer:** Tempo runs HRDetect from the `signature.tools.lib` package on whole-genome data to compute a composite score predicting BRCA1/BRCA2 deficiency based on six mutational features derived from SNVs, indels, SVs, and copy number data.

## What HRDetect Measures

HRDetect is a weighted logistic regression model that integrates six genomic features associated with homologous recombination repair (HRR) deficiency:

1. **del.mh.prop**: Proportion of deletions with microhomology at breakpoints. HRR-deficient tumors show elevated microhomology-mediated deletion repair.
2. **SNV3 (SBS3)**: Exposure to COSMIC signature 3, which is strongly associated with BRCA1/2 deficiency.
3. **SV3 (RefSigR3)**: Rearrangement signature 3, associated with tandem duplications characteristic of BRCA1 loss.
4. **SV5 (RefSigR5)**: Rearrangement signature 5, associated with deletions found in BRCA2-deficient tumors.
5. **hrd (HRD-LOH index)**: Count of LOH segments exceeding 15 Mb, derived from copy number data.
6. **SNV8 (SBS8)**: Exposure to COSMIC signature 8, associated with HRR deficiency.

The final HRDetect score ranges from 0 to 1, where scores above 0.7 are typically considered indicative of BRCA1/2 deficiency. This analysis is restricted to whole-genome data (`params.assayType == "genome"`) because exome data lacks sufficient SV and genome-wide CNV resolution.

## How the Process Works

The `HRDetect` process receives three input files per sample: the somatic MAF (SNVs and indels), the copy number file from FACETS, and the filtered somatic BEDPE (structural variants). It also takes the `HRDetect_wrapper.R` script as input.

The wrapper script performs extensive preprocessing:

1. **SV reformatting**: Converts the BEDPE into the format required by `signature.tools.lib`, mapping SV types (BND, INV, DEL, DUP) to rearrangement classes (translocation, inversion, deletion, tandem-duplication).
2. **Mutation splitting**: Separates the MAF into SNVs and indels, adjusting VCF-style allele representations for proper indel classification.
3. **CNV reformatting**: Converts FACETS output to the ASCAT-like format expected by the HRDetect pipeline (columns: seg_no, Chromosome, chromStart, chromEnd, total/minor copy number in normal and tumor).
4. **SNV catalogue construction**: Builds 96-channel trinucleotide substitution catalogues using `tabToSNVcatalogue()`.
5. **Indel classification**: Computes the microhomology deletion proportion using `tabToIndelsClassification()`.
6. **HRDetect pipeline execution**: Calls `HRDetect_pipeline()` with all preprocessed inputs. The function internally fits SNV signatures (COSMIC v2) and SV signatures to compute the six features and the final weighted score.

## Output File

The output file `{idTumor}__{idNormal}.hrdetect.tsv` is a tab-delimited file with one row per sample and columns for each of the six HRDetect features plus the final `Probability` (HRDetect score).

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/hrdetect/{idTumor}__{idNormal}.hrdetect.tsv
```

## Example

```bash
# Example output columns:
# sample  del.mh.prop  SNV3  SV3  SV5  hrd  SNV8  Probability
# T__N    0.234        0.42  0.15 0.08 12   0.03  0.92
```

A Probability of 0.92 strongly suggests BRCA1/2 deficiency in this sample.

## See Also

- `somatic-signatures.md` -- SNV mutational signatures (SBS3, SBS8 are HRDetect features)
- `somatic-sv-signatures.md` -- SV rearrangement signatures (SV3, SV5 are HRDetect features)
- `somatic-facets.md` -- Copy number data used for HRD-LOH index computation
- `cohort-aggregates.md` -- Cohort-level `hrdetect.tsv`
