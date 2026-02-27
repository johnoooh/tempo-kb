# Somatic Mutation Filtering in Tempo: From Caller VCFs to Final MAF

> **Quick answer:** Tempo merges Mutect2 and Strelka2 calls (union, with Mutect2 precedence), annotates with VEP/vcf2maf, applies soft filters for depth/VAF/gnomAD/PoN/blacklists, keeps only PASS variants, and rescues mutational hotspots that fail certain filters. The final MAF contains high-confidence somatic mutations.

## Step 1: Caller Merging (SomaticCombineChannel)

Tempo takes the FILTER=PASS VCFs from both Mutect2 and Strelka2 and computes set operations using `bcftools isec`:

- **Mutect2-only variants** are tagged with `MuTect2`.
- **Strelka2-only variants** are tagged with `Strelka2`.
- **Shared variants** are tagged with both `MuTect2` and `Strelka2`. If Strelka2 did not pass the variant at a shared site, the `Strelka2FILTER` flag is added (resulting in a `caller_conflict` filter downstream).

The three sets are concatenated, deduplicated, and sorted into a union VCF.

## Step 2: Blacklist and Database Annotation

The union VCF is annotated with:

1. **RepeatMasker regions:** Tags variants in repetitive elements.
2. **ENCODE DAC Mappability:** Tags variants in low-mappability regions.
3. **gnomAD allele frequencies:** Adds population allele frequencies. Exome runs use the non-cancer gnomAD exome database; genome runs use the gnomAD genome database.
4. **Panel of Normals (PoN):** Adds the count of normal samples where the variant was seen. Exome and genome have separate PoN files.
5. **Indel annotation:** `vt annotate_indels` adds flanking sequence context.

## Step 3: Custom Filter Annotation

The `filter-vcf.py` script applies soft filter flags to each variant. The FILTER column can contain any semicolon-separated combination of:

- **low_vaf:** Tumor VAF below threshold (exome/genome default: 0.05).
- **low_t_depth:** Tumor depth below threshold (default: 20).
- **low_t_alt_count:** Tumor alt read count below threshold (default: 3).
- **low_n_depth:** Normal depth below threshold (default: 10).
- **high_n_alt_count:** Normal alt count exceeds threshold (default: 3).
- **high_gnomad_pop_af:** gnomAD population AF exceeds threshold (default: 0.01).
- **PoN:** PoN count exceeds threshold (default: 10).
- **mappability / repeatmasker:** Variant in blacklisted genomic region.
- **strand_bias:** All supporting reads from one strand (with sufficient depth).
- **caller_conflict:** Both callers detected the variant but Strelka2 did not pass it.
- **part_of_mnv:** Variant is likely part of a multi-nucleotide variant.
- **multiallelic2:** Multiallelic locus from Strelka2.
- **low_mapping_quality:** For Strelka2 indels with low mapping quality.

After filter annotation, only variants with `FILTER=PASS` are retained.

## Step 4: Raw Read Count Addition

`GetBaseCountsMultiSample` (GBCMS) re-counts reads at every variant position using relaxed criteria (mapping quality 0), providing caller-independent allele counts for both tumor and normal. These raw counts populate the `*_raw` and strand-specific columns in the final MAF.

## Step 5: VEP Annotation and MAF Conversion (SomaticAnnotateMaf)

The PASS VCF is converted to MAF format using `vcf2maf`, which runs VEP for functional annotation. The `--custom-enst` flag applies preferred transcript isoforms, and all INFO/FORMAT fields from previous steps are retained in the MAF using `--retain-info` and `--retain-fmt`.

## Step 6: OncoKB Annotation

The MAF is annotated with OncoKB for mutation effect, oncogenicity, and drug actionability levels. Because Tempo does not know the cancer type, Level 1 and 2A annotations are not available.

## Step 7: Final R-Based Filtering

The `filter-somatic-maf.R` script applies the parameterized thresholds to produce the final filtered MAF. The parameters are:

| Parameter | Exome Default | Genome Default |
|-----------|--------------|----------------|
| tumorVaf | 0.05 | 0.05 |
| tumorDepth | 20 | 20 |
| tumorCount | 3 | 3 |
| normalDepth | 10 | 10 |
| normalCount | 3 | 3 |
| gnomadAf | 0.01 | 0.01 |
| ponCount | 10 | 10 |

## Hotspot Whitelisting (Rescue)

Mutational hotspots (`Hotspot=TRUE`) are rescued into the filtered MAF if they:

- Are flagged only with `low_vaf` and the tumor VAF is at least 0.02.
- Are flagged only with `low_mapping_quality`, `low_t_depth`, or `strand_bias`.

Combinations of multiple filter flags still result in filtering, even for hotspots.

## File Locations

- Merging process: `tempo/modules/process/SNV/SomaticCombineChannel.nf`
- MAF annotation process: `tempo/modules/process/SNV/SomaticAnnotateMaf.nf`
- Filter thresholds: `tempo/conf/exome.config` and `tempo/conf/genome.config`
- Filtering documentation: `tempo/docs/variant-annotation-and-filtering.md`

## Example

```bash
# Output files
<outDir>/somatic/TUMOR__NORMAL/combined_mutations/
  TUMOR__NORMAL.somatic.unfiltered.maf   # All variants with filter flags
  TUMOR__NORMAL.somatic.maf              # Final filtered MAF
  intermediate_files/
    TUMOR__NORMAL.union.annot.vcf.gz             # After blacklist annotation
    TUMOR__NORMAL.union.annot.filter.vcf.gz      # After filter flags added
    TUMOR__NORMAL.union.annot.filter.pass.vcf.gz # PASS + raw counts
```

## See Also

- `maf-format.md` -- MAF column definitions
- `sv-callers.md` -- SV filtering uses a parallel but different approach
- `signatures-msi.md` -- Mutation signatures use the filtered MAF
