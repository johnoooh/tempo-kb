# Quality-Checking FACETS Results in R

> **Quick answer:** Load the FACETS `.Rdata` output and inspect purity, ploidy, and the logR/logOR diagnostic plots. Samples with purity below 0.2 are unreliable, and disagreement between purity and hisens fits warrants manual review.

## What Is FACETS

FACETS (Fraction and Allele-Specific Copy Number Estimates from Tumor Sequencing) is a copy number analysis tool that estimates tumor purity, ploidy, and allele-specific copy number from paired tumor-normal sequencing data. Tempo runs FACETS automatically for both exome and genome assays.

## File Locations

FACETS outputs are stored per tumor-normal pair. The versioned subdirectory name (`facets{version}c{cval}pc{purity_cval}`) changes with FACETS version and parameters, so use a glob (`facets*/`) in scripts:

```
outDir/somatic/{pair}/facets/{pair}/facets*/{pair}_hisens.Rdata
outDir/somatic/{pair}/facets/{pair}/facets*/{pair}_purity.Rdata
outDir/somatic/{pair}/facets/{pair}/facets*/{pair}_hisens.cncf.txt
outDir/somatic/{pair}/facets/{pair}/facets*/{pair}_purity.cncf.txt
outDir/somatic/{pair}/facets/{pair}/facets*/{pair}_purity.out
outDir/somatic/{pair}/facets/{pair}/{pair}.facets_qc.txt
```

The `_purity` fit uses a higher critical value (more conservative segmentation), while `_hisens` uses a lower critical value to detect smaller copy number events.

## Example: Loading FACETS Results

```r
# Load the hisens fit
load("outDir/somatic/TUMOR__NORMAL/facets/TUMOR__NORMAL/facets0.5.14c100pc500/TUMOR__NORMAL_hisens.Rdata")

# Access key metrics
purity <- fit$purity
ploidy <- fit$ploidy
dipLogR <- fit$dipLogR

cat(sprintf("Purity: %.3f\n", purity))
cat(sprintf("Ploidy: %.3f\n", ploidy))
cat(sprintf("dipLogR: %.3f\n", dipLogR))

# Access segment-level copy number data
cncf <- fit$cncf
head(cncf)
```

The `.Rdata` file contains two objects: `fit` (the FACETS fit object) and `out` (the FACETS output object).

- `fit` (verified against a Tempo run, FACETS 0.5.14) has fields: `loglik`, `purity`, `ploidy`, `dipLogR`, `seglen`, `cncf`, `emflags`. Access purity/ploidy/dipLogR via `fit$purity`, `fit$ploidy`, `fit$dipLogR`.
- `fit$cncf` is the segment data frame — **17 columns** (no leading sample-ID column): `chrom, seg, num.mark, nhet, cnlr.median, mafR, segclust, cnlr.median.clust, mafR.clust, start, end, cf.em, tcn.em, lcn.em, cf, tcn, lcn`. Total/lesser/cellular EM estimates are `tcn.em`/`lcn.em`/`cf.em`.
- `out` has: `jointseg`, `out`, `nX`, `chromlevels`, `dipLogR`, `alBalLogR`, `IGV`. Use `plotSample(x = out, emfit = fit)` for diagnostic plots.

> **Tempo also writes a tab-delimited `*_hisens.cncf.txt`** alongside the `.Rdata`. This file has **18 columns** — the same segment fields as `fit$cncf` but with a leading `ID` column (e.g. `{tumor}__{normal}_hisens`) prepended, and the columns are reordered: `ID, chrom, loc.start, loc.end, seg, num.mark, nhet, cnlr.median, mafR, segclust, cnlr.median.clust, mafR.clust, cf, tcn, lcn, cf.em, tcn.em, lcn.em`. Prefer the `.Rdata` for in-memory analysis; the `.cncf.txt` is convenient for grep/awk and downstream Python use.

## QC Checklist

### 1. Purity Check

