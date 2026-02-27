# Germline MAF and SV Files: HaplotypeCaller and Strelka2 Germline Variant Calls

> **Quick answer:** Tempo produces germline MAF files per normal sample containing annotated SNVs/indels from HaplotypeCaller and Strelka2, plus germline structural variant BEDPE files from Delly, Manta, and SvABA.

## What the Germline MAF Contains

The germline MAF captures inherited variants identified in the normal sample. It follows a similar annotation pipeline to the somatic MAF but uses HaplotypeCaller instead of MuTect2 and applies germline-specific filtering, including gnomAD-based exclusion and clonal hematopoiesis (CH) flagging.

## How Germline MAFs Are Built

**Step 1 -- Combine and filter VCFs (`GermlineCombineChannel`).** HaplotypeCaller and Strelka2 VCFs are intersected using `bcftools isec`. The union of PASS variants is taken, giving precedence to HaplotypeCaller at overlapping sites. RepeatMasker and ENCODE DAC mappability annotations are layered on. Unlike the somatic pipeline, gnomAD filtering is applied directly here: variants exceeding a gnomAD allele frequency threshold are excluded. GetBaseCountsMultiSample (GBCMS) adds tumor read counts at germline variant loci by genotyping the tumor BAM.

**Step 2 -- Annotate with VEP (`GermlineAnnotateMaf`).** The filtered VCF is converted to MAF using `vcf2maf` with VEP for functional consequence prediction. An R script (`filter-germline-maf.R`) applies germline-specific filter flags and generates both the `unfiltered.maf` and a filtered intermediate MAF.

**Step 3 -- FACETS annotation (`GermlineFacetsAnnotation`).** The filtered MAF is annotated with FACETS hisens copy number data, and tumor zygosity of each germline variant is estimated. The expected tumor VAF calculation differs from the somatic case because the variant is assumed to be present in both tumor and normal. The result is the `final.maf`.

## Key Germline MAF Columns

The germline MAF shares many columns with the somatic MAF but includes germline-specific fields:

- **Caller flags:** `HaplotypeCaller`, `Strelka2`, `Strelka2FILTER`
- **Blacklist annotations:** `RepeatMasker`, `EncodeDacMapability`
- **gnomAD:** Allele counts and frequencies (column naming differs between WES and WGS, same as somatic)
- **BRCA Exchange:** `brca_exchange_id`, `brca_exchange_enigma`, `brca_exchange_clinvar` -- annotations from the BRCA Exchange database for BRCA1/BRCA2 variants
- **Clonal hematopoiesis:** `ch_gene` -- boolean flag for genes associated with CH (DNMT3A, TET2, TP53, ASXL1, and 32 others)
- **Germline filter flags:** `low_n_depth`, `low_n_vaf`, `ch_mutation` (CH gene with low normal VAF and tumor VAF below 0.25), `t_in_n_contamination` (tumor VAF > 3x normal VAF)
- **Zygosity:** Tumor zygosity based on FACETS-derived copy number

## Germline Structural Variants

Tempo also calls germline SVs using three callers: Delly, Manta, and SvABA (BRASS is not used for germline analysis). The germline SV workflow mirrors the somatic SV pipeline but runs on the normal sample only:

1. Individual callers produce VCFs
2. `mergesvvcf` merges calls across callers (200bp window, matching directionality)
3. Merged VCF is converted to BEDPE with `svtools vcftobedpe`
4. Annotation and filtering with mappability, RepeatMasker, and PCAWG blacklists
5. iAnnotateSV adds gene-level structural variant annotations

The `unfiltered.bedpe` contains all merged calls with filter flags. The `final.bedpe` retains only PASS variants. The minimum supporting caller threshold is 1 for exome and 2 for genome.

## File Locations

```
outDir/germline/{idNormal}/combined_mutations/
  {idTumor}__{idNormal}.germline.unfiltered.maf
  {idTumor}__{idNormal}.germline.final.maf
  intermediate_files/
    {idNormal}.union.vcf.gz
    {idNormal}.union.pass.vcf.gz
    {idTumor}__{idNormal}.germline.vcf.gz

outDir/germline/{idNormal}/combined_svs/
  {idNormal}.unfiltered.bedpe
  {idNormal}.final.bedpe
  intermediate_files/
    {idNormal}.merged.raw.vcf.gz
    {idNormal}.merged.vcf.gz
    {idNormal}.combined.bedpe
```

Note: Germline SNV/indel output is organized under the normal sample ID, but the MAF filenames include both tumor and normal IDs because tumor BAM genotyping is performed at germline variant sites.

## Example

```bash
# View germline final MAF
head -3 outDir/germline/NORMAL1/combined_mutations/TUMOR1__NORMAL1.germline.final.maf

# Count germline SVs
grep -v "^#" outDir/germline/NORMAL1/combined_svs/NORMAL1.final.bedpe | wc -l

# Find pathogenic BRCA variants
awk -F'\t' '$brca_exchange_clinvar ~ /Pathogenic/' TUMOR1__NORMAL1.germline.final.maf
```

## See Also

- `somatic-maf.md` -- Somatic MAF generation with Mutect2 + Strelka2
- `somatic-sv-vcf.md` -- Somatic structural variant calling pipeline
- `somatic-facets.md` -- FACETS copy-number data used for germline zygosity annotation
