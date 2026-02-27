# Structural Variant Callers in Tempo: Delly, Manta, SvABA, and BRASS

> **Quick answer:** Tempo uses three SV callers by default (Delly, Manta, SvABA) for both exome and genome data, plus BRASS for genome-only somatic analysis. Calls are merged using mergesvvcf with a 200bp window, requiring 2 supporting callers for genomes and 1 for exomes.

## Delly

**Algorithm:** Paired-end (PE) and split-read (SR) based detection. Delly identifies discordant read pairs that span breakpoints and refines breakpoint positions using split reads. It calls each SV type separately (DEL, DUP, INV, BND).

**Version in Tempo:** Delly v0.8.2 (EMBL.DELLYv0.8.2, confirmed from VCF headers).

**Key paper:** Rausch et al., 2012, Genome Research.

**Tempo implementation:** The `SomaticDellyCall` process runs `delly call` for each SV type with an exclude regions file, followed by `delly filter --filter somatic`. Additional soft filters are applied via bcftools:
- `tumor_read_supp`: Fewer than 5 discordant reads (DV) or fewer than 2 split reads (RV) in tumor.
- `normal_read_supp`: Any supporting reads in the normal sample.

## Manta

**Algorithm:** Graph-based SV detection and local assembly. Manta builds a breakend association graph from anomalous read pairs and split reads, then performs local assembly across candidate breakpoints to determine precise breakpoint sequences. It is particularly strong at detecting insertions and producing accurate breakpoint coordinates.

**Version in Tempo:** Manta/GenerateSVCandidates 1.5.0 (confirmed from VCF headers).

**Key paper:** Chen et al., 2016, Bioinformatics.

**Tempo implementation:** The `SomaticRunManta` process runs `configManta.py` followed by `runWorkflow.py`. For exome data, the `--exome` flag is added. Manta also produces `candidateSmallIndels.vcf.gz`, which is passed to Strelka2 to improve indel calling. The same soft filters as Delly are applied:
- `tumor_read_supp`: Fewer than 5 paired reads (PR) or fewer than 2 split reads (SR) in tumor.
- `normal_read_supp`: Any supporting reads in normal.

## SvABA

**Algorithm:** Local genome assembly using a string graph approach. SvABA performs local assembly of reads in regions with evidence of structural variation, identifying breakpoints from assembled contigs aligned back to the reference. This approach can detect variants that are difficult for PE/SR methods, including complex rearrangements and variants near repetitive regions.

**Version in Tempo:** SvABA (version from VCF source line shows the svaba command with `-p 16` threads and `--id-string` for pair naming).

**Key paper:** Wala et al., 2018, Genome Research.

**Tempo implementation:** The `SomaticRunSvABA` process runs `svaba run` with tumor and normal BAMs. For exome data, the `-k` flag restricts analysis to target regions. Germline calls are deleted. Sample names in the VCF header are corrected with `bcftools reheader`.

## BRASS (Genome Only)

**Algorithm:** Breakpoint assembly using reads spanning structural variant junctions. BRASS (BReakpoint AnalySiS) identifies rearrangements by grouping discordant read pairs, then performs local assembly to resolve breakpoint sequences. It requires ASCAT or FACETS sample statistics (purity, ploidy) for copy number-aware filtering.

**Tempo implementation:** BRASS is only available for WGS data. The workflow involves three steps:
1. `SomaticRunBRASSInput`: Prepares input data.
2. `SomaticRunBRASSCover`: Computes coverage profiles.
3. `runBRASS`: Runs the main BRASS algorithm with extensive reference files (high-depth regions, normal panel groups, viral sequences, bacterial sequences, cytoband data).

BRASS annotates results using a Vagrent cache for gene-level annotation.

## Merging Strategy

The `SomaticMergeSVs` process combines calls from all callers using `mergesvvcf`:
- Each call is normalized to a common breakpoint representation.
- Two breakpoints are merged if each breakend is within a **200bp window** and the relative strand orientation matches.
- A custom `filter-sv-vcf.py` script requires a minimum number of supporting callers:
  - **Genome:** 2 callers must support the variant (since 3-4 callers run).
  - **Exome:** 1 caller is sufficient (since 3 callers run but with lower SV sensitivity).

If a caller produced a filter flag for a variant, that caller is not counted as supporting the variant.

This merging approach follows the 2020 PCAWG publication on whole genomes (Nature, 2020).

## File Locations

- Delly: `tempo/modules/process/SV/SomaticDellyCall.nf`
- Manta: `tempo/modules/process/SV/SomaticRunManta.nf`
- SvABA: `tempo/modules/process/SV/SomaticRunSvABA.nf`
- BRASS: `tempo/modules/process/SV/BRASS/SomaticRunBRASS.nf`
- Merging: `tempo/modules/process/SV/SomaticMergeSVs.nf`
- SV workflow: `tempo/modules/subworkflow/sv_wf.nf`

## Example

```bash
# Individual caller outputs
<outDir>/somatic/TUMOR__NORMAL/delly/TUMOR__NORMAL_DEL.delly.vcf.gz
<outDir>/somatic/TUMOR__NORMAL/manta/TUMOR__NORMAL.manta.vcf.gz
<outDir>/somatic/TUMOR__NORMAL/svaba/TUMOR__NORMAL.reheader.svaba.somatic.sv.vcf.gz
<outDir>/somatic/TUMOR__NORMAL/brass/*.vcf.gz  # genome only

# Merged output
<outDir>/somatic/TUMOR__NORMAL/combined_svs/intermediate_files/TUMOR__NORMAL.merged.vcf.gz
```

## See Also

- `sv-format.md` -- SV types and BEDPE format details
- `copy-number-facets-vs-ascat.md` -- CNV source for BRASS sample statistics
- `svclone-clonality.md` -- Clonality analysis of merged SV calls
