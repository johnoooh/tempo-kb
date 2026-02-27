# Tempo QC: fastp FASTQ-Level Quality Control

> **Quick answer:** fastp runs on each FASTQ lane pair during the alignment step, producing per-lane HTML and JSON reports at `outDir/bams/{SAMPLE_ID}/fastp/`. It measures read quality, adapter content, duplication rate, and base composition before alignment.

## What fastp Does

fastp is a FASTQ preprocessing and quality assessment tool. In Tempo, fastp is executed within the `AlignReads` process, meaning it runs on each FASTQ read pair (R1 + R2) at the lane level before the reads are aligned with `bwa mem`. Tempo does not use fastp for trimming or filtering -- it runs in reporting mode to capture pre-alignment quality metrics. The actual command executed is:

```bash
fastp --html {SAMPLE_ID}@{RGID}.fastp.html \
      --json {SAMPLE_ID}@{RGID}.fastp.json \
      --in1 {fastq_R1} \
      --in2 {fastq_R2}
```

Because fastp runs per lane, a sample sequenced across multiple lanes will have multiple fastp reports (one HTML and one JSON per lane).

## Key Metrics in the fastp Report

The fastp HTML and JSON reports contain several important QC sections:

**Read Quality**
- Total reads before and after filtering
- Q20 and Q30 rates (percentage of bases with quality score >= 20 or >= 30)
- Mean quality score per read

**Base Content**
- Per-position nucleotide composition (A, T, G, C, N)
- GC content across the reads
- Per-position quality scores

**Adapter Detection**
- Detected adapter sequences
- Adapter content percentage
- Adapter-trimmed read counts (if trimming is enabled)

**Duplication**
- Estimated duplication rate from a subset of reads
- This is an early estimate; the definitive duplication rate comes later from Picard MarkDuplicates

**Insert Size (Paired-End)**
- Estimated insert size distribution from overlapping read pairs
- Useful for detecting library preparation issues

**Overrepresented Sequences**
- Sequences that appear more frequently than expected
- Can indicate contamination or library bias

## When to Check fastp

Review fastp reports when:

- A sample has unexpectedly low alignment rate (check for adapter contamination)
- Coverage is lower than expected (check total read counts and quality)
- Downstream QC tools flag quality issues (fastp provides the FASTQ-level baseline)
- You want to compare sequencing quality across lanes for the same sample

## File Locations

```
outDir/bams/{SAMPLE_ID}/fastp/
├── {SAMPLE_ID}@{RGID}{lane_part}.fastp.html    # Human-readable report
└── {SAMPLE_ID}@{RGID}{lane_part}.fastp.json    # Machine-parseable metrics
```

The `{RGID}` is derived from the FASTQ header and typically encodes the flowcell, lane, and barcode information. The `{lane_part}` segment distinguishes files when a single FASTQ is split across lanes.

If a sample was sequenced on three lanes, you will see three `.fastp.html` and three `.fastp.json` files in the `fastp/` directory.

The JSON files are consumed downstream by the per-sample MultiQC process (`SampleRunMultiQC`) and the cohort-level MultiQC process (`CohortRunMultiQC`) to produce aggregated quality summaries.

## Example

```bash
# View the fastp HTML report for a sample
open outDir/bams/SAMPLE-T/fastp/SAMPLE-T@HXXXXX@0001.fastp.html

# Parse Q30 rate from the JSON report
python3 -c "
import json
with open('outDir/bams/SAMPLE-T/fastp/SAMPLE-T@HXXXXX@0001.fastp.json') as f:
    d = json.load(f)
    print('Q30 rate R1:', d['read1_before_filtering']['q30_rate'])
    print('Q30 rate R2:', d['read2_before_filtering']['q30_rate'])
"

# Count how many lanes were sequenced
ls outDir/bams/SAMPLE-T/fastp/*.fastp.json | wc -l
```

## See Also

- `bam-files.md` for the full list of QC subdirectories per sample
- `qc-alfred.md` for post-alignment BAM-level QC metrics
- `qc-multiqc.md` for how fastp metrics are aggregated into MultiQC reports
