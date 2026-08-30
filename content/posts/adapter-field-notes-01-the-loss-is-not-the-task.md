---
title: "The Loss Is Not the Task"
date: 2026-08-30T19:30:00+08:00
draft: false
note: "14"
series: "Adapter Field Notes"
plate: "I / III"
description: "Teacher-forced validation loss fell at every checkpoint I trusted. The behaviors I actually cared about got worse."
summary: "A 50-step math adapter beat a 600-step one, and ViT-Large was best after one epoch—on a single Colab A100."
categories: ["Adapters"]
tags: ["qlora", "sft", "gsm8k", "evaluation", "a100"]
---

The Transformer folio ended with a warning I had only read: a base model continues documents; chat behavior arrives later. I then spent three days on one NVIDIA A100-SXM4-40GB doing the “later” part.

The prediction I carried in was simple. If held-out token loss keeps falling, the checkpoint is getting better. Pick the last one.

That rule selected the wrong model three times.

## Plate I.1 — the desk

The job was not to drain a Colab credit balance. It was to leave adapters (and one vision checkpoint) that could be compared to their bases on a frozen eval.

{{< folio-card label="Lab card / 2026-08-23 to 2026-08-25" >}}

| | |
|---|---|
| GPU | NVIDIA A100-SXM4-40GB, ~40 GB |
| Method | NF4 QLoRA, BF16, packing **off** |
| Math data | `open-r1/OpenR1-Math-220k` (verified traces) |
| Math eval | GSM8K test, seed `20260823`, **128** rows, greedy |
| Vision | full fine-tune of ViT on Imagenette `320px` |
| Persistence | private Hub adapters; VM torn down without `--keep` |

{{< /folio-card >}}

Packing stayed off because the TRL stack I had warned that packed samples could contaminate each other. Throughput lost. Boundaries kept.

GSM8K scoring extracted the last numeric token and compared `Decimal` values exactly. Every adapter in this plate parsed at 100%. The gaps are not parser failures.

## The math adapter that should have been worse

Qwen3-8B, rank-64 LoRA on all linear layers, 4,096 context.

| Checkpoint | Val token loss | GSM8K exact (128) |
|---|---:|---:|
| Base | — | 68 / 128 · 53.125% |
| 50 steps | 0.351 | **108 / 128 · 84.375%** |
| 600 steps | **0.341** | 101 / 128 · 78.906% |

From step 100 to 600 the held-out token loss improved at every checkpoint. Token accuracy crept up. The 600-step run processed about 16.8M tokens and spent two and a half hours in the trainer.

On the same 128 word problems it was seven exact matches worse than the 50-step pilot. Eleven formerly correct answers flipped wrong; four formerly wrong answers flipped right. The adapter also wrote longer traces: the eval wall time rose from 17 minutes (base) to 30 minutes.

Qwen3-14B repeated the shape. Fifty steps: 106 / 128. Two hundred steps, lower validation loss: 100 / 128.

The useful math checkpoint, under this recipe, was the short run.

## The vision run that should have needed more epochs

This plate is not a vision-language model. No captions, no LLaVA. A pretrained ViT, a new 10-class head, `frgfm/imagenette` 320px, all 3,925 official validation images.

The number before training is a **random head** on a pretrained backbone, not zero-shot classification. Those baselines jumped between about 3% and 17% across runs. Only the finished accuracies compare.

| Model | Epochs | Top-1 | Errors / 3,925 |
|---|---:|---:|---:|
| ViT-Base | 1 | 98.12% | — |
| ViT-Base | 15 | 99.39% | 24 |
| ViT-Large | **1** | **99.59%** | **16** |
| ViT-Large | 15 | 99.29% | 28 |

ViT-Large after one epoch beat ViT-Base after fifteen. Fifteen epochs on Large added twelve extra mistakes. Train loss was still happy to fall. The task was already saturated.

{{< folio-card label="Erratum / how far these numbers go" >}}

- GSM8K here is **128** rows, not the full 1,319. A one- or two-problem gap is noise. The seven-problem drop from 50 to 600 steps is the claim I am willing to stand on, still as a checkpoint ranking, not as a paper.
- I did not publish a prompt-only baseline. This is base vs adapter, not “does prompting already suffice.”
- I did not train DPO or GRPO.
- Mid-run Hub recovery of optimizer state was **not** proven. Do not assume a killed Colab session can resume from the remote repo.

{{< /folio-card >}}

## Prediction / result / correction

**Prediction.** Validation loss is how you pick the checkpoint. Longer supervised fine-tuning is the conservative choice.

**Result.** Token-level loss and the behavior I scored moved in opposite directions on math and on Imagenette.

**Correction.** Teacher-forced loss asks “how unsurprised is the model by the teacher’s next token?” GSM8K asks “does the last number match?” Classification asks “which of ten labels?” Those are different exams. A checkpoint is an answer to one of them, not to all of them.

> The loss is a training instrument. It is not the task.

## Unpaid debt

Math told me that **what** I optimized was the wrong proxy. Tool calling told me something sharper: even the tokens inside the loss can be the wrong object. That is [Plate II: The Mask Is the Objective](/posts/adapter-field-notes-02-the-mask-is-the-objective/).
