# HLA Typing and Loss of Heterozygosity Analysis (Polysolver and LOHHLA) in Tempo

> **Quick answer:** Tempo uses Polysolver to infer HLA class I genotypes (HLA-A, HLA-B, HLA-C) from the normal BAM, and LOHHLA to detect loss of heterozygosity at HLA loci in the tumor; however, LOHHLA is currently noted as temporarily disabled due to a known bug.

## Polysolver: HLA Typing

Polysolver (POLYmorphic loci reSOLVER) performs computational HLA class I typing from whole-exome or whole-genome sequencing data. It operates on the **normal** sample BAM only (not the tumor), because the normal genome represents the patient's germline HLA genotype without somatic alterations.

The `RunPolysolver` process calls the `shell_call_hla_type` script from the Polysolver container with the following parameters:

- Input BAM from the normal sample
- Race set to `Unknown` (population-agnostic mode)
- `includeFreq` set to 1 (include allele frequency priors)
- Genome build derived from `params.genome` (hg19 for GRCh37, hg38 for GRCh38)
- Data format set to STDFQ (standard FASTQ quality encoding)
- `insertCalc` set to 0

The output file `{idNormal}.hla.txt` is a three-row, three-column tab-delimited file. Each row corresponds to an HLA gene (HLA-A, HLA-B, HLA-C), and the two additional columns contain the two allele calls. The allele format uses underscores rather than colons (for example, `hla_a_02_01_01_01` instead of `HLA-A*02:01:01:01`).

## LOHHLA: HLA Loss of Heterozygosity

LOHHLA (Loss of Heterozygosity in Human Leukocyte Antigen) analyzes whether the tumor has lost one copy of an HLA allele. This is a mechanism of immune evasion: by losing an HLA allele, tumor cells can escape T-cell recognition of neoantigens presented by that allele.

The `RunLOHHLA` process requires:

- Tumor and normal BAMs
- Purity and ploidy estimates from FACETS
- HLA allele calls from Polysolver
- Reference HLA FASTA and DAT files

LOHHLA produces several output files including:
- **`{idTumor}__{idNormal}.DNA.HLAlossPrediction_CI.txt`**: Predictions of HLA allele loss with confidence intervals (39 columns including region, HLA types, log2 median coverage per allele, copy number estimates with/without BAF, confidence intervals, p-values, and loss/kept allele calls).
- **`{idTumor}__{idNormal}.DNA.IntegerCPN_CI.txt`**: Integer copy number estimates for each HLA allele with confidence intervals (24 columns: sample, missMatchseq1, logR_type1, TumorCov_type1, missMatchseq2, logR_type2, TumorCov_type2, NormalCov_type1, NormalCov_type2, logRcombined, BAFcombined, binlogRCombined, binlogRtype1, binlogRtype2, binNum, nAcombined, nBcombined, nAcombinedBin, nBcombinedBin, nAsep, nAsepBin, nBsep, nBsepBin, expectedBAF).
- **PDF plots**: Visualization of allele-specific coverage and copy number (`{idTumor}__{idNormal}.HLA.pdf`).
- **RData files**: Per-HLA-gene workspace dumps (`{idTumor}.hla_a.tmp.data.plots.RData`, `{idTumor}.hla_b.tmp.data.plots.RData`, `{idTumor}.hla_c.tmp.data.plots.RData`) containing ~200 R objects from the LOHHLA analysis including coverage data, copy number solutions, and diagnostic values. Note these files are named with the tumor ID only.

### Interpreting LOH Status

To determine whether an HLA allele has undergone LOH from the `HLAlossPrediction_CI.txt` file:

1. Check `PVal_unique` -- use < 0.05 as a standard threshold, or < 0.01 for higher confidence.
2. Check allele copy numbers: if `HLA_type1copyNum_withBAFBin` or `HLA_type2copyNum_withBAFBin` < 0.5, that allele is lost.
3. If **both** allele copy numbers are < 0.5, take the lower one as the lost allele.

```python
import pandas as pd

lohhla = pd.read_csv("TUMOR__NORMAL.DNA.HLAlossPrediction_CI.txt", sep="\t")
lohhla["LOH"] = (
    (lohhla["PVal_unique"] < 0.05) &
    ((lohhla["HLA_type1copyNum_withBAFBin"] < 0.5) |
     (lohhla["HLA_type2copyNum_withBAFBin"] < 0.5))
)
```

**Important caveat**: The Tempo documentation includes a warning that "LOHHLA is temporarily disabled due to a bug need future investigation. It will be enabled again in the future release." If LOHHLA fails, the process creates empty placeholder output files so downstream processes (metadata parser, aggregation) do not break.

## Integration with Other Processes

Polysolver HLA calls serve as input to two downstream analyses:
1. **Neoantigen prediction** (NetMHCpan) -- uses HLA alleles to predict which mutations generate neoantigens
2. **Metadata parser** -- incorporates HLA-A1, HLA-A2, HLA-B1, HLA-B2, HLA-C1, HLA-C2 into the sample metadata

## File Locations

```
# Polysolver HLA typing (in Nextflow work directory, not separately published)
work/<hash>/RunPolysolver/{idNormal}.hla.txt

# LOHHLA outputs
outDir/somatic/{idTumor}__{idNormal}/lohhla/{idTumor}__{idNormal}.DNA.HLAlossPrediction_CI.txt
outDir/somatic/{idTumor}__{idNormal}/lohhla/{idTumor}__{idNormal}.DNA.IntegerCPN_CI.txt
outDir/somatic/{idTumor}__{idNormal}/lohhla/{idTumor}__{idNormal}.HLA.pdf
```

## Example

```bash
# Polysolver HLA output format ({idNormal}.hla.txt):
# HLA-A  hla_a_02_01_01_01  hla_a_03_01_01_01
# HLA-B  hla_b_07_02_01     hla_b_44_02_01_01
# HLA-C  hla_c_07_02_01_03  hla_c_05_01_01_01
```

## See Also

- `somatic-neoantigen.md` -- Neoantigen prediction using Polysolver HLA alleles
- `metadata-parser.md` -- HLA genotypes in the consolidated metadata file
- `cohort-aggregates.md` -- Cohort-level LOHHLA files (`HLAlossPrediction_CI.txt`, `DNA.IntegerCPN_CI.txt`)
