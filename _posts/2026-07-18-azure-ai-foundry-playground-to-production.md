---
title: "From Playground to Production with Azure AI Foundry"
date: 2026-07-18
tags: ["Azure AI Foundry", "Enterprise AI Patterns"]
description: "The playground demo is the easy part. Here's the path I walk with enterprise teams to take an Azure AI Foundry project from a promising prototype to a production workload — model selection, evaluations, deployment, and observability."
---

Every enterprise AI project I've worked on starts the same way: someone builds a demo in an afternoon, leadership gets excited, and then the real question lands on the table — *"how do we actually ship this?"*

The gap between a playground prototype and a production workload is where most AI projects stall. Azure AI Foundry exists to close that gap, and after helping multiple enterprise teams walk this path, I've settled on a repeatable sequence. Here it is.

## Start with the model catalog — but decide with data

Foundry's model catalog gives you frontier models (GPT-4o class and beyond), open-weight options like Phi and Llama, and task-specific models behind one consistent interface. The mistake teams make is picking a model by reputation instead of by measurement.

My rule: **prototype with the strongest model, then evaluate your way down.** Build your scenario against the best available model to establish a quality ceiling. Then run the same evaluation set against smaller, cheaper models and let the numbers tell you where the acceptable trade-off lives. Teams are routinely surprised how often a small model handles a well-scoped task at a fraction of the cost and latency.

## Make evaluations your definition of done

This is the single biggest culture shift: **in AI engineering, evaluations are your unit tests.** If you can't measure quality, you can't ship responsibly — and you definitely can't upgrade models later without fear.

Foundry's built-in evaluators cover groundedness, relevance, coherence, and safety dimensions. Start there, then add scenario-specific evaluators that encode *your* definition of correct:

```python
from azure.ai.evaluation import evaluate, GroundednessEvaluator

groundedness = GroundednessEvaluator(model_config)

results = evaluate(
    data="eval_dataset.jsonl",
    evaluators={"groundedness": groundedness},
    evaluator_config={
        "groundedness": {
            "column_mapping": {
                "query": "${data.query}",
                "context": "${data.context}",
                "response": "${data.response}",
            }
        }
    },
)
```

Build an evaluation dataset from real (sanitized) user queries as early as you can — even 50 well-chosen examples beats a hundred synthetic ones. Wire the evaluation run into CI so every prompt change, model swap, or retrieval tweak produces a scorecard before it merges.

## Treat prompts and configuration as code

Playgrounds encourage tinkering; production demands versioning. Everything that shapes model behavior — system prompts, temperature, retrieval parameters, tool definitions — belongs in source control and flows through the same PR review as application code. A prompt change *is* a behavior change. Review it like one.

## Deploy behind an abstraction you control

When you move to production, put a thin gateway between your application and the model endpoint. Azure API Management's AI gateway capabilities give you token-based rate limiting, semantic caching, and load balancing across deployments. Even if you skip APIM on day one, keep the abstraction in your own code. You *will* change models — the teams that planned for it upgrade in days, the ones that didn't spend a quarter untangling hard-coded assumptions.

Two deployment details that bite people:

- **Provisioned throughput vs. pay-as-you-go.** Start pay-as-you-go, measure real token consumption, then buy provisioned capacity for the predictable baseline and let burst traffic spill over.
- **Regional strategy.** Model availability varies by region. Verify your compliance boundary and your model requirements intersect *before* you promise a go-live date.

## Observability: trace every token

Production AI without tracing is flying blind. Foundry integrates with Application Insights, and the OpenTelemetry-based tracing in the Azure AI SDKs captures each call's prompt, completion, token counts, and latency. At minimum, put these on a dashboard from day one:

| Signal | Why it matters |
|--------|----------------|
| Tokens per request (p50/p95) | Cost forecasting and prompt bloat detection |
| Latency (p95) | User experience and timeout tuning |
| Groundedness score trend | Quality drift after content or model changes |
| Content-filter block rate | Safety posture and false-positive tuning |

## The path, condensed

1. Prototype against the strongest model to set a quality ceiling
2. Build a real evaluation dataset and make it your definition of done
3. Version prompts and config like code, gated by CI evaluations
4. Deploy behind a gateway you control; plan for model swaps
5. Trace everything, dashboard the four signals above

None of this is glamorous, and that's the point — production AI is an engineering discipline, not a demo. Foundry gives you the platform pieces; the sequence above is how you assemble them into something your enterprise can actually run.

Next up in this series: what happens when a single model call isn't enough, and you need *agents* — with all the architectural implications that word carries.
