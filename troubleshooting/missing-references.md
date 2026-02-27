# Resolving Missing Reference Files and Genome Build Errors in Tempo

> **Quick answer:** Tempo requires reference files at the path defined by `params.reference_base` (default: `/juno/work/tempo/cmopipeline`). Most "file not found" errors are caused by incorrect paths, missing index files, or genome build mismatches between input data and the GRCh37/b37 reference that Tempo uses.

## Common Reference Error Messages

Reference file errors typically appear in `.command.err` with messages like:

```
FileNotFoundException: /juno/work/tempo/cmopipeline/ref/...
```

or tool-specific errors such as:

```
ERROR: Unable to open reference genome FASTA
htsjdk.samtools.SAMException: Cannot read non-existent file
```

These indicate that a required reference file is either missing, has the wrong path, or lacks an accompanying index file.

## Checking Reference File Paths

Tempo's reference base directory is set in the pipeline configuration:

```groovy
params.reference_base = "/juno/work/tempo/cmopipeline"
```

To verify that the reference directory and its contents are accessible:

```bash
ls -la /juno/work/tempo/cmopipeline/
ls -la /juno/work/tempo/cmopipeline/ref/
```

If you are running Tempo with a custom configuration that overrides `reference_base`, ensure the new path exists and contains all required subdirectories.

## Genome Build: GRCh37 (b37 with hg19 Decoys)

Tempo uses **GRCh37** as its default genome build, specifically b37 with hg19 decoy contigs. This is critical because:

- Input BAM files must be aligned to the same genome build (b37). BAMs aligned to GRCh38/hg38 will cause mismatches and errors in variant calling.
- Reference FASTA, known-sites VCFs, and interval files must all use b37 coordinates.
- Chromosome naming follows the `1, 2, 3, ...` convention (not `chr1, chr2, chr3`).

If you see errors about missing contigs or coordinate mismatches, check whether your input files use a different genome build.

## Required Reference Files

Tempo depends on several categories of reference files. If any are missing, the corresponding pipeline step will fail:

### Genome References
- Reference FASTA (`.fa`) with `.fai` index and `.dict` dictionary
- BWA index files (`.amb`, `.ann`, `.bwt`, `.pac`, `.sa`)

### Known Sites VCFs
- dbSNP VCF (with `.tbi` index)
- Mills and 1000 Genomes gold-standard indels
- 1000 Genomes phase 1 indels
- gnomAD VCF for germline filtering
- HapMap VCF for VQSR

### Analysis-Specific Files
- **FACETS SNP VCF** -- required for allele-specific copy number analysis. Missing this file causes DoFacets to fail immediately.
- **VEP cache** -- the Variant Effect Predictor cache directory must be pre-populated. VEP cannot download the cache during pipeline execution on compute nodes without internet access.
- **Panel of Normals (PoN)** -- used by Mutect2 for filtering artifacts in somatic calling.
- **Capture kit BED files** -- define the targeted regions for WES. The correct BED file must match the capture kit used in sequencing (e.g., Agilent SureSelect, IDT xGen).
- **HLA references** -- required for LOHHLA (loss of heterozygosity in HLA) analysis.

## Missing Index Files

Many tools require index files alongside the primary reference. A common oversight is having the FASTA or VCF present but missing its index:

| Primary File | Required Index |
|-------------|---------------|
| `reference.fa` | `reference.fa.fai`, `reference.dict` |
| `dbsnp.vcf.gz` | `dbsnp.vcf.gz.tbi` |
| `known_indels.vcf.gz` | `known_indels.vcf.gz.tbi` |

If an index is missing, regenerate it:

```bash
samtools faidx reference.fa
samtools dict reference.fa > reference.dict
tabix -p vcf dbsnp.vcf.gz
```

## Verifying All References

Run a quick check of the main reference directories:

```bash
# Check genome reference
ls -la /juno/work/tempo/cmopipeline/ref/

# Check known sites
ls -la /juno/work/tempo/cmopipeline/known_sites/

# Check VEP cache
ls -la /juno/work/tempo/cmopipeline/vep_cache/

# Check capture kit BED files
ls -la /juno/work/tempo/cmopipeline/bed_files/
```

## File Locations

| File | Location |
|------|----------|
| Reference base | `/juno/work/tempo/cmopipeline/` |
| Genome FASTA | `ref/` subdirectory |
| Known sites VCFs | `known_sites/` subdirectory |
| VEP cache | `vep_cache/` subdirectory |
| Pipeline config | `nextflow.config` and `conf/references.config` |

## Example

The DoFacets process fails with a missing SNP VCF error:

```bash
# Check the error
cat work/ab/cd1234567890/.command.err

# Verify the FACETS VCF exists
ls -la /juno/work/tempo/cmopipeline/facets_snps/

# If the path is wrong, override in params
nextflow run dsl2.nf \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  -profile juno \
  --outDir results \
  --facets_vcf /correct/path/to/facets_snps.vcf.gz \
  -resume
```

## See Also

- [Diagnosing Failed Processes](failed-processes.md) -- locating error messages in work directories
- [Juno HPC Cluster-Specific Issues](juno-specific.md) -- verifying filesystem access on Juno
- [Resuming Interrupted Runs](nextflow-resume.md) -- resuming after fixing reference paths
