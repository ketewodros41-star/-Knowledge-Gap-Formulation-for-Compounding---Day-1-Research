# Answering: How Do Attention Dynamics Cause Instruction Drift?

---

## The Short Answer

When a prompt gets long, an instruction written near the top has to **compete** with everything that comes after it at every single generation step. That competition is what causes drift. The model doesn't freeze your instruction in place — it re-evaluates what matters most each time it picks the next token.

---

## What "Attention Dynamics During Inference" Actually Means

A transformer doesn't read the prompt once and lock in the rules. At each decoding step, the model runs a **query** against all prior tokens and assigns attention weights — essentially deciding: *"given what I'm about to write, which parts of the context are most relevant right now?"*

Three things shape the outcome of that decision:

### 1. Position and Recency (Salience)
Tokens that are **close to the current generation point** tend to win more attention. A constraint written 2,000 tokens ago has to compete against everything written after it. As context grows, a distant instruction's influence weakens — not because the model forgets it, but because nearby evidence keeps outbidding it.

> Xiao et al. (2023) showed that early tokens can act as "attention sinks" — receiving disproportionate attention mass — but this doesn't mean every early instruction stays influential across a long completion.

### 2. Learned Instruction Hierarchy (Priority)
Not all text is treated equally. If a model has been trained to treat system-level instructions as **privileged**, those instructions start with a structural advantage regardless of position. If it hasn't, then a policy in the system prompt is just more text — and it can be overridden by strong local patterns.

> Wallace et al. (2024) showed this directly: instruction-following isn't purely semantic. It depends on whether the model has learned to respect placement and priority signals.

### 3. Local Decoding Pressure (Override)
Even when an instruction is present and understood, the model's next-token prediction can be pulled toward a **strong local pattern** in the surrounding context. If the enrichment context looks like confident, assertive language, the model may continue in that tone — even if a policy somewhere says "hedge when uncertain."

This is the subtlest mechanism. The instruction doesn't disappear; it just loses the local competition at the exact moment it matters.

---

## So What Determines Whether an Instruction Is Preserved or Overridden?

It's not one thing. It's all three acting together:

| Mechanism | Preserves Instruction | Overrides Instruction |
|---|---|---|
| **Salience** | Instruction is near generation site | Instruction is distant in a long prompt |
| **Learned Priority** | Model trained to treat it as privileged | Model treats it as ordinary text |
| **Local Decoding Pressure** | Surrounding context is neutral or supportive | Surrounding context pulls toward a competing pattern |

Instruction drift happens when all three factors work against you at once.

---

## The Practical Implication

Writing the instruction once at the top of a long prompt is often not enough. The safest pattern is to **reintroduce the constraint immediately before generation** — not because the model forgot it, but because proximity shifts the local competition in the instruction's favor.

The strongest evidence for this isn't an attention diagram. It's an **ablation**: run the same input with the policy only at the top vs. the policy repeated right before generation, and measure whether output behavior changes. If it does, proximity is doing real work.

---

## One-Sentence Summary

Instructions drift in long-context generation because a transformer re-evaluates what matters at every token step — and a distant constraint loses influence not through forgetting, but through being outcompeted by salience, weak learned priority, and strong local decoding pressure.

---

## Sources

- Xiao, G., et al. (2023). *Efficient Streaming Language Models with Attention Sinks.*
- Wallace, E., et al. (2024). *The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions.*
