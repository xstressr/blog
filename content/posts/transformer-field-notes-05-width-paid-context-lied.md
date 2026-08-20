---
title: "Width Paid, Context Lied, Last Step Lost"
date: 2026-08-20T11:35:00+08:00
draft: false
note: "09"
series: "Transformer Field Notes"
plate: "V / VI"
description: "Three controlled knobs produced one useful winner, one confounded story, and a checkpoint that was better before training ended."
summary: "Tiny Shakespeare ablations for context, depth, and width—and the caveats that matter more than the leaderboard."
categories: ["Transformers"]
tags: ["ablation", "model-width", "model-depth", "context-window"]
---

Before each run, I wrote a prediction. That small ritual mattered more than I expected: it turned tuning from a search for pleasing numbers into a record of which intuition survived.

The baseline stayed fixed at seed `42`, batch `32`, context `128`, four layers, four heads, width `128`, and 2,000 steps. Then I changed one main variable at a time.

## Plate V.1 — context length

**Prediction.** More context should improve validation loss, while costing more memory and time.

| Context | Train tokens | Best val | Wall | Peak memory |
|---:|---:|---:|---:|---:|
| 64 | 4.10M | 1.833 | 27.8s | 127 MB |
| 128 | 8.19M | 1.782 | 27.2s | 205 MB |
| 256 | 16.4M | 1.749 | 29.9s | 497 MB |

The direction matched the prediction, but the explanation did not stay clean. At the same number of steps, doubling `T` also doubles the number of training tokens. Context `256` did not merely see farther; it saw four times as many token positions as context `64`.

**Correction.** This experiment says the larger-context configuration won under a same-step budget. It does not isolate context length as the cause.

## Plate V.2 — depth

**Prediction.** More layers should improve validation loss and cost substantially more time.

| Layers | Parameters | Best val | Wall |
|---:|---:|---:|---:|
| 2 | 0.42M | 1.916 | 18.3s |
| 4 | 0.82M | 1.782 | 27.2s |
| 6 | 1.21M | 1.734 | 38.2s |

The token budget stayed at 8.19M, so this attribution is cleaner. Depth helped and the wall time rose with it.

## Plate V.3 — width

**Prediction.** More width should help, but parameter count should grow quickly.

| Width | Parameters | Best val | Wall | Peak memory |
|---:|---:|---:|---:|---:|
| 64 | 0.21M | 2.036 | 27.9s | 177 MB |
| 128 | 0.82M | 1.782 | 27.2s | 205 MB |
| 256 | 3.21M | **1.624** | 28.4s | 420 MB |

On this A100 and this small workload, width produced the largest validation improvement while wall time stayed nearly flat. The cost appeared in parameter count and memory instead.

This is not a perfectly pure width knob either. I kept `n_head=4`, so increasing `n_embd` also increased each head's dimension from `16` to `32` to `64`.

That is a local result, not a scaling law. The honest sentence is: **width won among these settings, on this corpus, under this budget.**

{{< folio-card label="Coda / the best checkpoint was not the last" >}}

I extended the width-256 run from 2,000 to 5,000 steps.

```text
best validation    1.552 @ step 3,750
final train        1.202 @ step 5,000
final validation   1.563 @ step 5,000
```

Training loss kept falling after the best held-out result. Saving only the last checkpoint would have preserved a slightly worse model.

{{< /folio-card >}}

## Prediction / result / correction

**Prediction.** More context, depth, width, and training should all help in increasingly obvious ways.

**Result.** All four could improve the score, but each attached a different bill: extra tokens, time, parameters, memory, or overfitting.

**Correction.** “More” is not an explanation. A result earns meaning only after the budget and confound are printed beside it.

> The cleanest number in an experiment is often the caveat that prevents it from becoming a slogan.

## Unpaid debt

The teaching model is now measurable. The next temptation is to equate a larger training system with a different intelligence. [Plate VI: The Extra Code Is Not Another Brain](/posts/transformer-field-notes-06-extra-code-is-not-another-brain/) separates the next-token spine from the factory around it.
