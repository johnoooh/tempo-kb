# Somatic Mutational Signature Analysis with tempoSig in Tempo

> **Quick answer:** Tempo decomposes each sample's trinucleotide mutation spectrum into COSMIC mutational signatures using the tempoSig tool, supporting either the 30 COSMIC v2 signatures or the 60 COSMIC v3 SBS signatures depending on the `--cosmic` flag.

## How Mutational Signature Inference Works

Mutational signatures are characteristic patterns of somatic nucleotide substitutions that arise from specific mutagenic processes such as UV exposure, APOBEC activity, defective DNA repair, or aging. Tempo performs signature decomposition in two steps within the `RunMutationSignatures` process.

First, the `maf2cat2.R` script converts the somatic MAF file into a trinucleotide context matrix. This script extracts each single-nucleotide variant along with its 5-prime and 3-prime flanking bases to classify it into one of 96 trinucleotide substitution categories (6 substitution types times 16 trinucleotide contexts).

Second, `tempoSig.R` fits the observed trinucleotide spectrum against the reference COSMIC signature matrix. The tool uses non-negative least squares to estimate the exposure (contribution) of each reference signature to the observed mutation catalog. It also computes empirical p-values for each signature via permutation testing (`--nperm 10000`), indicating whether the signature contribution is statistically significant above background noise. The random seed is fixed at 132 for reproducibility.

## The --cosmic Flag

The `--cosmic` parameter controls which reference signature set is used:

- **`--cosmic v2`**: Fits against the 30 COSMIC v2 signatures (SBS1 through SBS30). These are the original Alexandrov 2013 signatures.
- **`--cosmic v3`**: Fits against the 60 COSMIC v3 SBS signatures. These provide finer resolution, splitting several v2 signatures into distinct processes.

The default in Tempo is `v3` (set in `nextflow.config` as `cosmic = 'v3'`). The pipeline validates that the value is either `v2` or `v3` and exits with an error if an invalid value is provided.

## Output File Format

The output file `{idTumor}__{idNormal}.mutsig.txt` is a tab-delimited text file. Each row represents one sample. Columns include the sample identifier and one column per reference signature containing the estimated exposure (proportion of mutations attributed to that signature). When the `--pvalue` flag is used, additional columns with p-values for each signature are included.

## Integration with Metadata

Like the MSIsensor output, the mutational signatures file is not published to its own subdirectory. It is passed directly to the `MetaDataParser` process, which merges all signature columns into the consolidated sample metadata file. The Tempo documentation states that "All 60 Mutational Signatures" are included in the metadata (when using v3).

## File Locations

```
# Raw tempoSig output (in Nextflow work directory, not published separately)
work/<hash>/RunMutationSignatures/{idTumor}__{idNormal}.mutsig.txt

# Intermediate trinucleotide matrix
work/<hash>/RunMutationSignatures/{idTumor}__{idNormal}.trinucmat.txt

# Signature data incorporated into metadata
outDir/somatic/{idTumor}__{idNormal}/meta_data/{idTumor}__{idNormal}.sample_data.txt
```

## Example

```bash
# Pipeline command (executed internally):
maf2cat2.R sample_T__sample_N.somatic.maf sample_T__sample_N.trinucmat.txt

tempoSig.R --cosmic_v3 --pvalue --nperm 10000 --seed 132 \
  sample_T__sample_N.trinucmat.txt \
  sample_T__sample_N.mutsig.txt

# To switch to COSMIC v2 signatures, run the pipeline with:
nextflow run tempo --cosmic v2 ...
```

## See Also

- `somatic-sv-signatures.md` -- Structural variant signatures (genome-only)
- `metadata-parser.md` -- How signature values appear in the metadata file
- `somatic-maf.md` -- The somatic MAF that serves as input to signature analysis
- `somatic-hrdetect.md` -- HRDetect uses SNV signatures (specifically SBS3 and SBS8) as features
