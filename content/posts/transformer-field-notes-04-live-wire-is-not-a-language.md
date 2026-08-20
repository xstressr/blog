---
title: "A Live Wire Is Not a Language"
date: 2026-08-20T11:34:00+08:00
draft: false
note: "08"
series: "Transformer Field Notes"
plate: "IV / VI"
description: "The model could memorize one batch, lower validation loss, restore a checkpoint, and imitate a play—without learning honest English."
summary: "The complete Tiny Shakespeare baseline as an instrument: diagnostics, curve, checkpoint, and one revealing broken specimen."
categories: ["Transformers"]
tags: ["tiny-shakespeare", "experiment", "checkpoint", "overfitting"]
---

This was the first output that felt exciting (excerpted from one longer sample):

```text
ROMEO:
Fit that that please the me distred he besty:

[...]

First ORIOLANENCE:
He wilt we as gace thee? worther thee more mine

[...]

QUEEN SLIZABETH:
No, I shall I can god and his doubtager
```

It had character names, colons, line breaks, and the visual rhythm of a play. It also did not know English.

That combination made the sample useful. The model learned format before spelling, and a plausible silhouette before reliable language.

## Plate IV.1 — the complete desk card

{{< folio-card label="Experiment 01 / Tiny Shakespeare baseline" >}}

| Measure | Reading |
|---|---:|
| Hardware | NVIDIA A100-SXM4-40GB |
| Seed / dropout | 42 / 0.0 |
| Model | 4 layers, 4 heads, width 128 |
| Context / batch | 128 / 32 |
| Parameters | 818,048 |
| Budget | 2,000 steps · 8.19M tokens |
| Wall time | 27.2s, including periodic evaluation |
| One-batch diagnostic | `4.2077 → 0.0104` |
| Validation loss | `4.1982 → 1.7817` |
| Checkpoint | logits matched after restore |

{{< /folio-card >}}

## Three tests, three different claims

### 1. Can it memorize one batch?

I trained a smaller diagnostic model on the same fixed batch for 201 updates. Loss fell from `4.2077` to `0.0104`.

That proves the forward pass, loss, backward pass, and optimizer can cooperate. It does not prove generalization. A system that cannot memorize a tiny batch is broken; a system that can is merely eligible for the next test.

### 2. Does validation loss keep improving?

The baseline moved steadily:

| Step | Train | Validation |
|---:|---:|---:|
| 0 | 4.2003 | 4.1982 |
| 400 | 2.3069 | 2.3175 |
| 1,000 | 1.9347 | 2.0130 |
| 1,600 | 1.6982 | 1.8422 |
| 2,000 | 1.6089 | 1.7817 |

The train/validation gap widened, but validation was still at its minimum when the budget ended. This run had not yet shown the clear rebound I saw in a different classroom configuration. Those scores cannot be compared as if the models, context, and training budget were identical.

### 3. Can the checkpoint reproduce the model?

I loaded a fresh model and optimizer from the saved checkpoint, fed the same prompt, and used `torch.testing.assert_close` on the logits. It passed. The file stored step `2000` and 52 optimizer-state entries.

That verifies loading. It is not yet seamless resume: an exact training continuation would also need RNG state and the data-loader position.

## Prediction / result / correction

**Prediction.** If loss falls and the sample resembles Shakespeare, the model has learned language.

**Result.** The diagnostics proved progressively more specific things: the graph is live, held-out prediction improved, state can be restored, and local form emerged before spelling.

**Correction.** No single green check owns the word “learned.” Evidence has to keep its scope.

> `QUEEN SLIZABETH` is not a failure to hide. It is the cleanest specimen of what the model learned first.

## Unpaid debt

The baseline gives me an instrument. Now I can turn one knob at a time and see what moves. [Plate V: Width Paid, Context Lied, Last Step Lost](/posts/transformer-field-notes-05-width-paid-context-lied/) is the experiment that changed my intuition most.
