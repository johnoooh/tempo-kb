# Accessing Your Delivered Cohort Results

> **Quick answer:** Your results live in the WES repository under `cohort_level/<cohort_id>/` for the pipeline version your cohort was run on. You can reach them directly on MSK file storage or by mounting the SAMBA/SMB share from a PC or Mac. The **exact paths are printed in your delivery email** — use those, since the fileshare host and version change over time.

## Two Ways to Reach the Files

The delivery email gives you the precise location for your cohort. There are two access routes:

### 1. Direct file storage path

The email includes a direct path on MSK file storage, for example:

```
<fileshare_root>/Results/<version>/cohort_level/<cohort_id>
```

If you have shell access to the storage system (e.g. IRIS / the HPC), you can `cd` straight to this path. **Review the QC files before starting your analysis.**

### 2. SAMBA / SMB network share

Most investigators mount the WES repository as a network drive. The server and volume names are in your email; current values for the JUNO-backed share are server `hpc-share02.mskcc.org` and volume `juno_work_tempo_wesrepo`.

**On a PC:**
1. Open **Windows File Explorer**.
2. In the **Computer** tab, choose **Map Network Drive**.
3. Pick an available drive letter. Under **Folder**, enter the share path from your email, e.g. `\\hpc-share02.mskcc.org\juno_work_tempo_wesrepo`.
4. Click **Finish**. If it fails the first time, retry and check **"Connect using different credentials"**, then enter `mskcc\<your-MSK-username>` and your password.
5. Navigate to the delivery path from your email (`.../cohort_level/<cohort_id>`).

**On a Mac:**
1. Press **⌘ + K** in Finder (or **⌘ + Space** and type the address into Spotlight).
2. Enter the SMB URL from your email, e.g. `smb://hpc-share02.mskcc.org/juno_work_tempo_wesrepo/Results/<version>/cohort_level/<cohort_id>` and press **Enter**.
3. If prompted, enter your MSK username and password.
4. Alternatively, mount the volume, then double-click into it and navigate to `Results/<version>/cohort_level/<cohort_id>`.

> The delivery email may reference a different host (for example an IRIS share) depending on when and where your cohort was delivered. Always copy the host, volume, and path from your own email rather than hard-coding the values above.

## Who Has Access (endUsers)

Access is granted at delivery time to the MSK users listed in the **`endUsers`** field of the cohort request. This typically includes analysts, PIs, and investigators. **If someone is not listed as an endUser on the request, they will not have access when the cohort is delivered.** To add someone, contact your CMO Project Manager so the access list can be updated.

## What's in the Delivered Folder

Your cohort is delivered under `cohort_level/<cohort_id>/`, which holds the **aggregated** cross-sample results (combined MAFs, BEDPEs, segmentation files, copy-number tables, and a cohort-wide MultiQC report). For the full file list, see [`outputs/cohort-aggregates.md`](../outputs/cohort-aggregates.md).

Per-pair and per-sample results follow the standard Tempo layout (`somatic/{idTumor}__{idNormal}/`, `germline/{idNormal}/`, `bams/{sample_id}/`). See [`outputs/directory-structure.md`](../outputs/directory-structure.md) for the complete naming convention.

## See Also

- [delivery-email.md](delivery-email.md) — where these paths come from in the email
- [`outputs/directory-structure.md`](../outputs/directory-structure.md) — full output layout
- [`outputs/cohort-aggregates.md`](../outputs/cohort-aggregates.md) — what the cohort-level files contain
- [policies.md](policies.md) — what you may and may not do with the data
