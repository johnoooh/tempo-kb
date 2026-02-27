# Structural Variant Signatures in Tempo (Genome Only)

> **Quick answer:** For whole-genome sequencing data, Tempo infers structural variant (SV) rearrangement signatures using the `signature.tools.lib` R package, producing per-sample SV signature exposures and a catalogue visualization PDF.

## Overview

Structural variant signatures capture patterns of genomic rearrangements (deletions, tandem duplications, inversions, and translocations) that reflect distinct biological processes. For example, tandem duplication-enriched signatures are associated with BRCA1 deficiency, while certain deletion patterns correlate with homologous recombination deficiency. This analysis is only performed for whole-genome sequencing (`params.assayType == "genome"`), because exome data lacks the resolution to reliably detect SV signatures genome-wide.

## How the Process Works

The `RunSVSignatures` process takes the filtered somatic BEDPE file as input and runs the `sv_signatures_wrapper.R` script from the `signature.tools.lib` container. The wrapper performs the following steps:

1. **Preprocessing**: Reads the BEDPE file, filters to PASS-only variants, and reformats columns to the required format (`chrom1`, `start1`, `end1`, `chrom2`, `start2`, `end2`, `strand1`, `strand2`, `sample`, `svclass`). The SV types BND, INV, DEL, and DUP are mapped to translocation, inversion, deletion, and tandem-duplication respectively.

2. **Catalogue construction**: Calls `bedpeToRearrCatalogue()` from `signature.tools.lib` to classify each SV into a rearrangement catalogue based on size and type.

3. **Signature fitting**: Depending on the `signature.tools.lib` version:
   - Older versions (0.x or 1.x): Uses `SignatureFit_withBootstrap()` against the `RS.Breast560` reference matrix.
   - Newer versions (2.x+): Uses `signatureFit_pipeline()` with `RefSigv2` reference signatures and bootstrap-based fitting. This produces exposures for rearrangement signatures RefSigR1 through RefSigR20 (including R6a and R6b), along with percentage contributions and bootstrap p-values.

The genome build is automatically determined from the `params.genome` setting (GRCh38 maps to hg38, others map to hg19).

## Output Files

The process produces two output files:

- **`{idTumor}__{idNormal}_catalogues.pdf`**: A visualization of the SV rearrangement catalogue, showing the distribution of SVs across the classification bins.
- **`{idTumor}__{idNormal}_exposures.tsv`**: A tab-delimited file with columns for each rearrangement signature. For newer versions of `signature.tools.lib`, columns include raw exposure counts, percentage contributions (`_perc` suffix), and bootstrap p-values (`_pval` suffix) for each signature (RefSigR1 through RefSigR20, plus unassigned).

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/combined_svs/{idTumor}__{idNormal}_catalogues.pdf
outDir/somatic/{idTumor}__{idNormal}/combined_svs/{idTumor}__{idNormal}_exposures.tsv
```

## Example

```bash
# The pipeline internally runs:
Rscript sv_signatures_wrapper.R \
  -i sample_T__sample_N.final.clustered.bedpe \
  -g hg38 \
  -n 4 \
  -s sample_T__sample_N
```

## See Also

- `somatic-hrdetect.md` -- HRDetect uses SV signatures (SV3, SV5) as input features
- `somatic-signatures.md` -- SNV/indel mutational signatures (COSMIC SBS)
- `cohort-aggregates.md` -- Cohort-level files `sv_exposures.tsv` and `sv_catalogues.pdf`
- `somatic-sv-vcf.md` -- The BEDPE files that serve as input to SV signature analysis
