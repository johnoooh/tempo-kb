# ASCAT Copy Number Analysis: Genome-Only Allele-Specific CNA

> **Quick answer:** ASCAT is a WGS-only copy number tool in Tempo that produces allele-specific CNV segments and sample statistics, serving as an alternative to FACETS for providing purity/ploidy estimates to downstream tools like BRASS.

## What ASCAT Does

ASCAT (Allele-Specific Copy number Analysis of Tumours) estimates allele-specific copy number profiles from whole-genome sequencing data. In Tempo, ASCAT runs exclusively on genome assays (`params.assayType == "genome"`) and provides two main outputs: copy number segments in Caveman-compatible format and sample-level statistics including purity and ploidy.

## When to Use ASCAT vs FACETS

Both ASCAT and FACETS produce allele-specific copy number estimates, but they serve different roles in Tempo:

| Feature | FACETS | ASCAT |
|---------|--------|-------|
| Assay support | Exome and genome | Genome only |
| MAF annotation | Yes (hisens Rdata used for CCF and zygosity) | No |
| BRASS input | Yes (sample statistics) | Yes (sample statistics) |
| HRDetect input | Yes (CNV segments) | No |
| Gene/arm-level calls | Yes | No |
| QC reports | Yes (FacetsPreview) | No |
| Segmentation approach | CBS-based with log-ratio and BAF | ASPCF segmentation of LogR and BAF |

FACETS is the default and more heavily integrated tool in Tempo. It provides the copy number context used for MAF annotation (clonality and zygosity) and feeds multiple downstream analyses. ASCAT provides an independent CNA estimate and can serve as the source of sample statistics for BRASS when FACETS results are problematic. The `sv_wf` accepts sample statistics from either FACETS or ASCAT for the BRASS caller.

## Processing Steps

The ASCAT workflow (`ascat_wf`) runs in two stages:

**Step 1 -- Allele counting (`runAscatAlleleCount`).** The genome is divided into segments (up to 48 for GRCh37/GRCh38, 1 for non-standard genomes) that are processed in parallel. Each segment runs `ascat.pl` in `allele_count` mode, counting alleles at known SNP positions using the tumor and normal BAMs. Results are packaged as tar.gz archives.

**Step 2 -- ASCAT fitting (`runAscat`).** All allele count segments are gathered and extracted. The full `ascat.pl` pipeline runs to perform GC correction (using the provided SNP GC corrections file), segmentation, and copy number fitting. The script uses `-q 20` for minimum mapping quality, `-g L` for GC correction, and `-pr WGS` for the sequencing protocol. For GRCh37 the assembly parameter is set to 37; for GRCh38 it is set to 38.

## Key Output Files

ASCAT produces two main output types in the `ascatResults/` directory:

| File | Description |
|------|-------------|
| `*.copynumber.caveman.csv` | Allele-specific copy number segments in Caveman format. Contains chromosome, start, end, total copy number, and minor allele copy number for each segment. |
| `*.samplestatistics.txt` | Sample-level statistics including estimated tumor purity (aberrant cell fraction), ploidy, and goodness-of-fit metrics. Used by BRASS for SV calling. |

<!-- TODO: VERIFY WITH USER -->

## Downstream Usage

- **BRASS SV calling** -- The `sv_wf` workflow accepts sample statistics from ASCAT (`ascatSS`) as an alternative to FACETS sample statistics for the BRASS structural variant caller. BRASS uses purity and ploidy estimates to refine SV detection in WGS.
- **Independent CNA validation** -- ASCAT results can be compared against FACETS to validate copy number profiles, particularly in samples where one tool may struggle (e.g., low purity or highly polyploid tumors).

Note that unlike FACETS, ASCAT output is **not** used for MAF annotation. The somatic and germline MAFs always receive their CCF and zygosity annotations from FACETS hisens data.

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/
  ascatResults/
    {idTumor}__{idNormal}.copynumber.caveman.csv
    {idTumor}__{idNormal}.samplestatistics.txt
```

<!-- TODO: VERIFY WITH USER -->
Note: The exact publishDir for ASCAT is not explicitly defined in the process file; outputs are emitted to channels consumed by downstream processes. The file paths above reflect the expected output structure based on the `runAscat` process working directory.

## Example

```bash
# View ASCAT sample statistics (purity, ploidy)
cat outDir/somatic/TUMOR1__NORMAL1/ascatResults/TUMOR1__NORMAL1.samplestatistics.txt

# View copy number segments
head outDir/somatic/TUMOR1__NORMAL1/ascatResults/TUMOR1__NORMAL1.copynumber.caveman.csv
```

## See Also

- `somatic-facets.md` -- Primary CNA tool, used for both exome and genome, provides MAF annotation
- `somatic-sv-vcf.md` -- BRASS SV caller consumes ASCAT or FACETS sample statistics
- `somatic-maf.md` -- MAF annotation uses FACETS (not ASCAT) for CCF and zygosity
