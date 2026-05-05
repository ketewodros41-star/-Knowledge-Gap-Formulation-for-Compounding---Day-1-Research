# How Attention Dynamics Contribute to Instruction Drift in LLMs - and What to Do About It

**Author:** Kidus Gashaw
**Context:** Week 11 - Tenacious Bench, 10 Academy

---

## The Gap This Closes

In Week 11 benchmark development, the goal was to fine-tune a model to generate outbound outreach emails written from the perspective of an external outreach agent. During evaluation on the dev dataset, the model consistently produced emails from the company's own voice instead. Given a hiring signal, for example, the model would output something like:

> "We are hiring, here is a job opening…"

rather than framing it as an external party reaching out on behalf of a client. The explicit instruction to write as an outreach agent was present in the prompt. The model ignored it.

That failure has a specific cause. It is not that the model failed to read the instruction. It is that competing signals in the training data and in the inference context were stronger than the instruction at the moment generation happened.

---

## How Inference Actually Works: Prefill and Decode

Before naming what goes wrong, it helps to know what the model is doing mechanically during inference. There are two distinct phases, and instruction drift lives in the second one.

**Prefill** is when the model processes the entire input prompt at once. It computes attention across every token in the sequence — including the instruction, the enrichment data, and all context — building a set of key-value representations that will be used during generation. This phase has quadratic complexity with respect to input length (O(n²)): a prompt twice as long takes roughly four times as much compute. For a system like the Tenacious Conversion Engine that passes 2,000+ tokens of enrichment data, prefill is the dominant latency cost. Critically, this is also when the model forms its initial representation of the instruction — but that representation is not locked in. It feeds into decode as one signal among many.

**Decode** is when the model generates tokens one at a time, autoregressively. At each step, it attends to everything it has seen — the original prompt and all tokens generated so far — and produces a probability distribution over the next token. This phase is sequential and cannot be parallelized across tokens. This is where instruction drift actually occurs: the instruction was processed in prefill, but during decode it must keep winning a re-run attention competition at every single generation step.

This split explains why the problem is not simply "the model ignored the instruction." The model processed the instruction. The question is whether that instruction continues to have enough influence during decode, as each new token is added and the competition shifts.

---

## Two Layers of the Problem

Instruction drift in a fine-tuned model operates at two levels, and it is important to keep them separate.

**Level 1: Training distribution bias.** During fine-tuning, the model observes input-output pairs. If most of those examples associate hiring signals with company-voice completions (because the training data was scraped from company-authored emails or generated without strict perspective control), the model learns that association in its weights. At inference time, the token sequence "they are hiring" or "job opening" activates a completion pattern that was reinforced hundreds of times. The instruction to write as an outreach agent is present in the prompt, but it is competing against a weight-level prior that fires strongly on those same tokens.

**Level 2: Context-length attention attenuation.** Even when the training data is clean, a long inference prompt creates a second failure mode. As the model generates each token, it computes attention across all prior tokens. An instruction placed far from the generation site has to win a competition against all the nearby evidence. If the enrichment block is large and contains dense company-centric language (job titles, team names, company mission), those tokens are closer to the active generation step and exert more local influence. The distant instruction is still visible but is structurally disadvantaged.

Both levels are present in the Week 11 setup. The trained model carries learned associations from training data, and at inference time the hiring-signal context reinforces those associations further.

---

## What the Sources Actually Show

Two sources are load-bearing here.

Xiao et al., *Efficient Streaming Language Models with Attention Sinks* (2023), shows that early tokens can receive disproportionate attention mass and serve a stabilizing role in streaming generation. That does not prove every early instruction is always obeyed, but it establishes that position matters and that some prompt locations have structural advantages over others.

Wallace et al., *The Instruction Hierarchy* (2024), shows that models can be trained to treat privileged instructions differently from ordinary context. Instruction-following is not purely semantic. It depends on learned placement and priority signals. A model trained without strong instruction-hierarchy supervision treats an explicit perspective directive as roughly equal weight to the surrounding context it is competing against.

Together, these support a narrower and more precise claim: in a fine-tuned model, perspective drift is not just a prompt-engineering failure. It is partly a training-data problem (the model learned the wrong association) and partly an attention-salience problem (the instruction loses to nearby context at generation time).

---

## The Three Mechanisms That Determine Whether an Instruction Wins

Mechanism 2 is the direct answer to the question about attention dynamics during inference. Mechanisms 1 and 3 explain the conditions that make the attention competition harder to win — one is a pre-condition set during training, the other is a cascade effect during generation.

