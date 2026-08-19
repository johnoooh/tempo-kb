# Tempo Calculations & Thresholds Reference

> **Quick answer:** Every important formula, threshold, and hardcoded constant used by the Tempo pipeline's helper scripts, in one place, with the source file. Use this as a lookup; follow the links for context. Values are from the `develop` branch and are the documented defaults — a specific run may override them via config.

## Variant quantities

| Quantity | Formula / value | Source |
|----------|-----------------|--------|
| Tumor VAF (reporting) | `t_alt_count / (t_alt_count + t_ref_count)` | `filter-somatic-maf.R` |
| Expected VAF, somatic | `(p·alt_cn) / (p·tcn + 2·(1−p))` | `annotate-with-zygosity-somatic.R` |
| Expected VAF, germline | `(p·alt_cn + (1−p)) / (p·tcn + 2·(1−p))`, ×`(n_alt_freq/0.5)` ref-bias factor | `annotate-with-zygosity-germline.R` |
| FACETS purity fallback | `0.3` when fit purity is NA | `generate_samplestatistics.R` |
| Gender from chrX | male if `sum(nhet)/sum(num.mark) < 0.01` on chrX | `generate_samplestatistics.R` |

See [`recipes/resolve-purity-ploidy-wgd.md`](../recipes/resolve-purity-ploidy-wgd.md) and [`recipes/calculate-vaf.md`](../recipes/calculate-vaf.md).

## TMB

`TMB = (non-synonymous somatic coding mutations) / CDS_size_Mb`. Germline excluded. Non-synonymous classes: Missense, Nonsense, Nonstop, Frame_Shift_Ins/Del, In_Frame_Ins/Del, Translation_Start_Site, Splice_Site, Splice_Region.

| Assay | CDS denominator (Mb) |
|-------|----------------------|
| Agilent Exon 51MB v3 | 30.89918 |
| IDT Exome v1 FP | 36.00458 |
| WGS | 45.57229 |

Source: `create_metadata_file.py`. See [`recipes/calculate-tmb.md`](../recipes/calculate-tmb.md). **CDS denominators differ by assay — TMB is not comparable across bait sets without knowing which was used.**

## Somatic MAF filters (defaults)

| Tag | Condition | Default |
|-----|-----------|---------|
| low_vaf | `t_var_freq < x` | 0.05 |
| low_t_depth | `t_depth < x` | 20 |
| low_t_alt_count | `t_alt_count < x` | 3 |
| low_n_depth | `n_depth < x` | 10 |
| high_n_alt_count | `n_alt_count > x` | 3 |
| high_gnomad_pop_af | popmax AF > x | 0.01 |
| PoN | `PoN ≥ x` | 10 |
| low_mapping_quality | `MQ < 55` (non-SNP) | 55 |
| strand_bias (ref) | `MQ < 40` | 40 |
| Hotspot rescue | keep hotspot if only `low_vaf` and `t_var_freq ≥ 0.02` | 0.02 |

Source: `filter-somatic-maf.R`. See [`tools/maf-filtering.md`](../tools/maf-filtering.md).

## Germline MAF filters (defaults)

| Tag | Condition | Default |
|-----|-----------|---------|
| low_n_depth | `n_depth < x` | 20 |
| low_n_vaf (SNV) | non-CH gene, `n_var_freq < x` | 0.35 |
| low_n_vaf (indel) | non-CH gene, `n_var_freq < x` | 0.25 |
| ch_mutation | CH gene, `n_var_freq < 0.35` AND `t_var_freq < 0.25` | — |
| t_in_n_contamination | `t_var_freq > 3 × n_var_freq` | 3× |
| gnomAD popmax | AF > x | 0.02 |

CH = clonal-hematopoiesis gene (36-gene list incl. DNMT3A, TET2, TP53, ASXL1…). Source: `filter-germline-maf.R`.

## Structural variants

