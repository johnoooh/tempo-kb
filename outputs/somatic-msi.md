# Microsatellite Instability (MSI) Analysis with MSIsensor in Tempo

> **Quick answer:** Tempo runs MSIsensor on each tumor-normal pair to quantify microsatellite instability, producing a score that represents the percentage of unstable microsatellite sites in the tumor compared to the matched normal.

## What MSIsensor Measures

MSIsensor is a tool that detects microsatellite instability by comparing the distribution of microsatellite lengths between a tumor BAM and its matched normal BAM. Microsatellites are short tandem repeat sequences (1-6 bp motifs) scattered throughout the genome. In tumors with defective DNA mismatch repair (dMMR), these repeats accumulate insertions and deletions at a higher rate than normal tissue. The resulting phenotype, microsatellite instability-high (MSI-H), is clinically significant because it predicts response to immune checkpoint inhibitors and has prognostic implications in colorectal, endometrial, and other cancer types.

Tempo invokes MSIsensor with its `msi` subcommand, passing a pre-built list of microsatellite loci (`msiSensorList` from the reference configuration), the tumor BAM, and the normal BAM. The tool scans each listed microsatellite site, compares the repeat-length distributions between tumor and normal, and applies a chi-squared test to determine whether each site is somatically unstable.

## Output File and Columns

The process produces a single TSV file with the following columns:

- **Total_Number_of_Sites**: The total number of microsatellite loci examined from the reference list.
- **Number_of_Somatic_Sites**: The count of loci classified as somatically unstable (significantly different repeat-length distribution in the tumor).
- **%** (MSI score): The percentage of somatic sites relative to total sites. This is the primary MSI score.

The MSI score (the `%` column) is the key metric. A higher score indicates greater microsatellite instability.

## MSI-H Threshold

The standard threshold for classifying a sample as MSI-H varies by assay type and institution. Common thresholds in the literature are an MSI score of 3.5% for WES or 10% for WGS, but the exact cutoff used for your cohort should be confirmed with your bioinformatics team. For reference, a typical MSS exome sample shows an MSIscore around 0.03% (e.g., 3 somatic sites out of 11,300 total sites).

## Integration with Metadata

The MSIsensor output is not published to its own subdirectory in the final output tree. Instead, it is consumed directly by the `MetaDataParser` process, which extracts all three columns (`MSI_Total_Sites`, `MSI_Somatic_Sites`, `MSIscore`) and writes them into the consolidated per-pair metadata file. If you need the raw MSIsensor output, check the Nextflow work directory for the process `RunMsiSensor`.

## File Locations

```
# Raw MSIsensor output (in Nextflow work directory, not published separately)
work/<hash>/RunMsiSensor/{idTumor}__{idNormal}.msisensor.tsv

# MSI data incorporated into metadata
outDir/somatic/{idTumor}__{idNormal}/meta_data/{idTumor}__{idNormal}.sample_data.txt
```

## Example

```bash
# The MSIsensor command executed by the pipeline:
msisensor msi \
  -d /path/to/microsatellites.list \
  -t tumor.bam \
  -n normal.bam \
  -o sample_T__sample_N.msisensor.tsv

# Example output (sample_T__sample_N.msisensor.tsv):
# Total_Number_of_Sites  Number_of_Somatic_Sites  %
# 227856                 42                       0.018432
```

The MSI score of 0.018432 in this example would indicate a microsatellite-stable (MSS) tumor.

## See Also

- `metadata-parser.md` -- How MSI values are incorporated into the consolidated sample metadata
- `cohort-aggregates.md` -- MSI data in the cohort-level `sample_data.txt`
- `somatic-maf.md` -- Somatic mutations that may correlate with MSI status
