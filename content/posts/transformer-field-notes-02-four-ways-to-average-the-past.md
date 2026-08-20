---
title: "Four Ways to Average the Past"
date: 2026-08-20T11:32:00+08:00
draft: false
note: "06"
series: "Transformer Field Notes"
plate: "II / VI"
description: "Q, K, and V stopped feeling magical when I rebuilt attention as four increasingly selective ways to mix the same past."
summary: "From a loop mean to scaled causal self-attention, following every tensor shape through the implementation."
categories: ["Transformers"]
tags: ["attention", "qkv", "causal-mask", "tensor-shapes"]
---

My first mental model for attention was three labels: Query, Key, Value. It sounded explanatory, but I could not reconstruct the operation from those words.

What finally helped was to remove the labels and build the same causal mix four times.

## Plate II.1 — the matrix that changes its mind

Suppose the sequence has four positions. Every version produces a matrix `W` whose row `t` says how position `t` mixes earlier values.

| Version | How `W` is made | What changes |
|---|---|---|
| Loop mean | average `x[:t+1]` | correct but slow |
| `tril` mean | normalized lower triangle | same uniform answer, parallel |
| Mask + softmax | `0` for past, `-inf` for future | same answer, attention-ready form |
| QKV | scaled `q @ kᵀ`, then mask + softmax | weights become content-dependent |

The first three versions are not failed attention. They isolate the two ideas that are easy to mix together:

1. causal access: which positions are legal;
2. selective access: how strongly to weight each legal position.

## The production shape trace

The compact implementation creates Q, K, and V in one projection:

```python
q, k, v = self.c_attn(x).split(self.n_embd, dim=2)
```

Then the tensors change shape:

```text
x                         [B, T, C]
c_attn(x)                 [B, T, 3C]
split q, k, v             [B, T, C] each
view + transpose          [B, H, T, D]
q @ k.transpose(-2,-1)    [B, H, T, T]
softmax + @ v             [B, H, T, D]
merge heads               [B, T, C]
c_proj                     [B, T, C]
```

Here `H` is the number of heads and `D=C/H` is the size of one head.

{{< folio-card label="Erratum / what the scale actually controls" >}}

~~Divide by `√d_k` because the dimension is large.~~

If the components of `q` and `k` have unit-scale variance, the variance of their dot product grows roughly with `d_k`. Dividing by `√d_k` keeps the logits at a manageable scale before softmax. The problem is variance and saturation, not the abstract fact that a dimension is “big.”

{{< /folio-card >}}

## Communication, not computation

Attention moves information between token positions. That is its distinctive job.

```text
"it"  ──reads──>  "cat"
```

The MLP that follows does something different: it applies the same channel transformation independently at every position.

```text
attention    token mixing       positions communicate
MLP          channel mixing     each position computes alone
```

This distinction sounds simple, but it repaired a vague belief that the feed-forward network was just “more attention.” It is not. Attention chooses what to read; the MLP transforms what has been read.

## Prediction / result / correction

**Prediction.** Q, K, and V are the essence of attention.

**Result.** The essence was already visible in a lower-triangular average: each position mixes an allowed past. QKV makes that mixing selective and learned.

**Correction.** Attention is not magic attached to three letters. It is a causal routing operation whose shape can be traced all the way from `[B,T,C]` to `[B,H,T,T]` and back.

> The most useful attention diagram is not an illustrated sentence. It is the tensor shape I can no longer hand-wave.

## Unpaid debt

Attention returns another `[B,T,C]` tensor. But a Transformer has many blocks, and none of them should erase the path that came before. [Plate III: The Residual Stream Is the Model](/posts/transformer-field-notes-03-residual-stream-is-the-model/) follows that path.
