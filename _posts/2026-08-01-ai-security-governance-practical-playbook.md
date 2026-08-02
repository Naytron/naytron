---
title: "AI Security & Governance: A Practical Playbook"
date: 2026-08-01
tags: ["AI Security & Governance", "AI for Public Sector"]
description: "Security reviews kill more AI projects than technical failures do. A practical playbook for getting enterprise AI through security and governance — identity, data boundaries, content safety, and the paper trail — with notes for public sector teams."
---

Here's an uncomfortable observation from the field: more enterprise AI projects die in security review than die from technical failure. Not because the security teams are wrong — but because the AI teams showed up without answers to entirely predictable questions.

This is the playbook I use to make sure those questions have answers *before* they're asked. It maps cleanly to Azure services, but the principles travel anywhere.

## 1. Identity: no keys, ever

The fastest way to fail an AI security review is an API key in an app setting. The rule is absolute: **managed identity and Entra ID for every hop.**

- App → Azure OpenAI / AI Foundry: managed identity with `Cognitive Services OpenAI User` — nothing broader.
- App → AI Search, Storage, Cosmos DB: managed identity, scoped RBAC roles, per resource.
- Agents get their **own** identity (Entra Agent ID), distinct from the app's, so agent actions are attributable and individually revocable.

Disable local auth (key-based access) on every AI resource that supports it. If a key can't exist, it can't leak.

```bash
az cognitiveservices account update \
  --name my-foundry-resource \
  --resource-group rg-ai-prod \
  --set properties.disableLocalAuth=true
```

## 2. Data boundaries: decide what the model may see

Every governance conversation eventually reaches the same question: *what data can reach the model, and where does it go afterward?* Have crisp answers:

- **Prompts and completions are not training data.** Azure OpenAI doesn't use your data to train foundation models — put the official documentation link in your review packet, because the question will come from three different directions.
- **Draw the RAG trust boundary.** Retrieval must respect the *user's* permissions, not the app's. Use security trimming in AI Search so users can only retrieve documents they're already entitled to read. An AI system that leaks documents through summaries is still a data breach.
- **Classify before you index.** Content labeled Confidential or above needs an explicit decision — and an owner's signature — before it enters a vector index. Microsoft Purview labels give you the machinery; the discipline is on you.
- **Log prompts as sensitive data.** Prompt logs contain whatever users typed into them. Store them with the same controls as the most sensitive data they might contain.

## 3. Content safety: layered, tuned, and tested

Azure AI Content Safety gives you the layers; your job is tuning them to the workload:

1. **Input filtering** — block prompt-injection attempts and abusive input with Prompt Shields.
2. **Output filtering** — category filters (hate, violence, sexual, self-harm) at severity thresholds appropriate to the audience.
3. **Groundedness detection** — for RAG workloads, flag responses that drift from retrieved sources; that's your hallucination alarm.
4. **Blocklists and custom categories** — for domain-specific terms the generic filters can't know about.

Then **red-team it**. Run adversarial prompts against the full stack before go-live, and keep the transcript — passing red-team results are the single most persuasive artifact you can bring to a security review. The PyRIT toolkit automates a solid first pass.

## 4. Governance: the paper trail is the product

Security gets you deployed; governance keeps you deployed. The minimum viable paper trail:

| Artifact | Answers the question |
|----------|---------------------|
| Use-case register | What AI runs where, owned by whom, at what risk tier? |
| Model + prompt version history | What changed, when, and who approved it? |
| Evaluation scorecards per release | How do we know quality didn't regress? |
| Incident runbook | What happens when the model misbehaves? |
| Access review cadence | Who can touch the AI resources, verified how often? |

If you're subject to the EU AI Act, NIST AI RMF, or ISO/IEC 42001, you'll recognize these artifacts — they're the common denominator. Build the register once, map it to frameworks as needed.

## Public sector notes

Government teams carry extra weight, and it changes the architecture:

- **Cloud selection is a day-zero decision.** Azure Government and sovereign cloud options differ in model availability and feature lag behind commercial. Verify the models your design assumes are actually available in your boundary *before* the architecture review, not after.
- **FedRAMP / IL boundaries shape RAG.** Data can't leave the authorization boundary — retrieval indexes, embedding pipelines, and evaluation datasets all inherit the same impact level as their source data.
- **Explainability is often statutory.** Decisions affecting citizens may require documented human review. Design the human-in-the-loop gate as a legal requirement, not a UX preference.
- **Procurement outlives hype.** Choose patterns (managed identity, RBAC, standard APIs) that survive vendor and model churn, because your compliance artifacts will outlive the model version you launched with.

## The checklist

Before your next AI security review, be able to say yes to all of these:

- [ ] Every hop authenticates with managed identity; local auth disabled
- [ ] Agents have distinct, least-privilege identities
- [ ] RAG retrieval enforces user-level permissions (security trimming)
- [ ] Sensitive-label content has documented approval before indexing
- [ ] Content safety layers tuned and red-team tested, transcripts kept
- [ ] Use-case register, version history, and eval scorecards exist
- [ ] Incident runbook names a human owner

None of this slows you down — it's what *prevents* the slowdown, because the alternative is discovering these requirements one security-review rejection at a time. Govern early, ship faster.
