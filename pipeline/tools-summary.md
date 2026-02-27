# Tempo Pipeline Bioinformatic Tools Reference Table

> **Quick answer:** Tempo orchestrates over 30 bioinformatic tools across alignment, somatic and germline variant calling, copy number analysis, HLA typing, neoantigen prediction, mutational signatures, MSI detection, quality control, and annotation. Each tool runs in a versioned Docker container managed through Nextflow process definitions.

## Complete Tool Inventory

### Alignment

| Tool | Process Name | Purpose | Container |
|------|-------------|---------|-----------|
| fastp + BWA-MEM + samtools | `AlignReads` | FASTQ QC, read alignment to reference genome, BAM sorting | `cmopipeline/fastp-bwa-samtools:2.0.0` |
| GATK MarkDuplicates | `MergeBamsAndMarkDuplicates` | Merge lane-level BAMs and flag PCR duplicates | `broadinstitute/gatk:4.1.9.0` |
| GATK BaseRecalibrator + ApplyBQSR | `RunBQSR` | Base quality score recalibration | `broadinstitute/gatk:4.1.9.0` |

### Somatic SNV/Indel Calling

| Tool | Process Name | Purpose | Container |
|------|-------------|---------|-----------|
| Mutect2 | `RunMutect2` | Somatic SNV and indel detection | `broadinstitute/gatk:4.1.0.0` |
| Strelka2 | `SomaticRunStrelka2` | Somatic SNV/indel detection (second caller) | `cmopipeline/strelka2-manta-bcftools-vt:2.0.1` |
| vcf2maf + VEP | `SomaticAnnotateMaf` | VCF-to-MAF conversion with VEP annotation | `cmopipeline/vcf2maf:vep88_1.3.0` |
| FACETS annotation | `SomaticFacetsAnnotation` | Annotate MAF with FACETS copy number | `cmopipeline/facets-suite-preview-htstools:0.0.1` |

### Somatic Structural Variant Calling

| Tool | Process Name | Purpose | Container |
|------|-------------|---------|-----------|
| Delly | `SomaticDellyCall` | SV detection via split-read and paired-end analysis | `cmopipeline/delly-bcftools:0.0.1` |
| Manta | `SomaticRunManta` | SV and indel candidate detection | `cmopipeline/strelka2-manta-bcftools-vt:2.0.1` |
| SvABA | `SomaticRunSvABA` | SV detection via local assembly | `cmopipeline/svaba:0.0.1` |
| BRASS | `runBRASS*` | SV detection (genome-only) | `cmopipeline/brass:0.0.2` |
| iAnnotateSV | `SomaticAnnotateSVBedpe` | Annotate SVs with gene/functional impact | `cmopipeline/iannotatesv:0.0.2` |

### Copy Number Analysis

| Tool | Process Name | Purpose | Container |
|------|-------------|---------|-----------|
| SNP-Pileup + FACETS + facets-suite | `DoFacets` | Allele-specific copy number, purity, ploidy | `cmopipeline/facets-suite-preview-htstools:0.0.1` |
| facets-preview | `DoFacetsPreviewQC` | FACETS output QC and review | `cmopipeline/facets-suite-preview-htstools:0.0.1` |
| ASCAT (ascatNgs) | `runAscat*` (label: ascat) | Allele-specific copy number (genome-only) | `quay.io/wtsicgp/ascatNgs:4.4.0` |

### Germline Variant Calling

Germline processes mirror the somatic pipeline with `Germline`-prefixed process names and the same containers. Key differences: HaplotypeCaller replaces Mutect2 for SNVs, and BRASS is not used.

| Tool | Process Name | Purpose | Container |
|------|-------------|---------|-----------|
| HaplotypeCaller | `GermlineRunHaplotypecaller` | Germline SNV and indel calling | `broadinstitute/gatk:4.1.0.0` |
| Strelka2 | `GermlineRunStrelka2` | Germline SNV/indel calling (second caller) | `cmopipeline/strelka2-manta-bcftools-vt:2.0.1` |
| vcf2maf + VEP | `GermlineAnnotateMaf` | VCF-to-MAF with VEP annotation | `cmopipeline/vcf2maf:vep88_1.2.7` |
| Delly | `GermlineDellyCall` | Germline SV detection | `cmopipeline/delly-bcftools:0.0.1` |
| Manta | `GermlineRunManta` | Germline SV candidate detection | `cmopipeline/strelka2-manta-bcftools-vt:2.0.1` |
| SvABA | `GermlineRunSvABA` | Germline SV detection via local assembly | `cmopipeline/svaba:0.0.1` |
| iAnnotateSV | `GermlineAnnotateSVBedpe` | Annotate germline SVs | `cmopipeline/iannotatesv:0.0.2` |

