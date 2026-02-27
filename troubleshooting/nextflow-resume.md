# Resuming Interrupted Tempo Runs with Nextflow -resume

> **Quick answer:** Add `-resume` to your `nextflow run` command to skip already-completed processes and pick up where the pipeline left off. You must run from the same directory that contains the original `work/` folder.

## How Resume Works

Nextflow caches the result of every completed process in the `work/` directory. Each task gets a unique hash based on its inputs, code, and container image. When you add `-resume`, Nextflow recalculates these hashes and matches them against previously cached results. If a match is found, the task is skipped and the cached output is reused. Tasks that never completed or whose hashes changed will be re-executed.

A typical resumed Tempo command looks like this:

```bash
nextflow run dsl2.nf \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  -profile juno \
  --outDir results \
  -resume
```

## When Resume Will Not Work

The `-resume` flag will fail to use cached results if any of the following changed since the original run:

- **Input files** were modified, renamed, or moved to a different path.
- **Pipeline parameters** changed, such as switching `--outDir` or `--genome`.
- **Process code** was edited in the Nextflow script or module files.
- **Container image** was updated to a new version.
- **The work/ directory** was deleted or you launched from a different directory.

If inputs changed for only some samples, Nextflow will correctly re-run only the affected tasks while caching the rest.

## Checking Run History

Use `nextflow log` to see all previous runs in the current directory:

```bash
nextflow log
```

This outputs a table with timestamps, run names, session IDs, and status. Each run gets a human-readable name like `crazy_wilson` and a session ID (a UUID). You can use either to target a specific run for resumption.

## Resuming a Specific Session

If you have run the pipeline multiple times, Nextflow may try to resume from the most recent session by default. To resume from a specific earlier session, pass the session ID explicitly:

```bash
nextflow run dsl2.nf \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  -profile juno \
  --outDir results \
  -resume 1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d
```

You can find the session ID from the `nextflow log` output or from the `.nextflow.log` file.

## Cache Invalidation Explained

Nextflow computes a hash for each task based on several factors. Understanding what invalidates the cache helps avoid unnecessary reruns:

| Factor | Effect on Cache |
|--------|----------------|
| Input file content changed | Invalidated |
| Input file path changed | Invalidated |
| Process script changed | Invalidated |
| Container image tag changed | Invalidated |
| Resource settings (memory/CPU) changed | **Not invalidated** |
| Output directory changed | Depends on process |

Changing resource allocations (memory, CPUs, time limits) does not invalidate the cache, which is useful when retrying failed processes with higher resource limits.

## Cleaning the Work Directory Safely

The `work/` directory grows with each run and can consume substantial disk space. To clean old cached files while preserving the most recent run:

```bash
nextflow clean -but last
```

To clean a specific run by name:

```bash
nextflow clean crazy_wilson
```

To do a dry run first and see what would be deleted:

```bash
nextflow clean -n -but last
```

Never delete the `work/` directory manually if you plan to resume. If you do delete it, the entire pipeline must be re-run from scratch.

## File Locations

| File | Location |
|------|----------|
| Task cache | `work/{hash_prefix}/{hash_suffix}/` |
| Run history | `.nextflow/history` |
| Detailed log | `.nextflow.log` |
| Execution trace | `trace.txt` |

## Example

A Tempo run fails at the somatic variant calling step after completing alignment and QC for 20 samples. To resume without redoing alignment:

```bash
# Check the previous run
nextflow log

# Resume from where it stopped
nextflow run dsl2.nf \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  -profile juno \
  --outDir results \
  --aggregate \
  -resume
```

Nextflow will print `[skipped]` next to cached processes and `[submitted]` next to tasks being re-executed.

## See Also

- [Diagnosing Failed Processes](failed-processes.md) -- how to inspect why a process failed before resuming
- [Fixing Out-of-Memory and Wall Time Errors](resource-limits.md) -- adjusting resources before a resume
- [Juno HPC Cluster-Specific Issues](juno-specific.md) -- environment setup required before running Nextflow
