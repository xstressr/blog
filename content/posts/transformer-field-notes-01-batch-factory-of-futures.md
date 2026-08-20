---
title: "The Batch Is a Factory of Futures"
date: 2026-08-20T11:31:00+08:00
draft: false
note: "05"
series: "Transformer Field Notes"
plate: "I / VI"
description: "I thought training a language model meant teaching it to write. The first batch showed me something much more mechanical—and more useful."
summary: "A shifted batch, 4,096 simultaneous next-character bets, and the first numerical check that tells me whether the model is alive."
categories: ["Transformers"]
tags: ["transformer", "nanogpt", "next-token", "tensor-shapes"]
---

The first useful number was not a low loss. It was an ordinary one:

```text
step 0    train 4.2003    val 4.1982
```

Tiny Shakespeare has 65 characters. A model that knows nothing should spread its probability almost uniformly, so its expected loss is roughly:

```text
ln(65) = 4.174
```

`4.20` is not an achievement. It is a wire check. A value near `50`, or `NaN`, would tell me to stop admiring the architecture and find the broken connection.

## Plate I.1 — one sequence, many futures

I used to picture language-model training as generation: give the model a prefix, let it write a character, then repeat. That is how inference works. Training is different.

From one continuous token stream, `get_batch()` cuts windows of length `T`:

```text
x:  F i r s t   C i t i z e n :
y:  i r s t   C i t i z e n : \n
     ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑
     T next-character targets
```

`y` is simply `x` shifted one place to the left. The model receives tensors shaped `[B,T]` and returns logits shaped `[B,T,V]`.

```text
token ids        [B, T]
embeddings       [B, T, C]
logits           [B, T, V]
targets          [B, T]
```

With my baseline, `B=32` and `T=128`. One forward pass therefore trains `32 × 128 = 4,096` next-character predictions. Over 2,000 steps, that is 8,192,000 prediction positions.

{{< folio-card label="Desk card / the baseline batch" >}}

- Corpus: Tiny Shakespeare, character vocabulary `V=65`
- Batch: `B=32`, context `T=128`
- Model width: `C=128`
- Prediction budget: `2,000 × 32 × 128 = 8.19M` tokens
- Seed: `42`

{{< /folio-card >}}

## Why the future does not leak

The targets for every position are present at once, but the model is not allowed to read them. A causal mask makes the attention matrix lower-triangular:

```text
        key position
query   0  1  2  3
  0     ✓  ·  ·  ·
  1     ✓  ✓  ·  ·
  2     ✓  ✓  ✓  ·
  3     ✓  ✓  ✓  ✓
```

Position 2 may use positions `0..2` to predict position 3. It cannot peek at position 3 itself. That single constraint is what allows training to be parallel across `T` while generation remains sequential.

## Prediction / result / correction

**Prediction.** Training a language model means repeatedly asking it to generate the next token.

**Result.** Training scores every legal next-token problem in the window simultaneously. Generation is the slow loop; training is the factory.

**Correction.** The model is not learning “how to write a play” as one indivisible behavior. It is minimizing thousands of local conditional predictions per step. The play-like behavior has to emerge from that pressure.

> A falling loss is interesting later. At step zero, the honest question is smaller: does the number look like ignorance?

## Unpaid debt

I can now explain how one batch manufactures `T` futures. I still owe an explanation for how each position decides which pieces of its allowed past matter. That debt is the subject of [Plate II: Four Ways to Average the Past](/posts/transformer-field-notes-02-four-ways-to-average-the-past/).
