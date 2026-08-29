# Customer Support Tutorial

This folder contains the complete 60-minute tutorial for:

**LLM Evaluation in Production: A/B Testing and Observability**

The tutorial uses one fictional MaviSepet customer-support application from
start to finish. Agent, tools, and RAG make the application realistic, but they
are not the final goal. The main goal is to measure whether an LLM application
change is actually better.

- Notebook: [`customer_support.ipynb`](customer_support.ipynb)
- Evaluation data: [`evaluation_dataset.json`](evaluation_dataset.json)

## What you will learn

By the end of the notebook, you will be able to:

- trace a customer-support application with Langfuse,
- manage prompt versions outside Python code,
- add deterministic tools for private order data,
- build RAG first with an in-memory store and then with Qdrant,
- upload a reusable evaluation dataset to Langfuse,
- run the same dataset against two prompt variants,
- combine rule-based, LLM-as-a-judge, latency, token, and cost metrics,
- inspect failed traces before making a release decision,
- demonstrate stable user routing for a production-like A/B test.

## The customer-support scenario

MaviSepet is a fictional online store. The assistant receives questions such
as:

- “Please write a short apology for a late delivery.”
- “How many days do I have to return an unused product?”
- “How long does standard delivery normally take?”
- “Do the MaviSound Wave headphones have noise cancellation?”
- “What is the status of my order?”
- “Can I return this unused monitor?”
- “Can MaviSepet repair my coffee machine?”
- “Tell me the status and item name for this order.”

These questions cover general writing, company knowledge, private order data,
unknown information, edge cases, and safe abstention.

All order and policy data is local. No real customer service or customer data is
used.

## Mental model

```text
Customer input
      |
      +--> optional retrieval
      |       ├── LangChain InMemoryVectorStore
      |       └── QdrantVectorStore
      |                 |
      |                 +--> top two chunks
      |
      +--> Langfuse prompt compiled with {{context}}
                              |
                              v
                        LangChain agent
                        /             \
                  direct answer     tool request
                                          |
                                   deterministic data
                                          |
                                     final answer
                              |
                              v
             Langfuse trace + evaluation scores
```

## Where the agent is created

The notebook has one explicit agent factory:

```python
def build_support_agent(managed_prompt, tools: list, context: str):
    compiled_system_prompt = managed_prompt.compile(context=context)
    return create_agent(
        model=model,
        tools=tools,
        system_prompt=compiled_system_prompt,
    )
```

This combines:

1. the OpenRouter chat model,
2. the selected Langfuse prompt version,
3. the tools enabled for the current stage.

`agent.invoke(...)` runs the agent. If the tool list is empty, the model answers
directly. If tools are available, the model can select a tool, create its input,
read the deterministic output, and call the model again for the final answer.

RAG is prepared before the agent loop. Retrieved chunks are inserted into the
managed prompt through `{{context}}`. RAG is intentionally not a tool in this
notebook, so retrieval and tool use remain easy to explain separately.

## Stage 1: simple agent

Configuration:

```text
tools=[]
store_name=None
```

The assistant only receives the customer message and the Langfuse-managed
prompt. It can write a polite apology using the model's general language
ability, but it cannot know a private order status or current MaviSepet policy.

This stage creates the application baseline. In Langfuse, the trace contains a
model generation but no retrieval or tool steps.

## Stage 2: agent with deterministic tools

Two local tools are added:

- `get_order_status(order_id, email)` verifies the order ID and email before
  returning item and status.
- `check_return_eligibility(order_id, email)` applies the 14-day return rule to
  a verified order.

The model decides which tool to use, but it does not invent the tool result. The
Python function returns deterministic mock data.

For the saved order-status example, the agent correctly called
`get_order_status` and returned that order `ORD-1002`, containing a MaviKeys
keyboard, was **in transit**.

## Stage 3: in-memory RAG

The local knowledge base contains six documents:

- returns,
- shipping,
- products,
- warranty,
- membership,
- privacy.

The notebook demonstrates every RAG step:

1. prepare documents,
2. split them into overlapping word chunks,
3. create embeddings,
4. index the chunks in `InMemoryVectorStore`,
5. retrieve the two closest chunks,
6. compile them into the managed prompt,
7. generate a grounded answer.

