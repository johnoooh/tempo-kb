# Loading and Exploring Tempo MAF Files in Python

> **Quick answer:** Use `pd.read_csv()` with `sep='\t'` and `comment='#'` to load Tempo MAF files into a pandas DataFrame, then filter and select columns for downstream analysis.

## What Is a MAF File

MAF (Mutation Annotation Format) is a tab-delimited text file that contains one row per somatic mutation. Tempo produces MAF files annotated with variant caller information, functional annotations from VEP, oncogenicity from OncoKB, and copy number data from FACETS. The first few lines of a MAF file begin with `#` comment headers that must be skipped when reading.

## MAF File Location

Tempo writes the final filtered and annotated MAF to:

```
outDir/somatic/{idTumor}__{idNormal}/combined_mutations/{idTumor}__{idNormal}.somatic.final.maf
```

The double-underscore (`__`) separates the tumor sample ID from the matched normal sample ID. There is also an unfiltered MAF in the same directory, but you should use the `.somatic.final.maf` for most analyses.

## Loading the MAF with Pandas

```python
import pandas as pd

maf_path = "outDir/somatic/TUMOR_01__NORMAL_01/combined_mutations/TUMOR_01__NORMAL_01.somatic.final.maf"

maf = pd.read_csv(maf_path, sep='\t', comment='#', low_memory=False)
print(f"Loaded {len(maf)} mutations across {maf['Tumor_Sample_Barcode'].nunique()} sample(s)")
print(maf.columns.tolist())
```

The `comment='#'` argument tells pandas to skip any line starting with `#`, which handles the MAF header comments automatically. Use `low_memory=False` to avoid mixed-type warnings on large MAFs.

## Key Columns

These are the most commonly used columns for analysis:

| Column | Description |
|--------|-------------|
| `Hugo_Symbol` | Gene name (e.g., TP53, KRAS) |
| `Chromosome` | Chromosome of the variant |
| `Start_Position` | Genomic start coordinate |
| `Variant_Classification` | Functional consequence (e.g., Missense_Mutation) |
| `Variant_Type` | SNP, DNP, INS, DEL |
| `Tumor_Sample_Barcode` | Tumor sample identifier |
| `t_alt_count` | Alternate allele read count in tumor |
| `t_depth` | Total read depth in tumor |
| `HGVSp_Short` | Protein change shorthand (e.g., p.V600E) |
| `oncogenic` | OncoKB oncogenicity annotation |

Select a working subset of columns:

```python
key_cols = [
    'Hugo_Symbol', 'Chromosome', 'Start_Position', 'End_Position',
    'Variant_Classification', 'Variant_Type', 'Reference_Allele',
    'Tumor_Seq_Allele2', 'Tumor_Sample_Barcode',
    't_alt_count', 't_ref_count', 't_depth',
    'n_alt_count', 'n_depth',
    'HGVSp_Short', 'oncogenic'
]
maf_slim = maf[key_cols].copy()
```

## Filtering for PASS Variants

The final MAF from Tempo should already contain only filtered (PASS) variants. However, if you are working with an unfiltered MAF or want to apply additional filters, check the `FILTER` column:

```python
# Only PASS variants
maf_pass = maf[maf['FILTER'] == 'PASS'].copy()
print(f"PASS variants: {len(maf_pass)} out of {len(maf)} total")
```

## Filtering by Variant Classification

To focus on coding or non-silent mutations:

```python
non_silent = [
    'Missense_Mutation', 'Nonsense_Mutation', 'Frame_Shift_Del',
    'Frame_Shift_Ins', 'In_Frame_Del', 'In_Frame_Ins',
    'Splice_Site', 'Translation_Start_Site', 'Nonstop_Mutation'
]
maf_coding = maf[maf['Variant_Classification'].isin(non_silent)].copy()
print(f"Non-silent mutations: {len(maf_coding)}")
```

## Quick Exploration

```python
# Most frequently mutated genes
print(maf['Hugo_Symbol'].value_counts().head(20))

# Distribution of variant classifications
print(maf['Variant_Classification'].value_counts())

# OncoKB oncogenic variants only
oncogenic_muts = maf[maf['oncogenic'].isin(['Oncogenic', 'Likely Oncogenic'])]
print(f"Oncogenic mutations: {len(oncogenic_muts)}")
```

## File Locations

- Final MAF: `outDir/somatic/{idTumor}__{idNormal}/combined_mutations/{idTumor}__{idNormal}.somatic.final.maf`
- Unfiltered MAF: same directory, without the `.final` suffix
- Aggregated cohort MAF (if `--aggregate` was used): `outDir/cohort_level/`

## See Also

- [calculate-tmb.md](calculate-tmb.md) -- computing TMB from the MAF
- [calculate-vaf.md](calculate-vaf.md) -- computing variant allele frequency
- [merge-maf-clinical.md](merge-maf-clinical.md) -- merging MAF with clinical metadata
- [oncoplot-maftools.md](oncoplot-maftools.md) -- visualizing mutations with R maftools
