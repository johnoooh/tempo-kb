# Circos/BioCircos Genome Visualization in Tempo (Genome Only)

> **Quick answer:** For whole-genome data, Tempo generates an interactive circular genome plot (Circos-style) using BioCircos that combines structural variant and copy number data into a single HTML visualization for each tumor-normal pair.

## What the Circos Plot Shows

Circular genome plots (Circos plots) provide a compact visual summary of genome-wide structural alterations. The `SomaticRunSVCircos` process in Tempo generates an interactive HTML-based visualization using BioCircos (an R/JavaScript library) that displays:

- **Structural variants**: Arcs connecting chromosomal regions involved in translocations, inversions, deletions, and tandem duplications from the filtered somatic BEDPE file.
- **Copy number alterations**: Tracks showing gains and losses from the FACETS segmentation data overlaid on the circular chromosome ideogram.

This combined view allows quick visual assessment of the overall genomic landscape, making it easy to identify chromothripsis regions, recurrent breakpoint clusters, whole-arm copy number changes, and other large-scale structural patterns.

## How the Process Works

The `SomaticRunSVCircos` process takes two inputs per sample:

1. **Filtered somatic BEDPE**: The final structural variant calls
2. **Copy number segmentation**: From FACETS (either hisens or purity run, controlled by `params.svcnv`)

The process runs an R script (`biocircos_script`) that:

1. Reads the BEDPE and CNV files
2. Formats the data for the BioCircos R package
3. Renders an R Markdown document (`biocircos_Rmd`) that generates the interactive HTML output

The genome build is automatically set based on `params.genome` (GRCh38 maps to hg38, GRCh37 maps to hg19). The process has a guard condition that only runs for GRCh37 or GRCh38 reference genomes.

## Output File

The process produces a single interactive HTML file:

- **`{idTumor}__{idNormal}.circos.html`**: An interactive circular genome plot that can be opened in any web browser. The HTML file is self-contained with embedded JavaScript, so it can be shared and viewed without a server.

## File Locations

```
outDir/somatic/{idTumor}__{idNormal}/combined_svs/{idTumor}__{idNormal}.circos.html
```

Note that the Circos HTML is published to the `combined_svs/` subdirectory alongside the SV BEDPE files and SV signature outputs, since it visually represents SV and CNV data together.

## Example

```bash
# The output is a self-contained 1+ MB interactive HTML file
# HTML title: "SV and CNV Circos Plot"
open outDir/somatic/TUMOR__NORMAL/combined_svs/TUMOR__NORMAL.circos.html
```

## See Also

- `somatic-sv-vcf.md` -- The BEDPE structural variant calls visualized in the plot
- `somatic-facets.md` -- The copy number segmentation displayed as the CNV track
- `somatic-sv-signatures.md` -- SV signatures also published to the `combined_svs/` directory
