# How Attention Dynamics Contribute to Instruction Drift in LLMs - and What to Do About It

**Author:** Kidus Gashaw
**Context:** Week 10 - Tenacious Conversion Engine, 10 Academy

---

## The Gap This Closes

Your Week 10 outbound system depends on a simple rule: do not overstate a signal when the evidence is weak. That rule is easy to write and harder to keep alive once the prompt gets long. By the time the model is generating the email body, it has already consumed a large block of enrichment context, competitor-gap text, and prospect-specific details. The real question is not whether attention exists. It is whether a distant instruction still has enough influence to shape the final tokens after the prompt grows.

This matters because a long prompt does not just add information. It also increases the number of competing signals the model has to weigh at generation time. In practice, that makes instruction drift a real engineering problem, not a vague language-model complaint.

---

## What the Sources Actually Show

Two sources are load-bearing here.

Xiao et al., *Efficient Streaming Language Models with Attention Sinks* (2023), shows that early tokens can receive disproportionate attention mass and serve a stabilizing role in streaming generation. That does not prove every early instruction is always obeyed, but it does show that position matters and that some prompt locations have structural advantages.

Wallace et al., *The Instruction Hierarchy* (2024), shows that models can be trained to treat privileged instructions differently from ordinary text. In other words, instruction-following is not only about semantic content. It also depends on how the model has learned to interpret placement, privilege, and priority.

Taken together, those sources support a narrower claim: in long prompts, a distant constraint is easier to lose unless the system does something to keep it salient near generation time. They also point to a second mechanism: instruction priority is partly learned, so training can make system-level constraints more or less resistant to override.

---

## The Mechanisms That Matter

There are three mechanisms worth naming.

First, attention salience changes with context length. A transformer does not read the prompt once and freeze the rule in place. At each generation step, it compares the current query against prior tokens and chooses what matters most right then. As the context grows, a distant instruction has to compete with more nearby evidence, so its influence can become weaker even if it is still present.

Second, learned instruction hierarchy affects whether the model treats a system-level rule as privileged or just as more text. If the model has been trained to respect that hierarchy, the instruction starts with an advantage. If not, the prompt has to do more work to hold the rule in place.

Third, local decoding pressure can override a good instruction. Even when the policy is present, the next-token continuation can be pulled toward a strong local pattern. That is why a model may look compliant on one prompt and drift on another prompt with similar wording but different surrounding context.

That is the core distinction your friend’s question is asking for: instruction preservation is not controlled by one thing. It is the result of salience, learned priority, and local generation pressure acting together.

---

## How This Maps To Your Week 10 System

Your Week 10 pipeline, especially [agent/main.py](c:/Users/Davea/Downloads/trp%20week%2010/agent/main.py) and [agent/outreach_generator.py](c:/Users/Davea/Downloads/trp%20week%2010/agent/outreach_generator.py), assembles a lot of context before the email is written. That means the model is not just reading a prompt; it is reading a long evidence bundle that includes hiring signals, competitor gaps, bench availability, and style constraints.

That setup makes instruction drift plausible. If the rule that says "hedge when confidence is low" lives only near the top of the prompt, it has to survive the entire enrichment block before it can affect the final phrasing. If nearby context strongly resembles confident outreach language, the model may follow that local pattern instead of the distant policy.

So the practical issue is not "does the model understand the instruction?" It is "does the instruction still win when generation reaches the final token choice?"

---

## What SCAP v2 Is Actually Doing

SCAP v2 is an application-layer workaround for that salience problem. It does not change the model weights. It changes what the model sees immediately before generation.

If the policy is repeated in a short, structured block right before the model writes the email, the constraint is no longer a distant memory. It becomes part of the local decision surface. That does not guarantee compliance, but it makes compliance more likely because the instruction is now competing from a position of recency instead of only from the top of a long prompt.

That is the strongest defensible claim. It is weaker than saying attention alone explains the failure, but it is much easier to support from the sources and from your own system.

---

## What To Test

The strongest evidence is not an attention diagram. It is an ablation.

Use the same prospect and the same signal bundle, then compare three conditions: baseline with the policy only in the system prompt, long context with the same policy plus a larger enrichment block, and adjacent constraint with the policy repeated immediately before generation. Then measure the output behavior directly: does the model hedge when confidence is low, or does it over-assert?

If the adjacent constraint improves adherence, that is evidence that proximity helps. If the long-context version fails while the adjacent version holds, then you have a practical reason to change the prompt structure rather than guessing about the model.

---

## Limits Worth Naming

This explanation does not prove that a particular attention head caused your specific failure. It also does not prove that attention weights alone predict instruction adherence. Those are stronger mechanistic claims than the evidence here supports.

What it does support is a practical engineering conclusion: if your policy matters, keep it close to the generation step, measure whether that changes adherence under long-context conditions, and check whether you also need a stronger priority mechanism beyond proximity alone.

---

## One-Line Summary

Instructions do not stay effective just because they were written once. In long-context generation, their influence depends on salience, learned instruction priority, and local decoding pressure at the moment the model chooses the next token. The safest response is to test that directly and, when needed, reintroduce the policy near the generation site.

---

## Sources

- Xiao, G., et al. (2023). *Efficient Streaming Language Models with Attention Sinks.*
- Wallace, E., et al. (2024). *The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions.*