# Delivery Timeline & Cohort Status

> **Quick answer:** A cohort is delivered automatically once every required pipeline step has completed for all its pairs. Until then it is "in progress," and if it stalls it is flagged as "delayed" with a reason. There is no fixed turnaround time — it depends on cohort size, sequencing, and pipeline queue. If your cohort seems stuck, contact your CMO Project Manager.

## When Delivery Happens

Delivery is **gated on completion**, not on a clock. The delivery system watches each cohort's pipeline trace and only delivers once a fixed set of cohort-level steps have finished — the eight `SomaticAggregate*` steps plus `QcConpairAggregate` and `QcBamAggregate` (the `cohortCompleteSteps` list). In addition, the cohort-wide MultiQC must have finished and the cohort request must not have been edited since it was queued. Only when all of these line up is access set and the [delivery email](delivery-email.md) sent.

A practical consequence: a cohort whose somatic analysis is done but whose **cohort-level MultiQC never completes** will not deliver — this is a common cause of a cohort that "looks done" but hasn't been delivered.

Because completion depends on cohort size, sample QC, sequencing turnaround, and cluster load, there is **no single expected duration**. Larger cohorts and cohorts with samples that need rework take longer.

## "Delayed" Cohorts

If a cohort has not progressed, the system may flag it as **delayed** and notify the team with:

- the **cohort ID** and **project title**, and
- a **cause** — the reason it has not progressed (for example, a sample missing from the repository, a failed pipeline step, or pending input).

The system sends an automatic delay reminder **once, after a cohort has been stalled for 7 days** with no delivery (it is not re-sent daily). So the absence of repeated reminders does not mean the cohort is progressing. A delay does not mean your cohort is lost — it means a blocker needs to be resolved.

## What You Can Do

- **Haven't received a delivery email and it's been a while?** Contact your CMO Project Manager (**skicmopm@mskcc.org**) — they can see the cohort's status on the internal tracker and identify any blocker.
- **Cohort flagged delayed?** The cause usually points to the fix (e.g. a sample needs to be added or re-run). Your PM and the CCS team coordinate the resolution.
- **Pipeline-step or QC failures?** Questions about the technical cause can go to the CCS Tempo team (**zzPDL_CMO_TEMPO_Support@mskcc.org**). See also [`troubleshooting/`](../troubleshooting/) for common pipeline failure modes.

## Why might a single sample be missing from my cohort?

If the cohort delivered but one sample is absent, it was most likely filtered out before or during analysis. Concrete reasons (from the delivery system's intake logic — `assessCohort.py`, `reconcileVoyager.py`, `flimsylimsy.py`):

- **Not yet in the Tempo repository** — the sample's CMO ID isn't in the repo mapping; it surfaces in a `sample_problem_report.txt` for your PM.
- **Pairing mismatch** — the requested tumor–normal pairing doesn't match the repo's pairing; the pair goes to a `pair_problem_report.txt`.
- **FASTQs not on a valid storage mount** (or none present) — samples whose FASTQs aren't on the accepted mounts (e.g. `/juno/cmo`, `/juno/archive`) are excluded; a tumor is also dropped if its matched normal was excluded for this reason.
- **IGO not complete** — samples without `igocomplete = true` or without FASTQs are skipped during mapping/pairing creation.
- **Manually failed QC** — a pair marked failing in the QC tracker (any reason other than "Passed Tempo Criteria") is held back.

Note: a **FACETS failure does not drop a sample** — such samples are still delivered (with a fallback purity of 0.3), so a missing sample is almost always one of the intake reasons above, not a FACETS issue. Because these checks happen on the operations side, **contact your CMO Project Manager** (skicmopm@mskcc.org) to find out which one applies and how to resolve it.

> An "access" issue is different from a "data" issue: if you simply *can't open* a sample's files, you may not be on the cohort's `endUsers` list — see [accessing-results.md](accessing-results.md#who-has-access-endusers). That's an access fix, not a missing sample.

## See Also

- [overview.md](overview.md) — the full delivery lifecycle
- [delivery-email.md](delivery-email.md) — what arrives when delivery completes
- [`troubleshooting/failed-processes.md`](../troubleshooting/failed-processes.md) — why a pipeline step might fail
