# Troubleshooting Singularity and Docker Container Issues in Tempo

> **Quick answer:** Most container errors in Tempo stem from a missing or misconfigured `NXF_SINGULARITY_CACHEDIR`. Set it to `/juno/work/taylorlab/cmopipeline/singularity_images` where pre-downloaded images are stored, and avoid running `singularity pull` on login nodes.

## Common Container Error Messages

### "Failed to pull singularity image"

This is the most frequent container error in Tempo on Juno. The full error typically looks like:

```
FATAL: could not open image /path/to/image.sif: failed to retrieve path
```

or:

```
ERROR ~ Error executing process > 'ProcessName'
Caused by: Failed to pull singularity image
```

This occurs when Nextflow tries to convert a Docker Hub image to a Singularity `.sif` file and either the download fails, disk space is insufficient, or the cache directory is not set.

### "Descriptor not found"

This error occurs during the Docker-to-Singularity conversion process when the specified Docker image tag does not exist on Docker Hub. Verify the image name and tag in the pipeline configuration.

## Setting the Singularity Cache Directory

Tempo's Docker images have already been pre-downloaded and converted to Singularity format on Juno. Set the cache directory so Nextflow uses these existing images instead of downloading them:

```bash
export NXF_SINGULARITY_CACHEDIR=/juno/work/taylorlab/cmopipeline/singularity_images
```

Add this to your `~/.bashrc` or `~/.bash_profile` so it persists across sessions. The images in this directory follow the naming convention used by Nextflow for automatic cache lookups.

## Manually Pulling Images

If a specific image is missing from the cache, you can pull it manually. Do this on a compute node, not the login node, because pulls are slow and resource-intensive:

```bash
# Request an interactive compute session first
bsub -Is -q interactive bash

# Load Singularity
module load singularity/3.1.1

# Pull the image
singularity pull docker://cmopipeline/facets:0.5.14

# Move the .sif file to the shared cache
mv facets_0.5.14.sif /juno/work/taylorlab/cmopipeline/singularity_images/
```

The resulting `.sif` file name must match what Nextflow expects. Check the pipeline configuration or `.command.sh` in the failed work directory to see the expected image path.

## Docker Hub Rate Limits

Docker Hub imposes pull rate limits on anonymous and free accounts. If multiple users are pulling images simultaneously, you may see HTTP 429 errors. Using the pre-populated `NXF_SINGULARITY_CACHEDIR` avoids this issue entirely since no downloads are needed.

## Docker vs Singularity Behavior Differences

Tempo pipeline processes are defined with Docker image references (e.g., `cmopipeline/facets:0.5.14`). On Juno, the Singularity profile converts these automatically. Key differences to be aware of:

- **Filesystem binding**: Singularity automatically binds the user's home directory. Additional paths must be bound explicitly with `-B`. Nextflow handles standard bindings, but custom paths may need configuration.
- **User context**: Singularity runs as the calling user (not root), unlike Docker which runs as root by default. This avoids permission issues but means the container cannot write to root-owned paths.
- **Environment variables**: Singularity inherits the host environment by default, whereas Docker containers start with a clean environment. This can cause conflicts if host environment variables interfere with tools inside the container.

## Disk Space Issues

Singularity `.sif` files and temporary build layers consume disk space. Common locations that can fill up:

| Location | What it stores |
|----------|---------------|
| `NXF_SINGULARITY_CACHEDIR` | Converted .sif images |
| `/tmp` or `$TMPDIR` | Temporary build files during pull |
| `~/.singularity` | Default local cache |

If your `TMPDIR` is full, Singularity builds will fail. Ensure `TMPDIR` points to `/scratch/$USER` where space is more available.

## File Locations

| File | Location |
|------|----------|
| Pre-downloaded images | `/juno/work/taylorlab/cmopipeline/singularity_images/` |
| Container error logs | `work/{hash}/.command.err` |
| Container config in pipeline | `nextflow.config` (singularity profile section) |
| Singularity local cache | `~/.singularity/` |

## Example

A run fails with a container pull error for the FACETS image:

```bash
# Set the cache directory
export NXF_SINGULARITY_CACHEDIR=/juno/work/taylorlab/cmopipeline/singularity_images

# Verify the image exists
ls -la $NXF_SINGULARITY_CACHEDIR/*facets*

# If missing, pull on a compute node
bsub -Is -q interactive bash
module load singularity/3.1.1
cd $NXF_SINGULARITY_CACHEDIR
singularity pull docker://cmopipeline/facets:0.5.14

# Resume the pipeline
nextflow run dsl2.nf \
  --mapping mapping.tsv \
  --pairing pairing.tsv \
  -profile juno \
  --outDir results \
  -resume
```

## See Also

- [Juno HPC Cluster-Specific Issues](juno-specific.md) -- environment setup including module loading
- [Diagnosing Failed Processes](failed-processes.md) -- inspecting work directories for container errors
- [Resuming Interrupted Runs](nextflow-resume.md) -- resuming after fixing a container issue
