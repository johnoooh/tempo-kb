# Tempo Pipeline Configuration Profiles and How to Select Them

> **Quick answer:** Tempo provides six Nextflow profiles -- `juno`, `awsbatch`, `docker`, `singularity`, `test`, and `test_singularity` -- each bundling an executor, container engine, resource limits, and reference paths. Select one with `-profile juno` on the command line.

## Available Profiles

Profiles are defined in the `profiles` block of `nextflow.config`. Each profile includes a specific combination of configuration files that set the executor, container runtime, resource allocations, and reference file paths.

### juno

The primary production profile for running Tempo on the MSKCC Juno HPC cluster. It composes:

- `conf/singularity.config` -- enables Singularity with auto-mounts and loads the `singularity/3.1.1` module.
- `conf/juno.config` -- sets the LSF executor (`executor.name = "lsf"`), queue size of 5 billion, per-job memory limits, scratch directories, and a `beforeScript` that loads `singularity/3.1.1` and traps termination signals.
- `conf/containers.config` -- maps each process to its container image.
- `conf/references.config` -- genome references rooted at `params.reference_base = "/juno/work/tempo/cmopipeline"`.
- Assay-type-specific configs: `conf/exome.config` + `conf/resources_juno.config` when `params.assayType == "exome"`, or `conf/genome.config` + `conf/resources_juno_genome.config` when `params.assayType == "genome"`.

Key Juno settings include `maxRetries = 3`, an error strategy that retries up to 3 times then ignores, `max_memory = "128.GB"`, and wall time codes `wallTimeExitCode = '140,0,1,143'`.

### awsbatch

Runs Tempo on AWS Batch using Docker containers. It composes `conf/docker.config`, `conf/containers.config`, `conf/awsbatch.config`, `conf/resources_aws.config`, and `conf/references.config`, plus assay-type-specific resource overrides.

<!-- TODO: VERIFY WITH USER -- conf/awsbatch.config is referenced in nextflow.config but the file was not found in the repository. AWS Batch details may be incomplete. -->

### docker

A lightweight local profile that enables Docker (`docker.enabled = true`, `docker.fixOwnership = true`), loads container definitions, default resources from `conf/resources.config`, and reference paths from `conf/references.config`.

### singularity

Similar to `docker` but uses Singularity (`singularity.enabled = true`, `singularity.autoMounts = true`, `singularity.runOptions = "-B $TMPDIR"`). Also loads `singularity/3.1.1` via a `beforeScript` in the process scope. Pairs with default resources and reference paths.

### test

A testing profile using Docker. Includes `conf/test.config` for test-specific parameters, Docker containers, default resources, and references. Sets `params.scatterCount = 3` for faster test runs.

### test_singularity

The same testing profile but uses Singularity and the LSF executor (`executor.name = "lsf"`). Sets `params.scatterCount = 3`.

## How Profiles Compose Configurations

Each profile uses `includeConfig` directives to layer config files. Files loaded later override earlier settings. For example, in the `juno` profile, `conf/juno.config` overrides the default process memory of `conf/singularity.config` because it is included afterward. Assay-type branching happens via `if(params.assayType == "exome")` blocks inside the profile definition.

## How to Select a Profile

```bash
# Production on Juno
nextflow run tempo -profile juno --mapping mapping.tsv

# Local Docker run
nextflow run tempo -profile docker --mapping mapping.tsv

# Test run
nextflow run tempo -profile test
```

You can combine profiles with a comma, though Tempo profiles are typically used individually because each is self-contained.

## Global Settings Applied to All Profiles

Regardless of profile, `nextflow.config` enables trace (`trace.txt`), timeline (`timeline.html`), and report (`report.html`) output files. The DAG output is disabled by default. The environment variable `PYTHONNOUSERSITE = 1` is set globally.

## File Locations

- Profile definitions: `nextflow.config` (lines 53-132)
- Juno executor config: `conf/juno.config`
- Docker config: `conf/docker.config`
- Singularity config: `conf/singularity.config`

## Example

```bash
# Run exome analysis on Juno with LSF
nextflow run tempo -profile juno \
  --assayType exome \
  --mapping mapping.tsv \
  --pairing pairing.tsv
```

## See Also

- [config-resources.md](config-resources.md) -- CPU, memory, and time allocations per process
- [config-references.md](config-references.md) -- Reference genome paths and known-sites VCFs
- [config-containers.md](config-containers.md) -- Container images and versions
