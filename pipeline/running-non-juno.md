# Running Tempo on Non-Juno Infrastructure (SLURM/PBS/local)

> **Quick answer:** Tempo ships executor configs only for **LSF** (the `juno` profile) and **AWS Batch** (the `awsbatch` profile). To run on SLURM, PBS/SGE, or a single machine, you write a small custom Nextflow config that swaps the executor and pass it with `-c`. This page gives the exact Juno executor block to adapt and what to change. Scheduler-specific values below are **templates you must verify for your site** — Tempo does not ship or test them.

> 📖 **This page is unofficial / net-new.** The official Tempo docs cover only the **supported** environments — Juno ([juno-setup.md](https://github.com/mskcc/tempo/blob/develop/docs/juno-setup.md)) and AWS ([aws-setup.md](https://github.com/mskcc/tempo/blob/develop/docs/aws-setup.md)) — and [Nextflow basics](https://github.com/mskcc/tempo/blob/develop/docs/nextflow-basics.md). There is **no official SLURM/PBS/local guidance**; everything below is adapted from the Juno config using standard Nextflow conventions and is not validated by the Tempo team. If you can use Juno or AWS, follow the official setup docs instead.

## What the `juno` profile sets (verified — `conf/juno.config`)

```groovy
executor {
  name = "lsf"
  queueSize = 5000000000
  perJobMemLimit = true
}

process {
  memory = "8.GB"
  time = { task.attempt < 3 ? 3.h * task.attempt : 500.h }
  clusterOptions = ""
  scratch = true
  beforeScript = "module load singularity/3.1.1; unset R_LIBS; ..."   // signal-trap omitted
  maxRetries = 3
  errorStrategy = { task.attempt <= process.maxRetries ? 'retry' : 'ignore' }
}

params {
  max_memory = "128.GB"
  mem_per_core = true
  reference_base = "/juno/work/tempo/cmopipeline"
  // ... reference paths (see custom-references.md)
}
```

The pieces that are **LSF/Juno-specific** and must change off-Juno: `executor.name`, `perJobMemLimit`, `mem_per_core`, the `module load singularity` in `beforeScript`, and all `reference_base` paths.

## Adapting to SLURM (template — verify for your cluster)

Create `mysite.config`:

```groovy
// ⚠️ TEMPLATE — values below are not from the Tempo repo; confirm against your scheduler.
executor {
  name = "slurm"      // was "lsf"
  queueSize = 200     // max concurrent jobs your site allows
  // perJobMemLimit is LSF-specific — omit for SLURM (SLURM enforces --mem natively)
}

process {
  scratch = true
  queue = "<your_partition>"          // SLURM partition / PBS queue
  clusterOptions = "--account=<acct>" // any extra scheduler flags
  maxRetries = 3
  errorStrategy = { task.attempt <= process.maxRetries ? 'retry' : 'ignore' }
  // If Singularity isn't already on PATH on compute nodes:
  // beforeScript = "module load singularity"
}

params {
  mem_per_core = false   // set true only if your scheduler bills memory per core (LSF-style)
}
```

Run with a base profile for the container engine plus your config:

```bash
nextflow run dsl2.nf --mapping ... --pairing ... --assayType exome --workflows snv,qc \
  -profile singularity \
  -c mysite.config \
  --reference_base /path/to/your/refs   # see custom-references.md
```

Using `-profile singularity` (not `juno`) gives you container settings and resource defaults **without** the LSF executor and Juno paths; your `-c mysite.config` then layers the SLURM executor on top.

- **PBS/SGE:** same pattern, `executor.name = "pbs"` (or `"sge"`), set `queue`/`clusterOptions` to your scheduler's syntax.
- **Single machine (no scheduler):** use `-profile docker` or `-profile singularity` alone — the default executor is `local`. Constrain with `executor { cpus = N; memory = 'X GB' }`.

For per-process memory/time overrides (e.g., one OOM-killing step), see [config-resources.md](config-resources.md).

## What is NOT in the repo

Tempo provides no SLURM/PBS/SGE profile, no validated executor block for them, and no site-specific queue names. The SLURM template above follows standard Nextflow executor conventions but **was not taken from the Tempo repo** — treat it as a starting point and validate against your cluster's Nextflow documentation.

## See Also

- [installation.md](installation.md) — prerequisites and first run
- [custom-references.md](custom-references.md) — supplying reference data off-Juno
- [config-resources.md](config-resources.md) — per-process resource overrides
- [config-profiles.md](config-profiles.md) — the built-in profiles and what each includes
