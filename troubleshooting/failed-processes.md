# Diagnosing Failed Processes in Tempo Pipeline Runs

> **Quick answer:** When a Tempo process fails, find the process hash from the Nextflow output or `trace.txt`, navigate to `work/{hash_prefix}/{hash_suffix}/`, and inspect `.command.err` and `.command.log` to determine the root cause.

## Identifying Which Process Failed

Nextflow reports failures directly in the terminal output. A typical failure message looks like:

```
Error executing process > 'MutectSomaticVariantCalling (SampleA_TN)'
  Command exit status: 1
  Command output: (empty)
  Work dir: /path/to/work/ab/cd1234567890...
```

The work directory hash is the key to investigating the failure. You can also find failed processes from the pipeline reports:

- **trace.txt** -- a tab-separated file listing every process with its status, exit code, duration, and resource usage. Filter for rows where `status` is `FAILED`.
- **report.html** -- an interactive HTML report with a summary table. Failed processes are highlighted in red.
- **.nextflow.log** -- the full Nextflow log with stack traces and error context.

## Navigating the Work Directory

Each process execution lives in a unique work directory with the structure `work/{2-char prefix}/{remaining hash}/`. Inside that directory you will find several key files:

| File | Contents |
|------|----------|
| `.command.sh` | The exact shell script that Nextflow executed |
| `.command.run` | The LSF submission wrapper (bsub parameters) |
| `.command.out` | Standard output from the process |
| `.command.err` | Standard error from the process |
| `.command.log` | LSF job log (scheduler-level messages) |
| `.exitcode` | The numeric exit code |

Start your diagnosis with `.command.err` for application-level errors, then check `.command.log` for scheduler-level issues like out-of-memory kills or wall time exceeded.

## Common Failure Patterns

### Out of Memory (Exit Code 130)

The LSF scheduler killed the job because it exceeded its memory allocation. The `.command.log` file typically contains:

```
TERM_MEMLIMIT: job killed after reaching LSF memory usage limit.
```

See [Fixing Out-of-Memory and Wall Time Errors](resource-limits.md) for resolution steps.

### Wall Time Exceeded (Exit Code 140)

The job ran longer than its time allocation. The `.command.log` will show:

```
TERM_RUNLIMIT: job killed after reaching LSF run time limit.
```

### Missing Input Files

The `.command.err` file will reference a file that does not exist. This usually means an upstream process failed silently or the input mapping file has incorrect paths. Verify that input BAMs and reference files exist at the paths listed in `.command.sh`.

### Container Errors

Look for messages like `FATAL: could not open image` or `Failed to pull singularity image` in `.command.err`. See [Troubleshooting Container Issues](container-issues.md) for solutions.

### Application-Level Errors (Exit Code 1)

Exit code 1 indicates the tool itself returned an error. Read `.command.err` carefully for the specific error message. Common causes include malformed input files, incompatible file formats, or incorrect parameter values.

## Manual Resubmission for Debugging

You can manually resubmit a failed job to test fixes or gather more debugging information. Navigate to the work directory and submit the LSF wrapper directly:

```bash
cd /path/to/work/ab/cd1234567890
bsub < .command.run
```

You can also run the command interactively (without LSF) for faster debugging:

```bash
cd /path/to/work/ab/cd1234567890
bash .command.sh
```

This is useful for testing whether a container issue or environment problem was the cause.

## File Locations

| File | Location |
|------|----------|
| Process work directories | `work/{hash_prefix}/{hash_suffix}/` |
| Execution trace | `trace.txt` in the run directory |
| HTML report | `report.html` in the run directory |
| Timeline | `timeline.html` in the run directory |
| Nextflow log | `.nextflow.log` in the run directory |

## Example

A Tempo run fails with an unclear error. Here is how to investigate:

```bash
# Find the failed task hash from trace.txt
grep FAILED trace.txt

# Navigate to the work directory (use the hash from output)
cd work/ab/cd1234567890abcdef

# Check what command was run
cat .command.sh

# Check stderr for application errors
cat .command.err

# Check LSF log for scheduler-level kills
cat .command.log

# Check the exit code
cat .exitcode
```

## See Also

- [Resuming Interrupted Runs](nextflow-resume.md) -- how to resume after fixing a failed process
- [Fixing Out-of-Memory and Wall Time Errors](resource-limits.md) -- resolving resource-related failures
- [Troubleshooting Container Issues](container-issues.md) -- fixing Singularity image problems
