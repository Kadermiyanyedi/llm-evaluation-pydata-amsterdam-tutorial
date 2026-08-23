# Evaluating a simple LLM application

You change a prompt. Is the application better now?

Trying one question does not answer that. The model may be better on your question and worse on ten others. Evaluation replaces "it looks good" with a number you can compare.

## The three parts

Every evaluation in this tutorial has the same three parts.

| Part | What it is |
| --- | --- |
| **Dataset** | Test examples. Each one has an input and the answer we expect. |
| **Task** | The application under test. It takes one dataset item and returns an output. |
| **Evaluator** | A function that looks at the output and returns a score. |

An **experiment** runs the task over the whole dataset, then runs the evaluators on every output.

A dataset item looks like this:

```json
{
  "input": { "question": "What does RAG stand for?" },
  "expected_output": { "answer": "retrieval-augmented generation" }
}
```

## The evaluator in this lesson

`contains_reference` checks whether the expected text appears in the answer.

It is **deterministic**: the same answer always gets the same score. It costs nothing and runs instantly.

It is also strict. A correct answer written in other words scores `0`:

| Answer | Score | Why |
| --- | --- | --- |
| *"RAG stands for retrieval-augmented generation."* | 1 | The expected text is there. |
| *"RAG means retrieving documents and generating from them."* | 0 | Correct, but the words differ. |

That weakness is the point. Start with the cheap check, and only reach for an LLM judge when you can see the cheap check failing on good answers.

`pass_rate` is a second kind of evaluator. It runs once for the whole experiment instead of once per item, and reports how many examples passed.

## Change one thing at a time

The lesson runs two comparisons:

1. same dataset, same model, **two prompts**,
2. same dataset, same prompt, **two models**.

Never change both at once. If you swap the prompt and the model together and the score moves, you have learned nothing about which one caused it.

## Prompts live outside the code

The prompts are stored in Langfuse, not in the notebook.

This is not tidiness. If a prompt is in your code, changing it means deploying your application again. When it is managed outside, you can create a new version, point a **label** such as `production` at it, and the running application picks it up.

Old versions stay. You can compare two versions, and you can move the label back if the new one is worse.

## Traces

A **trace** is the record of one run: what went in, what each step did, what came out, how long it took and how many tokens it used.

You need it when a score is bad and you want to know why. The number tells you something is wrong. The trace tells you where.
