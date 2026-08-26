# Evaluating LLM applications

A hands-on tutorial for PyData Amsterdam.

## The problem

Building a first LLM application is easy. Knowing whether you made it better is not.

You change a prompt, try two questions, and it feels good. You swap the model because a newer one came out. You add retrieval. Every one of those changes could have made the application worse, and without evaluation you find out when a user does.

## The solution

Evaluation turns "it feels better" into a number you can compare.

It always has the same three parts: a **dataset** of test examples, the **application** under test, and an **evaluator** that scores what came out. Run them together and you get a score. Change one thing, run again, and the two numbers tell you whether the change helped.

The tools change, the idea does not. These lessons use Langfuse, and the same three parts fit any evaluation platform.

## The three lessons

Each lesson stands on its own, and each one adds a harder thing to measure.

| Lesson                                               | Application                      | The new problem                                                                            |
| ---------------------------------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| [01 simple](examples/01_simple/)                     | A prompt and a model             | What are a dataset, a task and an evaluator?                                               |
| [02 agent](examples/02_agent/)                       | An agent with tools              | The agent decides its own steps, so we score its actions, not only its words.              |
| [03 rag](examples/03_rag/)                           | Retrieval plus generation        | Two parts can fail, so we score them separately.                                           |
| [04 customer support](examples/04_customer_support/) | End-to-end production simulation | Offline evaluation, A/B routing, observability and rollout decisions in one product story. |

Every lesson has a README that explains the ideas, and a notebook that runs them.

## Requirements

| What                          | Why you need it                                                                                                                      |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| A free Langfuse Cloud project | Holds the prompts, datasets, traces and scores. You will spend as much time here as in the notebooks.                                |
| An OpenRouter key             | One key for models from many providers, so swapping a model is a one line change. A full run of all three lessons costs a few cents. |
| Docker or Podman, optional    | Only for the Qdrant version of lesson 03. The other version needs nothing.                                                           |

Both keys go in a `.env` file, copied from `.env.example`.
