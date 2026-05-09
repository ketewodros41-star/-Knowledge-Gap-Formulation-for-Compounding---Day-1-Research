# Week 12 Synthesis — Kidus Gashaw

**TRP1 Challenge | Week 12 | 4 Pairs Completed**
**Partner:** Mistire Daniel (Days 3–4), Amare Kassa (Day 2), rotating (Day 1)

---

## Section 1: The Eight Gaps

### Four Gaps I Named (Questions I Asked)

**Gap 1 — Day 1: Prefill vs. Decode Latency (Evaluation and Inference)**

My `outreach_generator.py` packed the prompt with Crunchbase firmographics, job-post velocity, layoffs.fyi data, and a competitor gap brief. Langfuse traces showed high latency but only a single coarse metric — total API wait time. I could not tell whether the bottleneck was in prefill (the model reading 2,000+ context tokens) or decode (the model writing the 300-word email).

The gap: I did not know that prefill and decode are fundamentally different computation phases. Prefill is a single forward pass across all input tokens simultaneously — it is parallelized and scales with context length. Decode is autoregressive: one token generated per forward pass, sequentially, with each step attending to all previous tokens. An optimization that reduces input context (chunked prompts, summarized briefs) targets prefill. An optimization that reduces output length targets decode. Without separate telemetry, every latency decision was a guess.

Closure: I added `ttft_ms` and `tokens_per_second` fields to `outreach_generator.py`. TTFT fires when the first decode token arrives. TPS is calculated over the full decode phase. Now a Langfuse trace can tell me which phase is the bottleneck before I decide where to optimize.

**Gap 2 — Day 2: Scaffolding Pre-Call vs. Model-Invoked Tool (Agent Architecture)**

I described the bench availability check in my Tenacious agent as a bench-gated constraint. The `method.md` framing implied the model was choosing whether to call a staffing tool. The code in `agent/main.py` told a different story: `check_bench_availability()` runs unconditionally before the model receives any input.

The gap: I could not defend whether this design was intentional or accidental, and I could not explain the reliability difference between the two approaches. A model-invoked tool is probabilistic — the model decides whether to call it, and that decision is subject to prompt phrasing, reasoning drift, and instruction competition. A scaffolding pre-call is deterministic — the runtime owns execution regardless of what the model generates.

Closure: I rewrote the bench-constraint paragraph in `method.md` to accurately describe what is actually enforcing the constraint. The old framing — "an autonomous agent choosing whether to use a staffing tool" — became: "a runtime-enforced enrichment pipeline where scaffolding executes the bench availability check before generation and injects the result into model context." The distinction matters for any FDE who needs a constraint that cannot be bypassed.

**Gap 3 — Day 3: SCAP Ceiling — Why Prompting Is Not Enough (Training vs. Inference)**

In Week 10, SCAP reduced signal over-claiming (P-005 cluster, trigger rate 0.70) but left residual violations. I justified training a LoRA judge in Week 11 by saying "the agent does not know when it is wrong." I could not explain what training the LoRA actually changed that SCAP's prompt instructions could not.

The gap: I conflated prompting with fine-tuning because both change model behavior, but I did not understand the mechanism behind either. Prompting adds context tokens that shift next-token probabilities for one inference run — the model's weights are unchanged and all pretrained tendencies remain intact. When the SCAP instruction competed with the model's fluency and plausibility objectives on a borderline input, those objectives could still win because the underlying weight function had not changed.

Closure: LoRA changes the computation itself: W_effective = W + BA means every forward pass through adapted layers produces different hidden states regardless of what the prompt says. The residual P-005 violations were the signature of a prompt ceiling — borderline inputs where the pretrained representation did not cleanly separate compliant from non-compliant outputs. I can now explain the design chain: SCAP first because prompting is cheaper and addresses the high-ROI majority; LoRA judge second because the residual requires moving a decision boundary in weight space. The explanation lives in a new paragraph in `method/method.md`.

**Gap 4 — Day 4: Verbosity Bias vs. Rubric Learning (Evaluation and Statistics)**

I trained a Qwen2.5-1.5B-Instruct judge on 69 preference pairs and validated it with Delta B = +0.3204 (wins=37/40 on held-out outputs). I cited that result as evidence the judge learned the Tenacious rubric. But my chosen outputs were DeepSeek V3.2 rewrites of failed Week 10 emails, and my rejected outputs were the original agent outputs that failed the rubric. DeepSeek V3.2 rewrites are structurally more elaborate than the originals.

The gap: I could not describe what the scores in `ablation_results.json` would look like differently if the judge had learned verbosity bias rather than rubric compliance. A positive Delta B is consistent with three distinct situations — rubric learning, length learning, or both entangled — and the aggregate lift cannot tell them apart.

