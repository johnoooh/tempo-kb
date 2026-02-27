# Tempo QC: Conpair Concordance and Contamination Analysis

> **Quick answer:** Conpair checks that tumor and normal samples in a pair come from the same patient (concordance) and estimates cross-sample contamination levels. Results are written to `outDir/somatic/{idTumor}__{idNormal}/conpair/` as concordance and contamination text files.

## What Conpair Does

Conpair is a two-step QC tool designed for paired tumor-normal sequencing analysis:

1. **Concordance verification** -- Confirms that the tumor and normal BAMs originate from the same individual by comparing genotypes at a panel of common SNP markers. This catches sample swaps, mislabeling, and pairing errors.

2. **Contamination estimation** -- Estimates the level of cross-sample DNA contamination in both the tumor and normal BAMs independently.

Conpair is critical for paired analyses because a sample swap or significant contamination can silently corrupt variant calls, copy number profiles, and all downstream results.

## How Conpair Works in Tempo

The analysis proceeds in two stages across separate Nextflow processes:

**Stage 1: Pileup generation (`QcPileup`)**

For each sample (tumor and normal independently), GATK pileup is run at a curated set of common SNP marker positions. These markers are high-MAF, well-separated SNPs selected from the 1000 Genomes Phase 3 data. The pileup files are saved to `outDir/bams/{SAMPLE_ID}/pileup/{SAMPLE_ID}.pileup`.

```bash
run_gatk_pileup_for_sample.py \
  --gatk={GATK_jar} \
  --bam={bam} \
  --markers={markers.bed} \
  --reference={genome.fa} \
  --outfile={SAMPLE_ID}.pileup
```

**Stage 2: Concordance and contamination (`QcConpair`)**

The tumor and normal pileup files are compared:

```bash
# Concordance
verify_concordances.py \
  --tumor_pileup={tumor.pileup} \
  --normal_pileup={normal.pileup} \
  --markers={markers.txt} \
  --normal_homozygous_markers_only \
  --outpre={idTumor}__{idNormal}

# Contamination
estimate_tumor_normal_contaminations.py \
  --tumor_pileup={tumor.pileup} \
  --normal_pileup={normal.pileup} \
  --markers={markers.txt} \
  --outpre={idTumor}__{idNormal}
```

The `--normal_homozygous_markers_only` flag restricts concordance analysis to positions where the normal is homozygous, improving accuracy.

**Cross-pair checking (`QcConpairAll`)**: Tempo also runs Conpair in an all-vs-all mode across every tumor-normal combination in the cohort. This helps detect unexpected relatedness or contamination between samples from different patients.

## Interpreting Results

**Concordance score** (in `{idTumor}__{idNormal}.concordance.txt`):
- A value near **99.0% or higher** indicates the tumor and normal come from the same patient.
- Values below **80%** strongly suggest a sample swap or mislabeling.
- Values in the **80-99%** range may warrant investigation -- they can indicate partial contamination or samples from related individuals.

**Contamination estimates** (in `{idTumor}__{idNormal}.contamination.txt`):
- Reports contamination percentage for both tumor and normal independently.
- Normal contamination above **1-2%** is a concern and may affect germline variant calls and somatic subtraction.
- Tumor contamination is harder to interpret due to tumor heterogeneity but very high values (above **5-10%**) suggest cross-sample contamination.

## When to Check Conpair

- **Always review concordance** before trusting somatic variant calls. Sample swaps are one of the most consequential errors in paired analysis.
- Check contamination when variant calls include an unexpectedly high number of low-VAF mutations.
- Review the cohort-level aggregated concordance and contamination tables (`outDir/cohort_level/{cohort}/concordance_qc.txt` and `contamination_qc.txt`) for a quick overview of all pairs.

## File Locations

```
outDir/bams/{SAMPLE_ID}/pileup/
└── {SAMPLE_ID}.pileup                                    # SNP pileup (intermediate)

outDir/somatic/{idTumor}__{idNormal}/conpair/
├── {idTumor}__{idNormal}.concordance.txt                  # Genotype concordance score
└── {idTumor}__{idNormal}.contamination.txt                # Contamination estimates

outDir/cohort_level/{cohort}/
├── concordance_qc.txt                                     # Aggregated concordance
└── contamination_qc.txt                                   # Aggregated contamination
```

## Example

```bash
# Check concordance for a tumor-normal pair
cat outDir/somatic/TUMOR__NORMAL/conpair/TUMOR__NORMAL.concordance.txt
# Output format (2 columns):
# concordance  TUMOR_ID
# NORMAL_ID    99.92

# Check contamination
cat outDir/somatic/TUMOR__NORMAL/conpair/TUMOR__NORMAL.contamination.txt
# Output format (3 columns):
# Sample_Type  Sample_ID      Contamination
# N            NORMAL_ID      0.112
# T            TUMOR_ID       0.249

# Quick cohort overview
column -t outDir/cohort_level/default_cohort/concordance_qc.txt
column -t outDir/cohort_level/default_cohort/contamination_qc.txt
```

## See Also

- `bam-files.md` for the pileup subdirectory location within the BAM directory
- `directory-structure.md` for the somatic output directory layout
- `qc-multiqc.md` for how Conpair metrics appear in MultiQC reports
