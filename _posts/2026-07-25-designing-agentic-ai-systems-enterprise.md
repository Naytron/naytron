---
title: "Designing Agentic AI Systems That Survive the Enterprise"
date: 2026-07-25
tags: ["Agentic AI", "Demos & Reference Architectures"]
description: "Agents are the most exciting — and most over-hyped — pattern in enterprise AI. A reference architecture for agentic systems on Azure, plus the guardrails that separate production agents from expensive chaos."
---

"Let's just make it an agent" has become the "let's just use microservices" of this decade — a reasonable architecture reflexively applied to problems that don't need it, with all the operational cost and none of the payoff.

I've now helped several enterprises take agentic systems to production on Azure, and the pattern that separates the successes from the stalled pilots isn't model choice or framework choice. It's **architectural discipline**. Here's the reference architecture I keep coming back to.

## First: do you actually need an agent?

An agent is a system where the model **decides the control flow** — which tools to call, in what order, and when the task is done. That autonomy is powerful and expensive. My decision test:

- **Known, fixed steps?** Build a workflow with LLM calls inside it. Deterministic, cheap, debuggable.
- **Dynamic steps, bounded toolset?** A single agent with function calling.
- **Multiple distinct domains of responsibility?** Multi-agent — but only when a single agent's tool list and instructions become unmanageably long.

If you can draw the flowchart in advance, you don't need an agent. Write the flowchart as code and put model calls in the boxes. Save agents for problems where the path genuinely can't be known ahead of time.

## The reference architecture

For systems that do warrant agents, this is the shape I deploy on Azure:

```text
┌──────────────┐     ┌───────────────────────────────┐
│   Client /   │────▶│  API layer (Container Apps)   │
│   Channel    │     │  auth, sessions, rate limits  │
└──────────────┘     └──────────────┬────────────────┘
                                    ▼
                     ┌───────────────────────────────┐
                     │  Orchestrator (Foundry Agent  │
                     │  Service / Semantic Kernel)   │
                     │  planning · tool routing      │
                     └──────┬──────────────┬─────────┘
                            ▼              ▼
                   ┌──────────────┐  ┌──────────────┐
                   │ Tools (APIs, │  │  Knowledge   │
                   │ Functions,   │  │  (AI Search, │
                   │ MCP servers) │  │  RAG stores) │
                   └──────────────┘  └──────────────┘
                            │              │
                            └──────┬───────┘
                                   ▼
                     ┌───────────────────────────────┐
                     │ Cross-cutting: Entra ID,      │
                     │ Content Safety, App Insights, │
                     │ Cosmos DB (state/memory)      │
                     └───────────────────────────────┘
```

Key decisions embedded in that diagram:

**The orchestrator is a service, not a script.** Azure AI Foundry Agent Service gives you managed agent runtime with threads, tool invocation, and state. If you need heavier customization, Semantic Kernel on Container Apps gives you the same shape with more control. Either way, agent state lives in a durable store (Cosmos DB is my default), never in process memory.

**Tools are contracts.** Every tool an agent can invoke is a typed, versioned API with narrow scope — increasingly exposed via Model Context Protocol (MCP) so the same tool works across agent runtimes. Write tool descriptions like you're writing docs for a brilliant intern: precise, with examples, and honest about limitations.

**Knowledge is retrieval, not memorization.** Ground the agent with Azure AI Search (hybrid + semantic ranking) rather than stuffing context. Retrieval quality caps agent quality — invest there before you invest in prompt cleverness.

## Guardrails that earn the "enterprise" label

The difference between a demo agent and a production agent is what happens when things go wrong:

1. **Identity per agent.** Each agent gets its own Entra Agent ID / managed identity. Tools authorize the *agent's* identity with least privilege — never a shared service principal with god-mode.
2. **Budget caps.** Max steps per task, max tokens per session, max spend per tenant per day. Agents fail by looping; make loops financially impossible.
3. **Human-in-the-loop gates.** Any irreversible action (send the email, issue the refund, modify the record) requires explicit approval above a risk threshold. Design the approval UX early — bolting it on later restructures everything.
4. **Full trace capture.** Every planning step, tool call, and intermediate result flows to Application Insights. When an agent misbehaves — and it will — you need the transcript, not a guess.

## Evaluate the trajectory, not just the answer

Single-model evaluation asks "was the answer right?" Agent evaluation adds "was the *path* right?" Track tool-selection accuracy, redundant call rate, and task completion against a scripted scenario suite. Run it on every change to instructions or tool definitions, exactly like the CI evaluation gates from [the Foundry post]({% post_url 2026-07-18-azure-ai-foundry-playground-to-production %}).

## Start smaller than you think

The successful pattern I've watched repeat: one agent, three to five tools, one domain, human approval on everything consequential. Ship that. Learn how it fails. Widen autonomy only where the evaluation data says you've earned it.

Multi-agent orchestras, autonomous background workers, computer-use agents — they're real, and they're coming to your roadmap. But they're all built on the same foundation: typed tools, scoped identity, budget caps, trajectory evals. Master the boring parts and the exciting parts follow.
