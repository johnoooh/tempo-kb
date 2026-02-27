# Tempo BAM Output Files: Location, Contents, and QC Subdirectories

> **Quick answer:** Tempo produces one BQSR-recalibrated, duplicate-marked BAM and its index per sample at `outDir/bams/{SAMPLE_ID}/{SAMPLE_ID}.bam`. Each sample directory also contains QC subdirectories for fastp, alfred, qualimap, pileup, and multiqc (plus collecthsmetrics for exomes).

## What the BAM Represents

The final BAM file produced by Tempo has been through the full alignment and post-processing pipeline:

1. **Alignment** -- FASTQ reads are aligned to the reference genome using `bwa mem`. Each lane or FASTQ pair is aligned separately with proper read group tags (`@RG`) set automatically from the FASTQ header.
2. **Sorting** -- Per-lane BAMs are coordinate-sorted using `samtools sort`.
3. **Merging and duplicate marking** -- All per-lane sorted BAMs for a given sample are merged and PCR/optical duplicates are marked using the `MergeBamsAndMarkDuplicates` process (Picard MarkDuplicates).
4. **Base Quality Score Recalibration (BQSR)** -- GATK's `BaseRecalibrator` and `ApplyBQSR` correct systematic errors in base quality scores using known variant sites (dbSNP and known indels).

The result is a single analysis-ready BAM per sample.

## Output Files per Sample

Each sample produces two files in its directory:

| File | Description |
|------|-------------|
| `{SAMPLE_ID}.bam` | Final BQSR-recalibrated, duplicate-marked, coordinate-sorted BAM |
| `{SAMPLE_ID}.bam.bai` | BAM index file |

Both tumor and normal samples get their own directory under `bams/`.

## QC Subdirectories

Inside each sample's BAM directory, Tempo publishes QC outputs in the following subdirectories:

| Subdirectory | Tool | What It Contains |
|-------------|------|------------------|
| `fastp/` | fastp | HTML and JSON reports for each FASTQ lane pair. Covers adapter trimming stats, read quality, duplication rate, and base content at the FASTQ level. |
| `alfred/` | Alfred | Per-sample and per-read-group alignment metrics as TSV and PDF files. Includes coverage, insert size distribution, alignment rate, and mapping quality. |
| `collecthsmetrics/` | GATK CollectHsMetrics | Hybridization-selection metrics (exome assays only). Reports on-target rate, fold enrichment, and target coverage. |
| `pileup/` | Conpair (GATK pileup) | SNP pileup file (`{SAMPLE_ID}.pileup`) used downstream by Conpair for concordance and contamination analysis. Not a QC report itself, but an intermediate required for pairing QC. |
| `qualimap/` | Qualimap | BAM alignment metrics as HTML report and text files. Provides genome-wide coverage statistics, GC content analysis, and mapping quality distributions. |
| `multiqc/` | MultiQC | Aggregated sample-level QC report combining metrics from fastp, alfred, qualimap, and collecthsmetrics (if exome). Produces `multiqc_report.html` and `multiqc_data.zip`. |

## File Locations

```
outDir/bams/{SAMPLE_ID}/
├── {SAMPLE_ID}.bam
├── {SAMPLE_ID}.bam.bai
├── fastp/
│   ├── {SAMPLE_ID}@{RGID}.fastp.html
│   └── {SAMPLE_ID}@{RGID}.fastp.json
├── alfred/
│   ├── {SAMPLE_ID}.alfred.tsv.gz
│   ├── {SAMPLE_ID}.alfred.tsv.gz.pdf
│   ├── {SAMPLE_ID}.alfred.per_readgroup.tsv.gz
│   └── {SAMPLE_ID}.alfred.per_readgroup.tsv.gz.pdf
├── collecthsmetrics/          # exome only
│   └── {SAMPLE_ID}.hs_metrics.txt
├── pileup/
│   └── {SAMPLE_ID}.pileup
├── qualimap/
│   ├── qualimapReport.html
│   ├── genome_results.txt
│   ├── css/
│   └── images_qualimapReport/
└── multiqc/
    ├── multiqc_report.html
    └── multiqc_data.zip
```

## Example

To locate the final BAM for a tumor sample called `DU874145-T`:

```bash
# BAM and index
ls outDir/bams/DU874145-T/DU874145-T.bam
ls outDir/bams/DU874145-T/DU874145-T.bam.bai

# Check coverage from qualimap
grep "mean coverageData" outDir/bams/DU874145-T/qualimap/genome_results.txt

# View FASTQ QC in a browser
open outDir/bams/DU874145-T/fastp/*.fastp.html
```

## See Also

- `directory-structure.md` for the overall output layout
- `qc-fastp.md` for details on fastp FASTQ-level QC
- `qc-alfred.md` for Alfred alignment metrics
- `qc-qualimap.md` for Qualimap coverage and GC content reports
- `qc-multiqc.md` for how MultiQC aggregates these per-sample metrics
