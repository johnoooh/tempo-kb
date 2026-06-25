# Alignment & BAM Preprocessing in Tempo (BWA, fastp, samtools, GATK)

> **Quick answer:** Tempo aligns FASTQs with BWA mem (after fastp trimming), sorts/merges per-lane BAMs with samtools, marks PCR duplicates with GATK MarkDuplicates, and recalibrates base qualities with GATK BQSR. The final analysis-ready BAM is the input to every downstream caller and QC tool.

## The preprocessing chain

```
FASTQ → fastp → BWA mem → samtools sort → (merge lanes) → MarkDuplicates → BQSR → analysis-ready BAM
```

### fastp (0.19.7) — `AlignReads`
Adapter trimming and read-level QC, run per lane before alignment. Produces the fastp JSON/HTML consumed later by MultiQC. See [outputs/qc-fastp.md](../outputs/qc-fastp.md). fastp metrics do **not** gate the pipeline but are surfaced in QC reports.

### BWA mem (0.7.17) — `AlignReads`
Aligns trimmed reads to the reference genome (GRCh37/hg19 or GRCh38/hg38, set by `params.genome`). Read-group (`@RG`) tags are assigned per lane so duplicates and QC can be tracked per read group. Output is piped directly to `samtools sort`.

### samtools (1.9) — `AlignReads`, `MergeBamsAndMarkDuplicates`
Converts SAM→BAM, coordinate-sorts, and **merges multiple lanes/flowcells** of the same sample into one BAM before duplicate marking. A sample sequenced across several lanes is only merged at this step — earlier QC (fastp, per-lane alignment) is per read group.

### GATK MarkDuplicates (4.1.9.0) — `MergeBamsAndMarkDuplicates`
Flags PCR/optical duplicates on the merged BAM. Duplicates are marked, not removed; the duplicate fraction is a QC metric (Alfred reports it, and it has a [pass/warn/fail threshold](../outputs/qc-thresholds.md)).

### GATK BaseRecalibrator + ApplyBQSR (4.1.9.0) — `RunBQSR`
Base Quality Score Recalibration corrects systematic sequencer error in reported base qualities using known-sites VCFs (dbSNP, known indels). The recalibrated BAM is the **analysis-ready BAM** that all callers and QC tools consume.

> In `tempodeliver`, `RunBQSR` is the single step in `bamCompleteSteps` — i.e., a sample's BAM is considered "complete" once BQSR finishes.

### GATK SplitIntervals (4.1.0.0) — `CreateScatteredIntervals`
Splits the genome (or bait targets) into interval lists so variant calling can be scattered across many parallel jobs and gathered afterward. This is a parallelization utility, not a biological step.

## Reference & assay notes

- Genome build comes from `params.genome`; the bait set (agilent/idt for WES, wgs for WGS) is set per sample via the `TARGET` column of the mapping file and selects the interval list and downstream denominators.
- The final BAM and its per-sample QC live under `outDir/bams/{SAMPLE_ID}/` (see [outputs/bam-files.md](../outputs/bam-files.md)).

## File Locations

- Alignment workflow: `tempo/modules/subworkflow/alignment_wf.nf`
- Processes: `tempo/modules/process/Alignment/`

## See Also

- [outputs/bam-files.md](../outputs/bam-files.md) — the BAM outputs and per-sample QC dirs
- [qc-tools.md](qc-tools.md) — QC run on the analysis-ready BAM
- [snv-callers.md](snv-callers.md) — the callers that consume the BAM
- [outputs/qc-thresholds.md](../outputs/qc-thresholds.md) — coverage/duplication thresholds
