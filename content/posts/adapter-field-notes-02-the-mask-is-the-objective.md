---
title: "The Mask Is the Objective"
date: 2026-08-30T19:40:00+08:00
draft: false
note: "15"
series: "Adapter Field Notes"
plate: "II / III"
description: "Eighteen hundred cleaner steps scored 0/128. Fifty steps that included the stop token beat the base. The labels were the experiment."
summary: "Three Qwen3 tool-calling masks: all assistant turns, first call without a terminator, then the terminator—and only the last one worked."
categories: ["Adapters"]
tags: ["qlora", "tool-calling", "sft", "evaluation", "a100"]
---

[Plate I](/posts/adapter-field-notes-01-the-loss-is-not-the-task/) showed a falling loss that picked a worse math checkpoint. Tool calling made the same mistake louder, because the base model was already good.

I did not download an opaque tool-use mixture. A converter pulled `tools` and `messages` out of Hermes, checked every call against that row’s JSON Schema, dropped duplicates, and split on fingerprints.

{{< folio-card label="Data card / Hermes → clean splits" >}}

| | |
|---|---:|
| Source rows | 7,102 |
| Rejected by strict checks | 1,997 |
| Kept | 5,103 |
| Train / val / test | 4,121 / 505 / 477 |
| One-call / multi-call | 2,279 / 2,824 |

{{< /folio-card >}}

Missing tools, illegal call JSON, orphaned tool responses, conflicting duplicate definitions, and schema failures were the common rejects. Volume was not the goal. Rows that could not be trusted did not enter training because the set would look larger.

Eval: 128 held-out test rows, seed `20260823`, greedy decoding, thinking off. Metrics: parse, exact function sequence, exact arguments, schema validity against the offered tools.

## Three objectives, one base

Qwen3-8B, same rank-64 QLoRA, same 128 rows.

| Supervision | Steps | Sequence exact | Args exact | Parse / schema |
|---|---:|---:|---:|---|
| Base | — | 121 / 128 · 94.53% | 115 / 128 | 126 / 128 |
| Every assistant turn | 600 | 86 / 128 · 67.19% | 84 / 128 | 91 / 128 |
| First call only, no stop | 1,800 | **0 / 128** | 0 / 128 | **128 / 128** |
| First call + turn terminator | **50** | **123 / 128 · 96.09%** | 116 / 128 | **128 / 128** |

Validation loss was a fan of the disasters. The 600-step run lowered it about 22% from step 100. The 1,800-step run reached 0.0058. Both lost to a 50-step model whose loss was not the smallest.

### 600 steps: the model stopped calling tools

Thirty-seven adapter rows failed to parse, versus two on the base. The typical failure was a fluent natural-language answer and no `<tool_call>` block.

The mask had supervised **all** assistant spans: the tool call *and* the post-tool closing sentence. In a multi-turn transcript those closing tokens are a large, easy next-token task. They diluted the first-turn decision—whether to call a function at all.

### 1,800 steps: the model never stopped

Parse and schema went to 128 / 128. Sequence exact went to zero.

The adapter usually picked a plausible function, then repeated the same legal `<tool_call>` until the 256-token cap. One row called `search_movies` seventeen times.

The mask had supervised the call body and omitted Qwen’s turn terminator. The model learned syntax. It did not learn “this turn is finished.”

### 50 steps: include the terminator

Putting the stop token in the supervised span, and stopping after fifty optimizer steps, was the first adapter that beat the base: +2 sequence, +1 arguments, parse/schema 128 / 128.

That is a small lift on a strong base. It is a large lift against the two longer runs. The lesson is not “always train fifty steps.” It is that **the mask is the loss function**, and a wrong mask cannot be outrun.

## 14B under the corrected mask

Same first-call-plus-terminator recipe.

| Run | Sequence | Args |
|---|---:|---:|
| 14B base | 124 / 128 · 96.88% | 115 / 128 |
| 14B · 50 steps | 124 / 128 | 117 / 128 |
| 14B · 200 steps | 125 / 128 | 117 / 128 |

Two hundred steps pressed validation loss from 0.0080 to 0.0066 and bought one extra sequence match. The ceiling was already low. Fine-tuning here is edge repair, not a new skill.

{{< folio-card label="Erratum / stack, not folklore" >}}

- Qwen3’s live chat template had no `{% generation %}` marker, so `return_assistant_tokens_mask=True` produced an empty mask. I parsed `<|im_start|>assistant` / `<|im_end|>` boundaries instead of trusting the template flag.
- Heterogeneous JSON tool schemas cannot sit in one Arrow struct column. The trainer templates first, then stores homogeneous integer `input_ids` / `labels`.
- TRL 1.10’s SFT preprocessing stripped `attention_mask` and then crashed. Already-tokenized rows went through a plain Transformers `Trainer`. Attempts that died before optimizer step 0 are not results.
- 128 rows again. Treat 1-row 14B gaps as noise.

{{< /folio-card >}}

## Prediction / result / correction

**Prediction.** Assistant-only loss is enough. More steps on a cleaner first-call objective should dominate a short run.

**Result.** Supervising the wrong assistant tokens taught the model to chat instead of call. Supervising the call without the terminator taught infinite calls. The short run with the terminator was the only one that beat the base.

**Correction.** `labels` are not bookkeeping. Everything set to `-100` is outside the objective. If the stop token is outside the objective, “stop” is not something the gradient can teach.

> The mask is the objective. The step count is a budget.

## Unpaid debt

Math and tools both said: do not buy more steps on a bad proxy. The remaining temptation was to buy a bigger base instead. That is [Plate III: Scale Bought the Base](/posts/adapter-field-notes-03-scale-bought-the-base/).
