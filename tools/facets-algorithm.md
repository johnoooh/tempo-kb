# FACETS Algorithm: Joint Segmentation and Allele-Specific Copy Number in Tempo

> **Quick answer:** FACETS uses SNP-Pileup counts at heterozygous SNP loci to jointly segment log-ratio (logR) and log-odds-ratio (logOR) signals, then fits allele-specific integer copy number with simultaneous purity and ploidy estimation. Tempo runs FACETS in two modes -- hisens and purity -- each optimized for different segmentation sensitivity.

## How FACETS Works

FACETS (Fraction and Allele-specific Copy number Estimates from Tumor Sequencing) was published by Shen and Seshan in 2016 (Nucleic Acids Research). It is the primary copy number tool in Tempo for both exome and genome data.

### Step 1: SNP-Pileup

Before FACETS runs its segmentation, the `snp-pileup-wrapper.R` script counts reference and alternate allele reads at known germline SNP positions in both the tumor and matched normal BAMs. The VCF of SNP positions is specified by the `facetsVcf` reference parameter (typically dbSNP). The `--pseudo-snps 50` flag inserts a pseudo-SNP every 50th genomic position to provide coverage information even in regions without heterozygous SNPs. The output is a gzipped pileup file (`*.snp_pileup.gz`).

### Step 2: Joint Segmentation

FACETS computes two signals from the pileup data:

- **logR (log-ratio):** The log2 ratio of tumor read depth to normal read depth at each SNP, reflecting total copy number changes.
- **logOR (log-odds-ratio):** The log-odds of the B-allele fraction at heterozygous SNPs, reflecting allelic imbalance.

FACETS performs joint segmentation of both signals simultaneously using a circular binary segmentation (CBS) variant. The `cval` parameter controls segmentation sensitivity -- lower values produce more segments (higher sensitivity), while higher values produce fewer, more confident segments.

### Step 3: Allele-Specific Copy Number Fitting

After segmentation, FACETS fits integer total copy number and minor allele copy number to each segment. This fitting simultaneously estimates:

- **Purity:** The fraction of tumor cells in the sample.
- **Ploidy:** The average total copy number of the tumor genome.
- **dipLogR:** The logR value corresponding to the diploid state, which anchors the copy number scale.

The dipLogR value is critical because it determines where "normal" (copy-neutral) segments sit on the logR axis. An incorrect dipLogR leads to systematically wrong copy number calls.

### Two FACETS Modes in Tempo

Tempo runs FACETS twice per sample, producing two complete sets of results:

- **Hisens (high sensitivity):** Uses the `cval` parameter (exome: 100, genome: 1000) for the segmentation step. This produces more granular segments and can detect smaller copy number events, but may over-segment noisy regions.
- **Purity:** Uses the `purity_cval` parameter (exome: 500, genome: 5000), which is higher than the hisens cval. This produces fewer, broader segments optimized for the most reliable purity and ploidy estimate.

Both modes share the same SNP-Pileup input and the same `min_nhet` (minimum number of heterozygous SNPs per segment, default 25) and `ndepth` (minimum normal depth, exome: 35, genome: 15) parameters.

### Retry Logic

The FACETS process includes a retry mechanism that tries up to 4 different random seeds (starting from `params.facets.seed`) if the initial run fails. This handles occasional convergence failures in the FACETS fitting algorithm.

### Post-Processing

After FACETS completes, Tempo runs `summarize_project.py` to produce a summary output file (`*_OUT.txt`) and `generate_samplestatistics.R` to extract sample statistics (purity, ploidy, dipLogR) for both hisens and purity fits. These statistics feed into downstream tools including BRASS, HRDetect, and SVclone. The FACETS Preview QC step (`DoFacetsPreviewQC`) generates additional genomic annotations and quality metrics using the `facetsPreview` R package.

## File Locations

- FACETS process: `tempo/modules/process/Facets/DoFacets.nf`
- FACETS QC process: `tempo/modules/process/Facets/DoFacetsPreviewQC.nf`
- FACETS workflow: `tempo/modules/subworkflow/facets_wf.nf`
- Exome config (cval, thresholds): `tempo/conf/exome.config`
- Genome config (cval, thresholds): `tempo/conf/genome.config`
- Output directory: `<outDir>/somatic/<tumor>__<normal>/facets/<tumor>__<normal>/`

## Example

```bash
# FACETS output files per sample:
<tumor>__<normal>.snp_pileup.gz         # Raw pileup counts
<tumor>__<normal>_hisens.seg             # Hisens segmentation
<tumor>__<normal>_purity.seg             # Purity segmentation
<tumor>__<normal>_hisens.Rdata           # Full hisens R object
<tumor>__<normal>_purity.Rdata           # Full purity R object
<tumor>__<normal>_hisens.cncf.txt        # Segment-level CN calls (hisens)
<tumor>__<normal>_purity.out             # Purity fit summary
<tumor>__<normal>_OUT.txt                # Combined run summary
```

## See Also

- `facets-interpreting.md` -- How to read and QC FACETS output
- `copy-number-facets-vs-ascat.md` -- Choosing between FACETS and ASCAT
- `hrdetect.md` -- Uses FACETS CNV output for HRD prediction
