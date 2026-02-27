# Tempo Pipeline Container Images and Tool Mapping

> **Quick answer:** All container image assignments are defined in `conf/containers.config`. Tempo uses Docker images from Docker Hub (primarily under the `cmopipeline/` and `broadinstitute/` namespaces). On Juno, Singularity pulls and caches these Docker images automatically via `singularity.enabled = true` and `singularity.autoMounts = true`.

## Container Runtime by Profile

| Profile | Container Engine | How Images Are Loaded |
|---------|-----------------|----------------------|
| juno | Singularity 3.1.1 | Pulled from Docker Hub, converted to SIF, cached locally |
| docker | Docker | Pulled directly from Docker Hub |
| singularity | Singularity 3.1.1 | Pulled from Docker Hub, converted to SIF |
| awsbatch | Docker | Pulled on AWS Batch compute nodes |

On Juno, the `beforeScript` directive in both `conf/singularity.config` and `conf/juno.config` runs `module load singularity/3.1.1` before every process. Singularity is configured with `autoMounts = true` and `runOptions = "-B $TMPDIR"` to bind the temporary directory.

## Singularity Image Caching on Juno

When Singularity encounters a Docker image URI (e.g., `cmopipeline/facets-suite-preview-htstools:0.0.1`), it automatically pulls the image from Docker Hub, converts it to a `.sif` file, and caches it in the Nextflow work directory. To control where images are cached, set the `NXF_SINGULARITY_CACHEDIR` environment variable before launching the pipeline.

```bash
export NXF_SINGULARITY_CACHEDIR=/juno/work/tempo/singularity_cache
```

## Container-to-Process Mapping

### Read Alignment

| Process | Container Image |
|---------|----------------|
| AlignReads | `cmopipeline/fastp-bwa-samtools:2.0.0` |
| MergeBamsAndMarkDuplicates | `broadinstitute/gatk:4.1.9.0` |
| RunBQSR | `broadinstitute/gatk:4.1.9.0` |
| CreateScatteredIntervals | `broadinstitute/gatk:4.1.0.0` |

### Somatic Variant Calling

| Process | Container Image |
|---------|----------------|
| RunMutect2 | `broadinstitute/gatk:4.1.0.0` |
| SomaticCombineMutect2Vcf | `cmopipeline/bcftools-vt:1.2.0` |
| SomaticRunManta | `cmopipeline/strelka2-manta-bcftools-vt:2.0.1` |
| SomaticRunStrelka2 | `cmopipeline/strelka2-manta-bcftools-vt:2.0.1` |
| SomaticCombineChannel | `cmopipeline/bcftools-vt:1.2.3` |
| SomaticAnnotateMaf | `cmopipeline/vcf2maf:vep88_1.3.0` |
| SomaticDellyCall | `cmopipeline/delly-bcftools:0.0.1` |
| SomaticMergeSVs | `cmopipeline/bcftools-vt-mergesvvcf:0.0.1` |
| SomaticSVVcf2Bedpe | `cmopipeline/svtools:0.0.3` |
| SomaticAnnotateSVBedpe | `cmopipeline/iannotatesv:0.0.2` |
| RunSvABA | `cmopipeline/svaba:0.0.1` |

### Copy Number and MSI

| Process | Container Image |
|---------|----------------|
| DoFacets | `cmopipeline/facets-suite-preview-htstools:0.0.1` |
| DoFacetsPreviewQC | `cmopipeline/facets-suite-preview-htstools:0.0.1` |
| SomaticFacetsAnnotation | `cmopipeline/facets-suite-preview-htstools:0.0.1` |
| RunMsiSensor | `vanallenlab/msisensor:0.5` |

### HLA, Neoantigen, and Signatures

| Process | Container Image |
|---------|----------------|
| RunPolysolver | `sachet/polysolver:v4` |
| RunLOHHLA | `cmopipeline/lohhla:1.1.7` |
| RunMutationSignatures | `cmopipeline/temposig:0.2.3` |
| RunNeoantigen | `cmopipeline/neoantigen:0.3.3` |
| MetaDataParser | `cmopipeline/metadataparser:0.5.9` |
| HRDetect | `cmopipeline/signaturetoolslib:0.0.1` |

### Germline Variant Calling

| Process | Container Image |
|---------|----------------|
| GermlineRunHaplotypecaller | `broadinstitute/gatk:4.1.0.0` |
| GermlineCombineHaplotypecallerVcf | `cmopipeline/bcftools-vt:1.1.1` |
| GermlineRunManta | `cmopipeline/strelka2-manta-bcftools-vt:2.0.1` |
| GermlineRunStrelka2 | `cmopipeline/strelka2-manta-bcftools-vt:2.0.1` |
| GermlineAnnotateMaf | `cmopipeline/vcf2maf:vep88_1.2.7` |
| GermlineDellyCall | `cmopipeline/delly-bcftools:0.0.1` |

### Quality Control

| Process | Container Image |
|---------|----------------|
| QcPileup, QcConpair, QcConpairAll | `cmopipeline/conpair:v0.3.3` |
| QcAlfred | `cmopipeline/alfred:v0.1.17` |
| QcCollectHsMetrics | `broadinstitute/gatk:4.1.0.0` |
| QcQualimap | `cmopipeline/qualimap:0.0.1` |
| multiqc_process (label) | `cmopipeline/multiqc:0.1.3` |

### Cohort Aggregation

Most aggregation processes (`SomaticAggregateMaf`, `SomaticAggregateFacets`, `SomaticAggregateSv`, `SomaticAggregateMetadata`, `SomaticAggregateNetMHC`) use `cmopipeline/bcftools-vt:1.1.1`. Germline aggregation processes use `cmopipeline/bcftools-vt:1.2.0`. The `QcBamAggregate` process uses `cmopipeline/metadataparser:0.5.7`.

## Key Container Namespaces

- **cmopipeline/**: MSKCC Center for Molecular Oncology custom images (the majority of pipeline tools)
- **broadinstitute/**: Broad Institute GATK images (alignment, BQSR, Mutect2, HaplotypeCaller, CollectHsMetrics)
- **sachet/**: Polysolver HLA typing image
- **vanallenlab/**: MSIsensor image
- **quay.io/wtsicgp/**: PCAP-core and ascatNgs images for BRASS and ASCAT workflows

## File Locations

- Container assignments: `conf/containers.config`
- Docker runtime config: `conf/docker.config`
- Singularity runtime config: `conf/singularity.config`

## Example

```nextflow
// Override a container in a custom config
process {
  withName:DoFacets {
    container = "cmopipeline/facets-suite-preview-htstools:0.0.2"
  }
}
```

## See Also

- [config-profiles.md](config-profiles.md) -- Which profiles use Docker vs Singularity
- [config-resources.md](config-resources.md) -- CPU and memory allocated to each container process
