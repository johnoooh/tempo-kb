# HRDetect: Predicting Homologous Recombination Deficiency in Tempo

> **Quick answer:** HRDetect is a genome-only lasso logistic regression classifier that combines six mutational features (SBS3, SBS8, SV signatures, HRD index, microhomology deletions, and short indels at repeats) to predict BRCA1/2 deficiency. Tempo runs HRDetect using the signaturetoolslib R package with inputs from the filtered MAF, FACETS CNV, and merged SV BEDPE.

## What HRDetect Does

Homologous recombination deficiency (HRD) is a clinically important phenotype associated with sensitivity to platinum-based chemotherapy and PARP inhibitors. While BRCA1/2 mutations are the most well-known cause of HRD, many tumors have HRD through other mechanisms (epigenetic silencing, mutations in other HR pathway genes).

HRDetect integrates multiple orthogonal mutational features that are consequences of defective homologous recombination, providing a comprehensive phenotypic readout rather than relying on genotype alone.

**Key paper:** Davies et al., 2017, Nature Medicine.

## The Six Features

HRDetect uses a lasso logistic regression model trained on breast cancers with known BRCA1/2 status. The six input features are:

1. **SBS3 (Signature 3) exposure:** A flat mutational signature associated with failure of double-strand break repair by homologous recombination. This is typically the single strongest predictor of HRD.

2. **SBS8 (Signature 8) exposure:** A mutational signature of uncertain etiology that correlates with HRD, potentially reflecting a secondary mutagenic process in HR-deficient cells.

3. **SV signature exposures:** Structural variant signatures, particularly SV signature 3 (tandem duplications of 1-10kb) and SV signature 5 (deletions of 1-10kb), which are enriched in BRCA1-deficient and BRCA2-deficient tumors respectively.

4. **HRD index:** A composite score derived from copy number data, combining three genomic scar metrics:
   - **LOH (Loss of Heterozygosity):** Number of LOH regions longer than 15Mb.
   - **TAI (Telomeric Allelic Imbalance):** Number of regions with allelic imbalance extending to the telomere.
   - **LST (Large-Scale State Transitions):** Number of breakpoints between adjacent segments of at least 10Mb after smoothing.

5. **Microhomology-mediated deletions:** The proportion of deletions with microhomology at the breakpoint junction. HR-deficient cells rely on alternative repair pathways (like MMEJ/theta-mediated end joining) that leave microhomology signatures.

6. **Short insertions and deletions at repeats:** The count of small indels occurring at repetitive sequences, reflecting replication-associated errors that accumulate when HR is impaired.

## Tempo Implementation

### When It Runs

HRDetect runs only when `params.assayType == "genome"`. It requires whole-genome data because several features (SV signatures, genome-wide HRD index) cannot be reliably computed from exome data.

### Input Files

The `HRDetect` process takes three inputs per sample:

1. **Somatic MAF** (`mafFile`): The filtered somatic MAF from the SNV workflow, providing the mutation catalog for SBS signature extraction and indel characterization.
2. **Copy number file** (`cnvFile`): FACETS copy number segments (from the `--svcnv` selected source -- hisens or purity filtered CNV). Used to compute the HRD index (LOH, TAI, LST).
3. **SV BEDPE** (`svFile`): The final filtered BEDPE from the SV workflow. Used to extract SV signatures.

### How It Runs

The process creates a sample manifest TSV linking the three input files, then runs an R script that uses the `signaturetoolslib` package to:

1. Extract the trinucleotide mutation catalog from the MAF.
2. Fit COSMIC mutational signatures (extracting SBS3 and SBS8 exposures).
3. Classify SVs and fit SV signatures from the BEDPE.
4. Compute the HRD index from the CNV segments.
5. Count microhomology deletions and repeat-associated indels from the MAF.
6. Apply the lasso logistic regression model to produce the final HRDetect probability.

The genome version (hg19 or hg38) is derived from `params.genome`.

### SV Signature Analysis

The `RunSVSignatures` process runs separately to characterize SV signatures from the BEDPE file, producing:
- A catalogue PDF showing the SV classification spectrum.
- An exposures TSV with the contribution of each SV signature.

These SV signatures feed into the HRDetect model.

## Interpreting the Output

The output file `*.hrdetect.tsv` contains the HRDetect probability score (0 to 1):

- **Score >= 0.7:** Strong prediction of BRCA1/2 deficiency (used as threshold in the original publication).
- **Score 0.3-0.7:** Uncertain; may warrant further investigation.
- **Score < 0.3:** Likely HR-proficient.

The TSV also includes the individual feature values, allowing inspection of which features contribute most to the prediction.

## File Locations

- HRDetect process: `tempo/modules/process/HRDetect/HRDetect.nf`
- SV signatures: `tempo/modules/process/HRDetect/RunSVSignatures.nf`
- Output: `<outDir>/somatic/<tumor>__<normal>/hrdetect/<tumor>__<normal>.hrdetect.tsv`
- SV signature output: `<outDir>/somatic/<tumor>__<normal>/combined_svs/<tumor>__<normal>_exposures.tsv`

## Example

```bash
# HRDetect output
cat TUMOR__NORMAL.hrdetect.tsv
# sample  Probability  SBS3  SBS8  SV3  SV5  HRDindex  del.mh  del.rep

# SV signature exposures
cat TUMOR__NORMAL_exposures.tsv
# sample  SV_sig1  SV_sig2  SV_sig3  SV_sig4  SV_sig5  SV_sig6
```

## See Also

- `facets-algorithm.md` -- CNV input for HRD index computation
- `signatures-msi.md` -- tempoSig runs separately for per-sample signature analysis
- `sv-callers.md` -- SV BEDPE input for SV signature extraction
- `copy-number-facets-vs-ascat.md` -- The `--svcnv` flag controls which CNV feeds HRDetect