**1. Training-weight salience (pre-condition, not an attention dynamic).**
When the model was trained, it adjusted weights to make high-likelihood completions more likely. If training examples frequently paired hiring-signal tokens with first-person company voice, the model's learned representations couple those concepts. At inference, the instruction to write as an outreach agent must overcome that coupling. If the coupling is strong enough, the model produces the company voice even when the instruction explicitly forbids it. This is distinct from the attention mechanism — it happens at the level of learned token-to-token affinities across the full vocabulary and embedding space. It is a pre-existing bias that shapes what the attention competition is even trying to fight against.

**2. Attention salience at generation time — this is the core attention dynamic.**
At each decoding step, the model computes a query vector Q from its current hidden state. It then scores every prior token by computing dot products between Q and each token's key vector K: score = Q·K / √d. These scores go through a softmax, which normalises them into a probability distribution that sums to 1. This is the critical point: attention is a zero-sum competition. If hiring-context tokens — company names, job titles, role descriptions — receive high dot-product scores because they are semantically close to what the model is currently generating, the instruction token's score is suppressed proportionally. The instruction does not disappear from context. It loses the competition. Position compounds this further: recent tokens are more likely to align with the current query's positional encoding, giving them a structural advantage over tokens placed far earlier in the prompt. A perspective instruction written 2,000 tokens before the generation site is competing against every enrichment token that came after it, and it is losing on two dimensions simultaneously — semantic distance and positional distance.

**3. Local decoding pressure (cascade effect).**
Even with a well-placed instruction and clean training data, the next-token prediction step can be pulled by a strong local continuation pattern. If the preceding generated tokens look like "Our team is growing and we'd love to connect about…", the probability distribution over the next token is dominated by continuations consistent with that pattern — regardless of what the instruction said. Each generation step is a new competition, and a drifted early token reshapes the query for the next step. A single drifted token makes the following token more likely to drift, because the query vector now encodes company-voice context rather than agent-voice context. This is how one slip at the start cascades into a fully drifted output.

---

## Why Hiring Signals Specifically Cause This

Hiring signals are a high-salience category. They appear in training corpora almost exclusively in first-person company communications: job posts, recruiting emails from the employer, LinkedIn posts authored by the company. The phrase "We are hiring" is overwhelmingly associated in pre-training and in most fine-tuning datasets with company-authored text. When the model sees that concept in context, the prior from training is extremely strong.

An outreach agent framing ("I noticed your company is hiring, and I wanted to reach out on behalf of…") is a minority pattern in most corpora. Unless the fine-tuning data deliberately overrepresented the outreach-agent framing for hiring signals, the model has no strong learned reason to prefer it.

This is why the drift is consistent rather than random. It is not noise. It is a reproducible, high-confidence wrong answer that the model is confident about — which is diagnostic of a training distribution problem, not just an attention-length problem.

---

## How This Maps to the Week 11 Setup

The fine-tuning dataset was designed to produce outreach emails from an agent's perspective. If the input side of those examples included hiring signals (job postings, headcount data, open roles), the model's ability to generalize depends entirely on how consistently the training outputs modeled the agent framing in response to those exact inputs.

If any training examples used company voice, or if the input format included company-authored text that the model could mirror, the model learned a mixed signal. At evaluation time, the explicit prompt instruction to write as an agent cannot overcome a weight-level association that was trained in.

The three unknowns in the original question map directly to the mechanisms:

- *How the model weighs instructions vs contextual tokens during inference* → Mechanism 2: the softmax over query-key dot products is the weighing operation. The instruction token's key vector scores lower than nearby company-context tokens because it is semantically and positionally further from the current generation query. Lower score after softmax means less influence on the output representation at that step.

- *Why certain tokens dominate generation despite explicit guidance* → Mechanism 1: hiring-signal tokens dominate because the model was trained on data that overwhelmingly paired those tokens with company-voice completions. The weight-level prior activates before the instruction has a chance to win the attention competition. The instruction is present but it is fighting a learned association, not just a distance problem.

- *What mechanisms in attention or token prediction lead to this drift* → Mechanism 3: local decoding pressure explains how drift propagates. Once a single token is generated in company voice, the query vector for the next step now encodes company-voice context, making the next company-voice token even more likely. The attention mechanism turns a small initial drift into a consistent output failure because each step builds on the last.

---

## What Measurement Reveals About the Problem

Instruction drift is hard to diagnose without the right telemetry. The same blind spot that hides latency problems also hides drift patterns: if you only log total generation time or a pass/fail label, you cannot see where in the generation the failure happened.

The prefill/decode split gives a useful diagnostic lens. A high **Time-To-First-Token (TTFT)** means the prefill phase was expensive — the input was large. That is a strong signal that the prompt contains enough competing context to create an attention-salience problem during decode. If TTFT scales with input length and drift also scales with input length, those two observations point to the same root cause: a long enrichment block that both slows prefill and weakens instruction influence during decode.

