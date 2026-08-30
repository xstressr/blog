---
title: "Learning Roadmap"
date: 2026-08-20T10:00:00+08:00
draft: false
summary: "The living map behind my zero-to-one AI journey."
hiddenInHomeList: true
---

This is a living map, not a syllabus I need to finish perfectly. Each stage should produce runnable work and a clearer explanation before I move on.

## 01 · Foundations

**Status: built**

- Python and tensor thinking
- Essential linear algebra and probability
- Data, train/validation splits, and baseline discipline
- Reading shapes and tracing values through code

## 02 · Neural networks

**Status: built**

- Backpropagation and computational graphs
- MLPs, activations, initialization, and normalization
- Training loops, optimization, and overfitting
- Controlled experiments: changing one variable at a time

## 03 · Transformers

**Status: in progress**

- Attention as communication
- Residual streams and MLP computation
- Tokenization, embeddings, and positional information
- Reproducing GPT-style training from first principles
- Scaling, batching, checkpoints, and evaluation
- Exporting learned weights and tracing token-by-token inference in C
- Handwritten GPT-2 training in C, graded against a Python answer sheet

Published sequence: [Transformer Field Notes I — The Batch Is a Factory of Futures](/posts/transformer-field-notes-01-batch-factory-of-futures/) through [VI — The Extra Code Is Not Another Brain](/posts/transformer-field-notes-06-extra-code-is-not-another-brain/), then [Inference Field Notes I — The Model File Has No Names](/posts/inference-field-notes-01-the-model-file-has-no-names/).

Current folio: [C Field Notes I — Python Writes the Answer Sheet](/posts/c-field-notes-01-python-writes-the-answer-sheet/) through [II — There Is No Autograd Here](/posts/c-field-notes-02-there-is-no-autograd-here/).

## 04 · AI systems

**Status: next**

- Retrieval and reranking
- Evaluation sets and error analysis
- Agents, tools, and reliable workflows
- Serving, observability, latency, and cost

## 05 · Original work

**Status: in progress**

- Form a testable question
- Build the smallest credible experiment
- Publish the method, failures, and result
- Let the evidence—not the novelty claim—carry the work

Published sequence: [Adapter Field Notes I — The Loss Is Not the Task](/posts/adapter-field-notes-01-the-loss-is-not-the-task/) through [III — Scale Bought the Base](/posts/adapter-field-notes-03-scale-bought-the-base/). One Colab A100, Qwen3 QLoRA and a ViT classifier, frozen 128-row evals. Not a production RAG system, and not a claim that bigger adapters win.
