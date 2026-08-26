# Customer support: production-like A/B testing

This capstone joins the ideas from lessons 01–03 in one product story. MaviSepet,
a fictional online shop, has a customer-support assistant. The assistant retrieves
company policy, looks up orders, decides whether to create a support ticket, and
answers the customer.

The workshop is explicit about one limitation: there is no real product traffic.
The notebook simulates production traffic. It demonstrates the same mechanics—
variant assignment, traces, scores, guardrails, comparison and rollback—but it
does not claim a real-world causal effect on customer satisfaction.

## Learning path

1. Inspect policies, orders and evaluation cases.
2. Run prompt A as the baseline.
3. Define deterministic product metrics.
4. Run a paired offline experiment on A and B.
5. Route synthetic traffic with stable 50/50 assignment.
6. Compare quality, policy violations, latency and cost.
7. Make a rollout, canary or rollback decision.

## Two run modes

- **Demo mode (default):** deterministic and free. It guarantees a reliable live
  workshop even when Wi-Fi or model providers fail.
- **Live mode:** uses OpenRouter for model calls and Langfuse for prompt versions,
  traces and experiment results. Set `WORKSHOP_MODE=live` after configuring `.env`.

Demo mode is a teaching simulator, not evidence about a real model. Presenters
should say this aloud before running it.

## Files

| File                     | Purpose                                                                                        |
| ------------------------ | ---------------------------------------------------------------------------------------------- |
| `customer_support.ipynb` | Complete presenter-led capstone; all essential application and evaluation code is visible here |
| `dataset.json`           | Evaluation and synthetic traffic cases                                                         |
| `policies.json`          | Small RAG knowledge base                                                                       |
| `orders.json`            | Fake order system                                                                              |

## Metrics

| Metric               | Why it matters                                         |
| -------------------- | ------------------------------------------------------ |
| `retrieval_recall`   | Did retrieval return every required policy?            |
| `action_correctness` | Did the assistant create a ticket exactly when needed? |
| `policy_compliance`  | Did it avoid prohibited promises?                      |
| `answer_coverage`    | Did it include the required next-step facts?           |
| `latency_ms`         | Did quality make the experience too slow?              |
| `estimated_cost_usd` | Is the improvement affordable?                         |

These are intentionally deterministic. Add an LLM judge only after inspecting
where rule-based checks reject genuinely good answers.
