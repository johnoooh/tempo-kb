# QC Pass / Warn / Fail Thresholds (WES & WGS)

> **Quick answer:** Tempo's per-sample and per-pair QC status (`pass`, `warn`, `fail`) shown in MultiQC and in the delivery email is computed by applying fixed thresholds to coverage, contamination, concordance, duplication, and enrichment metrics. The thresholds differ between exome and genome runs. They are defined in the MultiQC config (`lib/multiqc_config/exome_multiqc_config.yaml` and `wgs_multiqc_config.yaml`).

The single QC label you see (`pass`/`warn`/`fail`) is the **worst** status across the metrics below — one failing metric makes the sample `fail`.

## Exome (WES) thresholds

| Metric (source tool) | Pass | Warn | Fail |
|----------------------|------|------|------|
| Tumor mean coverage (Qualimap) | ≥ 60× | < 60× | < 40× |
| Normal mean coverage (Qualimap) | ≥ 30× | < 30× | < 20× |
| Tumor contamination (Conpair) | < 2% | > 2% | > 5% |
| Normal contamination (Conpair) | < 15% | > 15% | *(no fail level)* |
| T/N concordance (Conpair) | ≥ 90% | < 90% | < 50% |
| Fold enrichment (CollectHsMetrics) | ≥ 30 | < 30 | *(no fail level)* |
| % aligned reads (Qualimap) | ≥ 97% | < 97% | *(no fail level)* |
| Duplicate fraction (Alfred) | ≤ 55% | > 55% | > 75% |

## Genome (WGS) thresholds (differences from exome)

| Metric (source tool) | Pass | Warn | Fail |
|----------------------|------|------|------|
| Tumor mean coverage | ≥ 60× | < 40× | < 40× |
| Normal mean coverage | ≥ 30× | < 20× | < 20× |
| Tumor contamination (Conpair) | < 1% | > 1% | > 5% |
| Normal contamination (Conpair) | < 1% | > 1% | > 5% |
| T/N concordance (Conpair) | ≥ 90% | < 90% | < 75% |
| % aligned reads (Qualimap) | ≥ 97% | < 97% | < 60% |

> **Note the key WES vs WGS difference:** exome runs tolerate **normal contamination up to 15%** (no fail), because a small amount of tumor-in-normal is common with IMPACT recapture normals; WGS holds both tumor and normal contamination to **< 1%**. `CollectHsMetrics`/fold-enrichment is exome-only (it is a bait-capture metric).

## Where the status comes from

- Per-sample and per-pair status is written into the MultiQC `multiqc_general_stats.txt` table (inside `multiqc_data.zip`) in a column ending in `-Status`.
- The **delivery email's** "X of N tumor samples passed Tempo's QC criteria" count is computed by reading that `-Status` column and counting tumors with status `pass` (see [`delivery/delivery-email.md`](../delivery/delivery-email.md)). If `multiqc_data.zip` is missing, the QC note is silently omitted from the email.

## Should I exclude warn/fail samples from my analysis?

There is no single Tempo-enforced rule — this is an analyst decision — but a practical default:

- **`fail`** → **exclude by default**, or treat as low-confidence and confirm anything you rely on. A failed metric means a core assumption (sufficient coverage, correct pairing, low contamination) is violated. A **concordance fail** in particular suggests a possible sample swap — treat as blocking, not just low-confidence.
- **`warn`** → **usually keep, but with a sensitivity check.** Re-run your key result (e.g. TMB, a driver call) with warns included and excluded; if the conclusion is stable, keep them. If a warn sample is an outlier driving your finding, scrutinize it.
- **Match the threshold to the metric to your use.** A low-coverage warn matters most for low-VAF/subclonal and CN work; a high-duplication warn inflates apparent coverage; a contamination warn matters most for low-VAF "somatic" calls.

This is general analysis guidance, not a Tempo policy — document whatever inclusion rule you choose. When in doubt about a specific sample, ask the CCS Tempo team.

## Interpreting a warn/fail

A `warn` or `fail` does not necessarily invalidate results — it flags that a metric is outside the comfortable range and the affected calls should be treated with extra caution:

- **Low coverage** → reduced sensitivity for low-VAF and subclonal variants; CN segmentation noisier.
- **High contamination** → spurious low-VAF "somatic" calls; check Conpair before trusting borderline variants.
- **Low concordance** → possible sample swap or mispairing; results may belong to the wrong patient. Treat as blocking until resolved.
- **High duplication / low enrichment** → effective unique coverage lower than raw coverage suggests.

## See Also

- [qc-conpair.md](qc-conpair.md) — concordance & contamination details
- [qc-qualimap.md](qc-qualimap.md) — coverage & alignment metrics
- [qc-alfred.md](qc-alfred.md) — duplication and read-group metrics
- [`delivery/delivery-email.md`](../delivery/delivery-email.md) — how QC pass/warn/fail appears to investigators
- [`reference/calculations.md`](../reference/calculations.md) — all thresholds in one place