| Rule | Value | Source |
|------|-------|--------|
| Multi-caller PASS | ≥ 2 callers if >2 callers ran, else ≥ 1 | `SomaticMergeSVs.nf` / `filter-sv-vcf.py` |
| Drop near-duplicate breakpoints | both ends ≤ 1 bp apart | `filter-sv-vcf.py` |
| iAnnotateSV gene distance | 3000 bp | `run_iannotatesv.py` |
| SVclone max clusters | 6 | `svclone_config.ini` |
| SVclone min depth | 8 | `svclone_config.ini` |
| SVclone SV→CNV offset | 100,000 bp | `svclone_config.ini` |

See [`tools/sv-callers.md`](../tools/sv-callers.md), [`tools/svclone-clonality.md`](../tools/svclone-clonality.md).

## Copy number (FACETS)

| Param | WES | WGS |
|-------|-----|-----|
| hisens `cval` | 100 | 1000 |
| `purity_cval` | 500 | 5000 |
| `snp_nbhd` (bp) | 250 | 500 |
| `ndepth` (min normal depth) | 35 | 15 |
| `min_nhet` | 25 | 25 |
| pseudo-snps (bp) | 50 | 50 |
| seed | 100 (+1 per retry, ≤4) | 100 |

WGD status: `True` if any FACETS `wgd` flag is True, else `False`, else `NA` (`create_metadata_file.py`). See [`tools/facets-algorithm.md`](../tools/facets-algorithm.md) and [`recipes/resolve-purity-ploidy-wgd.md`](../recipes/resolve-purity-ploidy-wgd.md).

## Signatures / MSI / HRD

| Item | Value | Source |
|------|-------|--------|
| tempoSig COSMIC default | v3 (switchable to v2) | `RunMutationSignatures.nf` |
| tempoSig permutations / seed | 10,000 / 132 | `RunMutationSignatures.nf` |
| MSI score | MSIsensor `%` unstable loci — **Tempo applies NO MSI-H threshold** | `RunMsiSensor.nf` |
| HRDetect SNV signatures | fit against **COSMICv2** (SBS3, SBS8) | `HRDetect_wrapper.R` |
| HRDetect features | del.mh.prop, SNV3, SV3, SV5, hrd-LOH, SNV8 | `HRDetect_wrapper.R` |
| SV signatures reference | RefSigv2 (bootstrap) | `sv_signatures_wrapper.R` |

See [`tools/signatures-msi.md`](../tools/signatures-msi.md), [`tools/hrdetect.md`](../tools/hrdetect.md).

## HLA / neoantigen

| Item | Value | Source |
|------|-------|--------|
| LOHHLA min coverage | 10 reads | `RunLOHHLA.nf` |
| Neoantigen binder cutoff | **not applied in Tempo** — raw predictions only | `RunNeoantigen.nf` |

See [`tools/hla-neoantigen.md`](../tools/hla-neoantigen.md).

## QC pass/warn/fail

Full WES & WGS threshold tables are in [`outputs/qc-thresholds.md`](../outputs/qc-thresholds.md). Conpair markers: 1000G Phase 3, autosomes, MAF ≥ 0.4, LD r² ≤ 0.8, normal-homozygous only.

## Delivery operations (tempodeliver)

| Item | Value / rule | Source |
|------|--------------|--------|
| Cohort "complete" steps | 8 `SomaticAggregate*` + `QcConpairAggregate`/`QcBamAggregate` all present | `nextflow.config` `cohortCompleteSteps` |
| Delivery also requires | `.lock_cohort` flag + `CohortRunMultiQC` done + CRF unchanged since intake | `deliver_cohorts.nf` |
| Trace "failed" | `status==FAILED` AND `attempt==4` (final retry) | `auto_tracker.py` |
| Delay reminder | once, after 7 days with no `.delivered` | `deliver_cohorts.nf` |
| ID normalization | `C-…` ↔ `s_C_…` (`-`↔`_`, `s_` prefix) | `cohort_utils/utils.py`, `assessCohort.py` |
| ACL user set | `endUsers + defaultUsers + pmUsers` | `deliver_cohorts.nf` |

See [`delivery/timeline-and-status.md`](../delivery/timeline-and-status.md).

## See Also

- [`tools/tools-summary.md`](../tools/tools-summary.md) — the tool inventory these calculations belong to
