---
layout: post
title: "E5 timeseries EN: first test runs on P. evermanni and P. tuahiniensis"
date: 2026-11-04
categories: [E5-coral]
tags: [elastic-net, stability-selection, Peve, Ptua]
---

With 26.8 giving identical results on *A. pulchra*
([previous entry]({% post_url 2026-11-04-e5-en-step1-26.8-portability %})), I wanted a quick
end-to-end test on the other two species before starting runs on the full gene sets, which will take
hours.

**These are test runs.** The numbers below show that the pipeline works; they aren't results I'd
draw conclusions from yet.

## Setup

To keep the runs small and comparable, I used the same 50 ortholog genes as the *A. pulchra* test,
taking whichever ones exist in each species' biomineralization gene file: 46 for Peve and 35 for
Ptua. I didn't change any values, I just kept those rows.

Everything else matched the *A. pulchra* baseline: alpha 0.2, 10 stability-selection subsamples,
1 core, a 0.6 selection threshold, no variance filter, and no plots. I used the sample exclusions
from Kathleen's 26.6 commands. The methylation and gene files come from
`urol-e5/timeseries-molecular-calcification` (commit `b87f951`), and the miRNA and lncRNA files from
`urol-e5/timeseries_molecular` (commit `900110b5`).

<!-- TODO: add the 26.8 commit link -->

## Results

Both runs finished in about 10 minutes.

| | *A. pulchra* | *P. evermanni* | *P. tuahiniensis* |
|---|---|---|---|
| Samples × predictors | 39 × 29,829 | 31 × 18,435 | 32 × 23,616 |
| Genes tested | 48 | 42 | 35 |
| OOF R² > 0.5 | 14 | 11 | 10 |
| OOF R² < 0 | 10 | 14 | 9 |
| Median OOF R² | 0.26 | 0.26 | 0.38 |
| Stable predictor–gene pairs | 1,407 | 915 | 965 |
| …lncRNA / methylation / miRNA | 1,254 / 144 / 9 | 810 / 105 / 0 | 820 / 145 / 0 |

A few things stood out:

- **The predictor counts match Kathleen's 26.6 logs exactly** (18,435 for Peve, 23,616 for Ptua).
  That was reassuring, because it means I found the same input files she used.
- **The labelling fix in 26.8 made a real difference.** Under 26.7, all 19,570 Peve and 21,025 Ptua
  methylation features would have been labelled "Other".
- **No miRNA predictors were stable in Peve or Ptua.** There are only about 40–46 miRNAs per species,
  against thousands of lncRNA and methylation features.

## Same gene, different species

Because the response genes are ortholog groups, 30 of them were tested in all three species. I was
curious whether a gene that's easy to predict in one species is also easy to predict in the others.
Mostly, it isn't. The rank correlations of per-gene OOF R² were 0.03 (*A. pulchra* vs Peve), 0.25
(*A. pulchra* vs Ptua), and 0.46 (Peve vs Ptua). For example, OG_01121 was well predicted in Peve
and Ptua (0.75 and 0.79) but not at all in *A. pulchra* (−0.13).

I don't want to read much into this yet. With 10 colonies per species and only 10 subsamples, single
gene values are noisy, and the inner cross-validation folds are random. It's something to look at
again with the full runs.

## Caveats

- OOF R² tells us how well these regulatory layers *predict* a gene's expression in a colony the model
  hasn't seen. It doesn't show that a stable lncRNA or methylation feature *regulates* that gene;
  it's an association.
- With 10 fits per gene, selection probabilities only move in steps of 0.1, so the 0.6 threshold is
  coarse.
- About 9,500 (Peve) and 8,700 (Ptua) methylation features were dropped for having a missing value
  in at least one sample (see the previous entry).

## Next

Before the full runs, I want to check a few things with the lab:

1. Why 5 Peve samples were excluded even though they have data in every layer.
2. Whether *A. pulchra* should use `Apul-gene-methylation.tsv` (as in 26.6) or the `_75pct` version,
   to match the other two species.
3. Whether dropping every methylation feature with a missing value is intended.
4. Settings for the full runs: gene sets, number of subsamples, and whether to run on a lab server.
5. How they'd like the stability-selection and inner cross-validation issues handled.
