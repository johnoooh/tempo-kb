# MAF File Format: Structure and Tempo-Specific Columns

> **Quick answer:** A MAF (Mutation Annotation Format) file is a tab-delimited text file where each row represents a somatic or germline variant with genomic coordinates, functional annotations, and sample-level metrics. Tempo extends the standard MAF with caller metadata, filtering flags, gnomAD frequencies, hotspot annotations, OncoKB levels, raw allele counts, and CCF estimates.

## Standard MAF Columns

The MAF format was defined by the NCI Genomic Data Commons. Tempo MAF files are tab-delimited with no comment lines (unlike some other MAF producers that prepend `#version` headers). Key standard columns include:

- **Hugo_Symbol:** HGNC gene symbol (e.g., TP53, KRAS).
- **Chromosome, Start_Position, End_Position:** Genomic coordinates of the variant.
- **Reference_Allele, Tumor_Seq_Allele2:** Reference and alternate alleles.
- **Variant_Classification:** Functional consequence (e.g., Missense_Mutation, Nonsense_Mutation, Frame_Shift_Del, Splice_Site, Silent).
- **Variant_Type:** SNP, DNP, TNP, INS, or DEL.
- **Tumor_Sample_Barcode, Matched_Norm_Sample_Barcode:** Sample identifiers for tumor and matched normal.
- **HGVSc, HGVSp_Short:** HGVS coding and protein-level notation (e.g., p.V600E).
- **t_depth, t_alt_count:** Tumor total depth and variant-supporting read count (from the caller).
- **n_depth, n_alt_count:** Normal total depth and variant-supporting read count.
- **Transcript_ID:** Ensembl transcript used for annotation.

Tempo uses `vcf2maf` with VEP for functional annotation and supports custom preferred transcript isoforms via the `--custom-enst` flag.

## Tempo-Added Columns

### Caller Information

- **MuTect2:** Flag indicating the variant was called by MuTect2.
- **Strelka2:** Flag indicating the variant was called by Strelka2.
- **Strelka2FILTER:** Flag indicating Strelka2 detected but did not pass the variant.
- **Custom_filters:** Internal filter annotation field.

### Genomic Context and Blacklists

- **RepeatMasker:** Indicates the variant is in a RepeatMasker-annotated region.
- **EncodeDacMapability:** Indicates the variant is in a low-mappability region.
- **PoN:** Count of normal samples in the panel of normals where the variant was detected.
- **Ref_Tri:** Trinucleotide context of SNVs, normalized to pyrimidine base.

### gnomAD Allele Frequencies

For exomes, columns are prefixed with `non_cancer_` (non-TCGA gnomAD populations): `non_cancer_AC`, `non_cancer_AF`, `non_cancer_AF_popmax`, and population-specific columns (e.g., `non_cancer_AF_nfe`, `non_cancer_AF_afr`).

For genomes, columns use direct gnomAD naming: `AC`, `AF`, `AF_popmax`, and population-specific columns (e.g., `AF_nfe`, `AF_afr`, `AF_eas`).

### Raw Allele Counts (from GetBaseCountsMultiSample)

These unfiltered counts are independent of the variant callers:
- `t_alt_count_raw`, `t_ref_count_raw`, `t_depth_raw` (and `_fwd`/`_rev` strand-specific variants)
- `n_alt_count_raw`, `n_ref_count_raw`, `n_depth_raw` (and strand-specific)

### Strand Bias Flags

- **alt_bias:** True if all raw variant reads are on one strand (with raw depth >= 5).
- **ref_bias:** True if all raw reads are on one strand (with raw depth >= 5).

### Hotspot Annotations

- **snv_hotspot, threeD_hotspot:** Linear and 3D structural hotspot flags.
- **indel_hotspot, indel_hotspot_type:** Indel hotspot flag and type (`prior` or `novel`).
- **Hotspot:** TRUE if the variant is a linear SNV or indel hotspot.

### OncoKB Annotations

- **mutation_effect, oncogenic:** Functional effect and oncogenicity classification.
- **LEVEL_1 through LEVEL_R3, Highest_level:** Drug actionability levels.
- **citations:** OncoKB references.

Note: Tempo runs OncoKB without cancer type (ONCOTREE_CODE), so Level 1 and 2A annotations are not available. The highest possible level is 2B. Re-run OncoKB with cancer type for full annotations.

### Caller-Specific Metadata

From MuTect2: `MBQ`, `MFRL`, `MMQ`, `MPOS`, `OCM`, `RPA`, `STR`, `ECNT`.
From Strelka2: `MQ`, `SNVSB`, `FDP` (tumor/normal), `SUBDP` (tumor/normal), `RU`, `IC`.

### Clonality and Zygosity (added during MAF annotation with FACETS)

- **ccf_* columns:** Cancer cell fraction estimates under three copy-number configurations, with confidence intervals.
- **Zygosity columns:** Tumor zygosity based on observed VAF relative to expected VAF at the local copy number and purity.

## Output Files

Tempo produces two MAF files per tumor-normal pair:
- **Unfiltered MAF:** `<tumor>__<normal>.somatic.unfiltered.maf` -- all variants with FILTER column annotations (233 columns).
- **Filtered MAF:** `<tumor>__<normal>.somatic.final.maf` -- PASS variants plus rescued hotspots, with FACETS CCF annotation (276 columns).

## File Locations

- MAF annotation process: `tempo/modules/process/SNV/SomaticAnnotateMaf.nf`
- Output directory: `<outDir>/somatic/<tumor>__<normal>/combined_mutations/`

## Example

```bash
# View key columns from a Tempo MAF
cut -f1,5,6,9,10,40,41,42 TUMOR__NORMAL.somatic.maf | head
# Hugo_Symbol  Chromosome  Start_Position  Variant_Classification  ...  t_alt_count  Hotspot  oncogenic
```

## See Also

- `maf-filtering.md` -- How Tempo filters somatic mutations
- `facets-interpreting.md` -- FACETS output used for CCF annotation
- `signatures-msi.md` -- Mutation signatures derived from MAF trinucleotide context