In the saved run, six documents became twelve chunks. The return-policy query
retrieved both `returns` chunks and answered that an unused product can be
returned within 14 days, after requesting approval.

## Stage 4: the same RAG with Qdrant

This is a second RAG implementation, not a replacement for the in-memory stage.
The notebook indexes the same twelve chunks with the same embedding model in a
Qdrant collection named `pydata_customer_support`.

The same return-policy input is then sent through both backends:

| Backend | Retrieved source | Tokens | Estimated cost | Saved-run latency |
| --- | --- | ---: | ---: | ---: |
| Memory | `returns` | 300 | $0.000155 | 1440.7 ms |
| Qdrant | `returns` | 300 | $0.000155 | 2223.1 ms |

Both returned the same source and answer. Qdrant was slower in this single run,
but one request is not a benchmark. The useful result is that retrieval behavior
stayed consistent after changing the storage backend.

The saved notebook output confirms:

```text
Qdrant RAG available: True
RAG backend used by later prompt experiments: qdrant
Qdrant RAG executed: True
```

Start Qdrant from the repository root before running the notebook:

```bash
docker compose up -d
docker compose ps
```

The dashboard is available at `http://localhost:6333/dashboard`.

## Langfuse Prompt Management

The application system prompt is not stored as a Python string. The notebook
loads these text prompts from Langfuse:

| Prompt | Version/label | Purpose |
| --- | --- | --- |
| `pydata-customer-support` | version 1 / `baseline` | Helpful but loose application prompt |
| `pydata-customer-support` | version 2 / `candidate` | Stricter grounding, privacy, and abstention |
| `pydata-customer-support-groundedness` | `production` | LLM-as-a-judge prompt |

The prompt object is also linked to the generation trace. This makes it
possible to group outputs and scores by exact prompt version.

## What is traced

`run_support(...)` creates one top-level **Customer Support Request** agent
trace. It records:

- customer input,
- user ID and session ID,
- application stage and experiment variant,
- model name,
- prompt name and exact version,
- selected vector store,
- retrieval query and returned chunks,
- tool name, input, and output,
- final customer answer,
- latency,
- input/output/total token usage,
- estimated customer-facing model cost.

The trace separates retrieval, tool use, and model generation. A metric shows
that behavior changed; the trace helps locate where it changed.

## Evaluation dataset

[`evaluation_dataset.json`](evaluation_dataset.json) contains eight labelled
cases. Each item includes:

- `input`: message, user, session, and optional order credentials,
- `required_terms`: facts or behavior expected in the answer,
- `forbidden_terms`: claims that must not appear,
- `expected_tool`: the expected tool or no tool,
- `expected_sources`: required knowledge-base sources,
- `should_abstain`: whether the assistant should avoid a factual answer,
- `metadata.category`: the case type.

Expected values are never sent to the support agent. They are used only after
generation by the evaluators.

The notebook uploads the local file to a timestamped Langfuse Dataset, so a new
full run does not mix results with an older workshop run.

## Controlled prompt experiment

The experiment answers:

> Does a stricter support prompt reduce unsupported claims without hurting task
> success, tool use, latency, or cost?

The comparison keeps these fixed:

- dataset items,
- model and temperature,
- order tools and mock data,
- Qdrant store and retrieval top-k,
- evaluator functions.

Only the managed application prompt changes. Every item runs once with the
baseline prompt and once with the candidate prompt. This is a paired offline
experiment, not randomized live traffic.

## Metrics

| Metric | Type | What it checks |
| --- | --- | --- |
| `task_success` | Deterministic, rule-based | Required facts and safe abstention |
| `tool_correctness` | Deterministic, rule-based | Exact expected tool selection |
| `source_recall` | Deterministic, rule-based | Required source was retrieved |
| `safety` | Deterministic, rule-based | Forbidden exact claims are absent |
| `groundedness` | LLM-as-a-judge | Evidence supports factual claims |
| `latency_ms` | Measured | Customer waiting time |
| `total_tokens` | Measured | Model usage |
| `estimated_cost_usd` | Calculated | Approximate chat-model cost |

