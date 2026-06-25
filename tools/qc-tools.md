# QC Tools in Tempo (fastp, Alfred, Qualimap, CollectHsMetrics, Conpair, MultiQC)

> **Quick answer:** Tempo runs FASTQ QC (fastp), per-read-group alignment QC (Alfred), genome/target coverage QC (Qualimap), bait-capture metrics (GATK CollectHsMetrics, exome only), and tumor–normal concordance/contamination (Conpair). MultiQC aggregates everything at sample, somatic, and cohort levels. Pass/warn/fail thresholds are in [../outputs/qc-thresholds.md](../outputs/qc-thresholds.md).

## The QC tools

| Tool | Version | Level | What it measures | Output doc |
|------|---------|-------|------------------|-----------|
| fastp | 0.19.7 | FASTQ | adapter content, read quality, duplication estimate | [qc-fastp.md](../outputs/qc-fastp.md) |
| Alfred | 0.1.17 | read group | mapping rate, duplicate fraction, GC, insert size | [qc-alfred.md](../outputs/qc-alfred.md) |
| Qualimap bamqc | 2.2.2d | sample BAM | mean coverage, % aligned, on-target coverage (WES) | [qc-qualimap.md](../outputs/qc-qualimap.md) |
| GATK CollectHsMetrics | 4.1.0.0 | sample BAM | hybrid-selection / fold-enrichment (**WES only**) | — |
| Conpair | 0.3.3 | T/N pair | concordance (same patient?) + contamination | [qc-conpair.md](../outputs/qc-conpair.md) |
| MultiQC | 1.11 | sample/somatic/cohort | aggregates all of the above into HTML + stats table | — |

## How they fit together

- **fastp** runs at alignment time (per lane); **Alfred** and **Qualimap** run on the analysis-ready BAM (per sample); **CollectHsMetrics** adds bait-capture metrics for exome; **Conpair** runs per tumor–normal pair.
- **Conpair** depends on a pileup at common SNP markers (`QcPileup`, using legacy GATK 3.8-1). Markers are 1000 Genomes Phase 3 autosomal SNPs filtered to MAF ≥ 0.4 and LD r² ≤ 0.8, queried as normal-homozygous only — so contamination/concordance are robust across ancestries.
- **MultiQC** runs at three levels: `SampleRunMultiQC`, `SomaticRunMultiQC`, `CohortRunMultiQC`. The cohort-level `multiqc_general_stats.txt` (inside `multiqc_data.zip`) carries the per-sample `-Status` column that drives both the QC report and the delivery email's pass count.

## Pass / warn / fail

Each metric is compared against fixed exome or genome thresholds; the sample's single QC label is the worst across metrics. The full tables (coverage, contamination, concordance, duplication, enrichment, % aligned) are in **[../outputs/qc-thresholds.md](../outputs/qc-thresholds.md)**. Key facts:
- Exome tolerates normal contamination up to 15% (no fail); WGS holds both T and N to < 1%.
- Fold enrichment (CollectHsMetrics) is exome-only.

## File Locations

- QC subworkflows: `tempo/modules/subworkflow/sampleQC_wf.nf`, `samplePairingQC_wf.nf`, `somaticMultiQC_wf.nf`
- Processes: `tempo/modules/process/QC/`
- Thresholds: `tempo/lib/multiqc_config/{exome,wgs}_multiqc_config.yaml`

## See Also

- [../outputs/qc-thresholds.md](../outputs/qc-thresholds.md) — the pass/warn/fail numbers
- [alignment-preprocessing.md](alignment-preprocessing.md) — produces the BAM these tools QC
- [../delivery/delivery-email.md](../delivery/delivery-email.md) — how QC reaches investigators
