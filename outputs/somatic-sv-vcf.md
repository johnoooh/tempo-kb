# Somatic Structural Variant Calls: Multi-Caller SV Detection and BEDPE Output

> **Quick answer:** Tempo calls somatic structural variants with Delly, Manta, and SvABA (plus BRASS for WGS), merges them with `mergesvvcf`, converts the merged VCF to BEDPE format, and annotates with mappability, repeat masker, PCAWG blacklists, cDNA contamination detection, and iAnnotateSV gene annotations.

## SV Callers Used

Tempo runs multiple SV callers in parallel on each tumor-normal pair:

- **Delly** -- Calls five SV types (DEL, DUP, INV, BND, INS) using split reads and paired-end evidence. Each type is called separately, then combined with `DellyCombine`.
- **Manta** -- Identifies SVs and large indels using paired-end and split-read evidence. Run as part of the Strelka2 workflow and reused for SV merging.
- **SvABA** -- Performs local assembly around breakpoints for high-resolution SV detection. Accepts a targets BED file for exome runs.
- **BRASS** -- WGS only. Identifies rearrangements using read-pair clustering and provides additional breakpoint evidence. Requires FACETS or ASCAT sample statistics as input.

## Merging Pipeline

The SV workflow (`sv_wf`) follows a strict sequence:

**Step 1 -- Merge VCFs (`SomaticMergeSVs`).** All caller VCFs are merged using `mergesvvcf` with a 200bp window: two breakpoints within 200bp of each other with matching directionality are merged into one event. A custom script (`filter-sv-vcf.py`) filters events based on minimum supporting callers -- 2 callers required for genome, 1 for exome. Each caller's FILTER status determines whether it counts as a supporting caller; a caller that flagged the variant as non-PASS does not count as support.

**Step 2 -- Convert to BEDPE (`SomaticSVVcf2Bedpe`).** The merged VCF is converted to BEDPE format using `svtools vcftobedpe`. Sample names are normalized to TUMOR and NORMAL. The BEDPE is sorted with `svtools bedpesort`. Tumor and normal IDs are appended as additional columns.

**Step 3 -- Annotate BEDPE (`SomaticAnnotateSVBedpe`).** Six sequential annotation passes are applied:

1. **Mappability** (`mappability`) -- Flags breakends in ENCODE DAC low-mappability regions.
2. **RepeatMasker** (`repeat_masker`) -- Flags breakends in repetitive sequence regions.
3. **PCAWG blacklist BED** (`pcawg_blacklist_bed`) -- Flags breakends in PCAWG-identified problematic single regions.
4. **PCAWG blacklist BEDPE** (`pcawg_blacklist_bedpe`) -- Flags breakpoints matching PCAWG blacklisted pairs.
5. **PCAWG foldback BEDPE** (`pcawg_blacklist_fb_bedpe`) -- Flags likely foldback artifacts.
6. **PCAWG transposable element BEDPE** (`pcawg_blacklist_te_bedpe`) -- Flags likely transposable element artifacts.

After blacklist annotation, `detect_cdna.py` identifies possible cDNA contamination among deletions spanning splice sites (flagged but not filtered). Finally, `iAnnotateSV` adds gene-level annotations for both breakends.

The annotated file becomes the `unfiltered.bedpe`. The `final.bedpe` contains only rows where `FILTER=PASS`. Both BEDPE files include VCF-style meta-headers (starting with `##fileformat=BEDPEVCFv4.2`) with FILTER definitions and contig declarations.

## SV Types in Output

Each breakpoint is a single BEDPE record with coordinates and orientation of both breakends:

| Type | Abbreviation | Description |
|------|-------------|-------------|
| Breakend | BND | Events that cannot be classified as DEL/DUP/INV; typically translocations |
| Deletion | DEL | Loss of a segment between two joined breakends |
| Tandem Duplication | DUP | Extra copy immediately downstream in same orientation |
| Inversion | INV | Segment inserted in original position but reversed orientation |

## Read Support Filters

Before merging, Delly and Manta calls are subject to read-support filters:
- `tumor_read_supp`: Fewer than 5 discordant reads or fewer than 2 split reads in tumor
- `normal_read_supp`: Any read support in the normal sample

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/combined_svs/
  {idTumor}__{idNormal}.unfiltered.bedpe
  {idTumor}__{idNormal}.final.bedpe
  intermediate_files/
    {idTumor}__{idNormal}.merged.raw.vcf.gz
    {idTumor}__{idNormal}.merged.vcf.gz
    {idTumor}__{idNormal}.combined.bedpe

outDir/somatic/{idTumor}__{idNormal}/delly/       # Per-type VCFs (*_BND, *_DEL, *_DUP, *_INS, *_INV) + combined
outDir/somatic/{idTumor}__{idNormal}/manta/       # Individual Manta output
outDir/somatic/{idTumor}__{idNormal}/svaba/       # SvABA: filtered + unfiltered for both indels and SVs, plus reheadered SV VCF
outDir/somatic/{idTumor}__{idNormal}/brass/       # Individual BRASS output (WGS only)
```

## Example

```bash
# Count final somatic SVs
grep -v "^#" outDir/somatic/TUMOR1__NORMAL1/combined_svs/TUMOR1__NORMAL1.final.bedpe | wc -l

# View SV type distribution
grep -v "^#" TUMOR1__NORMAL1.final.bedpe | awk -F'\t' '{print $11}' | sort | uniq -c

# Extract translocations only
awk -F'\t' '$11 == "BND"' TUMOR1__NORMAL1.final.bedpe
```

## See Also

- `somatic-maf.md` -- Somatic SNV/indel MAF files
- `germline-maf.md` -- Germline SV calling uses same callers (minus BRASS)
- `somatic-facets.md` -- FACETS sample statistics feed into BRASS for WGS
