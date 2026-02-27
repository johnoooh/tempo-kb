# Cohort-Level Aggregate Outputs in Tempo

> **Quick answer:** When Tempo is run with the `--aggregate` flag, it concatenates per-sample output files across the cohort into single combined files in the `outDir/cohort_level/{cohort}/` directory, covering somatic/germline mutations, SVs, copy number, signatures, QC, and metadata.

## How Aggregation Works

Aggregation in Tempo is a post-processing step triggered by the `--aggregate` flag. It takes the per-sample or per-pair output files and naively concatenates them together, removing duplicate header rows. The result is one combined file per data type for the entire cohort, making it easy to perform cross-sample analyses without manually merging files.

Each cohort gets its own subdirectory under `cohort_level/`. The cohort name is defined in the aggregate input file. If you have multiple cohorts, each will have a separate folder (for example, `cohort_level/default_cohort/`, `cohort_level/cohort2/`).

## Complete List of Cohort-Level Files

### Somatic Mutation Files

- **`mut_somatic.maf`**: All somatic MAFs concatenated. Header from the first file is retained; subsequent headers are removed. Rows are sorted by chromosome and position.
- **`mut_somatic_neoantigens.txt`**: All neoantigen predictions concatenated from per-sample `*.all_neoantigen_predictions.txt` files.

### Germline Mutation Files

- **`mut_germline.maf`**: All germline MAFs concatenated and sorted by chromosome and position.

### Structural Variant Files

- **`sv_somatic.bedpe`**: All somatic SV BEDPE files concatenated. For genome data, these are the clustered BEDPEs; for exome data, the final BEDPEs.
- **`sv_germline.bedpe`**: All germline SV BEDPE files concatenated.

### Copy Number Files (FACETS)

- **`cna_purity_run_segmentation.seg`**: Concatenated FACETS purity-run segmentation files.
- **`cna_hisens_run_segmentation.seg`**: Concatenated FACETS hisens-run segmentation files.
- **`cna_facets_run_info.txt`**: Concatenated FACETS run information (OUT.txt files).
- **`cna_armlevel.txt`**: Arm-level copy number calls, filtered to exclude DIPLOID segments.
- **`cna_genelevel.txt`**: Gene-level copy number calls, filtered to non-DIPLOID segments that pass QC (PASS or RESCUE).

### Signature Files (Genome Only)

- **`sv_exposures.tsv`**: Concatenated SV rearrangement signature exposures from all samples.
- **`sv_catalogues.pdf`**: Merged PDF of all per-sample SV catalogue plots.

### HRDetect (Genome Only)

- **`hrdetect.tsv`**: Concatenated HRDetect scores and feature values for all samples.

### SVclone (Genome Only)

- **`svclone_sv_cluster_certainty.tsv`**: Concatenated SV cluster assignment files.
- **`svclone_snv_cluster_certainty.tsv`**: Concatenated SNV cluster assignment files.

### LOHHLA (HLA Loss of Heterozygosity)

- **`HLAlossPrediction_CI.txt`**: Concatenated LOHHLA HLA loss predictions.
- **`DNA.IntegerCPN_CI.txt`**: Concatenated LOHHLA integer copy number estimates.

### Metadata

- **`sample_data.txt`**: Concatenated per-pair metadata files containing purity, ploidy, WGD, MSI, signatures, HLA, TMB for all samples.

### QC Files

- **`alignment_qc.txt`**: Aggregated alignment QC metrics from Alfred across all samples.
- **`concordance_qc.txt`**: Conpair concordance values for all tumor-normal pairs.
- **`contamination_qc.txt`**: Conpair contamination estimates for all samples.
- **`multiqc_report.html`**: Combined MultiQC report with QC metrics for all samples in the cohort.
- **`multiqc_data.zip`**: Underlying data for the MultiQC report.

## File Locations

```
outDir/cohort_level/{cohort_name}/mut_somatic.maf
outDir/cohort_level/{cohort_name}/mut_germline.maf
outDir/cohort_level/{cohort_name}/mut_somatic_neoantigens.txt
outDir/cohort_level/{cohort_name}/sv_somatic.bedpe
outDir/cohort_level/{cohort_name}/sv_germline.bedpe
outDir/cohort_level/{cohort_name}/cna_purity_run_segmentation.seg
outDir/cohort_level/{cohort_name}/cna_hisens_run_segmentation.seg
outDir/cohort_level/{cohort_name}/cna_facets_run_info.txt
outDir/cohort_level/{cohort_name}/cna_armlevel.txt
outDir/cohort_level/{cohort_name}/cna_genelevel.txt
outDir/cohort_level/{cohort_name}/sv_exposures.tsv
outDir/cohort_level/{cohort_name}/sv_catalogues.pdf
outDir/cohort_level/{cohort_name}/hrdetect.tsv
outDir/cohort_level/{cohort_name}/svclone_sv_cluster_certainty.tsv
outDir/cohort_level/{cohort_name}/svclone_snv_cluster_certainty.tsv
outDir/cohort_level/{cohort_name}/HLAlossPrediction_CI.txt
outDir/cohort_level/{cohort_name}/DNA.IntegerCPN_CI.txt
outDir/cohort_level/{cohort_name}/sample_data.txt
outDir/cohort_level/{cohort_name}/alignment_qc.txt
outDir/cohort_level/{cohort_name}/concordance_qc.txt
outDir/cohort_level/{cohort_name}/contamination_qc.txt
outDir/cohort_level/{cohort_name}/multiqc_report.html
outDir/cohort_level/{cohort_name}/multiqc_data.zip
```

## Example

```bash
# Run Tempo with aggregation enabled:
nextflow run tempo \
  --aggregate \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  --outDir /results \
  ...

# The cohort-level files appear at:
# /results/cohort_level/default_cohort/mut_somatic.maf
# /results/cohort_level/default_cohort/sample_data.txt
# etc.
```

## See Also

- `metadata-parser.md` -- Per-pair metadata that gets aggregated into `sample_data.txt`
- `somatic-maf.md` -- Per-pair MAFs aggregated into `mut_somatic.maf`
- `somatic-sv-vcf.md` -- Per-pair BEDPEs aggregated into `sv_somatic.bedpe`
- `somatic-facets.md` -- Per-pair segmentation aggregated into CNA files
- `somatic-hrdetect.md` -- Per-pair HRDetect scores aggregated into `hrdetect.tsv`
