---
title: "The Extra Code Is Not Another Brain"
date: 2026-08-20T11:36:00+08:00
draft: false
note: "10"
series: "Transformer Field Notes"
plate: "VI / VI"
description: "Production nanoGPT adds a factory around the same next-token spine—and the product is still a document continuer, not a chatbot."
summary: "Flash Attention, mixed precision, DDP, shards, checkpointing, and HellaSwag without pretending I trained GPT-2."
categories: ["Transformers"]
tags: ["nanogpt", "gpt-2", "ddp", "hellaswag", "evaluation"]
math: true
---

The upstream training code was much longer than my teaching implementation. My first reaction was that it must contain a more advanced model.

It mostly contained a factory.

## Plate VI.1 — the spine and the factory

| Teaching desk | Training factory | What changed |
|---|---|---|
| Hand-written `QKᵀ` + mask | Flash Attention | less memory traffic |
| FP32 | BF16/FP16 autocast | less bandwidth and memory |
| One `AdamW` call | decay parameter groups, fused path | optimizer engineering |
| One device | DDP | data-parallel scale |
| In-memory character tensor | binary shards + memmap | data delivery |
| One batch per update | gradient accumulation | larger effective batch |
| Fixed learning rate | warmup + cosine decay | optimization schedule |
| Python execution | `torch.compile` | runtime optimization |

The core route remained recognizable:

```text
tokens → embeddings → Transformer blocks → logits
       → cross-entropy → backward → optimizer step
```

The extra code feeds, accelerates, distributes, evaluates, and preserves that route. It matters enormously, but it is not another brain hidden beside attention.

## Two arithmetic traps in the factory

### Gradient accumulation

`cross_entropy` already averages one micro-batch. PyTorch's `backward()` adds gradients. To simulate the mean of \(K\) micro-batches, each loss must be divided by \(K\) before backward:

$$
\mathcal{L}_{\mathrm{step}} = \frac{1}{K}\sum_{k=1}^{K}\mathcal{L}_k
$$

```python
loss = loss / gradient_accumulation_steps
loss.backward()
```

Without the division, the gradient is \(K\) times larger.

### DDP

Each GPU has a complete model and a different slice of data. DDP all-reduces gradients. If every worker starts from the same weights and applies the same averaged gradient with the same optimizer state, the weights remain synchronized. Copying the weights between GPUs every step is unnecessary.

## Evaluation for a model that cannot take a test

A base language model does not naturally answer “A, B, C, or D.” It continues text. HellaSwag can therefore be scored in the language the model actually speaks:

1. append each candidate continuation to the shared context;
2. compute token negative log-likelihood;
3. mask out the shared context;
4. average over the completion length;
5. choose the continuation with the lowest average NLL.

Length normalization matters. Otherwise, a longer answer pays more total loss simply for containing more tokens. Prompt formatting matters too, so scores from different evaluation harnesses are not automatically comparable.

{{< folio-card label="Erratum / what I did not do" >}}

- I did **not** train GPT-2 124M on FineWeb-Edu.
- I did **not** run an 8×A100, 19,073-step reproduction.
- I did **not** benchmark Flash Attention, BF16, DDP, or `torch.compile` in my teaching run.
- I studied how those parts wrap the same training spine, and I ran a char-level GPT experiment on Tiny Shakespeare.

That boundary makes the learning record stronger, not smaller.

{{< /folio-card >}}

## The base model is not a chatbot

Ask a base model:

```text
What is the capital of France?
```

It may continue with more quiz questions instead of replying “Paris.” That is not necessarily refusal or ignorance. Its objective is document continuation.

Supervised fine-tuning still uses next-token loss, but changes the data distribution to conversations and often masks user tokens so the loss focuses on assistant responses. Chat behavior arrives through later training and system design, not because the pretraining loop secretly understood the role of an assistant.

## Prediction / result / correction

**Prediction.** A production GPT repository contains a more sophisticated intelligence than the small model I built.

**Result.** It contains a much more sophisticated way to train the same kind of next-token model at scale.

**Correction.** Model architecture, training factory, evaluation harness, and chat alignment are separate layers. Confusing them makes every claim larger and every lesson weaker.

> The factory changes what can be trained. It does not change what evidence I personally produced.

## What this folio leaves behind

The six notes began with a shifted batch and ended with the boundary between a base model and a chatbot. The stable spine is now visible:

```text
next-token prediction
→ causal communication + per-token computation
→ residual stream
→ controlled training evidence
→ scalable systems
→ evaluation that matches the model
```

The next folio will leave Python's training desk and follow a single token through a lower-level inference loop—where the unpaid debt is no longer attention, but the KV cache.
