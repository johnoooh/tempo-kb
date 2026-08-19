# Troubleshooting FACETS Failures (DoFacets)

> **Quick answer:** `DoFacets` runs `run-facets-wrapper.R` and retries internally with up to 4 incrementing seeds; if all fail (non-zero exit), the Nextflow task then retries per `errorStrategy`. FACETS most often fails on low-purity / low-coverage samples with too few heterozygous SNPs per segment — governed by `min_nhet`, `ndepth`, and `cval`. Importantly, **a FACETS failure does not drop the sample from delivery** — the pipeline falls back to purity 0.3.

> 📖 **Related official docs:** general pipeline-failure debugging (Nextflow process, individual jobs, LSF and Singularity errors) is in the official [troubleshooting.md](https://github.com/mskcc/tempo/blob/develop/docs/troubleshooting.md). This page adds the **FACETS-specific** detail the official troubleshooting doc does not cover.

## The retry mechanism (verified — `modules/process/Facets/DoFacets.nf`)

Inside the process, FACETS is wrapped in a shell loop that tries up to **4 seeds**, incrementing the seed by 1 each time the R script exits non-zero:

```bash
seed=$((${params.facets.seed}-1)); attemptNumber=0
while [ $i -eq 1 ]; do
  attemptNumber=$(( attemptNumber + 1 ))
  if [ $attemptNumber -gt 4 ]; then break; fi
  seed=$((seed+i))
  Rscript run-facets-wrapper.R --cval ${params.facets.cval} --min-nhet ${params.facets.min_nhet} \
    --normal-depth ${params.facets.ndepth} --purity-cval ${params.facets.purity_cval} \
    --purity-min-nhet ${params.facets.purity_min_nhet} --seed $seed --everything ...
  i=$?
done
```

On top of this, the Juno `errorStrategy` retries the **whole Nextflow task** up to `maxRetries = 3` (with memory scaling). So a true failure means FACETS could not converge across all seeds and all task retries.

## Tunable parameters (verified — `conf/exome.config` / `conf/genome.config`)

| Param | Exome | WGS | `run-facets-wrapper.R` flag | When to change |
|-------|-------|-----|------------------------------|----------------|
| `facets.cval` | 100 | 1000 | `--cval` | lower → more segments (over-segment noisy data); raise to stabilize |
| `facets.purity_cval` | 500 | 5000 | `--purity-cval` | coarser purity fit |
| `facets.min_nhet` | 25 | 25 | `--min-nhet` | **lower if too few het SNPs per segment** (sparse/low-purity) |
| `facets.purity_min_nhet` | 25 | 25 | `--purity-min-nhet` | same, for the purity run |
| `facets.ndepth` | 35 | 15 | `--normal-depth` | min normal depth at a SNP; lower for shallow normals |
| `facets.snp_nbhd` | 250 | 500 | `--snp-window-size` | SNP sampling window |
| `facets.seed` | 100 | 100 | `--seed` | starting seed (auto-incremented in the loop) |

Override nested params via a custom config (they are not `--` flags):

```groovy
// rescue.config
params.facets.min_nhet = 15
params.facets.cval     = 50
```
```bash
nextflow run dsl2.nf ... -c rescue.config -resume
```

## What to check when FACETS fails

1. **Coverage / purity** — low tumor purity or shallow coverage leaves too few informative het SNPs. Check the sample's QC ([../outputs/qc-thresholds.md](../outputs/qc-thresholds.md)); a `warn`/`fail` coverage almost always explains a FACETS failure.
2. **`min_nhet` / `ndepth`** — if segments are being rejected for too few het SNPs, lower `min_nhet` (and `ndepth` for a shallow normal) and re-run with `-resume`.
3. **Assay/config mismatch** — confirm exome data used the exome config (`cval=100`), not WGS (`cval=1000`); a mismatch produces unstable fits. See [../tools/facets-algorithm.md](../tools/facets-algorithm.md).
4. **The actual R error** — open `.command.err`/`.command.log` in the process work dir (`nextflow log <run> -f workdir,name -F "name =~ /DoFacets/"`).

> **No explicit failure-cause messages** are emitted by `DoFacets.nf` itself — the loop only reacts to the R script's non-zero exit. The causes above are inferred from the parameters the wrapper exposes (`min-nhet`, `normal-depth`), not from named error strings in the repo.

## A failed FACETS does not exclude the sample

If all 4 seeds fail, `generate_samplestatistics.R` substitutes **purity = 0.3** (verified) and the sample continues — it is **not** dropped from the cohort. So a delivered sample can carry a fallback purity that silently propagates into CCF, zygosity, and TMB. See "Is a purity of 0.3 real or a fallback?" in [../tools/facets-interpreting.md](../tools/facets-interpreting.md), and the purity–ploidy checklist in [../recipes/resolve-purity-ploidy-wgd.md](../recipes/resolve-purity-ploidy-wgd.md).

## See Also

- [failed-processes.md](failed-processes.md) — generic process-failure debugging
- [resource-limits.md](resource-limits.md) — raising memory for a process
- [../tools/facets-algorithm.md](../tools/facets-algorithm.md) — parameters and modes
- [../tools/facets-interpreting.md](../tools/facets-interpreting.md) — reading FACETS QC and the 0.3 fallback
