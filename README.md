# Tempo Knowledge Base

A documentation knowledge base for MSKCC's **Tempo** WES/WGS tumor-normal analysis pipeline and its **cohort delivery** process. It answers two kinds of questions:

- **General Tempo questions** — how the pipeline works, what each tool does, and what the outputs mean.
- **Delivery questions** — how investigators get, read, and use their delivered cohort results.

> 📖 **Official Tempo docs are the authoritative source for running the pipeline:** **[`mskcc/tempo/docs`](https://github.com/mskcc/tempo/tree/develop/docs)** (on GitHub). This KB **complements** them — it adds the investigator/delivery side, a consolidated calculations reference, cross-tool synthesis, and unofficial non-Juno guidance. Where this KB and the official docs disagree about running Tempo, **trust the official docs** (the KB flags the few known exceptions inline).

## Sections

### [`delivery/`](delivery/) — Cohort delivery (for investigators)
How you receive and access your results after the pipeline runs.
- [overview.md](delivery/overview.md) — the delivery lifecycle and who to contact
- [accessing-results.md](delivery/accessing-results.md) — find your files and mount the share (PC & Mac)
- [delivery-email.md](delivery/delivery-email.md) — decode the delivery email and its attachments
- [timeline-and-status.md](delivery/timeline-and-status.md) — how long it takes; what "delayed" means
- [policies.md](delivery/policies.md) — embargo, firewall/IRB, dbGaP, and publication registration
- [requesting-a-cohort.md](delivery/requesting-a-cohort.md) — what a request is (PM-managed; informational)

### [`pipeline/`](pipeline/) — How the pipeline works (and how to run it yourself)
- [overview.md](pipeline/overview.md) — what Tempo does, stages, and key terminology
- **Run it yourself** (KB summaries of the [official docs](https://github.com/mskcc/tempo/tree/develop/docs), which remain authoritative): [installation.md](pipeline/installation.md) (prereqs & first run), [running-non-juno.md](pipeline/running-non-juno.md) (SLURM/PBS/local — *unofficial*), [custom-references.md](pipeline/custom-references.md) (your own reference data)
- Running it: [running-invocation.md](pipeline/running-invocation.md), [running-inputs.md](pipeline/running-inputs.md)
- Configuration: [config-profiles.md](pipeline/config-profiles.md), [config-references.md](pipeline/config-references.md), [config-resources.md](pipeline/config-resources.md), [config-containers.md](pipeline/config-containers.md)
- [tools-summary.md](pipeline/tools-summary.md) — the tools at a glance

### [`tools/`](tools/) — Per-tool deep dives
Start with [tools-summary.md](tools/tools-summary.md) — the **complete inventory of every tool** Tempo runs, by stage, with versions — and [tool-references.md](tools/tool-references.md) for each tool's **primary paper + source code**. Deep dives: [alignment & preprocessing](tools/alignment-preprocessing.md), [SNV callers](tools/snv-callers.md), [QC tools](tools/qc-tools.md), [FACETS](tools/facets-algorithm.md), MAF format & filtering, structural variants, HRDetect, SVclone, HLA/neoantigen, signatures/MSI.

### [`outputs/`](outputs/) — Output file reference
What each output file and directory contains, by category (somatic, germline, QC, BAMs, cohort aggregates) — start with [directory-structure.md](outputs/directory-structure.md). QC pass/warn/fail cutoffs (and whether to exclude warn/fail samples) are in [qc-thresholds.md](outputs/qc-thresholds.md); cohort copy-number files in [cna-genelevel-armlevel.md](outputs/cna-genelevel-armlevel.md).

### [`recipes/`](recipes/) — Task how-tos
Compute [TMB](recipes/calculate-tmb.md)/[VAF](recipes/calculate-vaf.md), [resolve a purity–ploidy / WGD ambiguity](recipes/resolve-purity-ploidy-wgd.md), [load & filter the cohort SV BEDPE](recipes/load-filter-sv-bedpe.md), build oncoplots, merge MAF with clinical data, batch-process samples, and more.

### [`reference/`](reference/) — Quick lookup
[calculations.md](reference/calculations.md) — every important formula, threshold, and hardcoded constant (TMB denominators, MAF filters, FACETS params, QC cutoffs, delivery rules) in one place, with source files.

### [`troubleshooting/`](troubleshooting/) — When things go wrong
Juno-specific issues, nextflow resume, missing references, resource limits, container and process failures, and [FACETS failures](troubleshooting/facets-failures.md).

## Contacts

- **CMO Project Management** (project/sample info, accessing results): skicmopm@mskcc.org
- **CMO Computational Science (CCS) / Tempo team** (pipeline output): zzPDL_CMO_TEMPO_Support@mskcc.org
- **Official Tempo docs**: <https://github.com/mskcc/tempo/tree/develop/docs>
