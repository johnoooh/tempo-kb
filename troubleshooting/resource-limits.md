# Fixing Out-of-Memory and Wall Time Errors in Tempo

> **Quick answer:** Tempo auto-retries failed processes up to 3 times with increased memory. If retries are exhausted, create a custom config file that overrides resource allocations for the specific process using `withName`, and pass it with `-c custom.config`.

## Understanding Resource Errors

Resource errors occur when a process exceeds its allocated memory or time on the Juno LSF cluster. These are the two most common resource failures in Tempo:

### Out of Memory (Exit Code 130)

LSF terminates jobs that exceed their memory limit. You will see this in `.command.log`:

```
TERM_MEMLIMIT: job killed after reaching LSF memory usage limit.
```

Memory-intensive processes in Tempo include FACETS, Mutect2 on large tumors, and VEP annotation. The default maximum memory ceiling is 128 GB.

### Wall Time Exceeded (Exit Code 140)

LSF terminates jobs that exceed their time allocation. The `.command.log` will contain:

```
TERM_RUNLIMIT: job killed after reaching LSF run time limit.
```

Wall time overruns commonly affect alignment (BWA-MEM) on large WGS samples or variant calling on high-depth regions.

## Automatic Retry Behavior

Tempo is configured with automatic retries in `conf/resources_juno.config`. Key settings include:

- **maxRetries = 3** -- each process can be retried up to 3 times.
- **Dynamic memory scaling** -- memory increases with each retry attempt using the `task.attempt` multiplier. For example, a process starting at 8 GB may scale to 16 GB on attempt 2 and 24 GB on attempt 3.
- **wallTimeExitCode** -- exit codes `140, 0, 1, 143` trigger wall time handling on retry.
- **Error strategy** -- after exhausting retries, the process is set to `ignore` so remaining pipeline steps can continue.

This means many transient resource failures resolve themselves without intervention. Check whether the pipeline already retried by looking at `trace.txt` -- you will see multiple rows for the same process with increasing attempt numbers.

## Manually Overriding Resource Allocations

If automatic retries are not sufficient, create a custom configuration file to increase resources for the failing process.

### Step 1: Identify the Process Name

Find the exact process name from the Nextflow output or `trace.txt`. Examples of Tempo process names include `DoFacets`, `SomaticAnnotateMaf`, `RunBwaMem`, `MutectSomaticVariantCalling`, and `RunVEP`.

### Step 2: Create a Custom Config

Create a file called `custom.config` with resource overrides:

```groovy
process {
    withName: 'DoFacets' {
        memory = '64.GB'
        time = '24.h'
    }
    withName: 'MutectSomaticVariantCalling' {
        memory = '32.GB'
        time = '48.h'
    }
}
```

### Step 3: Run with the Custom Config

Pass the custom config with the `-c` flag along with `-resume` to avoid redoing completed work:

```bash
nextflow run dsl2.nf \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  -profile juno \
  --outDir results \
  -c custom.config \
  -resume
```

The `-c` flag merges your custom config with the pipeline defaults, and your `withName` overrides take precedence for the specified processes.

## Default Resource Allocations

The default allocations in `conf/resources_juno.config` are tuned for typical WES samples. WGS samples or unusually high-depth WES samples may require increases. The upper ceiling for any single process is:

| Resource | Maximum |
|----------|---------|
| Memory | 128 GB |
| CPUs | Defined per process |
| Time | Defined per process |

If a process needs more than 128 GB, you must also override `max_memory` in your custom config.

## File Locations

| File | Location |
|------|----------|
| Default resource config | `conf/resources_juno.config` |
| LSF job log | `work/{hash}/.command.log` |
| Execution trace | `trace.txt` |
| Custom overrides | Your `custom.config` (user-created) |

## Example

The `DoFacets` process fails after 3 retries with exit code 130. To fix it:

```bash
# Check current allocation in trace.txt
grep DoFacets trace.txt

# Create override config
cat > custom.config << 'EOF'
process {
    withName: 'DoFacets' {
        memory = '64.GB'
    }
}
EOF

# Resume with increased memory
nextflow run dsl2.nf \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  -profile juno \
  --outDir results \
  -c custom.config \
  -resume
```

## See Also

- [Diagnosing Failed Processes](failed-processes.md) -- inspecting work directories to identify the root cause
- [Resuming Interrupted Runs](nextflow-resume.md) -- using resume to skip completed processes
- [Juno HPC Cluster-Specific Issues](juno-specific.md) -- LSF queue and resource context
