# LLM Evaluation in Production: A/B Testing and Observability

Hands-on material for the PyData Amsterdam tutorial.

This repository teaches how to evaluate LLM applications with repeatable
datasets, controlled experiments, and Langfuse observability. It includes three
small examples and one 60-minute end-to-end customer-support tutorial.

The central question is:

> When we change a prompt, model, tool, or retrieval system, how do we know the
> application is actually better?

## Tutorial

### Customer support

A 60-minute end-to-end application that brings the examples together: managed
prompts, deterministic tools, in-memory and Qdrant RAG, Langfuse tracing,
dataset experiments, metrics, and A/B testing.

- [Tutorial guide](tutorial/README.md)
- [Notebook](tutorial/customer_support.ipynb)
- [Evaluation dataset](tutorial/evaluation_dataset.json)

## Examples

The examples are independent lessons. Each folder includes its own
README and notebook.

### 01 — Simple LLM application

Learn the basic evaluation loop: datasets, tasks, evaluators, traces, and
controlled prompt or model comparisons.

- [Overview](examples/01_simple/README.md)
- [Notebook](examples/01_simple/langchain.ipynb)

### 02 — Agent

Evaluate agent behavior, tool calls, actions, and structured outputs.

- [Overview](examples/02_agent/README.md)
- [Notebook](examples/02_agent/langchain.ipynb)

### 03 — RAG

Evaluate retrieval and generation separately with in-memory and Qdrant vector
stores.

- [Overview](examples/03_rag/README.md)
- [In-memory notebook](examples/03_rag/langchain_with_inmemory.ipynb)
- [Qdrant notebook](examples/03_rag/langchain_with_qdrant.ipynb)

## Quick start

Requirements:

- Python 3.13
- [uv](https://docs.astral.sh/uv/)
- an OpenRouter API key
- a Langfuse project
- Docker for Qdrant examples

Install the environment and create your local configuration:

```bash
uv sync --locked
cp .env.example .env
```

Add your real keys to `.env`:

```env
OPENROUTER_API_KEY=
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_BASE_URL=https://cloud.langfuse.com
QDRANT_URL=http://localhost:6333
```

Do not commit `.env`.

Start Jupyter from the repository root:

```bash
uv run jupyter lab
```

## Langfuse prompts

The notebooks use Langfuse Prompt Management instead of hardcoded system
prompts. Create the prompt names and labels described inside the notebook you
want to run.

The customer-support tutorial requires:

| Prompt                                 | Labels                                     |
| -------------------------------------- | ------------------------------------------ |
| `pydata-customer-support`              | `baseline` and `candidate` on two versions |
| `pydata-customer-support-groundedness` | `production`                               |

The exact prompt bodies and UI steps are in
[`tutorial/customer_support.ipynb`](tutorial/customer_support.ipynb).

## Qdrant

The simple, agent, and in-memory RAG notebooks do not require Qdrant. Start the
root Qdrant service before running a Qdrant notebook:

```bash
docker compose up -d
docker compose ps
```

Qdrant is available at `http://localhost:6333`; its dashboard is at
`http://localhost:6333/dashboard`.

Stop it when finished:

```bash
docker compose down
```

For detailed setup, the full workshop flow, current experiment interpretation,
and troubleshooting, see [`tutorial/README.md`](tutorial/README.md).
