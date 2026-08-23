# Evaluating an agent

A chain does what you wrote. An agent decides for itself.

It chooses which tool to call, what arguments to pass, whether to call another one, and when to stop. That freedom is why agents are useful, and it is also why a bad result has more possible causes.

## What can break

Our example request:

> I am interested in NLP talks on Thursday.

The agent should find the three NLP talks on Thursday and save them to a file.

| What happened | Where is the problem? |
| --- | --- |
| The agent never called `search_talks`. It answered from memory and invented a talk. | It did not use its tools. |
| It called `search_talks("mlops")`. | Wrong arguments. |
| It found the right talks, listed them, and asked *"shall I save these?"* instead of saving. | It stopped too early. |
| It saved the Friday NLP talk too. | It did not filter by day. |

A single "is the answer correct?" score gives you one number for all four. So we measure more than the final text.

## Score the words and the actions

The agent produces two things, and we score them separately.

| Evaluator | Looks at | Catches |
| --- | --- | --- |
| `finds_expected_talks` | the answer text | wrong or missing talks in the reply |
| `saves_expected_talks` | the CSV file | the agent talked but never acted |

A talk only counts as found when its title, speaker, day and start time all appear in the answer. Checking the four fields separately, instead of matching one formatted line, keeps the score honest without failing over a stray space.

## Why a file is a good thing to evaluate

Text is hard to score. A file is easy.

The rows are either there or not. The columns are either right or not. No judge is needed, no prompt to tune, and the same output always gets the same score.

This is worth remembering beyond agents: when you can make your application produce something structured, evaluation gets much cheaper.

## Two experiments

Both experiments compare the **same two models**. Only the tools change between them.

1. **Without tools.** The whole programme is in the system prompt, so the model can answer from what it was given.
2. **With tools.** The programme is not in the prompt. The model has to search for the talks and save them itself.

Comparing "with tools" against "without tools" would not be a fair experiment, because only one of them can write a file. So the tools stay fixed inside each experiment, and the model is the thing that varies.

## What the results usually show

With the programme in its prompt, a model answers well. The programme is small.

That is the honest result, and it points at the next lesson. Tools let the agent **act**, not just answer. And when the data no longer fits in a prompt, you need retrieval, which is lesson 03.