### HLA Typing and Neoantigen Prediction

| Tool | Process Name | Purpose | Container |
|------|-------------|---------|-----------|
| Polysolver | `RunPolysolver` | HLA class I genotyping from WES/WGS BAMs | `sachet/polysolver:v4` |
| LOHHLA | `RunLOHHLA` | Loss of heterozygosity at HLA loci | `cmopipeline/lohhla:1.1.7` |
| NetMHCpan (via neoantigen-dev) | `RunNeoantigen` | Neoantigen prediction using MHC-I binding affinity | `cmopipeline/neoantigen:0.3.3` |

### Mutational Signatures and MSI

| Tool | Process Name | Purpose | Container |
|------|-------------|---------|-----------|
| tempoSig | `RunMutationSignatures` | SNV mutational signature decomposition | `cmopipeline/temposig:0.2.3` |
| signature.tools.lib | `RunSVSignatures` | Structural variant signature inference | `cmopipeline/signaturetoolslib:0.0.1` |
| MSIsensor | `RunMsiSensor` | Microsatellite instability scoring | `vanallenlab/msisensor:0.5` |

### Genome-Only Analyses

| Tool | Process Name | Purpose | Container |
|------|-------------|---------|-----------|
| HRDetect (signature.tools.lib) | `HRDetect` | Homologous recombination deficiency prediction | `cmopipeline/signaturetoolslib:0.0.1` |
| SVclone + CCube | `SomaticRunSVclone` | Joint SV/SNP clonality inference | `cmopipeline/svclone:0.0.1` |
| ClusterSV | `SomaticRunClusterSV` | Cluster SVs for chromothripsis detection | `cmopipeline/clustersv:0.0.1` |
| BioCircos | `SomaticRunSVCircos` | Circular genome visualization of SVs | `cmopipeline/biocircos:0.0.1` |

### Quality Control

| Tool | Process Name | Purpose | Container |
|------|-------------|---------|-----------|
| Alfred | `QcAlfred` | BAM-level QC metrics | `cmopipeline/alfred:v0.1.17` |
| Qualimap | `QcQualimap` | BAM QC and coverage statistics | `cmopipeline/qualimap:0.0.1` |
| Conpair | `QcPileup`, `QcConpair`, `QcConpairAll` | Tumor-normal concordance and contamination | `cmopipeline/conpair:v0.3.3` |
| CollectHsMetrics | `QcCollectHsMetrics` | Hybridization-selection metrics (exome only) | `broadinstitute/gatk:4.1.0.0` |
| MultiQC | `SampleRunMultiQC`, `SomaticRunMultiQC`, `CohortRunMultiQC` | Aggregate QC reports at multiple levels | `cmopipeline/multiqc:0.1.3` |

## File Locations

- Container definitions: `tempo/conf/containers.config`
- Process modules: `tempo/modules/process/<Category>/<ProcessName>.nf`
- Tool documentation: `tempo/docs/bioinformatic-components.md`

## Example

To check which container a specific process uses, search `containers.config`:

```bash
grep -A1 "withName:RunMutect2" tempo/conf/containers.config
# Output:
#   withName:RunMutect2 {
#     container = "broadinstitute/gatk:4.1.0.0"
```

## See Also

- [pipeline-overview.md](pipeline-overview.md) -- High-level pipeline architecture and execution flow
- [facets-copynumber.md](../somatic/facets-copynumber.md) -- Detailed FACETS copy number analysis
- [sv-calling.md](../somatic/sv-calling.md) -- Structural variant calling workflow details
- [qc-metrics.md](../qc/qc-metrics.md) -- Quality control thresholds and interpretation
