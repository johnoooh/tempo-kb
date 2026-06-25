# Supplying Your Own Reference Data (Off-Juno)

> **Quick answer:** All Tempo reference paths are built from `params.reference_base` (default `/juno/work/tempo/cmopipeline`) plus a fixed sub-path layout defined in `conf/references.config`. To run elsewhere, mirror that directory layout under your own `reference_base` and point Tempo at it with `--reference_base /your/path`. Individual files can also be overridden by their `--<param>` name. **GRCh37/b37 is the validated build; GRCh38 is not fully supported.**

> 📖 **Authoritative source:** the official [reference-files.md](https://github.com/mskcc/tempo/blob/develop/docs/reference-files.md) ([docs site](https://cmotempo.netlify.app/)) is the source of truth for reference data — and it has a detailed **"Custom target files"** section (how to build `targets.bed`/`baits.interval_list`/`coding.bed` for a new bait set, and the `targets_base` folder layout) that this page does not reproduce. Use it for custom **targets**; use the param table below to map the broader **reference_base** layout when running off-Juno.

## The base parameters (verified — `conf/juno.config`)

| Param | Default | Override |
|-------|---------|----------|
| `reference_base` | `/juno/work/tempo/cmopipeline` | `--reference_base /your/path` |
| `genome_base` | computed from `params.genome` (e.g. `…/igenomes/Homo_sapiens/GATK/GRCh37`) | `--genome_base …` |
| `targets_base` | `${reference_base}/mskcc-igenomes/${genome.toLowerCase()}/tempo_targets` | `--targets_base …` |

## GRCh37 reference layout (verified — `conf/references.config`)

Paths below are relative to `genome_base` unless they start with `${reference_base}`. Recreate this tree under your own base.

| Param (`--name` to override) | Path |
|------------------------------|------|
| `genomeFile` | `Sequence/WholeGenomeFasta/human_g1k_v37_decoy.fasta` |
| `genomeIndex` | `${genomeFile}.fai` |
| `genomeDict` | `Sequence/WholeGenomeFasta/human_g1k_v37_decoy.dict` |
| `bwaIndex` | `Sequence/BWAIndex/human_g1k_v37_decoy.fasta.{amb,ann,bwt,pac,sa}` |
| `intervals` | `Annotation/intervals/human.b37.genome.bed` |
| `dbsnp` | `Annotation/GATKBundle/dbsnp_138.b37.vcf` |
| `knownIndels` | `Annotation/GATKBundle/{1000G_phase1,Mills_and_1000G_gold_standard}.indels.b37.vcf` |
| `facetsVcf` | `${reference_base}/…/b37/dbsnp_137.b37__RmDupsClean__plusPseudo50__DROP_SORT.vcf` |
| `vepCache` | `${reference_base}/mskcc-igenomes/grch37/vep` (cache version 88) |
| `msiSensorList` | `Sequence/WholeGenomeFasta/human_g1k_v37_decoy.fasta.microsatellites.list` |
| `gnomadWesVcf` | `${reference_base}/…/grch37/gnomad/gnomad.exomes.r2.1.1.sites.non_cancer.vcf.gz` |
| `gnomadWgsVcf` | `${reference_base}/…/grch37/gnomad/gnomad.genomes.r2.1.1.sites.minimal.vcf.gz` |
| `exomePoN` / `wgsPoN` | `${reference_base}/…/grch37/annotation/{wes,wgs}.pon.vcf.gz` |
| `hlaFasta` / `hlaDat` | `${reference_base}/…/grch37/hla/{abc_complete.fasta,hla.dat}` |
| `neoantigenCDNA` / `neoantigenCDS` | `${reference_base}/…/grch37/neoantigen/Homo_sapiens.GRCh37.75.{cdna,cds}.all.fa.gz` |
| `snpGcCorrections` | `${reference_base}/…/grch37/ascat/SnpGcCorrections.tsv` |
| `acLoci` | `Annotation/ASCAT/1000G_phase3_20130502_SNP_maf0.3.loci` |
| `svBlacklistBed` | `${reference_base}/…/grch37/sv_calling/pcawg6_blacklist.slop.bed.gz` |
| `repeatMasker` / `mapabilityBlacklist` | `${reference_base}/…/grch37/annotation/{rmsk_mod.bed.gz, wgEncodeDacMapabilityConsensusExcludable.bed.gz}` |
| `dellyExcludeRegions` | `${reference_base}/…/grch37/delly/human.hg19.excl.tsv` |
| `isoforms` / `spliceSites` | `${reference_base}/…/grch37/{annotation/isoforms, splice_sites/splice_sites.bed}` |

**Per-kit target files** resolve under `${targets_base}/${targets_id}/` (where `targets_id` ∈ `agilent`, `idt`, `wgs` from the mapping `TARGET` column): `baits.interval_list`, `targets.interval_list`, `targets.bed`, `targets.bed.gz(.tbi)`, `coding.bed`. To **build these six files for a new bait set** (padding, `bedtools`/`gatk BedToIntervalList`, the `coding.bed` used for TMB) and lay out the `targets_base` folder, follow the official [Custom target files](https://github.com/mskcc/tempo/blob/develop/docs/reference-files.md#custom-target-files) section — it is the authoritative recipe and is not reproduced here.

## Two ways to point Tempo at your data

1. **Mirror the tree** under your base and override just the base:
   ```bash
   --reference_base /data/tempo_refs
   ```
2. **Override individual files** by param name when your layout differs:
   ```bash
   --genomeFile /refs/hg19.fa --dbsnp /refs/dbsnp_138.b37.vcf --facetsVcf /refs/facets_snps.vcf
   ```
   Any `params.*` is overridable as `--<name>`. **Nested params** (e.g. `params.facets.cval`) cannot be set with `--`; put them in a `-c custom.config`.

## GRCh38 caveat (verified)

The GRCh38 block in `conf/references.config` is only partially defined — `vepCache`, `facetsVcf`, `gnomadWesVcf/gnomadWgsVcf`, `exomePoN/wgsPoN`, `msiSensorList`, and SV blacklists are **not set**. The README states GRCh38 is **not fully supported**. Run GRCh37/b37 unless you are prepared to supply and validate the missing GRCh38 references yourself.

## What is NOT in the repo

The exact contents/versions of each reference file (how the FACETS dbSNP VCF was curated, the PoN construction) are not in the pipeline repo — they are pre-built artifacts hosted on Juno. To reproduce them off-Juno you must obtain or rebuild each; the repo gives the expected filenames and layout (above) but not build recipes.

## See Also

- [config-references.md](config-references.md) — the same references in narrative form
- [running-non-juno.md](running-non-juno.md) — executor config for your cluster
- [installation.md](installation.md) — prerequisites
- [../troubleshooting/missing-references.md](../troubleshooting/missing-references.md) — diagnosing a missing reference at runtime