The practical measurement is:

- **TTFT** ≈ prefill cost → proxy for how much the model had to process before generating
- **Output token rate** → characterizes decode; sudden slowdowns can indicate repetition or recovery from drift
- **Input token count** → the primary driver of prefill cost and instruction attenuation

In the Week 11 benchmark, logging input token count alongside drift rate by signal type would directly test whether larger inputs (more hiring-signal context) produce more perspective drift. If they do, that is evidence that the mechanism is attention attenuation, not just a training-data problem.

---

## What to Test

The strongest diagnostic is a controlled ablation across three prompt conditions:

**Condition A — Baseline:** prompt includes perspective instruction at the top, followed by hiring signal context, no repetition.

**Condition B — Adjacent constraint:** same prompt, but the perspective instruction is repeated immediately before the model begins generating the email body.

**Condition C — Rephrased constraint with negative example:** the instruction includes an explicit negative example: "Do NOT write as the company. Example of wrong output: 'We are hiring…'. Write as an external agent reaching out about this signal."

If Condition B outperforms A, that is evidence that proximity helps and the mechanism is attention-salience at generation time. If Condition C is needed to close the gap, that is evidence that the training prior is strong and the model needs a stronger override signal to beat it.

A separate diagnostic is to check the training data directly: count how many fine-tuning examples that included hiring signals had outputs written in first-person company voice. Even a small percentage of such examples can dominate if they are highly consistent with a pattern the pre-trained model already knew well.

---

## What SCAP v2 Is Actually Doing (Updated)

SCAP v2 addresses the attention-salience problem (mechanism 2) but not the training-weight problem (mechanism 1). By re-introducing the constraint immediately before generation, it gives the instruction a recency advantage in the attention competition. That helps when the drift is caused by context-length attenuation.

It does not help when the drift is caused by a strong weight-level prior from training. In that case, the constraint needs to be strong enough to overcome the learned association, which may require either a negative example in the prompt or retraining with better perspective-labeled data for the hiring-signal case specifically.

---

## What To Do (Practical Steps)

**1. Audit your training data first.**
Go through the fine-tuning examples that include hiring signals and check whether the outputs were written in company voice or agent voice. Even 10–15% company-voice examples in that subset are enough to create consistent drift. Relabel or remove those examples before retraining.

**2. Repeat the perspective instruction immediately before generation.**
Do not only place it at the top of the prompt. Add a short restatement right before the model starts writing the email — for example: `"Remember: write as an external outreach agent, not as the company."` This keeps the instruction in the high-attention zone at the moment it matters.

**3. Add a negative example to the instruction.**
Abstract instructions lose to strong learned patterns. Make the failure mode explicit: `"Do NOT write from the company's perspective. Wrong: 'We are hiring…'. Correct: 'I noticed your company is hiring and wanted to reach out…'."` A concrete wrong example gives the model a pattern to avoid rather than an abstract rule to follow.

**4. Prime the output with the first words.**
Prepend the start of the correct framing to the model's output space — for example, begin the generation with `"I came across"` or `"I noticed"` as a forced prefix. Because local decoding pressure compounds from the first token, starting in the right voice dramatically reduces the chance the model drifts mid-generation.

**5. If drift persists after the above, retrain on the corrected data.**
Prompt-level fixes address attention-salience problems. They cannot fully override a strong weight-level prior. If steps 2–4 reduce but do not eliminate drift, the training data is the root cause and retraining on a relabeled dataset is the correct fix.

---

## Limits Worth Naming

This explanation does not prove that a specific attention head caused the failure. It does not claim attention weights alone determine instruction adherence.

What it supports is a layered diagnostic: first check whether the training data created a strong hiring-signal-to-company-voice prior; then check whether adjacent constraint placement changes behavior at inference; then test whether explicit negative examples are needed to override what the model learned.

The combination of those three tests gives a principled path to diagnosis without requiring mechanistic interpretability tools.

---

## One-Line Summary

When a fine-tuned model generates from the wrong perspective despite explicit instructions, the cause is usually a training distribution prior (hiring signals were mostly modeled in company voice during training) compounding with attention-salience attenuation (the instruction is too far from the generation step to win the competition). The practical fix is to test adjacent constraint placement and to audit training data for perspective consistency on the specific signal types that cause drift.

---

## Sources

- Xiao, G., et al. (2023). *Efficient Streaming Language Models with Attention Sinks.*
- Wallace, E., et al. (2024). *The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions.*
- Dao, T., et al. (2022). *FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness.* — cited for prefill optimization and attention computation efficiency.
