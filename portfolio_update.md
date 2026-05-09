# Portfolio Update — Week 12 Grounding Commits

**Kidus Gashaw | TRP1 Challenge | Week 12**
**For:** FDE hiring review

---

## What This Document Is

Four code and documentation edits made to the Tenacious Conversion Engine during Week 12. Each edit was made in response to a knowledge gap I could not defend — identified through the paired research process — and changes a specific artifact in the project from an incorrect or incomplete claim to one I can explain mechanistically.

---

## Commit 1 — TTFT Telemetry in `outreach_generator.py`

**File:** `agent/outreach_generator.py`
**Change:** Added `ttft_ms` and `tokens_per_second` telemetry fields to Langfuse traces

**What it fixes:** The agent constructs dense prompts with Crunchbase firmographics, job-post velocity, layoffs.fyi data, and competitor gap briefs before generating outreach emails. Langfuse traces recorded only total API wait time. I could not tell whether latency spikes came from the model reading the long context (prefill) or writing the email (decode) — and those two phases require different optimizations.

**What I understand now:** Prefill is a single parallelized forward pass over all input tokens; it scales with context length. Decode is autoregressive — one token per forward pass — and scales with output length. Without `ttft_ms` logged at the moment the first decode token arrives, every latency optimization was a guess. The edit adds `ttft_ms` (time from API call to first token) and `tokens_per_second` (throughput across decode). A trace now tells me which phase to optimize before I touch the code.

**Why this matters for production:** Any agent that constructs long prompts in a latency-sensitive pipeline has this problem. Coarse end-to-end latency metrics hide the diagnosis. This change makes the telemetry actionable.

---

## Commit 2 — Scaffolding vs. Tool Distinction in `method.md`

**File:** `method/method.md`
**Change:** Rewrote the bench-constraint paragraph from "model choosing to call a tool" to "scaffolding pre-call injecting result as context"

**What it fixes:** I described the bench availability check as a "bench-gated constraint" in `method.md` using language that implied the model decides whether to call a staffing tool. The actual code in `agent/main.py` runs `check_bench_availability()` unconditionally before the model receives any input — the model cannot bypass this constraint because it never owned the decision.

**What I understand now:** Model-invoked tools are probabilistic — the model decides whether to call them, and that decision is subject to instruction drift. Scaffolding pre-calls are deterministic — the runtime owns execution. My bench-gated constraint was always deterministic. The `method.md` description was wrong. The corrected paragraph names what is actually enforcing the constraint and why it cannot be bypassed.

**Why this matters for production:** Any pipeline that advertises "hard constraints" implemented as model-invoked tools is making a reliability claim it cannot back up. Understanding this distinction changes both the design and the documentation of any constrained agent system.

---

## Commit 3 — SCAP Ceiling Explanation in `method/method.md`

**File:** `method/method.md`
**Change:** Added "Why SCAP Has a Ceiling" paragraph after the SCAP mechanism description

**What it fixes:** `method.md` described SCAP as "a prompt-time mechanism that directly attacks the highest-ROI failure mode that can be reduced by a prompt-time mechanism" — correct but circular. It left the ceiling unexplained, which meant the rationale for training a LoRA judge in Week 11 had no mechanistic foundation in the Week 10 artifact.

**What I understand now:** Prompting is runtime steering — it adds context tokens that shift next-token probabilities for one inference run, but the model's weights are unchanged. When SCAP's instruction competes with the model's fluency and plausibility objectives on a low-signal input, those objectives can still win because the weight function has not changed. LoRA changes the computation itself: W_effective = W + BA means every forward pass through adapted layers produces different hidden states regardless of what the prompt says. The residual P-005 violations after SCAP are the signature of a prompt ceiling, not a poorly written prompt.

**Why this matters for production:** Every constrained agent will hit the same decision: add another prompt instruction, or fine-tune. Without understanding what these two approaches change — one at inference time, one in the weights — you cannot know when prompt engineering is sufficient and when training is the only fix. The paragraph makes this design rationale explicit and permanent in the artifact.

---

## Commit 4 — Length Bias Check in `methodology_rationale.md`

**File:** `methodology_rationale.md` (Week 11 Tenacious-Bench)
**Change:** Added "Length Bias Check" paragraph after the Preference Pair Construction section

**What it fixes:** `methodology_rationale.md` described chosen outputs as DeepSeek V3.2 rewrites of failed emails and rejected outputs as original agent failures, with no analysis of whether the two groups differ in length or structural elaborateness. Delta B = +0.3204 was cited as evidence the judge learned the Tenacious rubric. I could not describe what the scores would look like differently if the judge had learned to reward output length instead.

**What I understand now:** A positive Delta B is consistent with rubric learning, length learning, or both entangled. The three-test protocol now documented in `methodology_rationale.md` — Wilcoxon signed-rank on training pair lengths, Spearman ρ on held-out scores, adversarial pairs as the definitive check — is the diagnostic path that resolves the ambiguity without rerunning training. The decision rule is explicit: ratio > 1.3x AND p < 0.05 on Test 1 triggers Test 2; ρ > 0.3 triggers retraining on length-matched pairs.

**Why this matters for production:** A judge that rewards elaborateness will quietly inflate scores on verbose production outputs while missing genuine rubric failures. The standard ablation result looks identical under both hypotheses. Without this check, a Delta B result cannot be trusted as evidence of capability.

---

## Summary

| Day | File | Change | Mechanism Fixed |
|-----|------|--------|-----------------|
| 1 | `outreach_generator.py` | Added TTFT + TPS telemetry | Prefill/decode phase isolation |
| 2 | `method/method.md` | Rewrote bench-constraint paragraph | Scaffolding vs. model-invoked tool |
| 3 | `method/method.md` | Added SCAP ceiling paragraph | Prompt vs. LoRA weight update |
| 4 | `methodology_rationale.md` | Added length bias check | Verbosity bias vs. rubric learning |

Each commit changes a claim I could not defend into one with a mechanistic explanation grounded in the artifact.
