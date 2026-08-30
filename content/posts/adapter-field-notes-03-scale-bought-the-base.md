---
title: "Scale Bought the Base"
date: 2026-08-30T19:50:00+08:00
draft: false
note: "16"
series: "Adapter Field Notes"
plate: "III / III"
description: "Qwen3-14B was a stronger base than 8B. After the same 50-step QLoRA, the three adapters sat on 108, 106, and 107 GSM8K problems."
summary: "8B versus 14B was a controlled comparison. 32B was a fit-on-40GB experiment. Neither scale won the adapter."
categories: ["Adapters"]
tags: ["qlora", "scaling", "gsm8k", "qwen3", "a100"]
---

[Plate I](/posts/adapter-field-notes-01-the-loss-is-not-the-task/) said loss is not the task. [Plate II](/posts/adapter-field-notes-02-the-mask-is-the-objective/) said the mask is the objective. The leftover superstition was size.

If a 50-step recipe helps 8B, the same recipe on 14B and 32B should help more. The bases did get stronger. The adapters did not separate.

## Plate III.1 — what actually changed

8B → 14B kept rank 64, context 4,096, 256 validation rows, and on-the-fly NF4 from the official BF16 weights. That pair is a controlled comparison.

14B → 32B is **not**. The official 32B repo tried to reconstruct seventeen shards and filled the Colab disk. The run that trained used a pre-quantized 4-bit base plus the official tokenizer (the quantized repo’s chat template was missing TRL generation markers). Rank dropped to 16, context to 2,048, validation to 64 rows.

{{< folio-card label="Scale card / 50-step math QLoRA" >}}

| | Qwen3-8B | Qwen3-14B | Qwen3-32B |
|---|---:|---:|---:|
| Parameters | 8.37B | 15.03B | 32.90B |
| LoRA trainable | 174.6M · 2.09% | 256.9M · 1.71% | 134.2M · 0.41% |
| Rank / context | 64 / 4,096 | 64 / 4,096 | 16 / 2,048 |
| 50-step train | 790 s | 1,196 s | 1,260 s |
| GSM8K base (128) | 53.125% | 59.375% | not measured |
| GSM8K adapter | **84.375% · 108** | 82.813% · 106 | 83.594% · 107 |

{{< /folio-card >}}

Adapter generation on the same 128 problems: 14B took 55 minutes (batch 4); 32B took 2.34 hours (batch 2). Those wall times are not a fair speed contest. They are a cost reminder.

## What scale did buy

Unadapted 14B beat unadapted 8B by eight GSM8K exact matches (76 vs 68). On tool calling, 14B base was already 124 / 128 sequence exact against 8B’s 121. Pretraining scale showed up **before** I attached LoRA.

After fifty identical QLoRA steps, the adapters sat on 108, 106, and 107. That spread is one or two problems on a 128-row slice. 8B is the nominal high score. A stronger base left less room for the same short recipe to move the number.

Tool calling under the corrected mask said the same thing in a smaller voice: 14B’s adapter tied sequence at 50 steps and picked up two argument matches. Two hundred further steps added one sequence match.

## What I will not claim about 32B

The 32B 50-step adapter is a feasibility result: the 40 GB A100 can go forward and backward at rank 16 and 2k context. It is not proof that 32B “ties 8B.”

A 200-step 32B math run **did** finish. Validation loss fell from 0.352 (50 steps) to 0.331. Immediately afterward, Colab rejected two fresh A100 allocations (`You may not have quota or entitlement for this accelerator`). There is no behavioral GSM8K number for that checkpoint. I am not interpolating one from the loss.

{{< folio-card label="Erratum / remaining debts" >}}

- No 32B base GSM8K on this slice.
- No 32B 200-step behavior score.
- No prompt-only arm. No DPO.
- Private adapters stay private. The public record is the protocol and the numbered evals, not a Hub URL.

{{< /folio-card >}}

## Prediction / result / correction

**Prediction.** Under a fixed QLoRA recipe, a larger Qwen3 is a better adapted model.

**Result.** Larger Qwen3 was a better **base**. The 50-step adapters did not rank by size. Stretching the recipe to 32B required weakening rank and context, then ran out of A100 quota before the interesting eval.

**Correction.** Scale is not a free multiplier on whoever happens to be running LoRA this week. It changes the starting point. Whether an adapter improves the starting point is a different measurement, and it needs a held-out behavior test the loss cannot sign.

> Scale bought the base. It did not buy the checkpoint.

## What this folio leaves behind

Three days, one A100, three task families:

```text
teacher-forced loss ≠ the exam
the label mask is the exam you actually set
scale moves the unadapted model first
```

The product I care about is not a lower `eval_loss` line. It is a frozen sample, a base number, an adapter number, and a written account of the runs that failed before step zero.

The unpaid debt is larger than another epoch. Serving numbers, a prompt-only control, and an evaluation set I would actually ship with an application are still ahead of more adapters.
