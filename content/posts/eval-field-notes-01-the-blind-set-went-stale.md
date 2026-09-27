---
title: "The Blind Set Went Stale"
date: 2026-09-27T10:00:00+08:00
draft: false
note: "17"
series: "Eval Field Notes"
plate: "I / II"
description: "My agent beat single-shot RAG by five questions on a blind set. Then I noticed I had been tuning on that blind set. Sixteen fresh questions later, the gap was gone."
summary: "An agent looked 12/14 vs 7/14 on a blind set I had quietly reused. On a freshly frozen set it tied single-shot RAG at 10/12—at three times the cost."
categories: ["Evaluation"]
tags: ["rag", "agents", "evaluation", "llm-as-judge", "blind-test"]
---

The Adapter folio ended on a lesson about proxies: the number you optimize is not the task. This folio starts one level up. Even the number you *report* can quietly turn into something you optimized.

The system is small. It answers questions about four public Chinese documents: three critical-illness insurance contracts and the industry's standard disease definitions. That comes to 565 chunks. It has two ways to answer:

- **Single-shot RAG.** Retrieve, expand to neighbouring chunks, answer with citations.
- **A LangGraph agent.** It can search, read a clause by number and call a few calculators before it submits an answer.

The question for this plate was whether the agent is worth it.

## Plate I.1 — the desk

{{< folio-card label="Lab card / 2026-09-26 to 2026-09-27" >}}

| | |
|---|---|
| Retriever | bge-m3 dense, product filter, top 5 |
| Generator | DeepSeek V4.1 Flash, temperature 0, cached |
| Judge | GLM-5.3, max reasoning, rubric `j2` |
| Judge checks | 15/15 on a calibration set locked before the judge ran; 6/6 adversarial; 5/5 held-out adversarial |
| Scoring | correctness and faithfulness judged separately |
| Sets | dev 33 · blind v1 18 · blind v2 16 |
| Code, sets, reports | [xstressr/insurance-rag-eval](https://github.com/xstressr/insurance-rag-eval) |

{{< /folio-card >}}

Correctness and faithfulness are judged separately on purpose. An answer can be fully grounded and still miss the point. It can also hit every point and smuggle in one invented sentence. Averaging the two hides which part of the pipeline failed.

## The number I liked

Blind v1 was written before the system existed. The questions were frozen, then labelled, and only then run. On its 14 answerable questions:

| System | Correct |
|---|---:|
| Single-shot RAG | 7 / 14 |
| Agent + citation guard | **12 / 14** |

Five questions is a big gap for a set this small. The agent read the second clause that single-shot retrieval never saw, and it compared products clause by clause instead of guessing.

Then I reread my own changelog.

- **Prompt `p2`.** The single-shot prompt was revised after reading blind v1 failures.
- **Reranker.** The decision not to ship a cross-encoder reranker was made on blind v1.
- **Agent prompt.** Its few worked examples were distilled from blind v1 questions the single-shot system had missed.

The blind set had become a development set. Nothing leaked in the usual sense: no answers ended up in the prompts. But every decision had been made while looking at the same 14 questions.

## Prediction

I expected the agent to keep most of its lead on fresh questions: maybe three or four questions out of fourteen, not five. The mechanism looked real. Multi-clause questions need more than one retrieval.

## Blind v2

Sixteen new questions:

- 12 answerable, 4 that should be refused.
- They cover conditions, cross-product comparisons, industry definitions, and things the contracts simply do not say.

The protocol was stricter this time:

1. The questions went in their own commit before any document was opened or any retrieval was run.
2. The labels went in a second commit. A script located each evidence sentence in the chunk text and asserted it existed. For the "not written" questions, it asserted that a search of the full text came back empty.
3. One question changed type during labelling. I had guessed it was answerable; the contract turned out to say nothing about it. The change is recorded in the question file, not silently fixed.
4. Both systems ran once, with configurations frozen days earlier. Nothing was tuned afterwards.

## Result

| System | Correct (answerable) | Refusals | Faithful | Median latency | $ / 1k questions |
|---|---:|---:|---:|---:|---:|
| Single-shot RAG | **10 / 12** | 3 / 4 | 100% | 6.1 s | 1.06 |
| Agent + citation guard | 9 / 12 → **10 / 12** after arbitration | 4 / 4 | 100% | 13.1 s | 3.28 |

The lead did not shrink. It disappeared.

Both systems stumbled on the same question. The claim-deadline question has three deadlines; both gave the decision deadline and dropped at least one of the other two. Neither invented anything.

The agent's single "incorrect" is the most interesting row. Asked whether a diagnosis at a foreign hospital counts, it found the contract's definition of a medical institution. That definition requires a licence from the PRC health authority, so the agent concluded that a foreign hospital does not qualify. My label said something softer: the contract requires a graded hospital, does not mention foreign ones, so ask the insurer. The judge applies a contradiction-first rule, so it marked the agent wrong.

I had missed that definition while labelling. The agent may be more right than the label. But I found this out by reading the system's output, and relabelling after seeing results is exactly the move this protocol exists to prevent. So I did not touch the label. The question went to a human arbiter, the learner who owns this evaluation, who sided with the agent. The fix shipped as a new version of the set, v2.1, with v2 left untouched and both results reported. Under v2.1 the agent's answer is correct, the single-shot answer ("the contract doesn't exclude foreign hospitals") becomes a contradiction, and the score is 10 vs 10.

I had predicted the single-shot score would drop to 9. It did not: that answer had only ever been partial, so it was never counted as correct. A small erratum, but it is the kind that is easy to make when you reason about scores instead of recomputing them.

{{< folio-card label="Erratum / how far these numbers go" >}}

- Twelve answerable questions. A one-question gap is noise; I claim "no measurable advantage", not "RAG is better".
- One run per configuration. Temperature 0 plus caching makes runs reproducible, not variance-free.
- The questions were written by an AI that had already seen the documents: semi-blind, not blind. A human who has never read the contracts should write the next set.
- The judge is an LLM. It was calibrated on a locked set and red-teamed, but it is still one model.
- Costs are list-price equivalents. The project actually ran on flat-rate plans.

{{< /folio-card >}}

## Prediction / result / correction

**Prediction.** A blind set stays blind as long as no answers leak into the prompts.

**Result.** Without a single leaked answer, the blind set had drifted into a development set, one decision at a time. On fresh questions, a 5-question lead turned into a 1-question deficit, then a tie once a label I had got wrong was arbitrated.

**Correction.** A blind set is a consumable. Every decision made while looking at it uses some of it up. The honest unit is not "the blind score" but "the first run on a set that no decision has touched". After that run, the set is spent too.

> A held-out set is only held out until you learn from it.

The agent is not useless. Its wins on blind v1 were questions whose answer sat in a second clause the first retrieval never surfaced, and that mechanism is real. What blind v2 showed is that such questions are not common enough to pay for on every request. Routing hard questions to the agent now has a price tag: roughly three times the cost for no measured gain on everything else.

## Unpaid debt

- A human-written blind v3: questions from someone who has not read the contracts.
- A router that sends only comparisons and multi-condition questions to the agent, measured against both baselines.

The next plate moves from "is it right" to "is it allowed": what happened when the same agent was asked, politely and then less politely, for data that was not its user's. That is [Plate II: The Model Asked, the Tool Said No](/posts/eval-field-notes-02-the-tool-said-no/).
