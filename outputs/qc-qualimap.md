# Tempo QC: Qualimap BAM Quality Control

> **Quick answer:** Qualimap produces per-sample BAM alignment metrics at `outDir/bams/{SAMPLE_ID}/qualimap/`, providing coverage statistics, GC content analysis, mapping quality distributions, and insert size information as both an HTML report and text files.

## What Qualimap Does

Qualimap's `bamqc` mode analyzes the final BQSR-recalibrated BAM to produce a detailed report on alignment quality. In Tempo, the `QcQualimap` process runs once per sample. For exome assays, Qualimap receives the target regions BED file via the `-gff` flag so that on-target coverage is reported. For WGS assays, it uses the `-gd HUMAN` flag for genome-wide analysis calibrated to the human genome.

The core command is:

```bash
qualimap bamqc \
  -bam {SAMPLE_ID}.bam \
  -gff {targets.bed}       # exome: on-target regions
  # OR -gd HUMAN           # WGS: genome-wide analysis
  -outdir {SAMPLE_ID} \
  -nt {threads} \
  -nw 300 \
  -nr {500 for WGS, 750 for exome} \
  --java-mem-size={mem}G
```

After the main analysis, Tempo packages the raw data into a tarball (`{SAMPLE_ID}_qualimap_rawdata.tar.gz`) containing `genome_results.txt` and `raw_data_qualimapReport/`. This tarball is later consumed by the MultiQC processes to include Qualimap coverage data in aggregated reports.

## Key Metrics

**Coverage Statistics** (from `genome_results.txt`)
- Mean coverage across the genome or target regions
- Standard deviation of coverage
- Percentage of the genome covered at various thresholds (1x, 5x, 10x, 15x, 20x, 30x, 50x)
- This is one of the most frequently checked metrics for confirming sequencing depth

**GC Content**
- GC content distribution across the sample
- Comparison against the expected GC content for the reference genome
- Deviations may indicate contamination, library bias, or sample quality issues

**Mapping Quality**
- Distribution of mapping quality scores across all aligned reads
- Mean mapping quality
- Helps identify samples with excessive multi-mapped or low-confidence alignments

**Insert Size**
- Insert size distribution and statistics
- Mean and median insert size
- Complements the insert size information from Alfred

**Coverage Across the Genome**
- Coverage uniformity and evenness plots
- Chromosome-by-chromosome coverage breakdown
- Coverage across GC content bins (helps detect GC-related coverage bias)

## WGS vs. Exome Differences

| Aspect | WGS | Exome |
|--------|-----|-------|
| Region flag | `-gd HUMAN` | `-gff {targets.bed}` |
| Number of reference regions (`-nr`) | 500 | 750 |
| Coverage reported over | Whole genome | Target regions only |

For exome, the coverage metrics focus exclusively on the targeted capture regions, making the mean coverage figure directly interpretable as on-target depth.

## When to Check Qualimap

Review Qualimap reports when:

- You need a quick check of mean coverage for a sample
- GC bias is suspected (check the GC content distribution plots)
- You want chromosome-level coverage breakdowns (useful for detecting large-scale aneuploidies)
- Coverage uniformity appears poor and you need to investigate which genomic regions are affected

## File Locations

```
outDir/bams/{SAMPLE_ID}/qualimap/
├── qualimapReport.html                   # Interactive HTML report
├── genome_results.txt                    # Summary text file with key metrics
├── {SAMPLE_ID}_qualimap_rawdata.tar.gz   # Packaged raw data for MultiQC
├── css/                                  # Stylesheets for HTML report
├── images_qualimapReport/                # Plots for HTML report
└── raw_data_qualimapReport/              # Raw data tables
```

## Example

```bash
# Quick check: mean coverage
grep "mean coverageData" outDir/bams/DU874145-T/qualimap/genome_results.txt

# Quick check: total mapped reads
grep "number of mapped reads" outDir/bams/DU874145-T/qualimap/genome_results.txt

# View the full interactive report
open outDir/bams/DU874145-T/qualimap/qualimapReport.html
```

## See Also

- `bam-files.md` for the complete BAM directory structure
- `qc-alfred.md` for complementary BAM-level metrics including per-read-group breakdowns
- `qc-multiqc.md` for how Qualimap data feeds into aggregated MultiQC reports
