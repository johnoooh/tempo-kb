# Copy Number Analysis: FACETS vs ASCAT and the --svcnv Flag

> **Quick answer:** Tempo supports two copy number tools: FACETS (exome and genome) and ASCAT (genome only). The `--svcnv` parameter controls which copy number source feeds into SV calling, BRASS, HRDetect, and SVclone. Options are `hisens` (default, FACETS high-sensitivity), `purity` (FACETS purity-optimized), or `ascat` (ASCAT, genome only).

## The --svcnv Parameter

The `--svcnv` flag (default: `hisens`) determines which copy number estimates are used by downstream structural variant and homologous recombination deficiency analyses. This parameter is set in `nextflow.config` and can be overridden on the command line:

```bash
nextflow run tempo --svcnv hisens   # Default: FACETS hisens mode
nextflow run tempo --svcnv purity   # FACETS purity mode
nextflow run tempo --svcnv ascat    # ASCAT (genome-only)
```

The downstream routing logic in `dsl2.nf` works as follows:

- **`hisens`:** Uses `FacetsHisensCNV4HrDetectFiltered` for copy number calls and `FacetsHisensSampleStatistics4BRASS` for purity/ploidy statistics passed to BRASS.
- **`purity`:** Uses `FacetsPurityCNV4HrDetectFiltered` for copy number calls and `FacetsPuritySampleStatistics4BRASS` for BRASS.
- **`ascat`:** Runs the ASCAT workflow instead. Uses ASCAT copy number calls and sample statistics. FACETS still runs separately for MAF annotation and LOH analysis.

## FACETS Overview

FACETS performs joint segmentation of total copy number (logR) and allelic imbalance (logOR) signals derived from matched tumor-normal SNP pileup counts. It works on both exome and genome data.

**Strengths:**
- Works with both WES and WGS data.
- Joint segmentation of logR and logOR captures allele-specific events naturally.
- Two modes (hisens and purity) give flexibility in sensitivity vs. stability.
- Integrated into multiple Tempo downstream analyses (MAF CCF annotation, LOH, neoantigen).

**Limitations:**
- Can struggle with highly aneuploid or polyploid tumors.
- Exome data may have limited resolution for broad copy number events.
- Occasional convergence failures (mitigated by the retry/seed mechanism).

## ASCAT Overview

ASCAT (Allele-Specific Copy number Analysis of Tumours) is an alternative copy number tool that is only available for genome (WGS) data in Tempo. It uses allele counts at known SNP positions from 1000 Genomes to compute logR and B-allele frequency (BAF) signals.

The Tempo ASCAT implementation uses `ascatNgs` (v4.4.0) and runs in two phases:

1. **Allele counting** (`runAscatAlleleCount`): Parallelized across up to 48 genomic chunks. Each chunk counts alleles at 1000 Genomes SNP loci (`acLoci`) in both tumor and normal BAMs.
2. **ASCAT fitting** (`runAscat`): Merges allele counts and runs the ASCAT algorithm with GC correction (`snpGcCorrections`), producing integer copy number segments and sample statistics.

**Strengths:**
- Mature algorithm with well-characterized behavior on WGS data.
- GC-content correction handles systematic biases.
- Widely used in PCAWG and other large-scale WGS studies.
- BRASS was originally designed to work with ASCAT sample statistics.

**Limitations:**
- Genome-only -- cannot be used with exome data.
- Requires additional reference files (1000G SNP loci, GC corrections).
- Runs as a separate workflow with significant compute time for allele counting.

## Algorithmic Differences

| Feature | FACETS | ASCAT |
|---------|--------|-------|
| Input data | SNP-Pileup at dbSNP sites | Allele counts at 1000G sites |
| Segmentation | Joint CBS of logR + logOR | Separate segmentation of logR and BAF |
| GC correction | Not explicit | Explicit GC bias correction |
| Assay support | WES + WGS | WGS only |
| Purity/ploidy | Grid search with EM | Grid search optimization |
| Output format | cncf.txt, Rdata, seg | copynumber.caveman.csv, samplestatistics.txt |

## When to Use Which

- **Exome data:** FACETS is the only option. The `--svcnv` flag is irrelevant for exomes since ASCAT requires WGS.
- **Genome data, default:** Use `hisens` (FACETS) for maximum sensitivity to focal copy number events.
- **Genome data, noisy samples:** Consider `purity` (FACETS) if hisens over-segments or produces unstable fits.
- **Genome data, PCAWG-style analysis:** Consider `ascat` for consistency with published PCAWG workflows, especially when running BRASS.

Regardless of the `--svcnv` setting, FACETS always runs for genome data when SNV, mutsig, or lohhla workflows are enabled, because the MAF annotation step requires FACETS hisens Rdata for CCF and zygosity calculations.

## File Locations

- FACETS process: `tempo/modules/process/Facets/DoFacets.nf`
- ASCAT allele count: `tempo/modules/process/Ascat/runAscatAlleleCount.nf`
- ASCAT fitting: `tempo/modules/process/Ascat/runAscat.nf`
- Routing logic: `tempo/dsl2.nf` (lines 206-222)
- Default svcnv setting: `tempo/nextflow.config` (line 49)

## Example

```bash
# Run Tempo with ASCAT for genome copy number (WGS only)
nextflow run tempo \
  --assayType genome \
  --svcnv ascat \
  --genome GRCh37

# Run Tempo with FACETS purity mode
nextflow run tempo \
  --assayType genome \
  --svcnv purity
```

## See Also

- `facets-algorithm.md` -- Detailed FACETS algorithm description
- `facets-interpreting.md` -- Reading FACETS output and QC
- `hrdetect.md` -- Uses CNV output from the selected svcnv source
