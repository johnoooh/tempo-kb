# SNV/Indel Callers in Tempo (Mutect2, Strelka2, HaplotypeCaller)

> **Quick answer:** Tempo calls somatic SNVs/indels with both Mutect2 and Strelka2 and takes their union (Mutect2 precedence); germline SNVs/indels are called with HaplotypeCaller and Strelka2. Raw VCFs are normalized, merged, annotated (VEP/vcf2maf, FACETS, OncoKB), and filtered into final MAFs. This page covers the callers themselves; the merge/filter math is in [maf-filtering.md](maf-filtering.md).

## Somatic callers

### GATK Mutect2 (4.1.0.0) — `RunMutect2`
Local-reassembly somatic caller run on the tumor–normal pair, scattered over intervals. `FilterMutectCalls` applies Mutect2's own orientation-bias and contamination filters before the calls enter Tempo's combine step. Mutect2 is the **precedence caller** when Mutect2 and Strelka2 disagree on shared sites.

### Strelka2 (2.9.10) — `SomaticRunStrelka2`
Tumor–normal somatic SNV/indel caller. Tempo feeds Strelka2 **Manta's** small-indel candidates to improve indel sensitivity (Manta runs first in the SV workflow). Strelka2's per-site PASS status is tracked: a site both callers found but Strelka2 did not pass gets a `caller_conflict` flag downstream.

### Combine logic
Mutect2-PASS and Strelka2-PASS VCFs are intersected with `bcftools isec`:
- Mutect2-only → tagged `MuTect2`
- Strelka2-only → tagged `Strelka2`
- Shared → tagged with both (plus `Strelka2FILTER` if Strelka2 didn't pass it)

The union is normalized (`vt`), gets caller-independent counts added (`GetBaseCountsMultiSample`), is annotated by VEP/vcf2maf + FACETS + OncoKB, and filtered to the final MAF. Full thresholds: [maf-filtering.md](maf-filtering.md).

## Germline callers

### GATK HaplotypeCaller (4.1.0.0) — `GermlineRunHaplotypecaller`
Calls germline SNVs/indels on the normal. `SelectVariants` splits the output into SNP and INDEL subsets, each hard-filtered with standard GATK `VariantFiltration` expressions.

### Strelka2 germline mode (2.9.10) — `GermlineRunStrelka2`
Independent germline caller on the normal. HaplotypeCaller and Strelka2 germline calls are combined and filtered (see germline thresholds in [../reference/calculations.md](../reference/calculations.md)) into the germline MAF ([outputs/germline-maf.md](../outputs/germline-maf.md)).

> Germline filtering treats **clonal-hematopoiesis (CH) genes** specially (lower tumor-contamination tolerance) and applies a lower VAF floor to indels than SNVs — see [../reference/calculations.md](../reference/calculations.md).

## Supporting tools

| Tool | Role |
|------|------|
| bcftools (1.9) | concat/sort/normalize/intersect/merge VCFs |
| vt (0.57721) | indel left-normalization; adds `Ref_Tri` (trinucleotide context, used by signatures) |
| GetBaseCountsMultiSample (1.2.2) | re-counts reads at every site → `*_raw` MAF columns, caller-independent |
| vcf2maf (1.6.17) + VEP (88) | VCF→MAF and functional annotation |
| OncoKB Annotator (1.1.0) | oncogenicity/actionability (no Level 1/2A — cancer type unknown) |

## File Locations

- Somatic SNV workflow: `tempo/modules/subworkflow/snv_wf.nf`
- Processes: `tempo/modules/process/SNV/`, `tempo/modules/process/GermSNV/`

## See Also

- [maf-filtering.md](maf-filtering.md) — merge + filter thresholds → final somatic MAF
- [maf-format.md](maf-format.md) — MAF columns
- [../recipes/calculate-vaf.md](../recipes/calculate-vaf.md) — VAF from MAF counts
- [outputs/germline-maf.md](../outputs/germline-maf.md) — germline output
