# Structural Variant Output Formats: VCF, BEDPE, and SV Types in Tempo

> **Quick answer:** Tempo merges SV calls into a combined VCF, converts to BEDPE format using svtools, then annotates and filters the BEDPE. The final output is a BEDPE file with SV type (DEL, DUP, INV, BND), filter status, iAnnotateSV gene annotations, blacklist flags, and ClusterSV clustering information.

## SV Types

Each structural variant in Tempo is classified into one of four types:

| Type | Full Name | Description |
|------|-----------|-------------|
| **DEL** | Deletion | Loss of a genomic segment spanned by two joined breakends. |
| **DUP** | Tandem Duplication | Extra copy of a segment immediately downstream, in the same orientation. |
| **INV** | Inversion | A segment inserted into its original position but in the opposite orientation. Simple inversions are balanced; complex inversions may have one unrescued breakend. |
| **BND** | Breakend | Any event that cannot be described as DEL, DUP, or INV. The majority of BND calls are translocations between different chromosomes. |

Note: Insertions (INS) may appear in individual caller outputs but are typically represented as BND events after merging.

## Merged VCF Format

After merging with `mergesvvcf`, the combined VCF contains standard VCF fields plus SV-specific INFO fields:

- **SVTYPE:** The structural variant type (DEL, DUP, INV, BND).
- **CHR2:** The chromosome of the second breakend (for inter-chromosomal events).
- **END:** Position of the second breakend (for intra-chromosomal events).
- **STRANDS:** Orientation of the breakpoint strands (e.g., `++`, `+-`, `-+`, `--`).
- **CALLERS:** Which callers support this variant.
- **NUM_CALLERS:** Number of supporting callers.

The merged VCF ID field is formatted as: `TEMPO_<SVTYPE>_<CHROM>_<POS>_<CHR2>_<END>_<STRANDS>`.

## BEDPE Format

The merged VCF is converted to BEDPE (Browser Extensible Data Paired-End) format using `svtools vcftobedpe`. BEDPE describes each breakpoint as a pair of genomic intervals. The standard columns are:

| Column | Description |
|--------|-------------|
| CHROM_A | Chromosome of breakend A |
| START_A, END_A | Coordinates of breakend A |
| CHROM_B | Chromosome of breakend B |
| START_B, END_B | Coordinates of breakend B |
| ID | Variant identifier |
| QUAL | Quality score |
| STRAND_A, STRAND_B | Strand orientation of each breakend |
| TYPE | SV type (DEL, DUP, INV, BND) |
| FILTER | Filter status (PASS or filter flags) |
| INFO_A, INFO_B | INFO fields for each breakend |
| FORMAT | Format field description |
| TUMOR | Tumor sample genotype data |
| NORMAL | Normal sample genotype data |
| TUMOR_ID | Tumor sample identifier |
| NORMAL_ID | Normal sample identifier |

## Annotation and Filtering of BEDPE

The `SomaticAnnotateSVBedpe` process applies sequential blacklist filters to the BEDPE file:

1. **mappability:** Breakend in a low-mappability region (ENCODE DAC).
2. **repeat_masker:** Breakend in a RepeatMasker-annotated region.
3. **pcawg_blacklist_bed:** Breakend in a PCAWG-blacklisted region.
4. **pcawg_blacklist_bedpe:** Breakpoint matches a PCAWG-blacklisted breakpoint pair.
5. **pcawg_blacklist_fb_bedpe:** Breakpoint is a likely foldback artifact.
6. **pcawg_blacklist_te_bedpe:** Breakpoint is a likely transposable element artifact.

These blacklists are sourced from the PCAWG SV merging publication.

Additional annotations:
- **cDNA contamination detection:** `detect_cdna.py` flags deletion events spanning splice sites as possible cDNA contamination (not filtered, only flagged).
- **Gene annotation:** `iAnnotateSV` adds gene-level breakpoint annotations.

## ClusterSV Analysis

The `SomaticRunClusterSV` process analyzes spatial clustering of SVs, producing:
- `*.sv_clusters_and_footprints.tsv`: Cluster assignments with footprint boundaries.
- `*.sv_distance_pvals`: Statistical significance of clustering.
- `*.clustered.bedpe`: The final BEDPE with appended cluster columns including `cluster_id`, `cluster_total_count`, footprint IDs, and p-values.

## Output Files

```
<outDir>/somatic/TUMOR__NORMAL/combined_svs/
  TUMOR__NORMAL.unfiltered.bedpe          # All SVs with filter flags
  TUMOR__NORMAL.final.bedpe               # PASS SVs only
  TUMOR__NORMAL.final.clustered.bedpe     # PASS SVs with cluster annotations
  intermediate_files/
    TUMOR__NORMAL.merged.vcf.gz           # Merged VCF before BEDPE conversion
    TUMOR__NORMAL.combined.bedpe          # BEDPE before filtering
```

## File Locations

- VCF-to-BEDPE: `tempo/modules/process/SV/SomaticSVVcf2Bedpe.nf`
- BEDPE annotation: `tempo/modules/process/SV/SomaticAnnotateSVBedpe.nf`
- ClusterSV: `tempo/modules/process/SV/SomaticRunClusterSV.nf`
- SV documentation: `tempo/docs/variant-annotation-and-filtering.md`

## Example

```bash
# View first few SVs from the final BEDPE
grep -v "^#" TUMOR__NORMAL.final.bedpe | head -3 | cut -f1-12
# chr1  1000  1001  chr1  50000  50001  TEMPO_DEL_chr1_1000_chr1_50000_+-  .  +  -  DEL  PASS
```

## See Also

- `sv-callers.md` -- How each SV caller works and how calls are merged
- `svclone-clonality.md` -- Clonality estimation for SVs using BEDPE input
- `hrdetect.md` -- SV signatures derived from BEDPE for HRD prediction
