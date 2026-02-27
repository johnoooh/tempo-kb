# Tempo Pipeline Reference Genomes, Known Sites, and Annotation Files

> **Quick answer:** Tempo defaults to GRCh37 (`params.genome = 'GRCh37'`). All reference paths are defined in `conf/references.config` and resolve relative to `params.reference_base` (on Juno: `/juno/work/tempo/cmopipeline`) and `params.genome_base` (the iGenomes GATK bundle directory). GRCh38 is partially supported but not fully validated.

## Genome Build Support

- **GRCh37** (default): Fully supported. Set with `--genome GRCh37` or by leaving the default.
- **GRCh38**: Defined in references.config but noted as "not fully supported" in the source comments.
- **smallGRCh37**: A reduced reference set used for testing purposes only.

The `params.genome` value controls which reference block is loaded from `conf/references.config`.

## Reference Base Paths (Juno)

On Juno, the reference hierarchy is:

- `params.reference_base = "/juno/work/tempo/cmopipeline"`
- `params.genome_base` for GRCh37: `${reference_base}/mskcc-igenomes/igenomes/Homo_sapiens/GATK/GRCh37`
- `params.targets_base`: `${reference_base}/mskcc-igenomes/grch37/tempo_targets`

<!-- TODO: VERIFY WITH USER -- params.targets_base uses ${params.genome.toLowerCase()} so GRCh38 targets would resolve to ${reference_base}/mskcc-igenomes/grch38/tempo_targets -->

## Reference FASTA and Indices (GRCh37)

| File | Path |
|------|------|
| Genome FASTA | `${genome_base}/Sequence/WholeGenomeFasta/human_g1k_v37_decoy.fasta` |
| FASTA index | `${genome_base}/Sequence/WholeGenomeFasta/human_g1k_v37_decoy.fasta.fai` |
| Sequence dict | `${genome_base}/Sequence/WholeGenomeFasta/human_g1k_v37_decoy.dict` |
| BWA index | `${genome_base}/Sequence/BWAIndex/human_g1k_v37_decoy.fasta.{amb,ann,bwt,pac,sa}` |
| Intervals BED | `${genome_base}/Annotation/intervals/human.b37.genome.bed` |

## Known Sites VCFs (GRCh37)

These files are used during Base Quality Score Recalibration (BQSR) and variant filtering:

- **dbSNP**: `${genome_base}/Annotation/GATKBundle/dbsnp_138.b37.vcf` (plus `.idx`)
- **Known indels**: `${genome_base}/Annotation/GATKBundle/{1000G_phase1,Mills_and_1000G_gold_standard}.indels.b37.vcf` (plus `.idx`)

The brace expansion resolves to two files: 1000G Phase 1 indels and Mills & 1000G Gold Standard indels.

## FACETS Reference VCF

FACETS uses a curated dbSNP VCF for allele-specific copy number analysis:

`${reference_base}/mskcc-igenomes/igenomes/Homo_sapiens/GATK/b37/dbsnp_137.b37__RmDupsClean__plusPseudo50__DROP_SORT.vcf`

## VEP Cache

Variant Effect Predictor annotation uses a local cache:

- **GRCh37 VEP cache**: `${reference_base}/mskcc-igenomes/grch37/vep`
- **GRCh38 VEP cache version**: `95` (set via `vepCacheVersion`)

<!-- TODO: VERIFY WITH USER -- GRCh37 vepCacheVersion is not explicitly set in the GRCh37 block of references.config; it is set to "88" only in the smallGRCh37 block. Confirm which VEP version is used in production. -->

## gnomAD VCFs (GRCh37)

Used for population allele frequency annotation and filtering:

- **Exome**: `${reference_base}/mskcc-igenomes/grch37/gnomad/gnomad.exomes.r2.1.1.sites.non_cancer.vcf.gz` (plus `.tbi`)
- **Genome**: `${reference_base}/mskcc-igenomes/grch37/gnomad/gnomad.genomes.r2.1.1.sites.minimal.vcf.gz` (plus `.tbi`)

## Panel of Normals (PoN)

Separate PoN VCFs are provided for exome and whole genome sequencing:

- **Exome PoN**: `${reference_base}/mskcc-igenomes/grch37/annotation/wes.pon.vcf.gz` (plus `.tbi`)
- **WGS PoN**: `${reference_base}/mskcc-igenomes/grch37/annotation/wgs.pon.vcf.gz` (plus `.tbi`)

## Capture Kit BED Files (Targets)

Target and bait intervals are resolved per capture kit using `params.targets_base` and a `targets_id` variable:

- `baitsInterval`: `${targets_base}/${targets_id}/baits.interval_list`
- `targetsInterval`: `${targets_base}/${targets_id}/targets.interval_list`
- `targetsBed`: `${targets_base}/${targets_id}/targets.bed`
- `targetsBedGz`: `${targets_base}/${targets_id}/targets.bed.gz` (plus `.tbi`)
- `codingBed`: `${targets_base}/${targets_id}/coding.bed`

<!-- TODO: VERIFY WITH USER -- The available targets_id values (e.g., agilent, idt, idt_v2, wgs) should be confirmed by listing the actual directories under the targets_base path on Juno. -->

## HLA Reference Files

Used by Polysolver and LOHHLA for HLA typing:

- `hlaFasta`: `${reference_base}/mskcc-igenomes/grch37/hla/abc_complete.fasta`
- `hlaDat`: `${reference_base}/mskcc-igenomes/grch37/hla/hla.dat`

## SV Calling Blacklists

Structural variant calling uses PCAWG blacklists:

- `svBlacklistBed`: `${reference_base}/mskcc-igenomes/grch37/sv_calling/pcawg6_blacklist.slop.bed.gz`
- `svBlacklistBedpe`: `${reference_base}/mskcc-igenomes/grch37/sv_calling/pcawg6_blacklist.slop.bedpe.gz`
- Foldback artefact and TE/pseudogene blacklists also provided.

## File Locations

- Reference definitions: `conf/references.config`
- Juno base path config: `conf/juno.config` (sets `params.reference_base`)

## Example

```bash
# Override genome build at the command line
nextflow run tempo -profile juno --genome GRCh37 --mapping mapping.tsv
```

## See Also

- [config-profiles.md](config-profiles.md) -- How profiles set reference_base and genome_base
- [config-containers.md](config-containers.md) -- Container images that consume these references
