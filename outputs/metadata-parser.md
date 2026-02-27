# Consolidated Sample Metadata File (MetaDataParser) in Tempo

> **Quick answer:** The MetaDataParser process collects results from FACETS, MSIsensor, mutational signatures, Polysolver HLA typing, and the somatic MAF to produce a single tab-delimited summary file per tumor-normal pair containing purity, ploidy, WGD status, MSI metrics, all mutational signature exposures, HLA genotypes, mutation count, and TMB.

## What the Metadata File Contains

The `{idTumor}__{idNormal}.sample_data.txt` file is a one-row-per-sample tab-delimited file that consolidates key results from multiple pipeline processes into a single convenient summary. The columns include:

- **sample**: The tumor-normal pair identifier (`{idTumor}__{idNormal}`)
- **purity**: Tumor purity estimated by FACETS (fraction of tumor cells in the sample)
- **ploidy**: Average tumor ploidy estimated by FACETS
- **WGD_status**: Whole-genome doubling status from FACETS QC (True, False, or NA)
- **MSI_Total_Sites**: Total microsatellite loci examined by MSIsensor
- **MSI_Somatic_Sites**: Number of somatically unstable microsatellite loci
- **MSIscore**: MSI percentage score (somatic sites / total sites)
- **Mutational signature columns**: All signature exposures from tempoSig (either 30 columns for COSMIC v2 or 60 columns for COSMIC v3, plus p-values)
- **HLA-A1, HLA-A2, HLA-B1, HLA-B2, HLA-C1, HLA-C2**: The six HLA class I allele calls from Polysolver
- **Number_of_Mutations**: Total number of somatic mutations in the MAF
- **TMB**: Tumor mutational burden, calculated as the number of non-synonymous coding mutations per megabase of coding sequence

The file typically contains ~149 columns. The signature columns follow the naming pattern `SBS{N}.observed` and `SBS{N}.pvalue` for each COSMIC SBS signature (e.g., `SBS1.observed`, `SBS1.pvalue`, `SBS2.observed`, `SBS2.pvalue`, etc.).

## How TMB Is Calculated

The MetaDataParser computes TMB by:

1. Reading the somatic MAF and filtering to non-synonymous variant classes: Missense_Mutation, Nonsense_Mutation, Nonstop_Mutation, Frame_Shift_Ins, Frame_Shift_Del, In_Frame_Del, In_Frame_Ins, Translation_Start_Site, Splice_Site, and Splice_Region.
2. Excluding germline mutations (Mutation_Status != 'GERMLINE').
3. Intersecting the remaining mutations with the assay-specific coding regions BED file using pybedtools.
4. Dividing the count of intersecting mutations by the total coding region size (in megabases) from the BED file.

The coding region size varies by assay and bait set. Common values include approximately 30.9 Mb for AgilentExon_51MB_b37_v3, 36.0 Mb for IDT_Exome_v1_FP, and ~45 Mb for WGS.

## Inputs to the Process

The MetaDataParser process receives outputs from six upstream processes:

1. **FACETS purity output** (`*_purity.out`): Purity and ploidy values
2. **FACETS QC output** (`*.facets_qc.txt`): WGD status
3. **Somatic MAF** (`*.somatic.final.maf`): For mutation counting and TMB
4. **MSIsensor output** (`*.msisensor.tsv`): MSI metrics
5. **tempoSig output** (`*.mutsig.txt`): Mutational signature exposures
6. **Polysolver output** (`*.hla.txt`): HLA genotypes
7. **Coding regions BED**: Assay-specific coding exon intervals

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/meta_data/{idTumor}__{idNormal}.sample_data.txt
```

## Example

```bash
# The pipeline internally runs:
create_metadata_file.py \
  --sampleID sample_T__sample_N \
  --tumorID sample_T \
  --normalID sample_N \
  --facetsPurity_out sample_purity.out \
  --facetsQC sample.facets_qc.txt \
  --MSIsensor_output sample_T__sample_N.msisensor.tsv \
  --mutational_signatures_output sample_T__sample_N.mutsig.txt \
  --polysolver_output sample_N.hla.txt \
  --MAF_input sample_T__sample_N.somatic.final.maf \
  --coding_baits_BED ensGene.all_CODING_exons.reference.bed

# The output file has one header row and one data row with all metrics
```

## See Also

- `somatic-facets.md` -- Source of purity, ploidy, and WGD status
- `somatic-msi.md` -- Source of MSI metrics
- `somatic-signatures.md` -- Source of mutational signature exposures
- `somatic-hla.md` -- Source of HLA genotypes
- `cohort-aggregates.md` -- Cohort-level `sample_data.txt` combining all pairs
