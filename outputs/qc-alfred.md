# Tempo QC: Alfred BAM-Level Alignment Metrics

> **Quick answer:** Alfred provides post-alignment QC at `outDir/bams/{SAMPLE_ID}/alfred/`, producing both aggregate (per-sample) and per-read-group metrics as compressed TSV and PDF files covering coverage, insert size, alignment rate, and mapping quality.

## What Alfred Does

Alfred is a BAM-level quality control tool that computes alignment statistics from the final BQSR-recalibrated BAM. In Tempo, Alfred runs twice per sample via the `QcAlfred` process: once with `--ignore` (aggregating all read groups into a single report) and once without (producing per-read-group breakdowns). This dual execution lets you assess overall sample quality and also pinpoint whether any individual sequencing lane or library is an outlier.

For exome assays, Alfred runs with the `--bed` option pointing to the target regions BED file, restricting coverage calculations to on-target regions.

The core command is:

```bash
# Aggregate (all read groups combined)
alfred qc --reference {genome.fa} --ignore --outfile {SAMPLE_ID}.alfred.tsv.gz {bam}

# Per read group
alfred qc --reference {genome.fa} --outfile {SAMPLE_ID}.alfred.per_readgroup.tsv.gz {bam}
```

After each run, an R script generates PDF visualizations from the TSV data:

```bash
Rscript --no-init-file /opt/alfred/scripts/stats.R {output.tsv.gz}
```

## Key Metrics

Alfred reports a comprehensive set of alignment statistics:

**Coverage Metrics**
- Mean coverage across the genome (or target regions for exome)
- Coverage uniformity and distribution
- Percentage of bases at various coverage thresholds (1x, 10x, 20x, 30x, etc.)

**Alignment Statistics**
- Total number of reads and mapped reads
- Percentage of reads mapped, properly paired, and with mapping quality >= 20
- Percentage of duplicate reads
- Secondary and supplementary alignment counts

**Insert Size Distribution**
- Median insert size and standard deviation
- Insert size distribution plot (in PDF)
- Useful for detecting library preparation problems or sample degradation

**Mapping Quality**
- Distribution of mapping quality scores
- Percentage of reads at various MAPQ thresholds

**Base Content and Quality**
- Per-position base quality along reads
- GC content profile

**Read Group Breakdown**
The per-read-group file (`*.per_readgroup.tsv.gz`) provides all the above metrics broken down by read group (typically one read group per sequencing lane). This is valuable for identifying whether a specific lane had lower quality or different insert sizes.

## When to Check Alfred

Review Alfred output when:

- Coverage appears lower than expected for the sequencing depth
- You need to verify insert size consistency across lanes
- Duplicate rates seem unusually high (compare Alfred's duplicate percentage with expectations)
- A downstream caller produces unexpected results and you want to rule out alignment issues
- You need per-lane quality breakdowns to decide whether to exclude a lane

## File Locations

```
outDir/bams/{SAMPLE_ID}/alfred/
├── {SAMPLE_ID}.alfred.tsv.gz                    # Aggregate metrics (all read groups)
├── {SAMPLE_ID}.alfred.tsv.gz.pdf                # Aggregate metrics PDF plots
├── {SAMPLE_ID}.alfred.per_readgroup.tsv.gz      # Per-read-group metrics
└── {SAMPLE_ID}.alfred.per_readgroup.tsv.gz.pdf  # Per-read-group PDF plots
```

The aggregate TSV files are consumed by the MultiQC processes (`SampleRunMultiQC` and `CohortRunMultiQC`) via the `parse_alfred.py` script, which converts them into MultiQC-compatible YAML modules.

## Example

```bash
# View the aggregate Alfred PDF
open outDir/bams/DU874145-T/alfred/DU874145-T.alfred.tsv.gz.pdf

# Extract mean coverage from the aggregate TSV
zcat outDir/bams/DU874145-T/alfred/DU874145-T.alfred.tsv.gz | head -2

# Compare insert size medians across read groups
zcat outDir/bams/DU874145-T/alfred/DU874145-T.alfred.per_readgroup.tsv.gz | cut -f1,15
```

## See Also

- `bam-files.md` for the complete BAM directory layout
- `qc-qualimap.md` for complementary BAM-level coverage and GC content analysis
- `qc-fastp.md` for pre-alignment FASTQ quality metrics
- `qc-multiqc.md` for how Alfred metrics feed into aggregated reports