Closure: Three tests on existing data resolve the ambiguity. Test 1 (Wilcoxon signed-rank on training pairs): if the chosen/rejected word-count ratio is above 1.3x and p < 0.05, the confound is present in the training data. Test 2 (Spearman ρ between word count and judge score on the 40 held-out outputs): if ρ > 0.3, length is a real driver. Test 3 (adversarial pairs): hand-written pairs where a short, rubric-compliant output is opposed to a long, rubric-violating output — the judge should prefer the short one. I added a "Length Bias Check" paragraph to `methodology_rationale.md` with the decision rule: run Test 2 if Test 1 triggers; retrain on length-matched pairs if ρ > 0.3.

---

### Four Gaps I Researched (Partner Questions I Answered)

**Gap 5 — Day 1: Instruction Drift in Long Prompts**

My partner's question asked how attention dynamics during inference cause instructions to lose salience in long-context generation. Researching this forced me to understand why my own prompts — stuffed with firmographic context — create instruction drift risk. Instructions placed early in a long prompt can receive lower relative attention weight at generation time as the model processes dense middle content. The practical signal: when behavioral constraints appear before 2,000+ tokens of content, test compliance on tasks where the instruction conflicts with the model's default response tendency.

**Gap 6 — Day 2: Pretraining Prior Dominance and the Forced Scratchpad Fix**

Amare Kassa's question revealed a gap that was also mine: his `generate_followup_email()` had a missing `objection` case causing every probe to fire the hardcoded fallback. He was measuring a deterministic fallback function, not his agent. Researching the fix forced me to understand why a plain-text policy label (`Tone mode: exploratory`) competes against pretraining priors and loses on generation, while a typed tool schema forces the model to satisfy a structured constraint before producing output. The forced scratchpad — requiring the model to produce a structured reasoning step that cannot be skipped — is the inference-time mechanism that converts a suggestion into a constraint.

**Gap 7 — Day 3: Statistical Power and Minimum Detectable Effects**

Mistire's question asked why p=0.585 on Delta A did not mean the adapter failed. Preparing the explainer forced me to work through the MDE calculation for his benchmark: at n=41 with 14.6% baseline, the benchmark can only detect effects above 18pp at 80% power. A genuine +10pp adapter improvement would be missed in over half of repeated experiments. Per-family deltas at n=7–8 (F2, F4) have MDE ≈ 42pp — one task changing is not a finding. This translates directly to my own evaluation design: any held-out set below 100 tasks is underpowered for detecting small improvements, and aggregate lift like Delta B needs a CI, not just a point estimate.

**Gap 8 — Day 4: Verbosity Bias Detection in LLM Judges**

The three-test framework Mistire gave me for my own question is itself a gap I closed while reading it. Knowing that a trained judge can learn verbosity bias rather than rubric compliance is obvious in retrospect. Knowing the specific operationalization — Wilcoxon on training pair lengths, Spearman ρ on held-out scores, adversarial pairs as the definitive check — is the judgment call that makes the concern actionable. Every FDE who trains a small judge on preference pairs and validates with aggregate score lift faces this ambiguity. The standard result cannot distinguish the two; the three tests can.

---

## Section 2: Most Surprising Learning

The most surprising learning across the week came from Day 3: that SCAP's ceiling and LoRA's advantage are not about quality — they are about where the correction lives.

Before this week, I thought of prompting and fine-tuning as two points on a capability dial: prompting is weaker, fine-tuning is stronger, choose based on how much improvement you need. That framing is wrong. The correct distinction is temporal and architectural. A prompt instruction is a conditional reminder at inference time — it competes with every other token in the context and with the pretrained weight function on every forward pass. A LoRA adapter changes the weight function itself: W_effective = W + BA is the computation for every input, regardless of what the prompt says.

This means there is a class of failures that prompting simply cannot fix — not because the instruction is too weak, but because the instruction must win a competition against the model's own pretrained tendencies on every single inference. When the failure is a borderline case where the model's internal representation does not cleanly separate compliant from non-compliant, no prompt revision will reliably change the output. Only changing the decision boundary in weight space will. SCAP was the right intervention first. LoRA was the right intervention second. The sequence was not escalation — it was the correct architectural diagnosis applied in the correct order.

---

## Section 3: Canonical Reading List

See `canonical_list.md` for full annotations. Core references across all four days:

1. **Hu et al. 2021** — LoRA: Low-Rank Adaptation of Large Language Models (ICLR 2022)
2. **Dror et al. 2018** — The Hitchhiker's Guide to Testing Statistical Significance in Natural Language Processing (ACL 2018)
3. **Cohen 1988** — Statistical Power Analysis for the Behavioral Sciences (Chapter 6)
4. **Dubois et al. 2024** — Length-Controlled AlpacaEval: A Simple Way to Debias Automatic Evaluators (arXiv)
5. **Zheng et al. 2023** — Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena (NeurIPS 2023)
6. **Li et al. 2025** — Preference leakage and rotation policy for LLM-as-a-judge training
7. **Wang et al. 2023** — Large Language Models Are Not Robust Multiple Choice Selectors (ICLR 2024)
