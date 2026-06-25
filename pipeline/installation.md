# Installing & First-Run Setup (Running Tempo Yourself)

> **Quick answer:** Tempo is a Nextflow DSL2 pipeline you clone and run with `nextflow run dsl2.nf`. You need Nextflow `>=25.04.7`, Java 11+, and a container engine (Docker or Singularity). On a non-Juno system you also supply your own reference data ([custom-references.md](custom-references.md)) and, for a non-LSF scheduler, your own executor config ([running-non-juno.md](running-non-juno.md)).

> 📖 **Authoritative source:** the official Tempo docs (in [`mskcc/tempo/docs`](https://github.com/mskcc/tempo/tree/develop/docs)) are the source of truth for running the pipeline — [installation.md](https://github.com/mskcc/tempo/blob/develop/docs/installation.md) · [juno-setup.md](https://github.com/mskcc/tempo/blob/develop/docs/juno-setup.md) · [aws-setup.md](https://github.com/mskcc/tempo/blob/develop/docs/aws-setup.md). This KB page summarizes them and adds cross-links; if it ever disagrees with the official docs, trust the official docs (except where noted below).
>
> ⚠️ **Version note (verified):** the official `installation.md` currently states "Nextflow 19.10.0 or later, Java 8" — but the repo's `nextflow.config` manifest requires **`nextflowVersion >= 25.04.7`** and `juno-setup.md` specifies **Java 11+**. The manifest is what Nextflow actually enforces, so use the values in the table below. The official install page appears out of date on this point.

## Prerequisites (verified against `nextflow.config` / `docs/`)

| Requirement | Value | Source |
|-------------|-------|--------|
| Nextflow | `>=25.04.7` | `nextflow.config` manifest `nextflowVersion` |
| Java | 11+ (Nextflow's requirement) | `docs/juno-setup.md` (`module load java/jdk-11.0.11` on Juno) |
| Container engine | Docker **or** Singularity | profiles `docker`, `singularity`, `juno` |
| (Singularity) image cache | `NXF_SINGULARITY_CACHEDIR` env var | `docs/juno-setup.md` |

## Install

```bash
# 1. Install Nextflow (https://www.nextflow.io) — needs Java 11+
curl -s https://get.nextflow.io | bash      # produces a `nextflow` launcher
# 2. Clone Tempo
git clone https://github.com/mskcc/tempo.git
cd tempo
```

There is **no** `nextflow run mskcc/tempo` remote-run shortcut documented — the repo must be cloned locally and you run `dsl2.nf` from inside it.

## First run (test profile)

The `test` / `test_singularity` profiles ship small inputs and set `scatterCount = 3`, so you can confirm the install before wiring up references:

```bash
nextflow run dsl2.nf -profile test            # Docker
nextflow run dsl2.nf -profile test_singularity # Singularity
```

## Minimal real run

```bash
nextflow run dsl2.nf \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  --assayType exome \
  --workflows snv,qc \
  --aggregate \
  -profile <docker|singularity|juno|awsbatch>
```

See [running-inputs.md](running-inputs.md) for the mapping/pairing files, [running-invocation.md](running-invocation.md) for flags and `--workflows`, and [config-profiles.md](config-profiles.md) for profiles.

## Container images & Singularity cache

Per-process images live in `conf/containers.config` (`withName:` selectors, images under the `cmopipeline/` Docker Hub org). With Singularity, Nextflow pulls `docker://…` automatically; point its cache at a writable directory so images are reused:

```bash
export NXF_SINGULARITY_CACHEDIR=/path/to/singularity_images
```

To override a single image (your own build/registry), see [config-containers.md](config-containers.md).

## Running outside Juno

The `juno` profile hardcodes MSKCC paths and an LSF executor. To run elsewhere you need two things, each documented separately:

1. **A scheduler/executor config** for your cluster (SLURM/PBS/local) — [running-non-juno.md](running-non-juno.md).
2. **Your own reference data** under a `reference_base` you control — [custom-references.md](custom-references.md).

> **Note:** GRCh38 is **not fully supported** in the reference config (several GRCh38 reference keys are undefined). GRCh37/b37 is the validated path. See [custom-references.md](custom-references.md).

## See Also

- [running-inputs.md](running-inputs.md) — mapping & pairing files
- [running-invocation.md](running-invocation.md) — flags, `--workflows`, `--aggregate`
- [running-non-juno.md](running-non-juno.md) — non-LSF schedulers
- [custom-references.md](custom-references.md) — supplying your own reference data
- [config-profiles.md](config-profiles.md) — the available profiles
