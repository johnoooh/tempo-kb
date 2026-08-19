# Resolving Purity–Ploidy Ambiguity and Checking a WGD Call

> **Quick answer:** When FACETS reports whole-genome doubling (WGD) but observed VAFs look like a non-doubled (diploid) genome, you are likely seeing purity–ploidy degeneracy. Anchor on the **expected-VAF formula** for a few clonal drivers: if observed VAF ≈ `purity/2` for a heterozygous mutation, the diploid solution is favored over the tetraploid one. This page gives the formula, how Tempo derives WGD, and a confirm/refute checklist.

## Why the ambiguity happens

FACETS fits purity and ploidy jointly by anchoring `dipLogR` (the logR of the diploid state). At moderate purity (~0.4–0.6), a **diploid tumor at higher purity** and a **tetraploid (WGD) tumor at lower purity** can produce nearly identical logR/logOR profiles. If `dipLogR` is anchored one copy-state too low, FACETS reports an inflated ploidy and a spurious WGD. The tie-breaker FACETS cannot always see — but you can — is the **mutation VAF**.

## The expected-VAF formula

This is exactly the formula Tempo uses internally to test whether an observed VAF is concordant with the FACETS copy-number state (`annotate-with-zygosity-somatic.R`).

For a **somatic** variant on a segment with total copy number `tcn`, with `alt_cn` mutant copies and `ref_cn = tcn − alt_cn` reference copies, at tumor purity `p`:

```
expected_VAF = (p · alt_cn) / [ (p · ref_cn + (1 − p)) + (p · alt_cn + (1 − p)) ]
             = (p · alt_cn) / (p · tcn + 2·(1 − p))
```

The `2·(1 − p)` term is the two reference copies contributed by every normal (stromal) cell. Worked cases at `p = 0.56`:

| Hypothesis | tcn | alt_cn | Expected VAF |
|-----------|-----|--------|--------------|
| Diploid, clonal heterozygous | 2 | 1 | 0.56 / (1.12 + 0.88) = **0.28** |
| WGD/tetraploid, 1 of 4 copies mutant | 4 | 1 | 0.56 / (2.24 + 0.88) = **0.18** |
| Diploid with LOH, 1 copy left, mutant | 1 | 1 | 0.56 / (0.56 + 0.88) = **0.39** |

So an observed VAF of ~0.30 for a known clonal heterozygous driver matches **diploid (0.28)**, not WGD (0.18) — evidence the WGD call is a purity–ploidy artifact.

> **Germline** variants use a different numerator (`annotate-with-zygosity-germline.R`): `expected_VAF = (p·alt_cn + (1−p)) / (p·tcn + 2·(1−p))`, because germline heterozygotes carry the allele in normal cells too. A reference-alignment-bias factor (`n_alt_freq / 0.5`) is also applied to germline expectations.

Rearranging the somatic formula lets you **solve for the purity** implied by a confidently-clonal-heterozygous driver: `p = 2·VAF / (1 + VAF·(2 − tcn))`. If several drivers imply a purity well above the FACETS estimate under the diploid model, stromal contamination likely pulled the FACETS purity down and triggered the wrong ploidy branch.

## How Tempo decides WGD

Tempo does not compute WGD itself — `MetaDataParser` (`create_metadata_file.py`) reads the `wgd` column from the pair-level facetsPreview summary file `{idTumor}__{idNormal}.facets_qc.txt` (passed as `--facetsQC` from `dsl2.nf`):

```
if sum(qc.wgd) > 0:                # any TRUE wgd → True
    WGD_status = True
elif sum(qc.wgd == False) > 0:     # otherwise any FALSE → False
    WGD_status = False
else:
    WGD_status = 'NA'
```

This file lives at the **pair root**, not inside `facets{version}.../`. There is also a `{pair}.qc.txt` **inside** the versioned facets subdir with per-fit (one row per `cval`) facetsPreview output — same columns minus the top-level `is_best_fit`/`purity`/`ploidy` block. The pipeline uses the pair-root `*.facets_qc.txt` for the WGD call.

MultiQC marks `wgd` as `fail` if facetsPreview's WGD detection itself failed. So a reported WGD is only as reliable as the underlying FACETS fit — which is the thing you are questioning.

## Confirm / refute checklist

| Check | Where | Supports "WGD is wrong" if… |
|-------|-------|------------------------------|
| Expected vs observed VAF for clonal het drivers | final MAF + formula above | observed ≈ `p/2` (diploid), not `p/4` |
| Hisens vs purity ploidy | `*_OUT.txt` / `_purity.out` / `_hisens.out` | ploidy differs by > 1.0 between fits |
| `dipLogR` | `*_OUT.txt` | baseline placed away from the most common copy state |
| Segment CN at the driver loci | `*_hisens.cncf.txt` (`tcn.em`, `lcn.em`) | driver sits on `tcn=2,lcn=1`, not `tcn=4` |
| Genome-wide logR plot | FACETS PNG | most segments cluster near logR ≈ 0 (flat), not shifted up |
| facetsPreview QC | `*.facets_qc.txt` | fit flagged / `wgd` = fail |
| ASCAT (WGS only) | `*.samplestatistics.txt` | independent ploidy ≈ 2 at higher purity |

If the VAF math, a flat logR, and a `tcn=2` segment at the driver all line up, the WGD call is best treated as a purity–ploidy artifact. A manual FACETS re-fit with a corrected `dipLogR`/purity (or trusting ASCAT for WGS) is the standard remedy.

## See Also

- [calculate-vaf.md](calculate-vaf.md) — VAF basics and the expected-VAF section
- [../tools/facets-interpreting.md](../tools/facets-interpreting.md) — reading FACETS QC, plots, and failure modes (incl. WGD)
- [../tools/facets-algorithm.md](../tools/facets-algorithm.md) — how purity/ploidy/dipLogR are fit
- [extract-ccf-clonality.md](extract-ccf-clonality.md) — CCF columns, which depend on the purity/ploidy fit
- [facets-qc-r.md](facets-qc-r.md) — loading the FACETS `.Rdata` to compare fits
