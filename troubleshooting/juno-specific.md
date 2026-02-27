# Juno HPC Cluster-Specific Issues and Setup for Tempo

> **Quick answer:** Before running Tempo on Juno, load the required modules (`java/jdk-11.0.11`, `singularity/3.1.1`), set `TMPDIR=/scratch/$USER`, export `NXF_SINGULARITY_CACHEDIR`, and launch the pipeline inside a `screen` or `tmux` session to prevent interruption on disconnect.

## Required Environment Setup

Every time you start a new session on Juno, the following environment must be configured before launching Tempo:

```bash
# Load required modules
module load java/jdk-11.0.11
module load singularity/3.1.1

# Set temporary directory to scratch space
export TMPDIR=/scratch/$USER
mkdir -p $TMPDIR

# Point to pre-downloaded Singularity images
export NXF_SINGULARITY_CACHEDIR=/juno/work/taylorlab/cmopipeline/singularity_images
```

Add these lines to your `~/.bashrc` to avoid repeating them each session.

### Verifying the Environment

Confirm everything is loaded correctly before launching a run:

```bash
java -version       # Should show 11.0.11 or higher
singularity version # Should show 3.1.1
echo $TMPDIR        # Should be /scratch/<your_username>
echo $NXF_SINGULARITY_CACHEDIR  # Should be set
which nextflow      # Should show a valid path
```

## Running Long Pipeline Jobs

Tempo runs can take hours to days depending on sample count and sequencing type. If your SSH session disconnects, the Nextflow head process terminates and the pipeline stops. Use one of these approaches to keep the process alive:

### Using screen (Recommended)

```bash
screen -S tempo_run
# Run your pipeline command here
# Detach with Ctrl+A then D
# Reattach later with: screen -r tempo_run
```

### Using tmux

```bash
tmux new -s tempo_run
# Run your pipeline command here
# Detach with Ctrl+B then D
# Reattach later with: tmux attach -t tempo_run
```

### Using nohup

```bash
nohup nextflow run dsl2.nf \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  -profile juno \
  --outDir results \
  > nextflow_stdout.log 2>&1 &
```

With `nohup`, check progress by tailing the log: `tail -f nextflow_stdout.log`.

## LSF Queue Submission

Tempo submits individual process tasks to the LSF scheduler automatically via the Juno profile. The Nextflow head process runs on the login node (or wherever you launched it) and orchestrates LSF job submissions. You do not need to `bsub` the Nextflow command itself, but you should not run heavy operations on the login node.

Common LSF commands for monitoring Tempo jobs:

```bash
bjobs -w          # List your running/pending jobs
bjobs -l <jobid>  # Detailed info on a specific job
bkill <jobid>     # Kill a specific job
bkill 0           # Kill all your jobs (use with caution)
```

## Scratch Space Management

The `/scratch` filesystem is intended for temporary files and has limited space per user. Tempo processes write intermediate files here when `TMPDIR` is set. Monitor your usage:

```bash
du -sh /scratch/$USER
```

Clean up old temporary files periodically. If `/scratch` fills up, processes will fail with "No space left on device" errors. Move completed results to your project directory under `/juno/work/` and remove scratch artifacts.

## Common Juno-Specific Issues

### Module Not Loaded

Symptom: `java: command not found` or `singularity: command not found`. Load the modules as described above. The default Java on Juno may be Java 8, which is incompatible with modern Nextflow.

### Wrong Java Version

Nextflow requires JDK 11 or higher. If you see errors about unsupported class file versions or `UnsupportedClassVersionError`, verify with `java -version` and reload the correct module.

### TMPDIR Full or Not Set

If `TMPDIR` is not set, processes default to `/tmp`, which has very limited space on Juno compute nodes. If `/scratch` is full, clean old files or request additional space from the HPC team.

### Filesystem Permissions

Files under `/juno/work/tempo/cmopipeline/` are shared and may be read-only for your account. Do not attempt to modify shared reference files. If you need custom references, copy them to your own directory.

## File Locations

| File | Location |
|------|----------|
| Scratch space | `/scratch/$USER` |
| Shared Singularity images | `/juno/work/taylorlab/cmopipeline/singularity_images/` |
| Reference files | `/juno/work/tempo/cmopipeline/` |
| Nextflow log | `.nextflow.log` in the run directory |

## Example

Full setup and test run on Juno:

```bash
# Start a persistent session
screen -S tempo_test

# Set up environment
module load java/jdk-11.0.11
module load singularity/3.1.1
export TMPDIR=/scratch/$USER
mkdir -p $TMPDIR
export NXF_SINGULARITY_CACHEDIR=/juno/work/taylorlab/cmopipeline/singularity_images

# Run the test pipeline
nextflow run dsl2.nf \
  --mapping test_inputs/local/full_test_mapping.tsv \
  --pairing test_inputs/local/full_test_pairing.tsv \
  -profile test_singularity \
  --outDir results \
  --aggregate \
  --workflows="SNV,qc,lohhla"
```

## See Also

- [Troubleshooting Container Issues](container-issues.md) -- Singularity-specific problems on Juno
- [Fixing Out-of-Memory and Wall Time Errors](resource-limits.md) -- LSF resource limits
- [Resolving Missing Reference Files](missing-references.md) -- reference paths on the Juno filesystem
