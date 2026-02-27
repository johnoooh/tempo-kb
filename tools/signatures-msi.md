# Mutation Signatures and Microsatellite Instability in Tempo

> **Quick answer:** Tempo uses tempoSig for mutational signature decomposition (COSMIC v2 or v3 signatures via maximum-likelihood estimation) and MSIsensor for microsatellite instability scoring (chi-square test comparing tumor vs. normal at microsatellite loci). Both tools run on exome and genome data.

## tempoSig: Mutational Signature Analysis

### What It Does

tempoSig decomposes the trinucleotide mutation spectrum of a tumor sample into contributions from known mutational processes (COSMIC signatures). Each COSMIC signature represents a distinct mutational process such as aging (SBS1), APOBEC activity (SBS2/SBS13), UV damage (SBS7), or homologous recombination deficiency (SBS3).

### Algorithm

tempoSig uses maximum-likelihood (ML) estimation to fit signature exposures to the observed trinucleotide mutation catalog. The approach:

1. **Build trinucleotide catalog:** The `maf2cat2.R` script extracts the 96-class trinucleotide context matrix from the filtered somatic MAF file using the `Ref_Tri` column.
2. **Fit signatures:** The `tempoSig.R` script fits the catalog against the reference signature matrix using ML decomposition.
3. **Significance testing:** With `--pvalue --nperm 10000`, tempoSig runs 10,000 permutations to assess the statistical significance of each signature's contribution.

### COSMIC Versions

Tempo supports two COSMIC signature sets, controlled by the `--cosmic` parameter (default: `v3`):

- **COSMIC v2:** 30 signatures (SBS1-SBS30). The original widely-used set.
- **COSMIC v3:** 60+ SBS signatures with refined definitions and additional signatures for rare processes.

The version is set in `nextflow.config` as `params.cosmic = 'v3'`.

### Output

The output file `*.mutsig.txt` contains the exposure (contribution) of each COSMIC signature for the sample, along with p-values indicating whether each signature's contribution is statistically significant above background.

### Interpreting Results

Key signatures to look for:
- **SBS1 (clock-like):** Correlates with age; expected in all samples.
- **SBS3 (HRD):** Associated with BRCA1/2 deficiency and homologous recombination defects.
- **SBS4 (tobacco):** Associated with smoking; expected in lung cancers.
- **SBS6, SBS15, SBS26 (MMR):** Associated with mismatch repair deficiency and MSI.
- **SBS7a/b (UV):** Associated with UV exposure; expected in melanoma.

Low mutation counts (fewer than approximately 50 SNVs) reduce the reliability of signature decomposition. The p-values help distinguish real contributions from noise.

## MSIsensor: Microsatellite Instability

### What It Does

MSIsensor detects microsatellite instability (MSI) by comparing the length distribution of microsatellite loci between tumor and matched normal samples. MSI is a hallmark of mismatch repair (MMR) deficiency, relevant for immunotherapy eligibility and Lynch syndrome screening.

### Algorithm

**Key paper:** Niu et al., 2014, Bioinformatics.

MSIsensor works in two steps:

1. **Scan:** A pre-computed list of microsatellite loci (provided as the `msiSensorList` reference) defines homopolymer and dinucleotide repeat sites across the genome.
2. **Score:** At each locus, MSIsensor compares the distribution of repeat lengths between tumor and normal BAMs using a chi-square test. The final MSI score is the percentage of unstable loci (those with significantly different length distributions).

### Score Interpretation

The MSI score ranges from 0 to 100 (percentage of unstable microsatellite loci):

<!-- TODO: VERIFY WITH USER - exact thresholds may vary by institution -->
- **MSI-H (high):** Score >= 10 (commonly used threshold) or >= 3.5 (some institutions use lower thresholds for WES).
- **MSS (stable):** Score < 3.5.
- **MSI-L (low):** Intermediate scores between MSS and MSI-H thresholds.

The appropriate threshold depends on the assay type and institutional standards. WGS data typically has more microsatellite loci evaluated, potentially requiring different thresholds than WES.

### Tempo Implementation

The `RunMsiSensor` process runs `msisensor msi` with the pre-computed locus list, tumor BAM, and normal BAM. The output is a TSV file containing the overall MSI score and per-locus results.

## File Locations

- Mutation signatures: `tempo/modules/process/MutSig/RunMutationSignatures.nf`
- MSIsensor: `tempo/modules/process/MSI/RunMsiSensor.nf`
- COSMIC version setting: `tempo/nextflow.config` (line 48)
- MSIsensor output: `<outDir>/somatic/<tumor>__<normal>/` (collected via metadata parser)
- Signature output: `<outDir>/somatic/<tumor>__<normal>/` (collected via metadata parser)

## Example

```bash
# tempoSig output: signature exposures per sample
cat TUMOR__NORMAL.mutsig.txt
# Columns: sample, SBS1, SBS2, SBS3, ..., SBS1_pvalue, SBS2_pvalue, ...

# MSIsensor output
cat TUMOR__NORMAL.msisensor.tsv
# Total_Number_of_Sites  Number_of_Somatic_Sites  %
# 423156                 127                       0.03
```

## See Also

- `maf-filtering.md` -- The filtered MAF is the input for tempoSig
- `hrdetect.md` -- Uses SBS3/SBS8 signature exposures as features for HRD prediction
- `maf-format.md` -- The Ref_Tri column provides trinucleotide context
