# Why Your Instruction Gets Ignored: Attention Dynamics and Instruction Drift in LLMs

*By Kidus Gashaw*

---

My partner came to me with a specific failure from her Week 11 benchmark work. She had fine-tuned a model to write outbound outreach emails from the perspective of an external outreach agent — but every time a hiring signal appeared in the context, the model wrote from the company's voice instead:

> "We are hiring, here is a job opening…"

The instruction was explicit. It was present in the prompt. The model had processed it. So why was it being overridden?

After researching this, the answer is not that the model ignored the instruction. It is that the instruction lost a competition — and understanding why requires knowing what the model is actually doing at each generation step.

---

## The Load-Bearing Mechanism: Attention Is a Zero-Sum Competition

LLM inference has two phases. During **prefill**, the model processes the entire input prompt at once and builds key-value representations for every token. During **decode**, the model generates one token at a time. At each step it computes a query vector from its current hidden state, scores every prior token by taking the dot product of that query with each token's key vector, scales by √d, and passes all scores through a softmax.

That softmax is the critical point. It normalises the scores into a probability distribution that sums to 1. Attention is not a parallel read of all tokens — it is a budget allocation. If some tokens receive more weight, others necessarily receive less. The instruction does not disappear from context. It loses the competition.

The illustrative example below makes this concrete. The following shows how the same prompt structure produces different outputs depending on instruction placement — one variable changed, everything else held constant.

---

**Condition A — Instruction at the top only**

*Prompt structure:*
```
[INSTRUCTION — top of prompt]
Write this email as an external outreach agent reaching out on behalf of a staffing firm.
Do not write as the company.

[2,000 tokens of context]
Company: Apex Technologies. Open roles: 4 Backend Engineers.
Hiring velocity: high. Recent funding: Series B, $40M.
Job post: "We are growing our backend team and looking for engineers..."
...

[Generation begins here]
```

*Output:*
> "We are expanding our backend team and are looking for strong engineers to join us. Our recent Series B funding has positioned us to scale aggressively, and we have four open roles across our engineering org..."

---

**Condition B — Instruction repeated immediately before generation**

*Prompt structure:*
```
[INSTRUCTION — top of prompt]
Write this email as an external outreach agent reaching out on behalf of a staffing firm.
Do not write as the company.

[2,000 tokens of context]
Company: Apex Technologies. Open roles: 4 Backend Engineers.
Hiring velocity: high. Recent funding: Series B, $40M.
Job post: "We are growing our backend team and looking for engineers..."
...

[INSTRUCTION REPEATED — immediately before generation]
Reminder: write as an external outreach agent, not as the company.

[Generation begins here]
```

*Output:*
> "I noticed Apex Technologies recently closed a Series B and has four backend roles open. I wanted to reach out — we have engineers on our bench with relevant experience and I thought it might be worth a conversation..."

---

Same instruction. Same context. The only difference is proximity to the generation site. In Condition A, the instruction was 2,000 tokens away when the first token was generated. In Condition B, it was the last thing the model read. The attention competition at that first token step looked completely different in each case — and everything after it was inherited from that first token.

---

This directly answers the three unknowns: how the model weighs instructions vs context is the softmax budget allocation at each decode step — lower score means less influence on the output representation; why certain tokens dominate despite explicit guidance is the training-weight prior activating before the attention competition even starts; what mechanism in token prediction leads to drift is the cascade — one drifted token reshapes the query for the next step, making every subsequent token more likely to drift in the same direction.

---

## Why Hiring Signals Make This Worse

Two adjacent mechanisms compound the attention problem.

**Training-weight salience.** Hiring signals appear in real-world corpora almost exclusively in company-authored text: job posts, recruiting emails, LinkedIn announcements. If her fine-tuning data included even a small fraction of examples where hiring inputs were paired with company-voice outputs, the model learned that association in its weights. At inference, the query vector itself already encodes company-voice context before the attention competition even starts — because the model's hidden state was shaped by what it was trained to predict. The instruction is fighting both a distance problem and a weight-level prior.

**Cascade from local decoding pressure.** Once the first generated token drifts toward company voice, the query vector for the next step encodes that context. The next company-voice token becomes more likely, which makes the one after it more likely still. A single slip at token one compounds into a fully drifted output. This is why the failure in her benchmark was never partial — the model produced company voice from the first word, consistently.

---

## What to Do About It

The fix depends on which layer is broken. If drift is consistent on a specific signal type regardless of input length, the training data is the root cause — audit the fine-tuning examples for that category and relabel. If drift worsens as input grows, the mechanism is attention attenuation — repeat the perspective instruction immediately before generation so it competes from recency rather than distance. Adding a negative example ("Do NOT write like this: 'We are hiring…'") strengthens this further by giving the model a pattern to avoid rather than an abstract rule to follow.

Neither fix is magic. Both work by changing what the model is attending to at the moment the first token is generated — because everything after that is inherited.

---

## Sources

- Xiao, G., et al. (2023). *Efficient Streaming Language Models with Attention Sinks.* [https://arxiv.org/abs/2309.17453](https://arxiv.org/abs/2309.17453) — establishes that token position creates structural attention advantages and that early tokens can receive disproportionate attention mass.

- Wallace, E., et al. (2024). *The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions.* [https://arxiv.org/abs/2404.13208](https://arxiv.org/abs/2404.13208) — shows that instruction-following depends on learned placement and priority signals, not only on semantic content; a model without instruction-hierarchy training treats a system directive as equal weight to surrounding context.
