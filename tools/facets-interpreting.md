# Interpreting FACETS Output: Copy Number Calls, QC Plots, and Failure Modes

> **Quick answer:** FACETS output includes integer total and minor copy number per segment, purity/ploidy estimates, and diagnostic plots. Check the QC metrics from FACETS Preview, inspect the logR/logOR plots for clean segmentation, and watch for common failure modes like noisy samples, low purity, or incorrect dipLogR.

## Key Output Files

Each FACETS run (hisens and purity) produces several output files. The most important are:

### Segment-Level Copy Number (cncf.txt)

The `*_cncf.txt` file contains one row per genomic segment with these critical columns:

- **ID:** Sample identifier (e.g., `TUMOR__NORMAL_hisens`).
- **chrom, loc.start, loc.end:** Genomic coordinates of the segment.
- **seg:** Segment number.
- **tcn.em:** Total copy number (integer) estimated by the EM algorithm. For a normal diploid region, this is 2.
- **lcn.em:** Minor (lesser) allele copy number. For a heterozygous diploid region, this is 1. A value of 0 indicates loss of heterozygosity (LOH).
- **cf.em:** Cellular fraction (clonality) of the copy number event.
- **cnlr.median:** Median log-ratio for the segment (the logR signal).
- **mafR:** Log-odds-ratio value for the segment.
- **num.mark:** Number of SNPs in the segment.
- **nhet:** Number of heterozygous SNPs in the segment.
- **segclust, cnlr.median.clust, mafR.clust:** Cluster-level values used in the EM fitting.

The major allele copy number can be derived as `tcn.em - lcn.em`.

### Summary Statistics

The `*_purity.out` and `*_hisens.out` files contain the key fitted parameters:

- **Purity:** Estimated tumor purity (0 to 1).
- **Ploidy:** Estimated average ploidy.
- **dipLogR:** The logR value at the diploid state.
- **loglik:** Log-likelihood of the fit.
- **flags:** Quality flags from the fitting process.

The `*_OUT.txt` file has 13 columns: `Sample, Facets, snp.nbhd, ndepth, purity_cval, cval, min.nhet, genome, Purity, Ploidy, dipLogR, loglik, flags`.

The `*_samplestatistics.txt` file reformats these values for downstream tools like BRASS and SVclone.

### QC File

The `*.facets_qc.txt` file from FACETS Preview contains automated quality assessments. This file evaluates whether the FACETS fit is reliable based on internal consistency metrics.

## Reading the Diagnostic Plots

FACETS produces PNG plots for both hisens and purity fits. Each plot typically shows:

1. **Top panel (logR):** Log-ratio values per SNP with segment means overlaid. Clean data shows tight clusters around segment means. Excessive scatter suggests noisy data.
2. **Bottom panel (logOR):** Log-odds-ratio at heterozygous SNPs. Symmetric bands around zero indicate balanced alleles; deviation indicates allelic imbalance or LOH.
3. **Copy number coloring:** Segments are colored by integer copy number state, making gains (red tones), losses (blue tones), and LOH events visually identifiable.

## Common Failure Modes

### Low Tumor Purity

When tumor purity is below approximately 20-30%, the copy number signal becomes very weak. FACETS may:
- Estimate purity as NA or an implausible value.
- Call everything as diploid (tcn=2, lcn=1) because the signal is indistinguishable from noise.
- Produce a dipLogR near zero with no visible segmentation structure.

### Noisy Samples

Excessive noise in the logR signal (often from low coverage, poor library quality, or FFPE artifacts) leads to:
- Over-segmentation in hisens mode (many tiny segments).
- Segments with very few heterozygous SNPs (`num.mark` values below `min_nhet`).
- Unstable purity/ploidy estimates that differ dramatically between hisens and purity modes.

### Incorrect dipLogR / Ploidy Solution

FACETS can converge on a wrong ploidy solution, especially in highly aneuploid tumors. Signs include:
- Purity and ploidy modes giving very different answers.
- The dipLogR placing the "diploid" baseline at a segment that is clearly not the most common copy state.
- Biologically implausible copy numbers (e.g., most of the genome called as 4N when the tumor is not known to be tetraploid).

### When Results Are Unreliable

Consider FACETS results unreliable when:
- Purity is reported as NA or below 0.1.
- The hisens and purity fits disagree substantially on purity (difference > 0.2).
- The QC file flags the sample as failing quality thresholds.
- The logR plot shows no clear segmentation structure.
- The run exhausted all 4 seed attempts (check the process logs).

## File Locations

- FACETS output: `<outDir>/somatic/<tumor>__<normal>/facets/<tumor>__<normal>/`
- QC file: `<outDir>/somatic/<tumor>__<normal>/facets/<tumor>__<normal>/<tumor>__<normal>.facets_qc.txt`
- Summary: `<outDir>/somatic/<tumor>__<normal>/facets/<tumor>__<normal>/<tumor>__<normal>_OUT.txt`

## Example

```bash
# Check purity and ploidy from the summary file
cat TUMOR__NORMAL_OUT.txt
# Columns include: Purity, Ploidy, dipLogR, cval, genome, seed

# Inspect the cncf file for segment-level calls
head -5 facets0.5.14c100pc500/TUMOR__NORMAL_hisens.cncf.txt
# ID  chrom  loc.start  loc.end      seg  num.mark  nhet  cnlr.median  mafR  segclust  cnlr.median.clust  mafR.clust  cf  tcn  lcn  cf.em  tcn.em  lcn.em
# TUMOR__NORMAL_hisens  1  569400  32209910  1  5358  432  -0.015  -0.005  12  -0.031  -0.003  1  2  1  1  2  1
```

## See Also

- `facets-algorithm.md` -- How the FACETS algorithm works
- `copy-number-facets-vs-ascat.md` -- When to use FACETS vs ASCAT
- `maf-filtering.md` -- MAF annotation uses FACETS purity for CCF estimation
