---
title: "The Model File Has No Names"
date: 2026-08-22T18:10:00+08:00
draft: false
note: "11"
series: "Inference Field Notes"
plate: "I / I"
description: "Following a TinyStories checkpoint from PyTorch export to mmap, KV cache, and token sampling in Karpathy's llama2.c."
summary: "A 28-byte header, 58 MB of unnamed floats, one C inference loop, and the contract that holds them together."
categories: ["Inference"]
tags: ["llama2.c", "c", "inference", "checkpoint", "kv-cache"]
---

The executable was 223 KB. The model file beside it was 58 MB. There was no Python environment, no PyTorch import, and no list of parameter names.

Yet this worked on my Windows machine:

```text
run.exe stories15M.bin
achieved tok/s: ~382
```

My previous notes ended with the claim that production training code is a factory around a recognizable next-token spine. The next question was physical: once training is over, what exactly crosses the border into a small C program?

I expected a model loader. What I found was closer to an agreement about where every byte must live.

## Plate I.1 — two artifacts leave training

The training script writes two different files:

| Artifact | Contents | Intended reader |
|---|---|---|
| `ckpt.pt` | named tensors, optimizer state, iteration, metadata | Python, for resuming training |
| `model.bin` | a small config header followed by ordered FP32 weights | `run.c`, for inference |

The distinction corrected a fuzzy idea I had about checkpoints. A training checkpoint is not automatically a deployable model. `ckpt.pt` is a Python object graph. It preserves enough state to continue optimization. The C program does not need AdamW's momentum, dictionary keys, or a pickle decoder.

It needs seven integers and a float array.

```text
train.py
  ├─ torch.save(...)       → ckpt.pt
  └─ model_export(..., 0)  → model.bin

model.bin
  └─ mmap + pointer offsets → run.c
```

`export.py` is therefore not a minor file-format utility. It is the border crossing between two runtimes.

## The header is the map

The legacy format starts with seven 32-bit integers:

```text
dim, hidden_dim, n_layers, n_heads,
n_kv_heads, vocab_size, seq_len
```

I inspected the first 28 bytes of the downloaded `stories15M.bin`:

| Field | Value |
|---|---:|
| model width | 288 |
| feed-forward width | 768 |
| layers | 6 |
| attention heads | 6 |
| KV heads | 6 |
| vocabulary | 32,000 |
| sequence length | 256 |

The whole file was 60,816,028 bytes. Subtract the 28-byte header and the remainder is exactly a sea of four-byte floats.

There are no tensor names and no shapes stored beside the tensors. Shape comes from the header. Meaning comes from order.

```text
token embeddings
attention RMS weights for every layer
all Wq, then all Wk, then all Wv, then all Wo
feed-forward RMS weights for every layer
all W1, then all W2, then all W3
final RMS weight
legacy RoPE tables
optional classifier weights
```

The exporter writes that sequence. The loader walks the same sequence. If one side changes its order, both programs remain syntactically valid and the model becomes nonsense.

> The file format is not merely storage. It is an ABI for learned behavior.

## `mmap` does not rebuild the model

I imagined `run.c` allocating tensors and copying weights into a reconstructed object. It does something more direct.

The operating system maps the file into the process address space. A pointer starts immediately after the config header. `memory_map_weights()` assigns names to regions by moving that pointer forward by known element counts:

```c
w->token_embedding_table = ptr;
ptr += vocab_size * dim;

w->rms_att_weight = ptr;
ptr += n_layers * dim;

w->wq = ptr;
ptr += n_layers * dim * dim;
```

The code is not deserializing named tensors. The pointer arithmetic *creates* the names.

This also explains a strange compatibility detail. Old exports contain two RoPE lookup tables. The current forward pass computes the rotations from position with `cos` and `sin`, so the loader advances past those floats without using them. The unused bytes remain because changing the historical layout would break the contract.

{{< folio-card label="Desk card / what I actually ran" >}}

- Source: Karpathy's `llama2.c`, checked out at commit `350e04f`
- Model: pretrained TinyStories 15M, FP32 legacy v0 export
- Binary: `run.exe`, built with Visual Studio Build Tools 2022
- Machine result: approximately 382 tokens/s on Windows
- Earlier comparison: approximately 106 tokens/s on my Mac
- Not completed: training my own 260K model and feeding its export back to C

{{< /folio-card >}}

## One token enters, logits leave

The forward function has a revealing signature:

```c
float* forward(Transformer* transformer, int token, int pos)
```

It receives one token at one position—not a complete sequence tensor.

For each layer, the function normalizes the current activation, computes Q, K, and V, applies RoPE to Q and K, attends over positions `0..pos`, runs the SwiGLU feed-forward block, and adds both residual updates. A final RMSNorm and classifier matrix produce the logits.

The architecture I had met in Python was still there:

```text
embedding
→ [RMSNorm → attention → residual
→  RMSNorm → SwiGLU   → residual] × layers
→ RMSNorm → vocabulary logits
```

The surprise was not a new mathematical component. It was memory discipline.

## The KV cache is part of the runtime state

At position `pos`, K and V are written directly into slots inside two preallocated caches. The temporary `k` and `v` pointers are aliases to those slots; they do not own separate allocations.

```text
key_cache[layer, pos, :]
value_cache[layer, pos, :]
```

On the next token, attention reuses every earlier K and V and computes only the new pair. Without this cache, autoregressive generation would repeatedly recompute the entire prefix.

This gave me a cleaner split:

| Persistent across program runs | Persistent across token steps |
|---|---|
| learned weights in `model.bin` | KV cache in allocated memory |
| produced by training | produced by the current prompt |
| read-only during inference | grows one position at a time |

Weights are the model's learned state. The KV cache is the conversation's temporary state.

## Prefill is inference without freedom

The generation loop also corrected my picture of prompting.

While prompt tokens remain, the program still calls `forward()` for every token, but it ignores the model's sampled choice and forces the next known prompt token. This is prefill: the work is necessary because each step must populate the KV cache.

Only after the prompt has been consumed does the program sample from logits:

```text
prompt remains → force the next prompt token
prompt ends    → sample the next generated token
```

The same forward function serves both phases. The difference is who chooses the next token: the prompt or the sampler.

Temperature zero selects the maximum logit. Otherwise the program applies temperature, softmax, and either multinomial or top-p sampling. The apparent behavior of the model therefore comes from three separate layers:

```text
weights define logits
KV cache carries context
sampler turns probabilities into one path
```

## Prediction / result / correction

**Prediction.** A C inference engine must reconstruct a structured model from a rich checkpoint before it can generate text.

**Result.** `run.c` maps an ordered float file, assigns structure with pointer offsets, allocates runtime state, and executes the same Transformer equations one token at a time.

**Correction.** The structure is not stored as names. It exists in the agreement between exporter and loader. The checkpoint layout, tensor shapes, runtime buffers, and generation loop are one connected system.

I have now consumed a pretrained export from end to end. I have not yet produced my own. That missing left half—training a tiny model, exporting its weights, and loading them into the same C binary—is a useful optional experiment, but it is no longer a mystery about formats.

The model file has no names. The code gives every byte a role.

The unpaid debt is the other half of the same bridge: C that *trains*, and a Python file that prints the numbers it must match. That is [C Field Notes I: Python Writes the Answer Sheet](/posts/c-field-notes-01-python-writes-the-answer-sheet/).

