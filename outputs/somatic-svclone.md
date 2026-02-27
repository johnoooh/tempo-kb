# SVclone and CCube Clonal Analysis in Tempo (Genome Only)

> **Quick answer:** For whole-genome data, Tempo runs SVclone with CCube to perform joint clonal inference on structural variants and single-nucleotide variants, assigning each variant to a clonal or subclonal cluster based on variant allele frequencies and copy number context.

## What SVclone Does

SVclone is a computational framework for clustering structural variants by their cancer cell fraction (CCF). It integrates with CCube, a Bayesian clustering method, to jointly model the clonal architecture of a tumor using both SVs and SNVs. This analysis reveals the clonal composition of the tumor, distinguishing mutations present in all cancer cells (clonal) from those present in only a subset (subclonal). Understanding clonal structure is important for assessing tumor heterogeneity, predicting treatment resistance, and understanding tumor evolution.

## Inputs

The `SomaticRunSVclone` process takes the following inputs:

- **Tumor and normal BAMs**: For reading variant allele counts at SV breakpoints
- **Somatic BEDPE**: Filtered structural variant calls
- **Filtered somatic MAF**: SNV/indel calls for joint clustering
- **Copy number profile**: FACETS CNV segmentation for copy number-aware CCF estimation
- **Purity/ploidy estimates**: From FACETS, used to convert variant allele frequencies to cancer cell fractions

The process is orchestrated by a Python wrapper script (`svclone_wrapper`) that formats inputs and configures SVclone using a template configuration file (`/config/svclone_config.ini`).

## How the Process Works

1. The wrapper script reformats the BEDPE, MAF, and CNV files into SVclone-compatible formats, writes them to the `svclone_in` directory, and sets purity/ploidy parameters.
2. SVclone runs its analysis, producing CCube clustering output in `{idTumor}__{idNormal}/ccube_out/post_assign/`.
3. Results are organized into separate `svs/` and `snvs/` subdirectories within the `svclone/` output folder.
4. Each cluster assignment file has the sample ID prepended to every row for downstream aggregation.

## Output Files

The published `svclone/` directory contains:

- **`svclone/svs/`**: SV cluster assignments
  - `*_cluster_certainty.txt`: Per-SV cluster assignment probabilities
  - `*.RData` and `*.pdf`: R data objects and visualization plots
- **`svclone/snvs/`**: SNV cluster assignments
  - `*_cluster_certainty.txt`: Per-SNV cluster assignment probabilities
  - `*.RData` and `*.pdf`: R data objects and visualization plots

The cluster certainty files are tab-delimited with a `sampleid` column prepended. They contain cluster membership probabilities for each variant, allowing identification of which variants belong to the same clone.

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/svclone/svs/*_cluster_certainty.txt
outDir/somatic/{idTumor}__{idNormal}/svclone/svs/*.RData
outDir/somatic/{idTumor}__{idNormal}/svclone/svs/*.pdf
outDir/somatic/{idTumor}__{idNormal}/svclone/snvs/*_cluster_certainty.txt
outDir/somatic/{idTumor}__{idNormal}/svclone/snvs/*.RData
outDir/somatic/{idTumor}__{idNormal}/svclone/snvs/*.pdf
```

## Example

```bash
# The pipeline internally runs the SVclone wrapper:
python svclone_wrapper.py \
  --cfg_template /config/svclone_config.ini \
  --bedpe sample_T__sample_N.final.clustered.bedpe \
  --maf sample_T__sample_N.somatic.final.maf \
  --purity_ploidy purity_ploidy.txt \
  --out_dir svclone_in \
  --sampleid sample_T__sample_N \
  --bam tumor.bam \
  --cnv facets_cnv.seg
```

## See Also

- `somatic-sv-vcf.md` -- SV BEDPE files that serve as input
- `somatic-facets.md` -- Purity, ploidy, and copy number used for CCF estimation
- `somatic-maf.md` -- Somatic SNVs used in joint clustering
- `cohort-aggregates.md` -- Cohort-level `svclone_sv_cluster_certainty.tsv` and `svclone_snv_cluster_certainty.tsv`
