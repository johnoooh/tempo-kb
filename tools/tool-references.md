# Tool References & Citations

> **Quick answer:** Primary publication and official code/homepage for every tool in the Tempo pipeline, so you can read the source for any specific tool. References were verified against PubMed (June 2026). Where a tool has no peer-reviewed paper (MSKCC utilities, some Sanger tools) or only a preprint, that is stated explicitly — cite the repository in those cases. Always confirm the version you actually ran (see [tools-summary.md](tools-summary.md) for versions).

Links: PMID → PubMed; DOI → publisher. Code links are the official/maintained repositories.

## Alignment & preprocessing

| Tool | Primary reference | Code / homepage |
|------|-------------------|-----------------|
| fastp | Chen et al., *Bioinformatics* 2018 — [PMID 30423086](https://pubmed.ncbi.nlm.nih.gov/30423086/) · [doi:10.1093/bioinformatics/bty560](https://doi.org/10.1093/bioinformatics/bty560) | [OpenGene/fastp](https://github.com/OpenGene/fastp) |
| BWA / bwa mem | Li & Durbin, *Bioinformatics* 2009 — [PMID 19451168](https://pubmed.ncbi.nlm.nih.gov/19451168/). BWA-MEM: Li 2013, [arXiv:1303.3997](https://arxiv.org/abs/1303.3997) (preprint only) | [lh3/bwa](https://github.com/lh3/bwa) |
| samtools | Li et al., *Bioinformatics* 2009 — [PMID 19505943](https://pubmed.ncbi.nlm.nih.gov/19505943/) | [samtools/samtools](https://github.com/samtools/samtools) |
| GATK (framework) | McKenna et al., *Genome Res* 2010 — [PMID 20644199](https://pubmed.ncbi.nlm.nih.gov/20644199/); DePristo et al., *Nat Genet* 2011 — [PMID 21478889](https://pubmed.ncbi.nlm.nih.gov/21478889/) | [broadinstitute/gatk](https://github.com/broadinstitute/gatk) |
| GATK MarkDuplicates / CollectHsMetrics (Picard) | **Software-only** — no paper; cite the toolkit | [broadinstitute/picard](https://github.com/broadinstitute/picard) |
| GATK BaseRecalibrator (BQSR) | No standalone paper; cite DePristo et al. 2011 — [PMID 21478889](https://pubmed.ncbi.nlm.nih.gov/21478889/) | [broadinstitute/gatk](https://github.com/broadinstitute/gatk) |

## Quality control

| Tool | Primary reference | Code / homepage |
|------|-------------------|-----------------|
| Alfred | Rausch et al., *Bioinformatics* 2019 — [PMID 30520945](https://pubmed.ncbi.nlm.nih.gov/30520945/) | [tobiasrausch/alfred](https://github.com/tobiasrausch/alfred) |
| Qualimap | Okonechnikov et al., *Bioinformatics* 2016 — [PMID 26428292](https://pubmed.ncbi.nlm.nih.gov/26428292/) | [qualimap.conesalab.org](http://qualimap.conesalab.org) |
| Conpair | Bergmann et al., *Bioinformatics* 2016 — [PMID 27354699](https://pubmed.ncbi.nlm.nih.gov/27354699/) | [nygenome/Conpair](https://github.com/nygenome/Conpair) |
| MultiQC | Ewels et al., *Bioinformatics* 2016 — [PMID 27312411](https://pubmed.ncbi.nlm.nih.gov/27312411/) | [MultiQC/MultiQC](https://github.com/MultiQC/MultiQC) |

## Somatic & germline SNV/indel + annotation

| Tool | Primary reference | Code / homepage |
|------|-------------------|-----------------|
| GATK Mutect2 | Benjamin et al. 2019, [bioRxiv (preprint)](https://doi.org/10.1101/861054); predecessor MuTect: Cibulskis et al., *Nat Biotechnol* 2013 — [PMID 23396013](https://pubmed.ncbi.nlm.nih.gov/23396013/) | [broadinstitute/gatk](https://github.com/broadinstitute/gatk) |
| Strelka2 | Kim et al., *Nat Methods* 2018 — [PMID 30013048](https://pubmed.ncbi.nlm.nih.gov/30013048/) | [Illumina/strelka](https://github.com/Illumina/strelka) |
| GATK HaplotypeCaller | Poplin et al. 2018, [bioRxiv (preprint)](https://doi.org/10.1101/201178); cite DePristo et al. 2011 — [PMID 21478889](https://pubmed.ncbi.nlm.nih.gov/21478889/) | [broadinstitute/gatk](https://github.com/broadinstitute/gatk) |
| bcftools | Li, *Bioinformatics* 2011 — [PMID 21903627](https://pubmed.ncbi.nlm.nih.gov/21903627/) | [samtools/bcftools](https://github.com/samtools/bcftools) |
| vt | Tan et al., *Bioinformatics* 2015 — [PMID 25701572](https://pubmed.ncbi.nlm.nih.gov/25701572/) | [atks/vt](https://github.com/atks/vt) |
| GetBaseCountsMultiSample | **Code-only** — no paper | [mskcc/GetBaseCountsMultiSample](https://github.com/mskcc/GetBaseCountsMultiSample) |
| vcf2maf | **Code-only** — no paper; Zenodo DOI per release ([10.5281/zenodo.1185418](https://doi.org/10.5281/zenodo.1185418)) | [mskcc/vcf2maf](https://github.com/mskcc/vcf2maf) |
| Ensembl VEP | McLaren et al., *Genome Biol* 2016 — [PMID 27268795](https://pubmed.ncbi.nlm.nih.gov/27268795/) | [Ensembl VEP](https://www.ensembl.org/info/docs/tools/vep/index.html) |
| OncoKB Annotator | Chakravarty et al., *JCO Precis Oncol* 2017 — [PMID 28890946](https://pubmed.ncbi.nlm.nih.gov/28890946/) | [oncokb/oncokb-annotator](https://github.com/oncokb/oncokb-annotator) |

## Structural variants

| Tool | Primary reference | Code / homepage |
|------|-------------------|-----------------|
| Manta | Chen et al., *Bioinformatics* 2016 — [PMID 26647377](https://pubmed.ncbi.nlm.nih.gov/26647377/) · [doi:10.1093/bioinformatics/btv710](https://doi.org/10.1093/bioinformatics/btv710) | [Illumina/manta](https://github.com/Illumina/manta) |
| Delly | Rausch et al., *Bioinformatics* 2012 — [PMID 22962449](https://pubmed.ncbi.nlm.nih.gov/22962449/) | [dellytools/delly](https://github.com/dellytools/delly) |
| SvABA | Wala et al., *Genome Res* 2018 — [PMID 29535149](https://pubmed.ncbi.nlm.nih.gov/29535149/) | [walaj/svaba](https://github.com/walaj/svaba) |
| BRASS | **Code-only** — no standalone paper (Sanger CGP) | [cancerit/BRASS](https://github.com/cancerit/BRASS) |
| PCAP-core | **Code-only** — no standalone paper (Sanger CGP) | [cancerit/PCAP-core](https://github.com/cancerit/PCAP-core) |
| mergeSVvcf | **Code-only** — no paper; fork of `ljdursi/mergevcf` | [papaemmelab/mergeSVvcf](https://github.com/papaemmelab/mergeSVvcf) |
| svtools | Larson et al., *Bioinformatics* 2019 — [PMID 31218349](https://pubmed.ncbi.nlm.nih.gov/31218349/) | [hall-lab/svtools](https://github.com/hall-lab/svtools) |
| iAnnotateSV | **Code-only** — no paper; Zenodo DOI [10.5281/zenodo.18929](https://doi.org/10.5281/zenodo.18929) | [rhshah/iAnnotateSV](https://github.com/rhshah/iAnnotateSV) |
| ClusterSV | **Code-only**; method/application: Li et al. (PCAWG), *Nature* 2020 — [PMID 32025012](https://pubmed.ncbi.nlm.nih.gov/32025012/) · [doi:10.1038/s41586-019-1913-9](https://doi.org/10.1038/s41586-019-1913-9) | [cancerit/ClusterSV](https://github.com/cancerit/ClusterSV) |
| BioCircos | Cui et al., *Bioinformatics* 2016 — [PMID 26819473](https://pubmed.ncbi.nlm.nih.gov/26819473/) (BioCircos.js; the R wrapper has no separate paper) | [lvulliard/BioCircos.R](https://github.com/lvulliard/BioCircos.R) · CRAN: `BioCircos` |

> Note: `svtools` (hall-lab) and `mergeSVvcf` are unrelated tools despite similar naming — Tempo's caller-merge step uses `mergeSVvcf`.

## Copy number & clonality

| Tool | Primary reference | Code / homepage |
|------|-------------------|-----------------|
| FACETS | Shen & Seshan, *Nucleic Acids Res* 2016 — [PMID 27270079](https://pubmed.ncbi.nlm.nih.gov/27270079/) · [doi:10.1093/nar/gkw520](https://doi.org/10.1093/nar/gkw520) | [mskcc/facets](https://github.com/mskcc/facets) |
| facets-suite | **Code-only** — cite FACETS (parent method) | [mskcc/facets-suite](https://github.com/mskcc/facets-suite) |
| facetsPreview | **Code-only** — cite FACETS (parent method); repo is hyphenated | [mskcc/facets-preview](https://github.com/mskcc/facets-preview) |
| ASCAT | Van Loo et al., *PNAS* 2010 — [PMID 20837533](https://pubmed.ncbi.nlm.nih.gov/20837533/) · [doi:10.1073/pnas.1009843107](https://doi.org/10.1073/pnas.1009843107) | [VanLoo-lab/ascat](https://github.com/VanLoo-lab/ascat) |
| ascatNgs | Raine et al., *Curr Protoc Bioinformatics* 2016 — [PMID 27930809](https://pubmed.ncbi.nlm.nih.gov/27930809/) · [doi:10.1002/cpbi.17](https://doi.org/10.1002/cpbi.17) | [cancerit/ascatNgs](https://github.com/cancerit/ascatNgs) |
| SVclone | Cmero et al., *Nat Commun* 2020 — [PMID 32024845](https://pubmed.ncbi.nlm.nih.gov/32024845/) · [doi:10.1038/s41467-020-14351-8](https://doi.org/10.1038/s41467-020-14351-8) | [mcmero/SVclone](https://github.com/mcmero/SVclone) |

## HLA typing & neoantigen

| Tool | Primary reference | Code / homepage |
|------|-------------------|-----------------|
| POLYSOLVER | Shukla et al., *Nat Biotechnol* 2015 — [PMID 26372948](https://pubmed.ncbi.nlm.nih.gov/26372948/) · [doi:10.1038/nbt.3344](https://doi.org/10.1038/nbt.3344) | [Broad CGA](https://software.broadinstitute.org/cancer/cga/polysolver) · fork [jason-weirather/hla-polysolver](https://github.com/jason-weirather/hla-polysolver) |
| LOHHLA | McGranahan et al., *Cell* 2017 — [PMID 29107330](https://pubmed.ncbi.nlm.nih.gov/29107330/) · [doi:10.1016/j.cell.2017.10.001](https://doi.org/10.1016/j.cell.2017.10.001) | [bitbucket.org/mcgranahanlab/lohhla](https://bitbucket.org/mcgranahanlab/lohhla) · fork [mskcc/lohhla](https://github.com/mskcc/lohhla) |
| NetMHC (4.0) | Andreatta & Nielsen, *Bioinformatics* 2016 — [PMID 26515819](https://pubmed.ncbi.nlm.nih.gov/26515819/) | [DTU NetMHC-4.0](https://services.healthtech.dtu.dk/services/NetMHC-4.0/) |
| NetMHCpan (4.0) | Jurtz et al., *J Immunol* 2017 — [PMID 28978689](https://pubmed.ncbi.nlm.nih.gov/28978689/) · [doi:10.4049/jimmunol.1700893](https://doi.org/10.4049/jimmunol.1700893) | [DTU NetMHCpan](https://services.healthtech.dtu.dk/services/NetMHCpan-4.1/) |
| neoantigen wrapper (MSKCC) | **Code-only** — wraps NetMHC/NetMHCpan (cite those) | MSKCC `cmopipeline/neoantigen` container |

## Signatures & MSI

| Tool | Primary reference | Code / homepage |
|------|-------------------|-----------------|
| tempoSig | **Code-only** — no paper | [mskcc/tempoSig](https://github.com/mskcc/tempoSig) |
| MSIsensor | Niu et al., *Bioinformatics* 2014 — [PMID 24371154](https://pubmed.ncbi.nlm.nih.gov/24371154/) · [doi:10.1093/bioinformatics/btt755](https://doi.org/10.1093/bioinformatics/btt755) | [ding-lab/msisensor](https://github.com/ding-lab/msisensor) |
| signature.tools.lib | Library: Degasperi et al., *Nat Cancer* 2020 — [PMID 32118208](https://pubmed.ncbi.nlm.nih.gov/32118208/). **HRDetect**: Davies et al., *Nat Med* 2017 — [PMID 28288110](https://pubmed.ncbi.nlm.nih.gov/28288110/). FitMS/reference signatures: Degasperi et al., *Science* 2022 — [PMID 35949260](https://pubmed.ncbi.nlm.nih.gov/35949260/) | [Nik-Zainal-Group/signature.tools.lib](https://github.com/Nik-Zainal-Group/signature.tools.lib) |
| sigfit | Gori & Baez-Ortega 2018, [bioRxiv (preprint)](https://doi.org/10.1101/372896) — no PMID | [kgori/sigfit](https://github.com/kgori/sigfit) |

## Notes on accuracy

- PMIDs/DOIs were confirmed against PubMed records (June 2026). **Preprint-only** tools (BWA-MEM, Mutect2, HaplotypeCaller, sigfit) have no peer-reviewed journal paper — the preprint is the primary description.
- **Code-only** tools (Picard utilities, several MSKCC tools, BRASS, PCAP-core, mergeSVvcf, iAnnotateSV, facets-suite, facetsPreview, tempoSig) have no canonical paper — cite the repository (and the listed parent method where noted).
- For tools where Tempo uses a **specific version** (e.g. POLYSOLVER v4, NetMHC/NetMHCpan 4.0), the cited paper is the version-appropriate reference; newer server versions (e.g. NetMHCpan 4.1) have their own papers. See [tools-summary.md](tools-summary.md) for the exact versions Tempo runs.

## See Also

- [tools-summary.md](tools-summary.md) — every tool with version, stage, and purpose
- [../reference/calculations.md](../reference/calculations.md) — the formulas/thresholds these tools produce
- [../pipeline/overview.md](../pipeline/overview.md) — how the tools chain together
