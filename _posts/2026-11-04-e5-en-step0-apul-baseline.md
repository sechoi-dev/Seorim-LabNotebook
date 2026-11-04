---
layout: post
title: "E5 timeseries EN: reproducing the A. pulchra baseline with 26.7"
date: 2026-11-04
categories: [E5-coral]
tags: [elastic-net, stability-selection, Apul, reproducibility]
---

My assignment is to get the elastic net + stability selection pipeline
(`M-multi-species/scripts/26.7-ElasticNet-stabsel-varfilter-OOF.R`) working on the two coral species
other than *Acropora pulchra*. Before touching any code, I wanted to run the original script on
*A. pulchra* exactly as it was last documented. That way I'd know my setup works, and I'd have a
reference result to check every later change against.

## What the pipeline does (in short)

For each gene, it tries to predict the gene's expression from three "regulatory" layers measured in
the same samples: miRNA counts, lncRNA counts, and gene-body DNA methylation. It does this with
elastic net regression, then asks two questions:

1. **How well can we predict this gene in a colony the model has never seen?** Version 26.7 answers
   this with leave-one-colony-out cross-validation: train on 9 colonies, predict the 10th, repeat.
   This gives an out-of-fold (OOF) R².
2. **Which predictors keep getting picked?** Stability selection refits the model on many random
   subsamples and records how often each predictor ends up with a nonzero coefficient. Predictors
   picked in at least 60% of fits are called "stable".

## Setup

I'm running this on Windows 11 with R 4.6.2 (glmnet 5.2, DESeq2 1.52.0). The script loads a lot of
packages it never actually uses (igraph, caret, vegan, and others), so I had to install those just
to get it to start. Only `pheatmap` is really needed, and only when plotting is turned on.

The repository has some filenames Windows can't handle, so I'm using a sparse checkout and pulled
the input files straight out of git with `git show`, checking that each file's hash matched.

I copied the command and settings from the
[2026-06-16 notebook entry](https://shedurkin.github.io/Roberts-LabNotebook/posts/projects/E5_coral/2026_06_16_timeseries_EN_OOF.html):

| Layer | File (in `urol-e5/timeseries_molecular`) |
|---|---|
| Response genes (50 biomineralization genes) | `M-multi-species/output/26.5-ElasticNet-permutation-sig/SUBSET50_apul_biomin_counts.csv` |
| miRNA | `M-multi-species/output/10.1-format-miRNA-counts-updated/Apul_miRNA_counts_formatted_updated.txt` |
| lncRNA | `D-Apul/output/31.5-Apul-lncRNA-discovery/lncRNA_counts.clean.filtered.txt` |
| Gene methylation | `D-Apul/output/40-Apul-Gene-Methylation/Apul-gene-methylation.tsv` |
| Metadata | `M-multi-species/data/rna_metadata.csv` |

```
Rscript 26.7-ElasticNet-stabsel-varfilter-OOF.R \
  <genes> <miRNA> <lncRNA> <methylation> ../../M-multi-species/data/rna_metadata.csv <output_dir> \
  ACR-225-TP1 \   # excluded sample
  0.2 \           # alpha
  10 \            # stability-selection subsamples
  1 \             # cores
  0.6 \           # selection threshold
  0 \             # variance filter (off)
  0               # plotting (off)
```

Script version: commit `900110b5` of `timeseries_molecular`. The seeds are fixed inside the script.

The one thing I changed from the documented run was using **1 core instead of 4**. The script
parallelises with `mclapply`, which relies on forking, and that doesn't work on Windows.

I wondered why ACR-225-TP1 is excluded, so I checked. Only 539 genes have any methylation signal in
that sample, compared with about 22,000 in a typical sample, so it looks like a failed methylation
library.

## Results

After filtering, the model had 39 samples from 10 colonies, 48 genes, and 29,829 predictors. It took
about 13 minutes on one core.

- **14 of 48 genes** had OOF R² above 0.5. The median was 0.26, and 10 genes scored below zero,
  meaning the model did worse than just predicting the average.
- **1,407 stable predictor–gene pairs** across 45 genes: 1,254 lncRNA, 144 gene-body methylation,
  and 9 miRNA.

A side note I found useful: before the real analysis, the script runs a quick test fit on one gene,
and that reported R² = 0.98. That model was fit and scored on the same samples, though. With about
30,000 predictors and only 39 samples, a model can fit the training data almost perfectly, which is
exactly why the held-out (OOF) numbers are the ones that matter. The same gene's OOF R² was 0.69.

## Comparing with the June entry

The headline number matched: 14 of 48 genes above 0.5. The individual gene values didn't. For
example, OG_00843 was 0.39 in June and 0.53 here, and the stable methylation count was 154 then
vs 144 now.

At first I thought this was randomness, but the git history explains it. All three committed
versions of 26.7 are dated **June 23**, a week *after* the June 16 entry, and one of them
("fix scaling") changed how the data are standardized. So the June numbers came from an earlier,
uncommitted version of the script, and they can't be reproduced exactly. From here on, this run
(commit `900110b5`) is my reference.

## Things I noticed

- The log says each stability-selection iteration produces "2 complementary-pair fits (20 total)",
  but the output shows **10 fits per gene**. The code takes one 80% subsample per iteration, not the
  paired half-samples described in the comments. With only 10 fits, selection probabilities can
  only be 0.6, 0.7, … 1.0.
- R crashes with a Windows error code *after* the script finishes and writes everything. The
  outputs are complete, but I still want to find the cause.

## Next

Make a new version of the script that can handle the other two species, and check that it gives
identical results on *A. pulchra*.
