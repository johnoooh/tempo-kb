# Tempo Pipeline Output Directory Structure and Naming Conventions

> **Quick answer:** Tempo organizes all outputs under four top-level directories within `outDir`: `bams/` for aligned BAMs and per-sample QC, `somatic/` for tumor-normal pair analyses, `germline/` for normal-only variant calls, and `cohort_level/` for aggregated cross-sample results.

## Top-Level Layout

When a Tempo run completes, the output root (`outDir`) contains these four directories:

```
outDir/
├── bams/
├── somatic/
├── germline/
└── cohort_level/
```

Each directory uses a distinct naming convention for its subdirectories.

## bams/

Contains one subdirectory per sample, named by sample ID (e.g., `DU874145-T`, `DU874145-N`). Both tumor and normal samples get their own folder here. Each folder holds the final BAM, its index, and per-sample QC subdirectories such as `fastp/`, `alfred/`, `qualimap/`, `collecthsmetrics/` (exome only), `pileup/`, and `multiqc/`.

The naming pattern is: `outDir/bams/{SAMPLE_ID}/`

## somatic/

Contains one subdirectory per tumor-normal pair. Subdirectories are named using the double-underscore convention: `{idTumor}__{idNormal}`. For example, `DU874145-T__DU874145-N`. Inside each pair directory you will find results from variant callers (mutect2, strelka2, delly, manta, svaba, brass), copy number analysis (facets), combined mutation and SV files, conpair QC, neoantigen predictions, HRDetect (WGS only), svclone (WGS only), meta_data summaries, and a somatic-level multiqc report.

The naming pattern is: `outDir/somatic/{idTumor}__{idNormal}/`

## germline/

Contains one subdirectory per normal sample, named by the normal sample ID (e.g., `DU874145-N`). Inside each folder you will find germline variant calls from HaplotypeCaller and Strelka2, germline SVs from Delly, Manta, and SvABA, and the combined mutation and SV outputs.

The naming pattern is: `outDir/germline/{idNormal}/`

## cohort_level/

Contains one subdirectory per cohort, named by the cohort identifier (defaults to `default_cohort`). This directory is only populated when the pipeline is run with the `--aggregate` flag. It contains concatenated results across all samples: aggregated MAFs, BEDPEs, segmentation files, copy number profiles, alignment QC summaries, concordance and contamination tables, and a cohort-wide MultiQC report (`multiqc_report.html` and `multiqc_data.zip`).

The naming pattern is: `outDir/cohort_level/{cohort_name}/`

## Naming Convention Summary

| Directory | Subdirectory Key | Example |
|-----------|-----------------|---------|
| `bams/` | `{SAMPLE_ID}` | `bams/DU874145-T/` |
| `somatic/` | `{idTumor}__{idNormal}` | `somatic/DU874145-T__DU874145-N/` |
| `germline/` | `{idNormal}` | `germline/DU874145-N/` |
| `cohort_level/` | `{cohort_name}` | `cohort_level/default_cohort/` |

The double-underscore (`__`) separator in somatic directories is important. It distinguishes the tumor and normal sample IDs within a single directory name and is used consistently across all somatic outputs including conpair, multiqc, combined mutations, and combined SVs.

## File Locations

- BAMs and per-sample QC: `outDir/bams/{SAMPLE_ID}/`
- Somatic variant calls: `outDir/somatic/{idTumor}__{idNormal}/`
- Germline variant calls: `outDir/germline/{idNormal}/`
- Cohort aggregates: `outDir/cohort_level/{cohort_name}/`

## Example

```
outDir/
├── bams/
│   ├── DU874145-T/
│   │   ├── DU874145-T.bam
│   │   ├── DU874145-T.bam.bai
│   │   ├── alfred/
│   │   ├── fastp/
│   │   ├── multiqc/
│   │   ├── pileup/
│   │   └── qualimap/
│   └── DU874145-N/
│       ├── DU874145-N.bam
│       ├── DU874145-N.bam.bai
│       └── ...
├── somatic/
│   └── DU874145-T__DU874145-N/
│       ├── combined_mutations/
│       ├── conpair/
│       ├── facets/
│       ├── multiqc/
│       └── ...
├── germline/
│   └── DU874145-N/
│       ├── combined_mutations/
│       └── ...
└── cohort_level/
    └── default_cohort/
        ├── multiqc_report.html
        ├── alignment_qc.txt
        └── ...
```

## See Also

- `bam-files.md` for details on BAM file contents and QC subdirectories
- `qc-multiqc.md` for MultiQC report levels (sample, somatic, cohort)
- `qc-conpair.md` for concordance and contamination QC in the somatic directory
