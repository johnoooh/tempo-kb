# Understanding Your Tempo Delivery Email

> **Quick answer:** The delivery email confirms your cohort is ready, tells you how many tumor-normal pairs were analyzed and how many passed QC, points you to your results on file storage, and attaches an ID-mapping file (and sometimes a historical-metadata file). This page explains each part.

## Anatomy of the Email

A delivery email contains the following pieces:

### Project and cohort identification

> *"The samples for your CMO WES recapture Project **"\<Project Title\> (\<subtitle\>)"** have been sequenced and analyzed through version \<version\> of the TEMPO pipeline. The cohort analysis (cohort ID: **\<cohort_id\>**) included \<N\> T/N pairs."*

- **Project Title / subtitle** — your project name as registered with CMO PM.
- **Version** — the Tempo pipeline version used (e.g. `v2.1.x`). Your results live under this version on the fileshare.
- **Cohort ID** — the identifier for this delivery (e.g. `CCS_XXXXXX`). This is the folder name under `cohort_level/`.
- **T/N pairs** — the number of matched tumor-normal pairs in the cohort.

### QC summary

> *"Of \<N\> tumor samples, \<M\> passed Tempo's QC criteria (others are warn/fail)."*

Tempo classifies each sample as **pass**, **warn**, or **fail** against its QC criteria. The email gives the count that passed; the rest are warn or fail. The pass count comes from the cohort MultiQC stats table (`multiqc_general_stats.txt`), so it reflects the same thresholds as the full report. A short QC summary is attached, and the **full QC report is in your deliverables** (the cohort-level MultiQC report). Review QC before analysis.

For the exact pass/warn/fail thresholds (coverage, contamination, concordance, etc., and how they differ between WES and WGS), see [`outputs/qc-thresholds.md`](../outputs/qc-thresholds.md). For what each QC metric means, see the QC pages under [`outputs/`](../outputs/) (e.g. [`outputs/qc-conpair.md`](../outputs/qc-conpair.md), [`outputs/qc-qualimap.md`](../outputs/qc-qualimap.md)).

### ID mapping attachment

The email attaches an **ID mapping file** linking the IDs you know (investigator/sample names) to the IDs used inside the pipeline outputs. Use it to connect your samples to the result files.

### Access instructions

The email lists both the direct file-storage path and the SAMBA/SMB mounting steps for your cohort. See [accessing-results.md](accessing-results.md) for a walkthrough.

### Historical metadata attachment (only sometimes)

If the email includes an attachment named **`<cohort_id>.historical.metadata.tsv`**, some samples in your cohort have **IDs or tumor/normal pairings that have since changed**. The file's leading column descriptions explain each field. Samples with no known ID or pairing changes are filtered out of this file, so if you don't receive it, no such changes are flagged for your cohort.

### Regeneration disclaimer (only sometimes)

Some deliveries include a disclaimer that, due to an ongoing systems issue, the project **may be regenerated and redelivered with updated sample IDs**, replacing the current delivery. If you see this, no action is needed from you — a new notification with updated metadata will follow if a redelivery happens.

## Who to Contact

The email closes with two contacts — pick based on your question:

- **Project & sample information, or help accessing results** → CMO Project Management: **skicmopm@mskcc.org**
- **Pipeline output / QC interpretation** → CMO Computational Science (CCS) Team: **zzPDL_CMO_TEMPO_Support@mskcc.org**, or see the [Tempo docs](https://github.com/mskcc/tempo/tree/develop/docs).

## See Also

- [accessing-results.md](accessing-results.md) — reach the files referenced in the email
- [policies.md](policies.md) — embargo and publication rules referenced in the email
- [timeline-and-status.md](timeline-and-status.md) — what happens if you haven't received an email yet
