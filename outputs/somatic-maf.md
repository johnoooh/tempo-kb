# Somatic MAF Files: Combined Mutation Calls from Mutect2 and Strelka2

> **Quick answer:** Tempo produces two somatic MAF files per tumor-normal pair: an `unfiltered.maf` containing all annotated somatic mutations, and a `final.maf` that retains only PASS variants with FACETS-derived copy number and zygosity annotations.

## What the Somatic MAF Contains

The somatic MAF (Mutation Annotation Format) file is the primary deliverable for somatic SNV and indel analysis in Tempo. It represents the union of variant calls from MuTect2 and Strelka2, annotated with functional effects from VEP, OncoKB actionability levels, copy number-based clonality and zygosity estimates, and extensive quality metadata.

## How Somatic MAFs Are Built

The pipeline constructs the somatic MAF through a multi-step chain spanning two main processes, `SomaticCombineChannel` and `SomaticAnnotateMaf`, followed by FACETS annotation in `SomaticFacetsAnnotation`.

**Step 1 -- Combine and filter VCFs (`SomaticCombineChannel`).** MuTect2 and Strelka2 VCFs are intersected with `bcftools isec`. The union of PASS calls is taken, with precedence given to MuTect2 at overlapping sites. Annotations are layered on: RepeatMasker, ENCODE DAC Mappability, gnomAD allele frequencies, and panel-of-normals (PoN) counts. A custom Python script (`filter-vcf.py`) applies filter flags such as `low_vaf`, `low_t_depth`, `strand_bias`, `mappability`, `high_gnomad_pop_af`, and `PoN`. Variants passing all filters get `FILTER=PASS`. Raw allele counts from GetBaseCountsMultiSample (GBCMS) are added for both tumor and normal.

**Step 2 -- Annotate with VEP and OncoKB (`SomaticAnnotateMaf`).** The PASS VCF is converted to MAF format using `vcf2maf`, which internally runs VEP to predict functional consequences using preferred transcript isoforms. OncoKB annotations (mutation effect, oncogenicity, levels of actionability) are added. An R script (`filter-somatic-maf.R`) applies depth-based filters and produces the `unfiltered.maf` (all calls with filter flags) and a filtered intermediate MAF.

**Step 3 -- FACETS annotation (`SomaticFacetsAnnotation`).** The filtered MAF is annotated with FACETS hisens copy-number data using `facets-suite`. Cancer cell fraction (CCF) columns (`ccf_*`) are added for three copy-number configurations. Tumor zygosity is estimated from observed VAF, purity, and local copy number. The result is published as the `final.maf`.

## Key MAF Columns

Beyond standard MAF columns and VEP annotations, the somatic MAF includes:

- **Caller flags:** `MuTect2`, `Strelka2`, `Strelka2FILTER`, `Custom_filters`
- **Blacklist annotations:** `RepeatMasker`, `EncodeDacMapability`
- **Population frequencies:** gnomAD allele counts and frequencies (column naming differs between WES and WGS)
- **Raw allele counts:** `t_alt_count_raw`, `t_ref_count_raw`, `t_depth_raw` (and `_fwd`/`_rev` strand-specific variants), same for normal (`n_*`)
- **Bias flags:** `alt_bias`, `ref_bias`
- **Hotspot annotations:** `snv_hotspot`, `threeD_hotspot`, `indel_hotspot`, `Hotspot`
- **OncoKB:** `mutation_effect`, `oncogenic`, `LEVEL_*`, `Highest_level`, `citations`
- **Copy number and clonality:** `ccf_expected_copies`, `ccf_expected_copies_lower`/`upper`, plus major-allele and single-copy CCF estimates. The final MAF has 276 columns (vs 233 in the unfiltered MAF), with the additional columns coming from FACETS annotation.
- **Zygosity:** tumor zygosity based on VAF and FACETS-derived local ploidy
- **Other:** `PoN`, `Ref_Tri` (trinucleotide context), caller-specific metadata (MBQ, MFRL, MMQ, MPOS, etc.)

## Whitelisting Logic

Mutational hotspots (`Hotspot=TRUE`) are rescued in the final MAF even when flagged with `low_vaf` (if tumor VAF >= 0.02), `low_mapping_quality`, `low_t_depth`, or `strand_bias` individually. Combinations of these flags still result in filtering.

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/combined_mutations/
  {idTumor}__{idNormal}.somatic.unfiltered.maf
  {idTumor}__{idNormal}.somatic.final.maf
  intermediate_files/
    {idTumor}__{idNormal}.union.annot.vcf.gz
    {idTumor}__{idNormal}.union.annot.filter.vcf.gz
    {idTumor}__{idNormal}.union.annot.filter.pass.vcf.gz
```

## Example

```bash
# View the first few somatic mutations
head -3 outDir/somatic/TUMOR1__NORMAL1/combined_mutations/TUMOR1__NORMAL1.somatic.final.maf

# Count PASS mutations in the unfiltered MAF
awk -F'\t' '$NF ~ /PASS/' outDir/somatic/TUMOR1__NORMAL1/combined_mutations/TUMOR1__NORMAL1.somatic.unfiltered.maf | wc -l

# Extract hotspot mutations
awk -F'\t' 'NR==1 || $Hotspot=="TRUE"' TUMOR1__NORMAL1.somatic.final.maf
```

## See Also

- `germline-maf.md` -- Germline MAF generation with HaplotypeCaller + Strelka2
- `somatic-facets.md` -- FACETS copy-number analysis that feeds into MAF annotation
- `somatic-sv-vcf.md` -- Structural variant calling and filtering