The exact-term checks are intentionally simple. They can mark a correct
paraphrase as a failure. `source_recall=1.0` also does not penalize irrelevant
extra chunks. Failed traces must be inspected before trusting an aggregate
score.

## Current saved experiment result

The notebook currently contains this completed Qdrant-backed run:

| Metric | Baseline | Candidate | Direction |
| --- | ---: | ---: | --- |
| Task success | 0.5000 | 0.5625 | Higher is better |
| Tool correctness | 0.875 | 0.750 | Higher is better |
| Source recall | 1.000 | 1.000 | Higher is better |
| Safety | 1.000 | 1.000 | Higher is better |
| Groundedness | 1.000 | 1.000 | Higher is better |
| Mean latency | 2237.9 ms | 1753.4 ms | Lower is better |
| Mean total tokens | 430.4 | 416.8 | Lower is better |
| Estimated cost/item | $0.000250 | $0.000227 | Lower is better |

Interpretation:

- candidate task success increased by 0.0625,
- candidate tool correctness decreased by 0.125,
- candidate latency was about 21.7% lower in this run,
- candidate used about 3.2% fewer tokens,
- estimated customer-facing model cost was about 9.2% lower,
- both variants passed the current simple safety, source-recall, and judge
  checks.

This is not enough evidence to promote the candidate. Task success is only
0.5625 and tool correctness is 0.75. The updated release gate therefore returns
`REVISE`, not `PROMOTE`. The next step is to inspect failed traces, improve one
thing, and rerun the same dataset.

Several failures are also measurement lessons:

- `sorry` versus `apologize` can fail an exact-term check despite equivalent
  meaning,
- `3–5 business days` versus `3 to 5 business days` can create a false failure,
- retrieving the expected source does not mean all retrieved chunks are
  relevant,
- an LLM judge can be too lenient and should be reviewed against human labels.

## From offline testing to an online A/B demo

The final section hashes `user_id` and assigns each user to a stable baseline or
candidate variant. The trace includes the selected variant, user, and session,
so production-like traffic can be filtered in Langfuse.

This cell demonstrates routing and observability only. A real online A/B test
also needs sufficient traffic, a predefined user outcome, privacy review,
statistical analysis, and a rollback owner.

## How the tutorial supports the talk abstract

| Abstract goal | Notebook implementation |
| --- | --- |
| Measure the effect of small LLM changes | Controlled baseline/candidate prompt comparison |
| Log user-facing outputs | Langfuse input, output, user, session, and prompt tracing |
| Compare prompt or model versions | Prompt version A/B experiment; model fields are also recorded |
| Use offline dataset experiments | Eight local cases uploaded to a Langfuse Dataset |
| Explain A/B testing | Paired offline experiment plus stable online routing demo |
| Track simple quality metrics | Task, tool, source, safety, and groundedness scores |
| Compare quality, latency, and cost | One summary table contains all three dimensions |
| Make a data-driven decision | Absolute release gates plus failed-trace diagnosis |
| Build repeatable workflows | Fixed data, deterministic tools, managed prompts, and idempotent Qdrant indexing |

The flow therefore matches the abstract well. Agent, tools, and RAG support the
evaluation story instead of replacing it. Roughly the final third of the
tutorial is dedicated to observability, experiments, metrics, failure analysis,
and the release decision.

## Run the tutorial

From the repository root:

```bash
uv sync --locked
cp .env.example .env
docker compose up -d
uv run jupyter lab
```

Create the three Langfuse prompt versions described in the notebook, open
`tutorial/customer_support.ipynb`, and run all cells from a clean kernel.

After changing the pinned Qdrant version, refresh the container with:

```bash
docker compose pull
docker compose up -d
```

## What to inspect in Langfuse

For a single request:

1. Open **Tracing**.
2. Select a **Customer Support Request** trace.
3. Inspect prompt version, retrieval, tools, model calls, answer, latency, and
   tokens.

For the offline experiment:

1. Open **Datasets**.
2. Select the timestamped `pydata-customer-support-*` dataset.
3. Open **Runs**.
4. Compare `prompt-a-baseline` and `prompt-b-candidate`.
5. Open low-scoring items and follow them to their traces.