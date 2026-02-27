# SVclone: Structural Variant Clonality Estimation in Tempo

> **Quick answer:** SVclone is a genome-only tool that estimates the cancer cell fraction (CCF) of structural variants using the CCube framework, then co-clusters SVs with SNVs to characterize clonal architecture. It requires BAMs, the filtered BEDPE, filtered MAF, FACETS copy number, and purity/ploidy estimates.

## What SVclone Does

SVclone addresses a key question in tumor biology: which structural variants are clonal (present in all cancer cells) and which are subclonal (present in only a fraction of cancer cells)? This information is critical for:

- Understanding tumor evolution and the order in which mutations were acquired.
- Identifying driver SVs that are clonal and thus likely early, selected events.
- Integrating SV clonality with SNV clonality for comprehensive clonal architecture reconstruction.

**Key paper:** Cmero et al., 2020, Nature Communications.

## Algorithm

SVclone uses the CCube framework (a Bayesian clustering approach) to estimate CCF for structural variants:

### Step 1: Read Count Extraction

For each SV breakpoint in the BEDPE, SVclone extracts:
- The number of reads supporting the variant (split reads and discordant pairs spanning the breakpoint).
- The total read depth at the breakpoint locus.

This is done directly from the tumor BAM file.

### Step 2: CCF Estimation

Using the read counts, local copy number (from FACETS), and sample purity/ploidy, SVclone estimates the CCF for each SV. The CCF calculation accounts for:
- The expected variant allele fraction given the copy number state.
- The tumor purity (fraction of tumor cells in the sample).
- The local ploidy at each breakpoint.

### Step 3: Joint SNV-SV Clustering

SVclone co-clusters SVs and SNVs based on their estimated CCFs using the CCube Bayesian mixture model. This joint clustering:
- Groups mutations (both SVs and SNVs) that exist at the same cellular frequency into clonal clusters.
- Identifies the major clonal population and any subclonal populations.
- Provides cluster assignments with certainty estimates for each variant.

## Tempo Implementation

### When It Runs

SVclone runs only for genome (WGS) data, as part of the `clonality_wf` workflow. It requires both the SV and SNV workflows to have completed.

### Inputs

The `SomaticRunSVclone` process takes:

1. **Tumor and normal BAMs** (with indices): For read count extraction at SV breakpoints.
2. **Filtered BEDPE** (`inBedpe`): The PASS structural variants from the SV workflow.
3. **Filtered MAF** (`mafFiltered`): The PASS somatic mutations from the SNV workflow.
4. **Copy number segments** (`cnv`): FACETS copy number calls for local copy number at each variant.
5. **Purity/ploidy file** (`ploidyIn`): Sample purity and ploidy estimates from FACETS.

### Processing Steps

A Python wrapper script (`svclone_wrapper`) prepares the inputs:
1. Converts the BEDPE and MAF into SVclone-compatible formats.
2. Generates a configuration file from the template (`/config/svclone_config.ini`).
3. Sets up the output directory structure.

SVclone then runs the CCube clustering, producing results in the `ccube_out/post_assign/` directory.

### Output Organization

Tempo copies and reformats the SVclone results into a structured output:

```
svclone/
  svs/
    *.cluster_certainty.txt   # SV cluster assignments with certainty
    *.RData                   # R objects with full model fit
    *.pdf                     # Visualization plots
  snvs/
    *.cluster_certainty.txt   # SNV cluster assignments with certainty
    *.RData                   # R objects with full model fit
    *.pdf                     # Visualization plots
```

Each `cluster_certainty.txt` file is prefixed with the sample ID (`sampleid` column) for aggregation across multiple samples.

## Interpreting Results

The `*cluster_certainty.txt` files contain per-variant cluster assignments with columns for variant identifier, cluster ID, CCF estimate, and certainty score. Key patterns:

- **One dominant cluster near CCF=1.0:** Most variants are clonal, consistent with a simple tumor.
- **Multiple clusters:** Indicates subclonal structure with distinct populations.
- **SVs in the clonal cluster:** Likely early events in tumor evolution.
- **SVs in subclonal clusters:** May represent ongoing instability or late-arising changes.

## Limitations

- Requires sufficient SV and SNV counts for reliable clustering (very few variants lead to unreliable clusters).
- Accuracy depends on the quality of the FACETS purity/ploidy and copy number estimates.
- Complex SVs near copy number breakpoints may have unreliable CCF estimates.
- Only available for WGS data.

## File Locations

- SVclone process: `tempo/modules/process/SVclone/SomaticRunSVclone.nf`
- Output: `<outDir>/somatic/<tumor>__<normal>/svclone/`
- SV cluster results: `<outDir>/somatic/<tumor>__<normal>/svclone/svs/`
- SNV cluster results: `<outDir>/somatic/<tumor>__<normal>/svclone/snvs/`

## Example

```bash
# SV cluster certainty output
head svclone/svs/*cluster_certainty.txt
# sampleid  variant  cluster  ccf  certainty
# TUMOR__NORMAL  TEMPO_DEL_chr1_1000_chr1_50000_+-  1  0.95  0.87
# TUMOR__NORMAL  TEMPO_DUP_chr3_5000_chr3_80000_++  2  0.42  0.73

# SNV cluster certainty output
head svclone/snvs/*cluster_certainty.txt
# sampleid  variant  cluster  ccf  certainty
# TUMOR__NORMAL  chr1:12345:A>T  1  0.98  0.91
```

## See Also

- `sv-callers.md` -- How SVs are called and merged before SVclone
- `sv-format.md` -- BEDPE format that serves as SVclone input
- `facets-algorithm.md` -- Purity/ploidy and copy number inputs for CCF estimation
- `hrdetect.md` -- Another genome-only analysis using SV and CNV data
