# Evaluating RAG applications

A RAG app works in two steps:

1. **Retrieve** : find text that may answer the question.
2. **Generate** : write an answer from that text.

Both steps can fail. So we measure both steps, not only the final answer.

## What can break

Our example question:

> Which room is the NLP talk in?

The correct answer is *Room 1*.

Three ways to get a bad answer:

| What happened | Where is the problem? |
| --- | --- |
| Search returns talks about MLOps. Room 1 is never mentioned. | Retrieval. The model cannot answer. |
| Search returns the right talk, but the model says *Room 2*. | Generation. The model made it up. |
| Search returns the right talk **and** nine wrong ones. The model picks a wrong room. | Retrieval. Too much noise. |

A single score like "is the answer correct?" gives you `0` in all three cases. It does not tell you what to fix. This is why we use separate metrics.

## Retrieval metrics

These look at the text we found, not at the answer.

### Context recall

**Question it asks:** did we find all the information we need?

**It needs:** the retrieved text and a reference answer.

**How it is measured:** split the reference answer into small facts. For each fact, check if the retrieved text contains it. The score is the number of facts found, divided by the number of all facts.

**Example**

Reference answer: *"The NLP talk is in Room 1 and starts at 10:00."*

Facts:
1. It is in Room 1.
2. It starts at 10:00.

The retrieved text only says *"NLP talk — Room 1"*. So we found 1 fact of 2.

Score: **0.5**

Low recall means the answer is impossible. The model does not have the information.

### Context precision

**Question it asks:** are the useful chunks at the top of the list?

**It needs:** the retrieved text and a reference answer (or a reference list of chunks).

**How it is measured:** mark every retrieved chunk as useful or not useful. The score is high when the useful chunks come first, and low when they come last.

**Example**

We retrieve three chunks:

1. A talk about MLOps — not useful
2. A talk about data engineering — not useful
3. The NLP talk — useful

The useful chunk is last, so precision is low. The answer may still be correct, but the model had to read a lot of noise first. Small models often fail here.

## Generation metrics

These look at the answer.

### Faithfulness

Also called **groundedness**.

**Question it asks:** is every part of the answer supported by the retrieved text?

**It needs:** the retrieved text and the answer. **It does not need a reference answer.** This is important: you can run it in production, on real user questions, where no correct answer exists.

**How it is measured:** split the answer into claims. Check each claim against the retrieved text. The score is the number of supported claims, divided by the number of all claims.

**Example**

Retrieved text: *"NLP talk — Room 1, 10:00."*

Answer: *"The NLP talk is in Room 1 and starts at 09:00."*

Claims:
1. It is in Room 1. → supported ✅
2. It starts at 09:00. → not supported ❌

Score: **0.5**

This is a hallucination. The model invented the time. Faithfulness is the metric that finds it.

### Answer relevancy

**Question it asks:** does the answer really answer the question?

**It needs:** the question and the answer.

**How it is measured:** an LLM reads the answer and writes questions that this answer would fit. Then we compare those questions with the real question. If they are close, the score is high.

**Example**

Question: *"Which room is the NLP talk in?"*

| Answer | Score | Why |
| --- | --- | --- |
| *"Room 1."* | high | It answers the question. |
| *"The NLP talk is very popular."* | low | True, but it is not an answer. |
| *"I do not have this information."* | low | It avoids the question. |

An answer can be faithful and still useless. Faithfulness and relevancy find different problems.

### Answer correctness

**Question it asks:** does the answer match the reference answer?

**It needs:** the answer and a reference answer.

**How it is measured:** compare the facts in the answer with the facts in the reference. Missing facts and wrong facts both lower the score.

This is the classic "is it right?" metric. It is useful, but only on a test dataset, because you need the correct answer first.

### Summary

| Metric | Needs a reference answer? | What it finds |
| --- | --- | --- |
| Context recall | yes | The search missed the information. |
| Context precision | yes | The search returned too much noise. |
| Faithfulness | **no** | The model made something up. |
| Answer relevancy | no | The answer does not fit the question. |
| Answer correctness | yes | The final answer is wrong. |

## What to check first

Fix problems in this order. Do not change everything at the same time.

1. **Context recall is low** → fix retrieval. Try other chunk sizes, another embedding model, a bigger `top_k`, or a better search query. Do not touch the prompt yet — the prompt cannot fix missing text.
2. **Recall is good, faithfulness is low** → fix generation. Change the prompt ("answer only from the text below"), or try another model.
3. **Faithfulness is good, relevancy is low** → the answer is true but not helpful. Fix the prompt or the answer format.

## What your dataset needs

The dataset looks like the ones in lesson 01 and lesson 02:

```json
{
  "inputs": {"question": "Which room is the NLP talk in?"},
  "outputs": {"answer": "Room 1."}
}
```

But the target must return **two** things:

```python
{
    "answer": "The NLP talk is in Room 1.",
    "retrieved_contexts": ["NLP talk — Room 1, 10:00.", "..."],
}
```

The signature is still `inputs: dict -> outputs: dict`, like in the other lessons. Only the `outputs` dict is bigger. Most RAG metrics cannot work without `retrieved_contexts`.

## Two kinds of evaluators

**Deterministic evaluators** — exact match, substring match, ROUGE. They are cheap and fast. They give the same score every time. But they are strict: a good answer with other words gets `0`.

**LLM judges** — faithfulness, relevancy, correctness. They understand meaning, so they are more flexible. But they cost money, they are slow, and they can give a different score on the next run.

Start with deterministic evaluators. Use an LLM judge only when you cannot measure the thing in Python. Always read a few judge scores by hand before you trust them.

## Where Ragas fits

[Ragas](https://docs.ragas.io) is an open source library. It gives you these metrics, so you do not have to write the judge prompts yourself. You can use Ragas metrics **inside** a Langfuse experiment.

The metric names in Ragas:

| Metric here | Ragas class |
| --- | --- |
| Context recall | `ContextRecall` |
| Context precision | `ContextPrecisionWithReference` |
| Faithfulness | `Faithfulness` |
| Answer relevancy | `AnswerRelevancy` |
| Answer correctness | `AnswerCorrectness` |

You give Ragas your own LLM and embedding model.
