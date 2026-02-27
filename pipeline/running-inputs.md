# Tempo Input Files: Mapping, Pairing, and Aggregate TSV Formats for WES and WGS

> **Quick answer:** Tempo requires a tab-separated mapping file (FASTQ or BAM) that links samples to sequencing files and bait sets, plus a pairing file that defines tumor-normal pairs. An optional aggregate file groups pairs into cohorts. Headers are mandatory but column order does not matter, and extra columns are allowed.

## FASTQ Mapping File (`--mapping <tsv>`)

Used when starting from raw sequencing reads. Each row maps a sample to one pair of paired-end FASTQ files.

**Required columns:** `SAMPLE`, `TARGET`, `FASTQ_PE1`, `FASTQ_PE2`

| SAMPLE | TARGET | FASTQ_PE1 | FASTQ_PE2 |
|---|---|---|---|
| normal_sample_1 | agilent | /data/normal1_L001_R01.fastq.gz | /data/normal1_L001_R02.fastq.gz |
| normal_sample_1 | agilent | /data/normal1_L002_R01.fastq.gz | /data/normal1_L002_R02.fastq.gz |
| tumor_sample_1 | agilent | /data/tumor1_L001_R01.fastq.gz | /data/tumor1_L001_R02.fastq.gz |
| tumor_sample_1 | agilent | /data/tumor1_L002_R01.fastq.gz | /data/tumor1_L002_R02.fastq.gz |

A single sample can span multiple rows to represent multiple lanes. Tempo merges them and scans FASTQ read names to generate read group IDs for BQSR. If the filename contains a lane pattern like `_L001_`, scanning is skipped. FASTQ files must use the `.fastq.gz` extension.

## BAM Mapping File (`--bamMapping <tsv>`)

Used when starting from pre-aligned BAM files. Each sample may appear only once (the pipeline does not merge BAMs).

**Required columns:** `SAMPLE`, `TARGET`, `BAM`, `BAI`

| SAMPLE | TARGET | BAM | BAI |
|---|---|---|---|
| normal_sample_1 | agilent | /data/normal1.bam | /data/normal1.bai |
| tumor_sample_1 | agilent | /data/tumor1.bam | /data/tumor1.bai |

BAM index files (`.bai` or `.bam.bai`) must exist in the same directory as the BAM. The `BAI` column is validated but not directly used. When using `--bamMapping`, at least one sub-workflow and a `--pairing` file must be provided.

## TARGET Column Values

The `TARGET` column specifies the bait set or sequencing type, and must be consistent with `--assayType`:

| assayType | Valid TARGET values | Notes |
|---|---|---|
| `exome` (default) | `agilent`, `idt` | Can be mixed within a run |
| `genome` | `wgs` | Cannot be mixed with exome targets |

Tumor and normal samples in the same pair must use the same TARGET value.

## Pairing File (`--pairing <tsv>`)

Defines which tumor samples are matched with which normal samples. Required for all sub-workflows except alignment-only runs.

**Required columns:** `NORMAL_ID`, `TUMOR_ID`

| NORMAL_ID | TUMOR_ID |
|---|---|
| normal_sample_1 | tumor_sample_1 |
| normal_sample_2 | tumor_sample_2 |

Sample names must exactly match the `SAMPLE` column in the mapping file. All samples referenced in the pairing file must appear in the mapping file, and vice versa.

## Aggregate File (`--aggregate <tsv>`)

Groups tumor-normal pairs into cohorts for combined reporting.

**Required columns:** `NORMAL_ID`, `TUMOR_ID`, `COHORT`, `PATH` (PATH required only in aggregate-only mode)

| NORMAL_ID | TUMOR_ID | COHORT | PATH |
|---|---|---|---|
| normal_sample_1 | tumor_sample_1 | cohort1 | /results/tempo_v1/result |
| normal_sample_2 | tumor_sample_2 | cohort1 | /results/tempo_v1/result |
| normal_sample_3 | tumor_sample_3 | cohort2 | /results/tempo_v2/result |

When `--aggregate true` is used instead of a TSV file, all samples are grouped into a single cohort named "default cohort."

## Validation Rules

Tempo validates inputs at startup via `lib/TempoUtils.groovy`. The pipeline will exit with an error if any check fails:

1. **Headers mandatory** -- Required column names must be present. Column order does not matter. Extra columns are allowed.
2. **Duplicate rows** -- Identical rows within a file are rejected.
3. **Duplicate files** -- The same FASTQ or BAM path assigned to different samples is rejected.
4. **TARGET consistency** -- TARGET values must match the `--assayType` parameter, and tumor-normal pairs must share the same TARGET.
5. **File path validation** -- All FASTQ and BAM paths must point to existing files.
6. **File extension checks** -- FASTQs must end in `.fastq.gz`, BAMs in `.bam`, indices in `.bai`.
7. **BAI co-location** -- Index files must exist alongside BAM files (as `*.bai` or `*.bam.bai`).
8. **Cross-file sample validation** -- All samples in the pairing file must appear in the mapping file, and all samples in the mapping file must appear in the pairing file.

## File Locations

- Input validation logic: `lib/TempoUtils.groovy`
- Test mapping files: `test_inputs/local/`
- Input documentation: `docs/running-the-pipeline.md`

## Example

A minimal exome run with two tumor-normal pairs requires three files:

**mapping.tsv:**
```
SAMPLE	TARGET	FASTQ_PE1	FASTQ_PE2
normal_1	idt	/data/N1_R1.fastq.gz	/data/N1_R2.fastq.gz
tumor_1	idt	/data/T1_R1.fastq.gz	/data/T1_R2.fastq.gz
```

**pairing.tsv:**
```
NORMAL_ID	TUMOR_ID
normal_1	tumor_1
```

## See Also

- [Pipeline Overview](overview.md) -- What Tempo does and its analysis stages
- [Running and Invocation](running-invocation.md) -- Command-line flags and execution modes
