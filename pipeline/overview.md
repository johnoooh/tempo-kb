# Tempo Pipeline Overview: MSKCC Nextflow DSL2 Workflow for WES and WGS Tumor-Normal Analysis

> **Quick answer:** Tempo is MSKCC's CMO CCS Nextflow DSL2 pipeline that processes whole-exome (WES) and whole-genome (WGS) sequencing data from matched tumor-normal pairs. It performs end-to-end analysis from FASTQ alignment through somatic and germline variant calling, copy number analysis, structural variant detection, mutational signatures, microsatellite instability, HLA typing, and neoantigen prediction.

## What Tempo Does

Tempo is a comprehensive bioinformatics pipeline developed at Memorial Sloan Kettering Cancer Center (MSKCC) for analyzing paired tumor and normal DNA sequencing data. It is built using Nextflow DSL2, enabling modular sub-workflow execution and reproducible analyses on HPC clusters (Juno/LSF) or AWS Batch.

Tempo supports two assay types:

- **Exome (WES):** Uses `agilent` or `idt` bait sets, specified via the `TARGET` column in the mapping file.
- **Genome (WGS):** Uses the `wgs` target value. Genome mode enables additional analyses not available for exome data.

Samples from mixed sequencing platforms cannot be processed together in a single run.

## Pipeline Stages

Tempo processes data through the following high-level stages:

1. **Alignment** -- BWA mem alignment, lane merging with samtools, PCR duplicate marking with GATK MarkDuplicates, and base quality score recalibration with GATK BQSR.
2. **Quality Control** -- FASTQ QC (fastp), BAM QC (Alfred, Qualimap), hybridization-selection metrics (CollectHsMetrics, exome only), and contamination/concordance analysis (Conpair). MultiQC reports are generated at BAM, somatic, and cohort levels.
3. **Somatic SNV/Indel Calling** -- MuTect2 and Strelka2 calls are combined, annotated, and filtered into a final somatic MAF.
4. **Somatic Structural Variant Detection** -- Delly, Manta, and SvABA calls are merged, with BRASS added for WGS. Output is provided in BEDPE format.
5. **Copy Number Analysis** -- FACETS produces locus-specific copy number, purity, and ploidy estimates. These are integrated with SNV calls for clonality and zygosity analysis.
6. **Germline SNV/Indel Calling** -- HaplotypeCaller and Strelka2 calls are combined and filtered into a germline MAF.
7. **Germline Structural Variants** -- Delly, Manta, and SvABA calls are merged and filtered.
8. **HLA Typing and LOH** -- POLYSOLVER performs HLA genotyping; LOHHLA assesses loss of heterozygosity at HLA loci.
9. **Neoantigen Prediction** -- NetMHC 4.0 predicts class I MHC binding affinity, integrated into somatic SNV/indel calls.
10. **Mutational Signatures and MSI** -- tempoSig infers COSMIC mutational signatures; MSIsensor detects microsatellite instability.
11. **Aggregation** -- Cohort-level summary files consolidate per-sample results into combined MAFs, BEDPE files, segmentation files, and QC reports.

## Genome-Only Analyses

The following analyses run exclusively in WGS mode:

- **ASCAT** -- Allele-specific copy number analysis.
- **BRASS** -- Structural variant caller contributing to combined SV calls.
- **HRDetect** -- Homologous recombination deficiency (BRCA1/BRCA2 deficiency) detection.
- **SVclone** -- Clonality inference using structural variants jointly with SNPs.
- **SV Signatures** -- Structural variant signature inference using signature.tools.lib.
- **Circos** -- Interactive BioCircos HTML visualization of structural variants and copy number across the circular genome.

## Key Terminology

- **Tumor-Normal Pair:** A matched set of tumor and normal samples from the same patient, defined in the pairing file.
- **Mapping File:** A TSV file that links sample IDs to their FASTQ or BAM files and target/bait-set information.
- **Pairing File:** A TSV file that specifies which normal sample is matched with which tumor sample.
- **Sub-workflows:** Modular pipeline components (e.g., `snv`, `sv`, `facets`, `qc`) that can be selectively enabled via the `--workflows` flag.

## Output Naming Convention

Somatic outputs are organized into directories named using the pattern `{idTumor}__{idNormal}` (double underscore separator). For example, a tumor sample `DU874145-T` paired with normal `DU874145-N` produces output in:

```
outDir/somatic/DU874145-T__DU874145-N/
```

Germline outputs are organized by the normal sample ID under `outDir/germline/{idNormal}/`.

## File Locations

- Pipeline entrypoint: `dsl2.nf`
- Sub-workflow modules: `modules/subworkflow/`
- Input validation: `lib/TempoUtils.groovy`
- Documentation: `docs/`

## See Also

- [Running and Input Files](running-inputs.md) -- Input TSV format and validation rules
- [Running and Invocation](running-invocation.md) -- Command-line usage, flags, and execution modes
