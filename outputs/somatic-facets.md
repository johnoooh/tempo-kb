# FACETS Copy Number Analysis: Purity, Ploidy, and Allele-Specific Copy Number

> **Quick answer:** Tempo runs FACETS in two modes (hisens and purity) per tumor-normal pair, producing allele-specific copy number profiles, purity/ploidy estimates, gene-level and arm-level calls, segmentation files, and QC reports that feed into MAF annotation, HRDetect, and BRASS.

## What FACETS Does

FACETS (Fraction and Allele-Specific Copy Number Estimates from Tumor Sequencing) is the primary copy number analysis tool in Tempo, used for both exome and genome assays. It estimates tumor purity, ploidy, and allele-specific integer copy number across the genome from heterozygous SNP read counts in matched tumor-normal pairs.

## Two FACETS Modes

Tempo runs FACETS in two configurations through a single process (`DoFacets`), producing output for each:

- **Purity mode** (`*_purity.*`) -- Uses a higher critical value (`purity_cval`) and minimum heterozygous SNP count (`purity_min_nhet`) for segmentation. Produces fewer, larger segments optimized for accurate purity and ploidy estimation.
- **Hisens mode** (`*_hisens.*`) -- Uses a lower critical value (`cval`) and minimum heterozygous SNP count (`min_nhet`), creating more granular segmentation that is more sensitive to focal copy number events. The hisens Rdata file is used for MAF annotation (clonality and zygosity).

Both modes are parameterized through the Nextflow config (`params.facets.cval`, `params.facets.purity_cval`, etc.). The FACETS output directory name encodes these parameters: `facets{R_lib}c{cval}pc{purity_cval}`.

## Processing Steps

1. **SNP pileup** -- `snp-pileup-wrapper.R` from facets-suite generates allele counts at common SNP positions from both BAMs, producing a `.snp_pileup.gz` file.
2. **FACETS fitting** -- `run-facets-wrapper.R` performs segmentation and copy number fitting. If fitting fails, Tempo retries up to 4 times with incremented random seeds.
3. **Summary** -- `summarize_project.py` generates a summary output file (`*_OUT.txt`).
4. **Sample statistics** -- `generate_samplestatistics.R` produces sample statistics files for both hisens and purity modes, used downstream by BRASS.
5. **QC** -- `DoFacetsPreviewQC` runs `facetsPreview` to generate genomic annotations and a `facets_qc.txt` quality control report.

## Key Output Files

The output directory is nested: `facets/{idTumor}__{idNormal}/{outputDir}/`. The `outputDir` encodes FACETS parameters using the pattern `facets{version}c{cval}pc{purity_cval}` -- e.g., `facets0.5.14c100pc500` means FACETS version 0.5.14 with cval=100 and purity_cval=500. This directory name will change if the FACETS version or cval parameters are updated, so use a glob (`facets*/`) rather than hardcoding the name in scripts.

| File Pattern | Description |
|---|---|
| `*.snp_pileup.gz` | Raw SNP allele counts for tumor and normal |
| `*_hisens.Rdata` | Hisens fit R object (used for MAF annotation) |
| `*_purity.Rdata` | Purity fit R object |
| `*_hisens.seg` / `*_purity.seg` | Segmentation files (IGV-compatible) |
| `*_hisens.out` / `*_purity.out` | Purity/ploidy estimates and fit diagnostics |
| `*_hisens.CNCF.png` / `*_purity.CNCF.png` | Copy number profile plots |
| `*_hisens.cncf.txt` / `*_purity.cncf.txt` | Segment-level copy number data (18 columns: ID, chrom, loc.start, loc.end, seg, num.mark, nhet, cnlr.median, mafR, segclust, cnlr.median.clust, mafR.clust, cf, tcn, lcn, cf.em, tcn.em, lcn.em) |
| `*.arm_level.txt` | Arm-level copy number calls |
| `*.gene_level.txt` | Gene-level copy number calls |
| `*.qc.txt` | FACETS internal QC metrics |
| `*.facets_qc.txt` | FacetsPreview QC report |
| `*_OUT.txt` | Summary file from summarize_project.py |
| `*_hisens.facets.copynumber.csv` | CNV segments for HRDetect |
| `*_hisens.samplestatistics.txt` | Sample statistics for BRASS |

## Downstream Usage

FACETS outputs feed into multiple downstream analyses:

- **Somatic MAF annotation** -- Hisens Rdata provides copy number context for CCF estimation and zygosity calls (`SomaticFacetsAnnotation`)
- **Germline MAF annotation** -- Same hisens Rdata used for germline variant zygosity (`GermlineFacetsAnnotation`)
- **BRASS** -- Sample statistics (purity, ploidy) required for WGS SV calling
- **HRDetect** -- Copy number segments used for homologous recombination deficiency detection
- **Cohort aggregation** -- Gene-level, arm-level, and segmentation files concatenated across samples

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/facets/{idTumor}__{idNormal}/
  {idTumor}__{idNormal}.snp_pileup.gz
  {idTumor}__{idNormal}_OUT.txt
  {idTumor}__{idNormal}.facets_qc.txt
  facets{R_lib}c{cval}pc{purity_cval}/
    {idTumor}__{idNormal}_hisens.Rdata
    {idTumor}__{idNormal}_hisens.seg
    {idTumor}__{idNormal}_hisens.out
    {idTumor}__{idNormal}_hisens.CNCF.png
    {idTumor}__{idNormal}_purity.Rdata
    {idTumor}__{idNormal}_purity.seg
    {idTumor}__{idNormal}_purity.out
    {idTumor}__{idNormal}_purity.CNCF.png
    {idTumor}__{idNormal}.arm_level.txt
    {idTumor}__{idNormal}.gene_level.txt
    {idTumor}__{idNormal}.qc.txt
```

## Example

```bash
# Check purity and ploidy from the purity-mode output
cat outDir/somatic/TUMOR1__NORMAL1/facets/TUMOR1__NORMAL1/facets*/TUMOR1__NORMAL1_purity.out

# View gene-level copy number calls
head outDir/somatic/TUMOR1__NORMAL1/facets/TUMOR1__NORMAL1/facets*/TUMOR1__NORMAL1.gene_level.txt
```

## See Also

- `somatic-maf.md` -- FACETS annotation adds CCF and zygosity to the somatic MAF
- `somatic-ascat.md` -- Alternative CNA tool for WGS; ASCAT can replace FACETS for BRASS input
- `germline-maf.md` -- FACETS also annotates germline MAFs with zygosity
