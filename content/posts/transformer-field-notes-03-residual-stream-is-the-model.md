---
title: "The Residual Stream Is the Model"
date: 2026-08-20T11:33:00+08:00
draft: false
note: "07"
series: "Transformer Field Notes"
plate: "III / VI"
description: "I stopped picturing a Transformer as layers replacing one another. A better picture is one long highway with repeated reads and writes."
summary: "Pre-LN, identity gradients, residual initialization, and why a block is two writes into one persistent stream."
categories: ["Transformers"]
tags: ["residual-stream", "pre-layernorm", "mlp", "gpt"]
---

I used to picture a deep model like a stack of filters: layer 1 produces a representation, layer 2 consumes and replaces it, and eventually only the top layer remains.

The code told a different story:

```python
x = x + self.attn(self.ln_1(x))
x = x + self.mlp(self.ln_2(x))
```

The `x` on the left is not discarded. It is the road.

## Plate III.1 — two writes per block

```text
residual stream x
      │
      ├── LN ── Attention ──┐
      └───────────────────── + ── x'
                              │
                              ├── LN ── MLP ──┐
                              └─────────────── + ── x''
```

Attention and the MLP read a normalized view of the stream, compute a change, and add that change back. This is Pre-LN: normalization lives inside the branch rather than blocking the main path.

During backpropagation, an addition creates an identity term:

```text
∂x_next / ∂x = I + ∂branch / ∂x
```

Even when a branch has an awkward gradient, the `I` remains. That is the gradient highway I was missing when I summarized residual connections as merely “helping with vanishing gradients.”

## The block's parameter chant

For channel width `C`, the large matrices in one GPT-style block are approximately:

```text
Attention: Q, K, V, output projection    4C²
MLP:       C → 4C → C                    8C²
Block total                               12C²
```

Attention is the communication branch. The MLP is the larger computation branch. Both write into the same stream.

{{< folio-card label="Bench evidence / two details in model.py" >}}

**Residual projection initialization**

```python
std = 0.02 / math.sqrt(2 * n_layer)
```

There are two residual writes per block. Scaling their output projections by `1/√(2N)` controls the variance accumulated across `2N` writes.

**Weight tying**

```python
self.transformer.wte.weight = self.lm_head.weight
```

The table that maps token IDs into the stream is also used, transposed in effect, to score which token best matches the final stream state.

{{< /folio-card >}}

## A layer is not a room

The new picture is less architectural and more editorial. The residual stream is a manuscript passed through the whole model. Attention inserts information from other positions. The MLP rewrites features at the current position. Neither starts a fresh document.

This also explains why swapping ReLU for GELU does not create a new kind of language model. It changes a branch operation. The persistent spine—embedding, residual stream, repeated communication and computation, final normalization, vocabulary scores—survives.

## Prediction / result / correction

**Prediction.** Each Transformer layer replaces the previous representation with a better one.

**Result.** Each block performs two residual writes into a stream that can carry identity information across the full depth.

**Correction.** The layers are not the model in isolation. The continuously preserved and revised stream is the model's working state.

> A twelve-layer GPT is not twelve sealed rooms. It is one `C`-wide road with twenty-four opportunities to write in the margins. My four-layer bench model is the same picture at eight writes.

## Unpaid debt

The architecture now has a path for information and gradients. That proves nothing about learning. [Plate IV: A Live Wire Is Not a Language](/posts/transformer-field-notes-04-live-wire-is-not-a-language/) puts the small model on the bench.
