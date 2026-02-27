# Tempo Invocation: Command-Line Usage, Workflow Flags, Profiles, and Execution Modes

> **Quick answer:** Run Tempo with `nextflow run dsl2.nf --mapping <tsv> --pairing <tsv> -profile juno --workflows="snv,qc"`. Use `--workflows` to select sub-workflows (snv, sv, mutsig, germSNV, germSV, lohhla, facets, qc, msisensor), and Tempo will automatically enable any dependencies. Resume interrupted runs with `-resume`.

## Basic Command

```shell
nextflow run dsl2.nf \
    --mapping <mapping.tsv> \
    --pairing <pairing.tsv> \
    -profile juno \
    --workflows="snv,qc" \
    --aggregate true
```

Replace `--mapping` with `--bamMapping` when starting from pre-aligned BAM files.

## The `--workflows` Flag

The `--workflows` flag accepts a comma-separated list (in quotes) of sub-workflows to execute. All values are case-insensitive.

| Workflow | Description |
|---|---|
| `snv` | Somatic SNV/indel calling (MuTect2 + Strelka2), annotation, filtering, neoantigen prediction |
| `sv` | Somatic structural variant calling (Delly, Manta, SvABA; plus BRASS for WGS) |
| `mutsig` | Mutational signature inference (tempoSig) |
| `germSNV` | Germline SNV/indel calling (HaplotypeCaller + Strelka2) |
| `germSV` | Germline structural variant calling (Delly, Manta, SvABA) |
| `lohhla` | Loss of heterozygosity at HLA loci (LOHHLA) |
| `facets` | Copy number analysis (FACETS) |
| `qc` | Quality control metrics (Conpair concordance/contamination, MultiQC) |
| `msisensor` | Microsatellite instability detection (MSIsensor) |

Example enabling multiple workflows:

```shell
--workflows="snv,sv,qc,facets,msisensor"
```

## Auto-Dependency Resolution

When a sub-workflow depends on another's output, Tempo automatically enables the dependency. Key examples:

- `lohhla` auto-enables `facets`.
- `snv` auto-enables `facets`, `manta`, and `loh` (POLYSOLVER).
- `mutsig` auto-enables `snv`, `facets`, `manta`, and `loh`.
- `germSNV` auto-enables `facets` and scatter intervals.

You do not need to manually specify dependencies.

## Profiles

The `-profile` flag (single dash, Nextflow syntax) loads environment-specific configuration:

| Profile | Environment |
|---|---|
| `juno` | MSKCC Juno HPC cluster (LSF scheduler) |
| `awsbatch` | AWS Batch cloud execution |
| `docker` | Local Docker execution |
| `singularity` | Local Singularity execution |
| `test` | Test profile with minimal test data |
| `test_singularity` | Test profile for Singularity on Juno |

## Execution Modes

Tempo supports three primary execution modes depending on which input files are provided:

### 1. Mapping-Only (Alignment)

Provide `--mapping` without `--pairing` and no `--workflows`. Only alignment steps (BWA, merging, MarkDuplicates, BQSR) will run.

```shell
nextflow run dsl2.nf --mapping mapping.tsv -profile juno
```

### 2. Mapping + Pairing (Full Analysis)

Provide both `--mapping` (or `--bamMapping`) and `--pairing`, plus `--workflows` to select analyses. This is the standard execution mode.

```shell
nextflow run dsl2.nf \
    --mapping mapping.tsv \
    --pairing pairing.tsv \
    -profile juno \
    --workflows="snv,sv,qc,facets"
```

### 3. Aggregate-Only

Provide `--aggregate <tsv>` with no mapping or pairing files. The aggregate TSV must include a `PATH` column pointing to existing Tempo output directories. Tempo aggregates per-sample results into cohort-level summary files.

```shell
nextflow run dsl2.nf --aggregate aggregate.tsv -profile juno
```

In this mode, all sub-workflows are implicitly enabled for aggregation purposes.

## Optional Flags

| Flag | Description | Default |
|---|---|---|
| `--outDir <path>` | Output directory (created if it does not exist) | Current working directory |
| `--genome <version>` | Reference genome version | `GRCh37` |
| `--assayType <type>` | `exome` or `genome` | `exome` |
| `--cosmic <version>` | COSMIC signature version: `v2` (30 signatures) or `v3` (60 signatures) | `v2` |
| `--splitLanes` | Scan FASTQs for lane identifiers and demultiplex | `true` |
| `-publishAll` | Retain intermediate output files | `true` |
| `-resume` | Resume from Nextflow cache after interruption | -- |
| `-work-dir <path>` | Directory for intermediate cached files | Current working directory |

Note the distinction: flags with double dash (`--`) are Tempo parameters; flags with single dash (`-`) are Nextflow parameters.

## Juno bsub Submission

On the MSKCC Juno cluster, submit the Nextflow leader process via `bsub`:

```shell
bsub -W 80:00 -n 2 -R "rusage[mem=8]" \
    -o nf_output.out \
    -e nf_output.err \
    nextflow run /path/to/dsl2.nf \
    --mapping test_inputs/local/WES_25TN.tsv \
    --pairing test_inputs/local/WES_25TN_pairing.tsv \
    --outDir results \
    -profile juno \
    --workflows="snv,qc,lohhla" \
    --aggregate true
```

Set `-W` generously; large batches and WGS take longer. The leader process needs minimal resources (`-n 2`, 8 GB). If the leader job dies, resume with `-resume`.

## Resuming Interrupted Runs

Add `-resume` to continue from cached results. To resume a specific prior run, use `nextflow log` to find the run name, then pass it:

```shell
nextflow run dsl2.nf -resume mighty_boyd --mapping mapping.tsv --pairing pairing.tsv -profile juno --workflows="snv,qc"
```

## File Locations

- Pipeline entrypoint: `dsl2.nf`
- Workflow flag logic: `dsl2.nf` (lines 56-112)
- Sub-workflow definitions: `modules/subworkflow/`
- Profile configurations: `conf/`, `nextflow.config`

## See Also

- [Pipeline Overview](overview.md) -- What Tempo does and its analysis stages
- [Input File Formats](running-inputs.md) -- Mapping, pairing, and aggregate TSV specifications
