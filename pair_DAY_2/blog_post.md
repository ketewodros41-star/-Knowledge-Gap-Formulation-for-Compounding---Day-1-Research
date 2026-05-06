# Why Tool Outputs Get Ignored: Two Failures Inside a Real Agent System

*By Kidus Gashaw*

---

My partner came to me with a failure from his Week 11 `tenacious-bench` system. He had built an evaluation pipeline where probes supplied structured constraints to the agent — failure category, hiring signal confidence, tone flags, overcommitment warnings. The agent was receiving them. But the outputs kept coming back generic, as if the constraints had never arrived.

His question was precise: inside a modern tool-using agent, what determines whether a tool output actually shapes the model's response — versus being present in the context but having no real effect?

After looking at the actual code, the answer was sharper than either of us expected. There were two separate failures happening — and only one of them was the LLM ignoring context.

---

## Failure One: The Context Never Reached the Model

The probe evaluation harness in `run_probes.py` builds a `FakeAnalysis` object that carries the probe's structured context — `confidence: low`, the weak signal evidence, the failure category. But when `generate_followup_email()` in `email_agent.py` is called, it reads only `reply_type` and routes to a template. The structured context is attached to the object but never passed to the LLM.

More critically: the function has no case for `reply_type = "objection"` — which is exactly what the test harness sets for every probe:

```python
if analysis.reply_type == 'interested':          return { ... }
if analysis.reply_type == 'information_request': return { ... }
if analysis.reply_type == 'defer':               return { ... }
if analysis.reply_type == 'unclear':             return { ... }
# no objection case — falls through to:
return {'body': 'Understood. Thanks for the reply.', 'source': 'fallback'}
```

All 32 probes in `probe_trace_log.json` return the same output: `"Understood. Thanks for the reply."` That is not the LLM ignoring structured context. That is a hardcoded fallback firing every time. The model is never called during probe evaluation at all.

This matters because it means the probe results were measuring the fallback, not the agent. Any conclusion about the agent's constraint-following behaviour drawn from those traces is measuring the wrong thing.

---

## Failure Two: Policy Constraints as Advisory Text

For the cases where the LLM is actually called — in `_generate_email_with_llm()` — the policy constraints arrive as plain-text key-value pairs in the user prompt:

```
Tone mode: exploratory
Claim strength: soft
```

These are not tool outputs. They are not schema-validated. They carry the same weight as any other text in the prompt — which means the model's pretraining prior for what a good B2B sales email sounds like competes directly with them. And the prior frequently wins.

This is the real LLM-context-ignoring problem.

---

## The Load-Bearing Mechanism

The model generates each token by sampling the highest-probability continuation of its context. It has two competing signals at every step: the pretraining prior — strong statistical patterns from millions of B2B outreach examples — and the prompt guidance sitting in the context window.

When the model is not forced to generate reasoning tokens that explicitly reference the constraint before writing the response body, the prior dominates. The output matches the surface form of a policy-aware response while being driven entirely by the prior. The constraint is acknowledged; the reasoning trajectory is unchanged.

The fix is forcing a scratchpad step. Compare these two prompt endings:

```python
# Advisory text — prior wins
"""
Tone mode: exploratory
Claim strength: soft
Prospect reply: {prospect_reply}
"""

# Forced scratchpad — constraint enters the reasoning path
"""
Policy:
- tone_mode: {policy_result['tone_mode']}
- claim_strength: {policy_result['claim_strength']}
- allow_capacity_language: {policy_result['allow_capacity_language']}

Prospect reply: {prospect_reply}

Before drafting, complete this:
Constraint check: Given tone_mode={policy_result['tone_mode']} and claim_strength={policy_result['claim_strength']}, I must [complete this sentence].

Response:
"""
```

The `Constraint check:` line forces the model to generate tokens that encode the policy into the probability distribution of everything that follows. The constraint is no longer a passive label — it is a load-bearing token sequence the model must complete before producing the reply.

---

## Three Concepts That Make This Gap Worth Closing

**Tool schema design is the missing layer.** The system has no tool schemas — `policy_result` is a Python dict serialised to plain text. A real OpenAI-format tool schema changes this fundamentally. When a model is given a typed tool definition, it must emit a structured JSON token sequence to invoke the tool — the schema forces the model into a specific generation path. Plain-text labels carry no such forcing. The difference between `Tone mode: exploratory` in a prompt and a typed `tone_mode` field in a tool schema is the difference between advice the model can ignore and a generation constraint it must satisfy to proceed.

**Position compounds the problem.** Liu et al. (2023) showed that information buried mid-prompt — between a long system prompt and the generation instruction — sits in the position where attention weight is lowest. Moving the policy constraints to the top of the user message, or repeating them immediately before the response instruction, measurably improves integration with no architectural change.

**ReAct makes it structural.** Yao et al. (2022) formalise the scratchpad fix: interleaving `Thought:` and `Action:` steps forces the model to generate reasoning tokens before output. Because the LLM path is still probabilistic, a cheap second-pass judge call — checking whether the response contradicts the boolean fields already computed in `policies.py` (`allow_capacity_language`, `require_handoff`) — adds a hard verification layer on top of the soft probabilistic one.

---

## What Needs to Change

Three fixes, ordered by impact:

1. **Add the objection case** to `generate_followup_email()`. Every probe currently fires the fallback. The LLM is never involved in probe evaluation.
2. **Pass probe context into the generation prompt.** The `confidence`, `signals`, and `failure_category` fields exist but never reach the model.
3. **Force a constraint check step** before the response body. Replace plain-text labels with a sentence the model must complete before drafting.

The first fix changes what is being measured. The second and third change whether the measurement is meaningful.

---

## Sources

1. **Liu et al. (2023). "Lost in the Middle: How Language Models Use Long Contexts."** arXiv:2307.03172. Canonical empirical paper on position-dependent attention and the U-shaped performance curve in long-context tasks. Load-bearing for the attention-position section above.

2. **Yao et al. (2022). "ReAct: Synergizing Reasoning and Acting in Language Models."** arXiv:2210.03629. Original paper establishing interleaved reasoning-action traces as the mechanism for making retrieved context load-bearing. The scratchpad fix is a direct application of this pattern.
