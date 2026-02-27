# Tempo Pipeline CPU, Memory, and Time Resource Allocations

> **Quick answer:** Default resource allocations are defined in `conf/resources.config` (typically 2 CPUs and 6 GB per process). The Juno profile overrides these in `conf/resources_juno.config` with dynamic, attempt-scaled memory and higher CPU counts for alignment and variant calling. You can override any process allocation using `withName` in a custom Nextflow config.

## Default Resource Allocations (resources.config)

The file `conf/resources.config` sets baseline resources for all processes. Most processes receive 2 CPUs and 6 GB of memory. Lighter processes such as `CrossValidateSamples`, `SplitLanesR1`, and `SplitLanesR2` receive 1 CPU and 1 GB. Aggregation processes (`SomaticAggregateMaf`, `GermlineAggregateMaf`, etc.) also use 1 CPU and 1 GB. The `multiqc_process` label applies 1 CPU and 3 GB scaled by attempt.

Notable default allocations:

| Process | CPUs | Memory |
|---------|------|--------|
| AlignReads | 2 | 6 GB |
| RunMutect2 | 2 | 6 GB |
| DoFacets | 2 | 6 GB |
| QcQualimap | 2 | 6 GB (scales on retry) |
| QcPileup | 1 | 2 GB (scales on retry) |

These defaults are intentionally conservative and are meant to be overridden by environment-specific configs.

## Juno Resource Overrides (resources_juno.config)

On Juno, `conf/resources_juno.config` replaces the defaults with production-grade allocations that scale with retry attempts. Key differences:

| Process | Default CPUs/Mem | Juno CPUs/Mem |
|---------|-----------------|---------------|
| AlignReads | 2 / 6 GB | 8 / 12 GB |
| RunBQSR | 2 / 6 GB | 4 / 6 GB + attempt |
| SomaticRunManta | 2 / 6 GB | 8 / 1 GB * attempt |
| SomaticRunStrelka2 | 2 / 6 GB | 8 / 1 GB * attempt |
| RunPolysolver | 2 / 6 GB | 8 / 1 GB * attempt |
| DoFacets | 2 / 6 GB | 1 / 20 GB * attempt |
| SomaticAnnotateMaf | 2 / 6 GB | 1 / 8 GB * attempt |
| RunNeoantigen | 2 / 6 GB | 4*attempt / 4 GB * attempt * 2 |
| RunSvABA | -- | 8 / 2 GB * attempt |

The Juno config uses dynamic memory expressions like `{ 20.GB * task.attempt }` so that retried tasks automatically request more resources. The maximum memory available on Juno is `128 GB` (set via `params.max_memory` in `conf/juno.config`).

## AWS Resource Overrides (resources_aws.config)

AWS allocations in `conf/resources_aws.config` use similar attempt-scaling patterns but with different baselines. Notable differences include `AlignReads` at 8 CPUs / 96 GB and `RunBQSR` at 8 CPUs / 24 GB + attempt scaling, reflecting the larger instance types available on AWS.

## Wall Time Settings

The Juno profile defines wall time in `conf/juno.config`:

- **Default process time**: `{ task.attempt < 3 ? 3.h * task.attempt : 500.h }` -- starts at 3 hours, doubles on retry, then jumps to 500 hours on the third attempt.
- `params.minWallTime = 3.h`
- `params.medWallTime = 6.h`
- `params.maxWallTime = 500.h`
- `params.wallTimeExitCode = '140,0,1,143'` -- exit codes that trigger wall-time-based retry behavior.

The `RunSvABA` process has a custom time override: `{ task.attempt < 3 ? 30.h * task.attempt : 500.h }`.

## Retry and Error Strategy

On Juno, `maxRetries = 3` and the error strategy is `{ task.attempt <= process.maxRetries ? 'retry' : 'ignore' }`. This means a process retries up to 3 times with increasing resources before being silently ignored.

## How to Override Resources in a Custom Config

Create a custom Nextflow config file and pass it with `-c`:

```nextflow
// custom.config
process {
  withName:DoFacets {
    cpus = 4
    memory = '64 GB'
    time = '48h'
  }
  withName:AlignReads {
    cpus = 16
    memory = '32 GB'
  }
}
```

```bash
nextflow run tempo -profile juno -c custom.config --mapping mapping.tsv
```

Settings in `custom.config` override profile defaults because `-c` configs are applied last.

## File Locations

- Default resources: `conf/resources.config`
- Juno overrides: `conf/resources_juno.config`
- AWS overrides: `conf/resources_aws.config`
- Juno executor and wall time: `conf/juno.config`

## Example

```nextflow
// Check current allocation for a process
// In conf/resources_juno.config:
withName:DoFacets {
  cpus = { 1 }
  memory = { 20.GB * task.attempt }
}
```

## See Also

- [config-profiles.md](config-profiles.md) -- How profiles compose resource configs
- [config-containers.md](config-containers.md) -- Which container runs each process
