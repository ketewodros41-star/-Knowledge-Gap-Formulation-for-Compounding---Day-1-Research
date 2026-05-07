# day 1 Submission

This folder contains the pair-day submission for the Week 12 inference-time mechanics topic.

The package is built around one sharpened question and the artifacts that show how it was refined, answered, and signed off.

## What's Inside

- [question.md](question.md) - the final question for the day.
- [morning_call_summary.md](morning_call_summary.md) - the question-sharpening notes from the morning call.
- [explainer.md](explainer.md) - the explainer written in response to the partner's question.
- [thread.md](thread.md) - the short public version of the explainer.
- [evening_call_summary.md](evening_call_summary.md) - the evening call summary.
- [signoff.md](signoff.md) - the final gap-closure judgment.
- [question_sources.md](question_sources.md) - the canonical sources used in the explainer.

## Public Artifacts

- **Blog post:** [Why Your Instruction Gets Ignored: Attention Dynamics and Instruction Drift in LLMs](https://open.substack.com/pub/kidusgashaw/p/why-your-instruction-gets-ignored?r=8bo4le&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true)
- **Tweet thread:** [https://x.com/Kidus5T99409/status/2051633404085502442?s=20](https://x.com/Kidus5T99409/status/2051633404085502442?s=20)

## Grounding

The question is grounded in the Week 10 conversion engine, especially the latency tracking gap in [agent/outreach_generator.py](c:/Users/Davea/Downloads/trp%20week%2010/agent/outreach_generator.py). The explainer focuses on separating prefill and decode latency so Time-To-First-Token can be measured directly instead of being hidden inside one coarse API timing bucket.

---

# Pair Day 2 — Agent and Tool-Use Internals

## What's Inside

- [pair_DAY_2/question.md](pair_DAY_2/question.md) — Kidus's sharpened question: the difference between a scaffolding pre-call and a model-invoked tool, grounded in the bench-gated constraint in `method.md`
- [pair_DAY_2/morning_call_summary.md](pair_DAY_2/morning_call_summary.md) — how both partners explained and sharpened each other's questions
- [pair_DAY_2/explainer.md](pair_DAY_2/explainer.md) — Kidus's explainer for Amare's question: two failure modes inside a real agent system and the mechanism behind both
- [pair_DAY_2/blog_post.md](pair_DAY_2/blog_post.md) — public-facing version of the explainer
- [pair_DAY_2/thread.md](pair_DAY_2/thread.md) — 6-tweet thread compressing the explainer
- [pair_DAY_2/evening_call_summary.md](pair_DAY_2/evening_call_summary.md) — feedback from both partners on each other's explainers
- [pair_DAY_2/signoff.md](pair_DAY_2/signoff.md) — Amare's gap-closure judgment on Kidus's explainer
- [pair_DAY_2/grounding_commit.md](pair_DAY_2/grounding_commit.md) — Kidus's edit to `method.md` based on what Amare's explainer revealed
- [pair_DAY_2/sources.md](pair_DAY_2/sources.md) — canonical sources and tool used

## Public Artifacts

- **Blog post:** [Why Tool Outputs Get Ignored: Two Failures Inside a Real Agent System](https://kidusgashaw.substack.com/p/why-tool-outputs-get-ignored-two)
- **Tweet thread:** [https://x.com/Kidus5T99409/status/2052066470230659446?s=20](https://x.com/Kidus5T99409/status/2052066470230659446?s=20)

## Grounding

Kidus's question is grounded in `agent/main.py` and `agent/enrichment.py` from the Week 10 Conversion Engine. The gap: `check_bench_availability()` was described in `method.md` as a bench-gated constraint, but was actually a scaffolding pre-call the model never controlled. Amare's explainer revealed the distinction between runtime-enforced scaffolding (deterministic) and model-invoked tools (probabilistic), which led to a rewrite of the bench-constraint paragraph in `method.md`.

---

# Pair Day 3 — Training and Post-Training Mechanics

## What's Inside

- [pair_DAY_3/question.md](pair_DAY_3/question.md) — Kidus's sharpened question: the mechanistic difference between a prompt-based constraint (SCAP) and a LoRA weight update, grounded in `probes/target_failure_mode.md` (P-005 trigger rate 0.70) and `method/method.md`
- [pair_DAY_3/morning_call_summary.md](pair_DAY_3/morning_call_summary.md) — how both partners sharpened each other's questions: Kidus's from "why did SCAP fail" to the prompt-vs-weight-update mechanism; partner's from a broad LoRA question to rank-shapes-confidence grounded in her `ablation_results.json` CI `[0.0, 16.1%]`
- [pair_DAY_3/explainer.md](pair_DAY_3/explainer.md) — Kidus's explainer for his partner's question: how LoRA rank controls expressivity and why r=16 on 51 preference pairs produces overconfident scores on uncertain inputs
- [pair_DAY_3/blog_post.md](pair_DAY_3/blog_post.md) — public-facing blog post version of the explainer
- [pair_DAY_3/thread.md](pair_DAY_3/thread.md) — 6-tweet thread compressing the explainer
- [pair_DAY_3/evening_call_summary.md](pair_DAY_3/evening_call_summary.md) — feedback from both partners on each other's explainers and revisions made
- [pair_DAY_3/signoff.md](pair_DAY_3/signoff.md) — partner's gap-closure judgment on Kidus's explainer
- [pair_DAY_3/grounding_commit.md](pair_DAY_3/grounding_commit.md) — Kidus's edit to `methodology_rationale.md` adding the missing rank justification
- [pair_DAY_3/sources.md](pair_DAY_3/sources.md) — Hu et al. (2021), Aghajanyan et al. (2020), PEFT library

## Public Artifacts

- **Blog post:** [https://open.substack.com/pub/kidusgashaw/p/lora-rank-and-the-confidence-problem?r=8bo4le&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true]
- **Tweet thread:** [https://x.com/Kidus5T99409/status/2052453988382003529?s=20](https://x.com/Kidus5T99409/status/2052453988382003529?s=20)

## Grounding

Kidus's question is grounded in `probes/target_failure_mode.md` and `method/method.md` from the Week 10 Conversion Engine — SCAP reduced signal over-claiming (P-005, trigger rate 0.70) but could not eliminate it, and the method.md never explains why a prompt-time mechanism has a ceiling that training does not. The explainer Kidus wrote is grounded in his partner's `training/train_lora_judge.py` (r=16, 51 preference pairs) and `ablations/ablation_results.json` (Delta A CI [0.0, 16.1%]), revealing that high rank on a small dataset causes the critic to memorize surface patterns of chosen examples rather than learning the rubric boundary — which led to adding a rank-justification paragraph to her `methodology_rationale.md`.
