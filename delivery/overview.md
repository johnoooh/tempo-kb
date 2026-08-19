# Tempo Cohort Delivery: Overview for Investigators

> **Quick answer:** "Delivery" is the step after the Tempo pipeline finishes analyzing your samples, where your cohort's results are made accessible to you on MSK file storage and a delivery email is sent. Delivery is managed by the CMO Project Management (CMO PM) and CMO Computational Science (CCS) teams — you receive results, you do not run the pipeline yourself.

## What This Section Covers

These pages answer the questions investigators and end users most often have **after** requesting a WES recapture cohort:

- [Accessing your results](accessing-results.md) — where the files live and how to mount the share (PC & Mac)
- [Understanding the delivery email](delivery-email.md) — what every part of the email and its attachments mean
- [Delivery timeline & status](timeline-and-status.md) — how long it takes and what "delayed" means
- [Data use policies](policies.md) — embargo, firewall/IRB, dbGaP, and publication registration
- [Requesting a cohort](requesting-a-cohort.md) — what a request is and what your CMO PM needs from you

For questions about the **science** of the pipeline outputs (MAFs, FACETS, QC metrics, etc.), see the [`tools/`](../tools/), [`outputs/`](../outputs/), and [`recipes/`](../recipes/) sections of this knowledge base.

## The Delivery Lifecycle

A WES recapture cohort moves through these stages:

1. **Request** — your CMO Project Manager submits a cohort request on your behalf (see [requesting-a-cohort.md](requesting-a-cohort.md)). This is a PM-managed step.
2. **Analysis** — Tempo aligns, calls variants, and performs the full somatic/QC analysis for every tumor-normal pair in the cohort. See [`pipeline/overview.md`](../pipeline/overview.md).
3. **Completion check** — the delivery system waits until all required pipeline steps have finished for the cohort. See [timeline-and-status.md](timeline-and-status.md).
4. **Access & delivery** — file permissions are set so that everyone listed as an `endUser` on the request can read the results, and a delivery email is sent to your team.

## Who Does What

| Team | Responsibility | Contact |
|------|----------------|---------|
| **CMO Project Management (CMO PM)** | Submits cohort requests; project & sample info; help accessing results | skicmopm@mskcc.org |
| **CMO Computational Science (CCS) / Tempo team** | Runs the pipeline; questions about pipeline output and QC | zzPDL_CMO_TEMPO_Support@mskcc.org |

The official Tempo docs are at <https://github.com/mskcc/tempo/tree/develop/docs>.

## See Also

- [accessing-results.md](accessing-results.md) — get to your files
- [delivery-email.md](delivery-email.md) — decode the delivery email
- [`pipeline/overview.md`](../pipeline/overview.md) — what the pipeline actually does
