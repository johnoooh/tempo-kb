# Creating Oncoplots from Tempo MAF Files Using R maftools

> **Quick answer:** Use the R `maftools` package to read Tempo MAF files directly with `read.maf()` and generate oncoplots with `oncoplot()`. Tempo MAFs use standard MAF column names, so no reformatting is needed.

## What Is an Oncoplot

An oncoplot (also called a waterfall plot or OncoPrint) is a visualization that shows the mutational landscape of a cohort of samples. Each column represents a sample, each row represents a gene, and cells are colored by mutation type. It is one of the most common figures in cancer genomics publications and is useful for identifying frequently mutated genes and co-occurrence patterns.

## Prerequisites

Install maftools from Bioconductor:

```r
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")
BiocManager::install("maftools")
```

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/combined_mutations/{idTumor}__{idNormal}.somatic.final.maf
```

For cohort-level analysis, you may want to combine multiple MAF files first (see [merge-maf-clinical.md](merge-maf-clinical.md)).

## Example: Basic Oncoplot

```r
library(maftools)

# Read a single MAF file
maf <- read.maf("outDir/somatic/TUMOR__NORMAL/combined_mutations/TUMOR__NORMAL.somatic.final.maf")

# Print summary
print(maf)

# Basic oncoplot showing top 20 mutated genes
oncoplot(maf, top = 20)
```

Tempo MAFs contain standard MAF column names (`Hugo_Symbol`, `Variant_Classification`, `Tumor_Sample_Barcode`, etc.), so `read.maf()` works without any column remapping.

## Example: Cohort Oncoplot

To create an oncoplot across multiple samples, merge individual MAF files into one:

```r
library(maftools)
library(data.table)

# List all final MAFs
maf_files <- list.files(
  path = "outDir/somatic/",
  pattern = "\\.somatic\\.final\\.maf$",
  recursive = TRUE,
  full.names = TRUE
)

# Combine into a single MAF
combined <- rbindlist(lapply(maf_files, function(f) {
  fread(f, skip = "#version")
}))

# Write to temp file and read with maftools
tmp <- tempfile(fileext = ".maf")
fwrite(combined, tmp, sep = "\t")
maf <- read.maf(tmp)

# Oncoplot with top 30 genes
oncoplot(maf, top = 30)
```

## Key Oncoplot Parameters

```r
# Show specific genes
oncoplot(maf, genes = c("TP53", "KRAS", "PIK3CA", "BRAF", "APC"))

# Remove samples without mutations in the displayed genes
oncoplot(maf, top = 20, removeNonMutated = FALSE)

# Sort by a clinical annotation
oncoplot(maf, top = 20, sortByAnnotation = TRUE,
         clinicalFeatures = "Tumor_Type")

# Customize colors for variant classifications
vc_cols <- c(
  Missense_Mutation = "#2196F3",
  Nonsense_Mutation = "#F44336",
  Frame_Shift_Del   = "#FF9800",
  Frame_Shift_Ins   = "#4CAF50",
  Splice_Site       = "#9C27B0",
  In_Frame_Del      = "#00BCD4",
  In_Frame_Ins      = "#CDDC39"
)
oncoplot(maf, top = 20, colors = vc_cols)
```

## Additional Useful maftools Functions

### MAF Summary Plot

```r
plotmafSummary(maf, rmOutlier = TRUE, addStat = 'median')
```

This displays a multi-panel figure with variant classification counts, variant type distribution, SNV class breakdown, and mutations per sample.

### Lollipop Plot

```r
lollipopPlot(maf, gene = "TP53", AACol = "HGVSp_Short", showMutationRate = TRUE)
```

Shows protein-domain-level distribution of mutations along the protein sequence for a single gene.

### Somatic Interactions

```r
somaticInteractions(maf, top = 25, pvalue = c(0.05, 0.01))
```

Tests for mutually exclusive or co-occurring mutation pairs using a pairwise Fisher exact test.

### Rainfall Plot

```r
rainfallPlot(maf, detectChangePoints = TRUE, pointSize = 0.5)
```

Visualizes inter-mutation distance along the genome to identify hypermutated regions (kataegis).

## Saving Plots

```r
pdf("oncoplot_cohort.pdf", width = 14, height = 10)
oncoplot(maf, top = 20)
dev.off()
```

## See Also

- [load-maf-python.md](load-maf-python.md) -- loading MAF files in Python
- [merge-maf-clinical.md](merge-maf-clinical.md) -- merging MAFs with clinical data
- [batch-process-samples.md](batch-process-samples.md) -- collecting MAFs across samples
