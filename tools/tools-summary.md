# Tempo Tool Inventory (Every Tool, by Stage)

> **Quick answer:** This is the authoritative list of every bioinformatics tool the Tempo pipeline invokes, grouped by stage, with version, purpose, the Nextflow process that calls it, and assay scope (WES/WGS/both). Versions are taken from the container definitions on the `develop` branch and may differ in tagged releases. Each tool with a dedicated page is linked.

## How to use this page

- Want to know **what a tool is and why it runs**? Find it below.
- Want the **math/thresholds** behind an output? See the linked deep-dive page or [`outputs/`](../outputs/).
- "Calling process" is the `.nf` process name — useful when reading a trace file or `tempodeliver` completion steps.

## Alignment & Preprocessing → [alignment-preprocessing.md](alignment-preprocessing.md)

| Tool | Version | Purpose | Calling process | Assay |
|------|---------|---------|-----------------|-------|
| fastp | 0.19.7 | FASTQ adapter trimming & QC | `AlignReads` | both |
| BWA mem | 0.7.17 | Read alignment to reference | `AlignReads` | both |
| samtools (view/sort/merge) | 1.9 | BAM conversion, sort, lane merge | `AlignReads`, `MergeBamsAndMarkDuplicates` | both |
| GATK MarkDuplicates | 4.1.9.0 | PCR duplicate marking | `MergeBamsAndMarkDuplicates` | both |
| GATK BaseRecalibrator / ApplyBQSR | 4.1.9.0 | Base quality score recalibration | `RunBQSR` | both |
| GATK SplitIntervals | 4.1.0.0 | Scatter genome into parallel intervals | `CreateScatteredIntervals` | both |

## Quality Control → [qc-tools.md](qc-tools.md)

| Tool | Version | Purpose | Calling process | Assay | Output doc |
|------|---------|---------|-----------------|-------|-----------|
| fastp | 0.19.7 | FASTQ-level QC metrics | `AlignReads` | both | [outputs/qc-fastp.md](../outputs/qc-fastp.md) |
| Alfred | 0.1.17 | Per-read-group alignment QC | `QcAlfred` | both | [outputs/qc-alfred.md](../outputs/qc-alfred.md) |
| Qualimap bamqc | 2.2.2d | Genome/target coverage & quality | `QcQualimap` | both | [outputs/qc-qualimap.md](../outputs/qc-qualimap.md) |
| GATK CollectHsMetrics | 4.1.0.0 | Hybrid-selection / bait metrics | `QcCollectHsMetrics` | WES | — |
| GATK 3 pileup | 3.8-1 | SNP-marker pileup for Conpair | `QcPileup` | both | [outputs/qc-conpair.md](../outputs/qc-conpair.md) |
| Conpair | 0.3.3 | T/N concordance & contamination | `QcConpair`, `QcConpairAll` | both | [outputs/qc-conpair.md](../outputs/qc-conpair.md) |
| MultiQC | 1.11 | Aggregates QC into HTML | `SampleRunMultiQC`, `SomaticRunMultiQC`, `CohortRunMultiQC` | both | — |

QC **pass/warn/fail thresholds** for these tools are in [outputs/qc-thresholds.md](../outputs/qc-thresholds.md).

## Somatic SNV / Indel → [snv-callers.md](snv-callers.md), [maf-filtering.md](maf-filtering.md)

| Tool | Version | Purpose | Calling process | Assay |
|------|---------|---------|-----------------|-------|
| GATK Mutect2 + FilterMutectCalls | 4.1.0.0 | Somatic SNV/indel calling | `RunMutect2` | both |
| Strelka2 | 2.9.10 | Somatic SNV/indel calling | `SomaticRunStrelka2` | both |
| bcftools | 1.9 | VCF merge/normalize/intersect | `SomaticCombineMutect2Vcf`, `SomaticCombineChannel` | both |
| vt | 0.57721 | Indel left-normalization, Ref_Tri | `SomaticCombineChannel` | both |
| GetBaseCountsMultiSample | 1.2.2 | Caller-independent allele counts (`*_raw`) | `SomaticCombineChannel` | both |
| vcf2maf | 1.6.17 | VCF→MAF conversion | `SomaticAnnotateMaf` | both |
| VEP | 88 | Functional consequence annotation | `SomaticAnnotateMaf` | both |
| OncoKB Annotator | 1.1.0 | Actionability annotation | `SomaticAnnotateMaf` | both |
| facets-suite annotate-maf | 2.0.8 | Adds CN/zygosity/CCF to MAF | `SomaticFacetsAnnotation` | both |

## Germline SNV / Indel → [snv-callers.md](snv-callers.md), [outputs/germline-maf.md](../outputs/germline-maf.md)

| Tool | Version | Purpose | Calling process | Assay |
|------|---------|---------|-----------------|-------|
| GATK HaplotypeCaller | 4.1.0.0 | Germline SNV/indel calling | `GermlineRunHaplotypecaller` | both |
| GATK SelectVariants / VariantFiltration | 4.1.0.0 | Hard-filter germline SNP/INDEL | `GermlineRunHaplotypecaller` | both |
| Strelka2 (germline mode) | 2.9.10 | Germline SNV/indel calling | `GermlineRunStrelka2` | both |

