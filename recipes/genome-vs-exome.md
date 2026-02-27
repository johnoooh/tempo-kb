# Genome vs Exome Analysis: What Runs in Each Tempo Assay Mode

> **Quick answer:** Tempo supports WES (exome) and WGS (genome) assay types. Most analyses run in both modes, but genome mode adds ASCAT, BRASS, HRDetect, SVclone, SV signatures, ClusterSV, and Circos plots.

## Overview

Tempo adjusts its analysis pipeline based on the assay type specified in the run configuration. Exome (WES) and genome (WGS) modes share a core set of analyses including alignment, somatic variant calling, FACETS copy number analysis, and mutational signature extraction. Genome mode extends the pipeline with additional structural variant and homologous recombination deficiency analyses that require whole-genome coverage.

## Analysis Comparison Table

| Analysis | Exome | Genome | Description |
|----------|:-----:|:------:|-------------|
| Alignment + QC | Yes | Yes | BWA-MEM alignment, duplicate marking, BQSR |
| Somatic SNV/Indel (Mutect2 + Strelka2) | Yes | Yes | Combined calling and filtering |
| FACETS | Yes | Yes | Purity, ploidy, allele-specific copy number |
| MSIsensor | Yes | Yes | Microsatellite instability scoring |
| tempoSig | Yes | Yes | Mutational signature decomposition |
| Polysolver + LOHHLA | Yes | Yes | HLA typing and LOH at HLA |
| Neoantigen prediction | Yes | Yes | NetMHCpan-based neoantigen calling |
| Germline SNV/SV | Yes | Yes | HaplotypeCaller and Manta germline calls |
| ASCAT | No | Yes | Alternative copy number (requires WGS) |
| BRASS | No | Yes | Structural variant breakpoint analysis |
| HRDetect | No | Yes | Homologous recombination deficiency score |
| SVclone | No | Yes | Clonality estimation for structural variants |
| SV Signatures | No | Yes | Structural variant signature extraction |
| ClusterSV | No | Yes | Structural variant clustering |
| Circos | No | Yes | Genome-wide circular visualization |

## Specifying the Assay Type

### The --assayType Flag

When launching Tempo, specify the assay type:

```bash
nextflow run tempo.nf --assayType genome ...
nextflow run tempo.nf --assayType exome ...
```

This flag determines which pipeline branches are activated and which reference files (bait sets, target regions) are used.

### The TARGET Column in the Pairing/Mapping File

The mapping file includes a `TARGET` column that specifies the capture platform:

| TARGET Value | Assay Type | Description |
|-------------|------------|-------------|
| `wgs` | Genome | Whole genome sequencing |
| `idt` | Exome | IDT xGen capture panel |
| `agilent` | Exome | Agilent SureSelect capture panel |

Example mapping file entries:

```
SAMPLE_01    FASTQ_R1    FASTQ_R2    idt
SAMPLE_02    FASTQ_R1    FASTQ_R2    wgs
```

The TARGET value determines which bait set and target region BED files are used for coverage calculations, variant filtering, and other region-dependent steps.

## Mixing Platforms

Samples from different assay types (exome vs genome) or different capture platforms (IDT vs Agilent) cannot be mixed within a single Tempo run. All samples in a run must share the same `--assayType` and TARGET platform. If you have samples from multiple platforms, run them as separate Tempo executions.

## Key Differences in Practice

### Coverage and Sensitivity

- **Exome**: Deep coverage (~150-300x) over coding regions (~30 Mb). Better sensitivity for low-frequency coding variants.
- **Genome**: Moderate coverage (~30-60x) across the entire genome (~3 Gb). Enables detection of structural variants, non-coding mutations, and genome-wide copy number profiles.

### TMB Calculation

The coding region size used as the denominator for TMB differs:

- Exome: ~30 Mb (varies by bait set)
- Genome: ~2800 Mb (whole genome callable region)

### Structural Variant Analysis

BRASS, SVclone, SV signatures, and ClusterSV require whole-genome data because structural variants in non-coding regions are not captured by exome sequencing.

### HRDetect

HRDetect is a weighted model that combines multiple mutational features (including SV signatures and rearrangement patterns) to predict homologous recombination deficiency. It requires whole-genome data and is available only in genome mode.

## Choosing Between Exome and Genome

| Consideration | Exome | Genome |
|---------------|-------|--------|
| Cost per sample | Lower | Higher |
| Coding variant detection | Excellent | Good |
| Structural variants | Limited | Comprehensive |
| Non-coding mutations | No | Yes |
| HRD assessment | Not available | HRDetect available |
| TMB accuracy | Standard (panel-based) | Genome-wide |
| Turnaround time | Faster (smaller data) | Slower (larger data) |

## See Also

- [calculate-tmb.md](calculate-tmb.md) -- TMB calculation with region size considerations
- [facets-qc-r.md](facets-qc-r.md) -- FACETS QC (runs in both modes)
- [batch-process-samples.md](batch-process-samples.md) -- collecting outputs from either mode
