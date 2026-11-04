---
layout: post
title: "E5 timeseries EN: reproducing the 26.7 A. pulchra baseline (Step 0)"
date: 2026-11-04
categories: [E5-coral]
---

## Goal

Before changing 26.7 to support *P. evermanni* and *P. tuahiniensis*, run the **unmodified** script on
*A. pulchra* to (1) confirm the local setup works and (2) produce a reference result that later code
changes must reproduce exactly.

## Setup

- Script: `M-multi-species/scripts/26.7-ElasticNet-stabsel-varfilter-OOF.R` at commit `900110b5`
  (fork `sechoi-dev/timeseries_molecular`, branch `generalize-elastic-net`, identical to `main`).
- R 4.6.2 (Windows 11), glmnet 5.2, DESeq2 1.52.0. Missing packages installed (igraph, psych,
  tidygraph, ggraph, reshape2, ggcorrplot, corrplot, pheatmap, caret, factoextra, vegan, ggfortify,
  pbapply, edgeR, genefilter); only pheatmap is actually used by the script (and only when plotting).
- Inputs exported byte-for-byte from git at `900110b5` (blob hashes verified):

| Layer | Path in repo |
|---|---|
| Response (50 biomin genes, OG IDs) | `M-multi-species/output/26.5-ElasticNet-permutation-sig/SUBSET50_apul_biomin_counts.csv` |
| miRNA | `M-multi-species/output/10.1-format-miRNA-counts-updated/Apul_miRNA_counts_formatted_updated.txt` |
| lncRNA | `D-Apul/output/31.5-Apul-lncRNA-discovery/lncRNA_counts.clean.filtered.txt` |
| Gene methylation | `D-Apul/output/40-Apul-Gene-Methylation/Apul-gene-methylation.tsv` |
| Metadata | `M-multi-species/data/rna_metadata.csv` (hardcoded in script) |

## Command

PowerShell reproduction command, using the repository-relative input paths in the table
above. Start in the repository root, then change to the scripts directory so 26.7's
hardcoded metadata path resolves correctly. This is a corrected runnable equivalent of
the recorded settings, not a transcript of the original invocation. If inputs were
exported elsewhere locally, replace the first four paths with their actual locations.
The output directory below is a new reproduction destination.

```powershell
Set-Location "M-multi-species/scripts"

$runArgs = @(
    "26.7-ElasticNet-stabsel-varfilter-OOF.R"
    "../output/26.5-ElasticNet-permutation-sig/SUBSET50_apul_biomin_counts.csv"
    "../output/10.1-format-miRNA-counts-updated/Apul_miRNA_counts_formatted_updated.txt"
    "../../D-Apul/output/31.5-Apul-lncRNA-discovery/lncRNA_counts.clean.filtered.txt"
    "../../D-Apul/output/40-Apul-Gene-Methylation/Apul-gene-methylation.tsv"
    "../../M-multi-species/data/rna_metadata.csv"
    "../../E5_runs/apul_26.7_reproduction"
    "ACR-225-TP1"  # excluded sample
    "0.2"          # alpha
    "10"           # n_subsamples
    "1"            # n_cores; Windows does not support mclapply with >1 core
    "0.6"          # pi_thr
    "0"            # var_filter_n (off)
    "0"            # plot_enabled (off)
)

& Rscript @runArgs
```

`Rscript` must be available in the current terminal's PATH. Sparse-checkout users
must also materialize the input files listed above before running this command.

Seeds are fixed inside the script (global 703, OOF 5703, stability selection 2703 + per-gene offsets).

Why ACR-225-TP1 is excluded: only 539 genes have any methylation signal in that sample vs ~22,000
in a typical sample, consistent with a failed WGBS library.

## Results

- After filtering: 39 samples (10 colonies), 48 response genes, 29,829 predictors.
- Leave-one-colony-out OOF R²: **14 / 48 genes > 0.5**; median 0.257; 10 genes < 0 (worse than predicting the mean).
- Stable predictors (selection prob ≥ 0.6): 1,407 associations across 45 genes —
  lncRNA 1,254 (45 genes), gene-body methylation 144 (37 genes), miRNA 9 (7 genes).
- Runtime on 1 core: 6.6 min OOF + 6.6 min stability selection = 13.2 min.

### Comparison with the 2026-06-16 entry

The count of genes with OOF R² > 0.5 matches (14/48), but per-gene values and stable-predictor counts
differ (e.g. OG_00843: 0.391 then vs 0.530 now; stable GbM 154 vs 144). This is expected: all three
committed versions of 26.7 are dated 2026-06-23, *after* that entry, and one of them (`82f25edd`,
"fix scaling") changed z-scaling from per-sample to per-feature/per-gene. The 06-16 numbers come from
an earlier, uncommitted version, so this run (commit `900110b5`) becomes the reference baseline.

## Issues noticed

- Log says each stability-selection iteration "produces 2 fits (20 total fits per gene)", but the
  output's `N_Total_Fits` is 10: the code takes one 80% subsample per iteration, not complementary
  half-sample pairs. With 10 fits, selection probabilities occur in increments of 0.1; stable associations
  at the 0.6 threshold can have probabilities 0.6, 0.7, … 1.0.
- R exits with a Windows crash code (0xC00000FF) *after* writing all outputs and "END OF SCRIPT";
  results are complete, but the cause still needs checking.

## Next steps

- Make 26.8 (copy of 26.7) with portability fixes; re-run Apul and confirm identical output.
- Locate the Peve/Ptua gene-level methylation and response count files.

Update (Step 1): these inputs were located publicly in the separate
[`timeseries-molecular-calcification` repository](https://github.com/urol-e5/timeseries-molecular-calcification).
Kathleen's [March 4 notebook entry](https://shedurkin.github.io/Roberts-LabNotebook/posts/projects/E5_coral/2026_03_04_timeseries_EN_stability_selection.html)
records the earlier run commands and sample exclusions. See the Step 1 entry for details.
