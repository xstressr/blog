---
title: "Python Writes the Answer Sheet"
date: 2026-08-27T10:20:00+08:00
draft: false
note: "12"
series: "C Field Notes"
plate: "I / II"
description: "I expected llm.c's Python file to be the real trainer. It is mostly a GPT-2 reference plus a machine that prints the answers C has to match."
summary: "train_gpt2.py as nanoGPT-shaped model, token-river loader, and a write_state bridge that dumps weights, grads, logits, and loss."
categories: ["C Implementation"]
tags: ["llmc", "gpt-2", "checkpoint", "dataloader"]
---

[Inference Field Notes I](/posts/inference-field-notes-01-the-model-file-has-no-names/) followed exported weights into `run.c`. After that, I assumed the next Python file would be “the trainer,” with C as a faster rewrite.

`train_gpt2.py` is not primarily a trainer. The first 300 lines are a GPT-2 that looks like nanoGPT. The rest is glue: a dataloader that reads a river of tokens, and writers that flatten weights, gradients, logits, and loss into `.bin` files C can grade itself against.

The default `main` overfits a few steps so those files exist. It is not an attempt to pretrain GPT-2.

## Plate I.1 — the same bridge, a heavier payload

Both codebases use the same idea: named PyTorch tensors become a nameless blob. The jobs diverge.

| | `llama2.c` | `llm.c` |
|---|---|---|
| Python | `train.py` trains; `export.py` dumps | `train_gpt2.py` is the reference |
| `.bin` | Config + weights | weights **and** `x`, `y`, logits, loss, grads |
| C program | `run.c` infers | `train_gpt2.c` trains, then checks the answers |

`write_model` is another `export.py`. The new piece is `write_state`: the same batch, with intermediates, so a C forward and backward can be compared to PyTorch.

The header is no longer seven ints. `gpt2_build_from_checkpoint` reads 256 `int`s, checks magic `20240326` and version `3`, then reads `maxT`, `V`, `L`, `NH`, `C`, and padded vocab `Vp`. A bad version prints a hint: re-run the Python file.

`lm_head` is not written as its own tensor. It is tied to `wte`. `LLMC_SKIP_INIT = 1` so the classifier is not initialized twice. Softmax later loops only to the real vocab `V`; `Vp` (for example 50304) exists so GPU matmuls can be cut on a multiple of 128. Padding is an empty operation for the language model, not a larger dictionary.

Write order is again by kind, across layers:

```text
wte (V, C)
wpe (T, C)
ln_1 weight / bias     × L
c_attn weight / bias   × L
c_proj weight / bias   × L
ln_2 weight / bias     × L
c_fc weight / bias     × L
mlp c_proj weight / bias × L
ln_f weight / bias
```

C must read in that exact order. There are still no names in the file.

## A dataloader that is not DataLoader

`DistributedDataLoader` is not PyTorch’s `DataLoader`. Disk holds one already-tokenized stream. Header: 256 `int32`s (magic `20240520`, version `1`, `ntok`), then `uint16` tokens.

Each rank starts at `rank * B * T` and jumps `B * T * num_processes` after every batch. `next_batch` reads `B*T+1` tokens and splits them:

```text
x = buf[:-1]
y = buf[1:]
view(B, T)
```

That `view` folds a contiguous slice into `B` rows. It does not sample `B` random documents. On one process, `ddp_rank=0` and `ddp_world_size=1`, it is sequential. `--overfit_single_batch 1` resets every step and chews the same morsel—useful for checking that C and Python agree, useless as pretraining.

## Two leftovers from the Python model

**MLP is not “the predictor.”** Inside a block it is the feed-forward: `768 → 3072 → GELU → 768` per token. Tokens mix in attention. Next-token prediction is `lm_head` after the last block.

**`configure_optimizers` does not train.** It groups AdamW parameter tensors. Rank ≥ 2 (matrices, embeddings) get weight decay. Rank < 2 (bias, LayerNorm scale/bias) do not. The 41-step C `main` later sets decay to `0`, so those particular updates are Adam.

{{< folio-card label="Erratum / what this Python file did on my machine" >}}

- Read on 2026-08-23. Interpreter: Windows `.venv`, `torch 2.13.0+cpu`, no CUDA in that environment.
- I used it as a map and a contract, not as a training run that I claim as a result.
- GPU training, if it happens, is a later Windows RTX 3060 6GB job—not this file’s default batch.

{{< /folio-card >}}

## Prediction / result / correction

**Prediction.** The Python in `llm.c` is the real model; C is a performance port.

**Result.** Python is the answer key. C is supposed to reproduce logits, loss, and gradients from the same weights and the same batch.

**Correction.** A port without a numerical contract is a rewrite you cannot debug. The `.bin` files are not an implementation detail. They are the exam.

> If C disagrees with Python, C is wrong until proven otherwise.

## Unpaid debt

The answer sheet is useless unless C can take the exam without autograd. That is [Plate II: There Is No Autograd Here](/posts/c-field-notes-02-there-is-no-autograd-here/).
