# HLA Typing, Loss of Heterozygosity, and Neoantigen Prediction in Tempo

> **Quick answer:** Tempo uses Polysolver for Bayesian HLA class I typing from normal BAMs, LOHHLA to detect allele-specific HLA loss in tumors, and NetMHCpan to predict neoantigen binding affinity for somatic mutations. Together these tools characterize the immunogenomic landscape of a tumor.

## Polysolver: HLA Class I Typing

### What It Does

Polysolver (POLYmorphic loci reSOLVER) infers the patient's HLA class I genotype (HLA-A, HLA-B, HLA-C alleles) from the normal sample BAM file.

**Key paper:** Shukla et al., 2015, Nature Biotechnology.

### Algorithm

Polysolver uses a Bayesian framework to:

1. Extract reads mapping to the HLA region of the genome.
2. Align these reads against a comprehensive library of known HLA allele sequences.
3. Use a Bayesian classifier to determine the most likely pair of alleles for each HLA gene, accounting for read quality, mapping ambiguity, and allele frequency priors.

The algorithm reports two alleles per gene (one per chromosome), producing a 6-allele HLA type.

### Tempo Implementation

The `RunPolysolver` process takes the normal BAM as input and runs `shell_call_hla_type` with:
- Race: `Unknown` (uses uninformative prior).
- Include frequency information: enabled (`1`).
- Genome build: derived from `params.genome` (hg19 for GRCh37, hg38 for GRCh38).

The output `*.hla.txt` contains three lines (HLA-A, HLA-B, HLA-C), each with two allele calls in Polysolver's underscore-delimited format (e.g., `hla_a_01_01_01_01`).

## LOHHLA: HLA Loss of Heterozygosity

### What It Does

LOHHLA (Loss Of Heterozygosity in Human Leukocyte Antigen) detects allele-specific loss of HLA genes in the tumor, a mechanism of immune evasion where cancer cells lose one HLA allele to escape T-cell recognition.

**Key paper:** McGranahan et al., 2017, Cell.

### Algorithm

LOHHLA:

1. Takes the Polysolver HLA calls and extracts HLA allele-specific sequences.
2. Aligns tumor and normal BAM reads to these patient-specific HLA sequences.
3. Estimates allele-specific copy number using tumor purity and ploidy from FACETS.
4. Applies a statistical test to determine if one allele has significantly reduced copy number.

### Tempo Implementation

The `RunLOHHLA` process requires:
- Tumor and normal BAMs.
- Polysolver HLA output (`winners.hla.txt`).
- FACETS purity output (purity and ploidy values extracted from `*_purity.out`).
- HLA reference files (FASTA and DAT).

The `minCoverageFilter` parameter (default: 10) sets the minimum coverage threshold for reliable HLA copy number estimation.

### Output Files

- `*.DNA.HLAlossPrediction_CI.txt`: Per-allele loss predictions with confidence intervals (39 columns). Key columns include the HLA allele, copy number estimates, and p-values for LOH.
- `*.DNA.IntegerCPN_CI.txt`: Integer copy number estimates for each HLA allele (24 columns).
- `*.HLA.pdf`: Visualization of allele-specific coverage.
- `*.hla_{a,b,c}.tmp.data.plots.RData`: Per-HLA-gene R workspace dumps (named with tumor ID only).

### Interpreting LOH Calls

From `HLAlossPrediction_CI.txt`, determine LOH status as follows:

1. **p-value**: `PVal_unique` < 0.05 (or < 0.01 for stricter confidence).
2. **Copy number**: `HLA_type1copyNum_withBAFBin` or `HLA_type2copyNum_withBAFBin` < 0.5 indicates loss of that allele.
3. **Both low**: If both allele copy numbers are < 0.5, the allele with the lower value is the lost allele.

**Note:** LOHHLA output is optional -- if the analysis fails (e.g., due to insufficient coverage), Tempo creates empty placeholder files and continues.

## NetMHCpan: Neoantigen Prediction

### What It Does

NetMHCpan predicts the binding affinity of mutant peptides to the patient's HLA class I molecules. Strong binders are candidate neoantigens that may trigger immune recognition of the tumor.

### Algorithm

For each somatic mutation in the filtered MAF:

1. The mutant and wild-type peptide sequences are generated from the cDNA/CDS reference.
2. NetMHCpan predicts binding affinity (IC50 in nM) of each peptide to each of the patient's 6 HLA alleles.
3. Peptides with strong predicted binding (typically IC50 < 500 nM) are flagged as candidate neoantigens.

### Tempo Implementation

The `RunNeoantigen` process runs `neoantigen.py` with:
- The Polysolver HLA file.
- The filtered somatic MAF.
- Reference cDNA and CDS FASTA files.

The process is computationally intensive and has dynamic time allocation based on MAF file size (larger MAFs get more wall time).

### Output

- `*.all_neoantigen_predictions.txt`: All peptide-HLA binding predictions with affinity scores, organized by mutation.
- `*.neoantigens.maf`: The somatic MAF annotated with neoantigen prediction results, passed to downstream MAF annotation steps.

## File Locations

- Polysolver: `tempo/modules/process/LoH/RunPolysolver.nf`
- LOHHLA: `tempo/modules/process/LoH/RunLOHHLA.nf`
- Neoantigen: `tempo/modules/process/SNV/RunNeoantigen.nf`
- Polysolver output: `<outDir>/` (used internally, not directly published)
- LOHHLA output: `<outDir>/somatic/<tumor>__<normal>/lohhla/`
- Neoantigen output: `<outDir>/somatic/<tumor>__<normal>/neoantigen/`

## Example

```bash
# Polysolver HLA typing result
cat NORMAL.hla.txt
# HLA-A  hla_a_02_01_01_01  hla_a_03_01_01_01
# HLA-B  hla_b_07_02_01     hla_b_44_03_01_01
# HLA-C  hla_c_07_02_01_03  hla_c_16_01_01_01

# LOHHLA loss prediction
cat TUMOR__NORMAL.DNA.HLAlossPrediction_CI.txt
# sample  HLA_allele  pval_unique  copy_number  ...

# Neoantigen predictions
head TUMOR__NORMAL.all_neoantigen_predictions.txt
# sample  Hugo_Symbol  HLA_allele  peptide  IC50  ...
```

## See Also

- `facets-interpreting.md` -- FACETS purity/ploidy feeds into LOHHLA
- `maf-filtering.md` -- Filtered MAF is the neoantigen prediction input
- `maf-format.md` -- Neoantigen columns added to the annotated MAF