## Structural Variants → [sv-callers.md](sv-callers.md), [sv-format.md](sv-format.md)

| Tool | Version | Purpose | Calling process | Assay |
|------|---------|---------|-----------------|-------|
| Manta | 1.5.0 | Somatic/germline SV; Strelka2 indel candidates | `SomaticRunManta`, `GermlineRunManta` | both |
| Delly | 0.8.2 | SV calling (DEL/DUP/INV/TRA) | `SomaticDellyCall`, `DellyCombine` | both |
| SvABA | ~1.1.3 (commit 4a0606e) | Assembly-based SV/indel calling | `SomaticRunSvABA` | both |
| BRASS | 6.3.4 | PCAWG-style rearrangement calling | `runBRASS*` | WGS |
| pcap-core bam_stats | 5.5.0 | `.bas` stats required by BRASS | `generateBasFile` | WGS |
| mergeSVvcf | 1.0.2 | Consensus merge of caller VCFs | `SomaticMergeSVs`, `GermlineMergeSVs` | both |
| svtools | 0.5.1 | VCF→BEDPE conversion & sort | `SomaticSVVcf2Bedpe` | both |
| iAnnotateSV | commit a2f8654 | Gene/region annotation of BEDPE | `SomaticAnnotateSVBedpe` | both |
| ClusterSV | commit 1d7eeea | SV clustering (chromothripsis/kataegis) | `SomaticRunClusterSV` | both |
| BioCircos (R) | unknown (R 4.2.2) | Interactive Circos HTML | `SomaticRunSVCircos` | both |

## Copy Number → [facets-algorithm.md](facets-algorithm.md), [copy-number-facets-vs-ascat.md](copy-number-facets-vs-ascat.md)

| Tool | Version | Purpose | Calling process | Assay |
|------|---------|---------|-----------------|-------|
| FACETS | 0.5.14 | Allele-specific CN, purity/ploidy | `DoFacets` | both |
| facets-suite | 2.0.8 | snp-pileup & run-facets wrappers | `DoFacets` | both |
| facetsPreview | 2.1.4 | FACETS QC review & annotation | `DoFacetsPreviewQC` | both |
| ASCAT / ascatNgs | 4.4.0 | Allele-specific CN, purity/ploidy (WGS) | `runAscat` | WGS |

## Clonality → [svclone-clonality.md](svclone-clonality.md)

| Tool | Version | Purpose | Calling process | Assay |
|------|---------|---------|-----------------|-------|
| SVclone | 1.1.1 | Joint SV+SNV clonal clustering | `SomaticRunSVclone` | WGS |

## HLA / Neoantigen → [hla-neoantigen.md](hla-neoantigen.md)

| Tool | Version | Purpose | Calling process | Assay |
|------|---------|---------|-----------------|-------|
| POLYSOLVER | v4 | HLA class-I typing from normal | `RunPolysolver` | both |
| LOHHLA | 1.1.7 | HLA loss-of-heterozygosity | `RunLOHHLA` | both |
| neoantigen pipeline | 0.3.3 | Neoantigen prediction wrapper | `RunNeoantigen` | both |
| NetMHC / NetMHCpan | 4.0a | MHC class-I binding affinity | `RunNeoantigen` | both |

## Signatures / MSI → [signatures-msi.md](signatures-msi.md), [hrdetect.md](hrdetect.md)

| Tool | Version | Purpose | Calling process | Assay |
|------|---------|---------|-----------------|-------|
| tempoSig | 0.2.3 | COSMIC SBS signature fitting | `RunMutationSignatures` | both |
| MSIsensor | 0.5 | Microsatellite instability score | `RunMsiSensor` | both |
| signature.tools.lib (HRDetect) | 2.1.2 | HRD composite score | `HRDetect` | WGS |
| signature.tools.lib (SV signatures) | 2.1.2 | SV signature decomposition | `RunSVSignatures` | WGS |

## Aggregation / Visualization

| Tool | Version | Purpose | Calling process | Assay |
|------|---------|---------|-----------------|-------|
| MultiQC | 1.11 | Cohort-level QC aggregation | `CohortRunMultiQC` | both |
| metadataparser (Tempo helper) | — | Per-pair metadata + TMB + WGD/MSI status | `MetaDataParser` | both |
| Ghostscript | system | Merge per-sample SV-signature PDFs | `SomaticAggregateSvSignatures` | WGS |

## Version caveats

A few tools are pinned to git commits rather than released versions (SvABA `4a0606e`, iAnnotateSV `a2f8654`, ClusterSV `1d7eeea`); the BioCircos R-package and the ASCAT R package (bundled inside `ascatNgs`) are not separately pinned, so their exact versions are **unknown** from the repo. Conpair's pileup step uses legacy **GATK 3.8-1** while all other GATK steps use 4.1.x. Always confirm against the container definitions of the exact pipeline version you ran.

## See Also

- [`pipeline/overview.md`](../pipeline/overview.md) — pipeline stages in narrative form
- [`outputs/qc-thresholds.md`](../outputs/qc-thresholds.md) — pass/warn/fail QC thresholds
- [`reference/calculations.md`](../reference/calculations.md) — every important formula/threshold in one place
