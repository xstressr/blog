---
title: "There Is No Autograd Here"
date: 2026-08-27T10:30:00+08:00
draft: false
note: "13"
series: "C Field Notes"
plate: "II / II"
description: "llm.c's CPU reference does not build a graph. Each layer is a handwritten forward paired with a handwritten backward, and training is forty-one updates of floats."
summary: "Paired layer functions, mean-loss seeds, AdamW on a flat parameter tape, and the 41-step sanity loop I read but have not yet run."
categories: ["C Implementation"]
tags: ["llmc", "backprop", "adamw", "gpt-2"]
math: true
---

micrograd attached a `_backward` closure to every scalar. makemore lecture 4 did the same at tensor scale in Python. `train_gpt2.c` is that idea with the framework removed.

There is no computational graph. There is no `loss.backward()`. Each layer is a pair:

```text
encoder_forward   / encoder_backward
layernorm_forward / layernorm_backward
matmul_forward    / matmul_backward
attention_forward / attention_backward
gelu_forward      / gelu_backward
residual_forward  / residual_backward
```

`gpt2_forward` and `gpt2_backward` only assemble those pairs in GPT-2 order. Training changes `params_memory`, not the C source.

## Plate II.1 — one index language

There is no `out[b][t][o]`. A row is width times row index, plus the lane inside the row:

```text
weight[o*C + i]          W[o, i]
inp[bt*C + i]            token bt, channel i
logits + b*T*Vp + t*Vp   vocab scores at (b, t)
bt = b*T + t
```

`C` is 768 for GPT-2 124M. `V` is 50257. `Vp` is the padded vocab. Softmax iterates to `V`.

A few layer facts that were easy to get wrong:

- **Encoder.** `out = wte[id] + wpe[t]`. Backward scatters `dwte[id] += dout`. Integer token ids have no `dinp`. Tied `lm_head` / `wte` accumulate into the same `grads.wte`.
- **LayerNorm.** Same formula as PyTorch, written as `rstd = 1/sqrt(var+eps)`. Variance is `v/C` (biased, `correction=0`). `mean` and `rstd` are cached per `(b,t)` for backward. `weight` / `bias` are γ / β, not statistics of `x`.
- **Attention.** Input is fused QKV of shape `(B,T,3C)`. Head size is `C/NH`. `preatt` / `att` are `(B,NH,T,T)`.
- **Residual backward** is addition: `dout` is added onto both branches.

## Forward computes loss; backward computes grad

```text
batch x, y
  → gpt2_forward     probs + mean_loss
  → gpt2_zero_grad
  → gpt2_backward    grads
  → gpt2_update      AdamW on params_memory
```

The seed of the chain rule is not mysterious:

```c
float dloss_mean = 1.0f / (B*T);
for (int i = 0; i < B*T; i++) { grads_acts.losses[i] = dloss_mean; }
```

`mean_loss` is already the average over `B*T` positions. Backward therefore starts from \(1/(BT)\). Every `+=` in the layer backwards is why `zero_grad` exists.

`gpt2_update` is AdamW on a flat tape of floats: first moment, second moment, bias correction, then

```text
param -= lr * (m_hat / (sqrt(v_hat) + eps) + weight_decay * param)
```

`main` passes `weight_decay=0`, so these 41 steps are Adam. The thing being minimized is **loss**. Gradients are the descent direction, not the objective.

## Forty-one steps are a wire check

```c
for (int step = 0; step <= 40; step++) {
```

That is 41 updates, each ending in `gpt2_update`.

| When | What | What it is not |
|---|---|---|
| every 10 steps | val: forward only, mean of 5 batches | a full evaluation suite |
| steps 20 and 40 | sample with `targets=NULL` | a proper generate loop |
| every step | train forward → zero → backward → update | pretraining GPT-2 |

The generate path in this file is intentionally wasteful. The comment says so: each new token reruns the whole `(B,T)` forward from scratch. There is no KV cache here. This is a sanity print, not `run.c`.

The checkpoint is `gpt2_124M.bin`. The data is Tiny Shakespeare if present, otherwise TinyStories. The loop continues a pretrained model for a few steps. It does not train 124M from scratch.

{{< folio-card label="Erratum / what I have and have not done" >}}

- I **read** `train_gpt2.c` through `main` (2026-08-24 to 2026-08-25). I have **not** compiled it, and I have **not** run the 41-step loop.
- I have **not** trained GPT-2 124M, and I have **not** run `train_gpt2.cu`.
- GPU work is planned on a Windows RTX 3060 6GB: start with `train_gpt2fp32cu` at `B=4`, `T=64`. Default llama2.c `train.py` (`batch_size=128`) would OOM there.
- 7B, GPT-2 1.6B, and nanochat are out of scope for this folio.

{{< /folio-card >}}

## Prediction / result / correction

**Prediction.** A C training file must hide a tiny autograd engine, or it cannot be a real trainer.

**Result.** The engine is the pair of functions per layer, plus three whole-network verbs: forward, backward, update. Python’s job is to print the numbers those verbs must match.

**Correction.** “Training” in this file is a change to a float buffer. The architecture is the order of the function calls. If I want a different model, I change that order—or the floats—not a graph object I never built.

> The C does not learn. The floats do.

## What this folio leaves behind

Inference Field Notes followed a nameless `.bin` through mmap and a KV cache. These two plates add the training side:

```text
layout contract + one-token loop
→ Python as answer key
→ paired forwards/backwards, no graph
```

The spine from the Transformer folio is still the spine. What changed is the runtime: mmap instead of `state_dict`, a cache instead of a batched window, and gradients that exist only because someone wrote them.

The unpaid debt on this desk is still a machine: compile the CPU 41-step check, then see whether a 6GB card can move the same loop without breaking the contract.

A later folio left the C trainer and scored adapters by frozen behavior tests instead of teacher-forced loss. That is [Adapter Field Notes I: The Loss Is Not the Task](/posts/adapter-field-notes-01-the-loss-is-not-the-task/).