```r
if (purity < 0.2) {
  warning("Low purity (<0.2): copy number estimates are unreliable")
} else if (purity < 0.3) {
  warning("Borderline purity (0.2-0.3): interpret with caution")
} else {
  message("Purity is acceptable")
}
```

Samples with purity below 0.2 produce unreliable allele-specific copy number calls. Consider excluding these from copy number-dependent analyses such as CCF estimation, LOH detection, and HRDetect.

### 2. Purity vs Hisens Agreement

```r
load("outDir/somatic/TUMOR__NORMAL/facets/TUMOR__NORMAL/facets0.5.14c100pc500/TUMOR__NORMAL_purity.Rdata")
purity_fit <- fit

load("outDir/somatic/TUMOR__NORMAL/facets/TUMOR__NORMAL/facets0.5.14c100pc500/TUMOR__NORMAL_hisens.Rdata")
hisens_fit <- fit

purity_diff <- abs(purity_fit$purity - hisens_fit$purity)
cat(sprintf("Purity difference: %.3f\n", purity_diff))

if (purity_diff > 0.1) {
  warning("Large purity disagreement between fits -- manual review recommended")
}
```

If the purity and hisens fits disagree substantially (difference greater than 0.1), the solution may be ambiguous. Manual review of the diagnostic plots is strongly recommended.

### 3. Inspect dipLogR

The `dipLogR` value represents the logR value corresponding to the diploid state. Values far from 0 may indicate a highly aneuploid tumor or a fitting issue.

```r
if (abs(dipLogR) > 1.0) {
  warning("dipLogR is far from 0 -- check for fitting artifacts")
}
```

### 4. Diagnostic Plots

```r
library(facets)

# Generate the logR and logOR spider plot
load("outDir/somatic/TUMOR__NORMAL/facets/TUMOR__NORMAL/facets0.5.14c100pc500/TUMOR__NORMAL_hisens.Rdata")
plotSample(x = out, emfit = fit, sname = "TUMOR__NORMAL")
```

In the diagnostic plot, look for:

- **logR track**: Segments should be clearly separated with minimal noise
- **logOR track**: Heterozygous SNP clusters should show clear separation
- **Segment fits**: Copy number calls should align with the data
- **Noisy profiles**: Excessive noise may indicate low tumor content or poor library quality

### 5. facetsPreview QC File

Tempo writes **two** facetsPreview QC files per pair:

- **`{pair}.facets_qc.txt`** at the pair root — one row, the best-fit summary used by `MetaDataParser` to set `WGD_status` (`wgd` column), plus per-filter `*_filter_pass` booleans and an overall `facets_qc` verdict.
- **`{pair}.qc.txt`** inside `facets{version}.../` — one row per `cval` (purity + hisens), the raw per-fit facetsPreview output without the best-fit aggregation.

The first one is what the rest of the pipeline keys off; read it first.

```r
qc <- read.delim("outDir/somatic/TUMOR__NORMAL/facets/TUMOR__NORMAL/TUMOR__NORMAL.facets_qc.txt")
cat("WGD:", qc$wgd, " purity:", qc$purity, " ploidy:", qc$ploidy,
    " facets_qc:", qc$facets_qc, "\n")
# Per-filter flags: homdel_filter_pass, diploid_bal_seg_filter_pass,
# waterfall_filter_pass, hyper_seg_filter_pass, high_ploidy_filter_pass,
# valid_purity_filter_pass, em_cncf_icn_discord_filter_pass,
# dipLogR_too_low_filter_pass, subclonal_genome_filter_pass,
# icn_allelic_state_concordance_filter_pass, contamination_filter_pass
failed <- names(qc)[grepl("_filter_pass$", names(qc)) & qc == FALSE]
if (length(failed)) cat("Failed filters:", paste(failed, collapse=", "), "\n")
```

## See Also

- [extract-ccf-clonality.md](extract-ccf-clonality.md) -- CCF depends on FACETS purity/ploidy
- [batch-process-samples.md](batch-process-samples.md) -- batch-collecting FACETS outputs
- [genome-vs-exome.md](genome-vs-exome.md) -- FACETS runs in both assay modes
