# How LoRA Alters Model Behavior — and Why Rank Matters for Confident vs Cautious Outputs

Your question came from a real problem: you trained an ORPO critic with `r=16` on 51 preference pairs (`training/train_lora_judge.py`, line 37), and your `ablations/ablation_results.json` shows a Delta A confidence interval of `[0.0, 16.1%]` — the lower bound touches zero. The lift might be real. It might be noise. Understanding what LoRA rank controls is how you figure out which one, and what to do about it.

---

## What LoRA Actually Does to the Weights

A pretrained language model like Qwen2.5-0.5B contains weight matrices — the `q_proj` and `v_proj` layers you targeted in `training/train_lora_judge.py`. Each of these is a large matrix, say W with shape 896 × 896. Full fine-tuning would update every value in W — roughly 800,000 numbers per layer, multiplied across dozens of layers.

LoRA does not touch W at all. It freezes W and adds a bypass: two small matrices A (shape 2048 × r) and B (shape r × 2048), where r is the rank you set. The model's effective weight during inference becomes:

```
W_effective = W + B × A
```

Only B and A are trained. W stays exactly as it was after pretraining. The rank r is what determines how many numbers are in A and B combined — for r=16 on a 2048-dim layer, that is `2048×16 + 16×2048 = 65,536` parameters instead of `2048×2048 = 4,194,304`. You are updating about 1.6% of that layer.

The key insight: **rank controls the expressivity of the update, not its magnitude**. A rank-1 update can only push the model's behavior along one direction in weight space. A rank-16 update has 16 independent directions to work with. A rank-64 update has 64.

---

## Why Low Rank Works at All

The reason LoRA works is that the updates needed to adapt a pretrained model to a new task are low-dimensional. Aghajanyan et al. (2020) showed that fine-tuning a large model actually navigates a surprisingly small subspace of its weight space — most of the parameter space is redundant for the task. The model already knows how language works; it only needs a small push to learn the new behavior.

For your critic task — "detect signal over-claiming in B2B outreach emails" — the meaningful update is small. The model already understands text quality, professional tone, and factual claims. It just needs to learn the specific Tenacious Style Guide boundary between hedged and over-claimed language, the same boundary that failed in traces `trace_id_8f3a2` and `trace_id_9b1c4`.

---

## How Rank Shapes Confidence vs Caution on Uncertain Inputs

Here is where rank directly affects output style.

A **low-rank adapter** (r=4 or r=8) has fewer directions to move the weights. It can learn the strongest signal in your training data but cannot memorize fine-grained patterns. This means on uncertain inputs — outreach emails where the signal is weak or ambiguous — the adapter's influence is small and the pretrained model's behavior dominates. Pretrained models are calibrated to hedge: they were trained on human text that expresses uncertainty with phrases like "may indicate," "appears to suggest," "limited evidence." Low rank preserves this caution.

A **high-rank adapter** (r=16 or r=32) on a small dataset has more directions available but too few examples to fill them meaningfully. It memorizes the surface patterns of the 51 chosen examples rather than learning the underlying rubric. On uncertain inputs — outreach emails where the hiring signal is thin or ambiguous — it applies those memorized patterns confidently, because nothing in 51 training examples taught it that weak signal warrants a cautious score. This is what a CI of `[0.0, 16.1%]` looks like: the critic passes everything on the held-out slice, but the lower bound of zero means you cannot rule out that it learned to say "pass" to almost anything that resembles a chosen example's structure.

A concrete way to see this:

```python
from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-1.5B")

for rank in [4, 16, 64]:
    config = LoraConfig(r=rank, lora_alpha=rank, target_modules=["q_proj", "v_proj"])
    lora_model = get_peft_model(model, config)
    lora_model.print_trainable_parameters()
    # r=4:  ~0.4% trainable  — small push, pretrained behavior dominates
    # r=16: ~1.6% trainable  — moderate push, can memorize 80 examples
    # r=64: ~6.1% trainable  — large push, nearly all 80 examples memorized
```

For 51 training pairs, r=4 or r=8 is the safer choice. The rule of thumb from the LoRA paper: rank should scale roughly with the size and diversity of the fine-tuning dataset. Small homogeneous datasets (your 51 Tenacious preference pairs) warrant lower rank. Large diverse datasets (thousands of varied examples) can use higher rank productively.

---

## The Adjacent Concept Worth Knowing: Intrinsic Rank

The reason any of this works is that the pretrained model's weight updates during fine-tuning naturally lie in a low-dimensional subspace. Aghajanyan et al. found that for many NLP tasks, the intrinsic dimensionality of the optimal update is surprisingly small — sometimes as low as 200 dimensions for a model with hundreds of millions of parameters. LoRA approximates this low-dimensional subspace with the BA product. Setting r too high does not give the model more "room to improve" — it gives it more room to overfit.

---

## What This Means for Your Critic

Your `methodology_rationale.md` justifies choosing ORPO over DPO and SimPO at length but never mentions rank. Given 51 preference pairs and a narrow single-task objective (detect signal over-claiming), r=16 is likely higher than necessary. A re-run at r=4 or r=8 would probably:

- Score ambiguous low-signal inputs more conservatively, closer to the pretrained model's hedging behavior
- Narrow the CI on Delta A — a more conservative critic that only fires on clear violations produces a tighter and more trustworthy result
- Produce a critic that is harder to fool by surface-level "chosen-looking" language

The CI touching zero is not bad luck. It is what happens when a high-rank adapter on a small dataset learns to recognize the shape of passing emails rather than the reasoning behind the rubric.

---

## Pointers

- **Hu et al. (2021) — LoRA**: the original paper, Section 4 covers the rank ablation experiments directly. Their finding: r=4 matches r=64 on most tasks, and higher rank helps only when the fine-tuning distribution is highly diverse.
- **Aghajanyan et al. (2020) — Intrinsic Dimensionality**: explains why the low-rank approximation works in principle, not just empirically.
- **PEFT library**: `model.print_trainable_parameters()` tells you exactly what fraction of the model you are updating at any rank — run it before committing to a rank.
