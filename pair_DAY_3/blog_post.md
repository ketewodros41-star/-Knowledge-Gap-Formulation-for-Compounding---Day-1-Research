# LoRA Rank and the Confidence Problem: What Nobody Tells You When You Copy the Default

*By Kidus Gashaw*

---

When I trained an ORPO critic for my Week 11 project, I set `r=16` in `training/train_lora_judge.py` and moved on. It was the default. I never questioned it. My critic passed the ablation. Delta A was positive. I shipped it.

Then I looked closely at `ablations/ablation_results.json`. Delta A lift: **+6.5%**. Confidence interval: **[0.0, 16.1%]**. The lower bound is zero. The result could be real or it could be noise, and I could not tell which — because I had not thought about what `r=16` on 51 preference pairs actually produces.

That CI sent me back to a question I had avoided: what does LoRA rank actually control, and why would getting it wrong make a critic overconfident on exactly the inputs — weak signals, ambiguous hiring data — where it should be most cautious?

---

## What LoRA Is Actually Doing

A pretrained language model like Qwen2.5-0.5B contains large weight matrices in its attention layers — `q_proj` and `v_proj`. Full fine-tuning would update every value in every one of those matrices. For a production-scale model, that is hundreds of millions of parameters, expensive compute, and a high risk of destroying the pretrained behavior you relied on.

LoRA takes a different approach. It freezes the original weight matrix W entirely and inserts a bypass made of two small matrices, A and B:

```
W_effective = W + B × A
```

Only A and B are trained. W never changes. For a layer with dimension 2048 and rank r=16, the bypass has shape `2048×16` and `16×2048` — about 65,000 parameters instead of 4 million. You are updating roughly 1.6% of that layer.

The rank r is the size of the bottleneck between A and B. It controls how many independent directions the adapter can push the model's behavior. Rank 1 means one direction. Rank 16 means sixteen. Rank 64 means sixty-four.

```python
from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-1.5B")

for rank in [4, 16, 64]:
    config = LoraConfig(r=rank, lora_alpha=rank, target_modules=["q_proj", "v_proj"])
    lora_model = get_peft_model(model, config)
    lora_model.print_trainable_parameters()

# r=4  → ~0.4% trainable parameters
# r=16 → ~1.6% trainable parameters
# r=64 → ~6.1% trainable parameters
```

Higher rank is not always better. It depends entirely on your dataset.

---

## Why Low Rank Works at All

Before getting to the confidence problem, it helps to understand why fine-tuning with a tiny fraction of the parameters works at all.

Aghajanyan et al. (2020) studied what actually happens to weights during fine-tuning and found something surprising: the updates that matter live in a surprisingly small subspace of the full weight space. A model with hundreds of millions of parameters effectively navigates a much lower-dimensional space when adapting to a new task — often as few as a few hundred meaningful dimensions. Everything else is redundant.

The pretrained model already knows how language works, how to assess quality, how to express uncertainty. Fine-tuning only needs to redirect a small part of that knowledge toward the new task. LoRA approximates this redirection with a low-rank matrix. The rank is your estimate of how large that small subspace needs to be.

---

## The Confidence Problem: What High Rank Does on Small Data

Here is the part that matters for anyone training a critic on a small preference dataset.

A **low-rank adapter** (r=4 or r=8) has few directions to move the weights. It can learn the strongest signal in the training data but cannot memorize fine-grained patterns. This means on uncertain inputs — outreach emails where the hiring signal is thin, AI maturity is ambiguous, or the prospect sits at the edge of ICP — the adapter's contribution is small and the pretrained model's behavior fills the gap. Pretrained models hedge. They were trained on human text that expresses uncertainty with phrases like "may suggest," "appears to indicate," "limited evidence." Low rank preserves this calibration.

A **high-rank adapter** (r=16 or r=32) on a small dataset is a different story. It has more directions available but too few training examples to use them meaningfully. It learns what "chosen" examples look like on the surface — the vocabulary, the structure, the phrasing patterns of a passing email — rather than what makes an output genuinely satisfy the rubric. On uncertain inputs, it applies those surface patterns confidently, because nothing in 51 training examples showed it that a weak signal brief warrants a cautious score.

This is what produces a CI of `[0.0, 16.1%]`. The critic passes the entire held-out slice — path_b_pass of 1.0 — but the lower bound of the confidence interval is zero. The critic may have learned to recognize the shape of passing emails rather than the reasoning behind the Tenacious Style Guide. Traces `trace_id_8f3a2` and `trace_id_9b1c4` show the exact failure mode the critic was trained to catch: asserting "scaling aggressively" from a single junior role, pitching AI transformation to an AI-maturity-1 company. A critic that overfits at r=16 will mark emails with that same confident phrasing as "pass" — because confident phrasing was the surface feature of chosen examples.

---

## The Rule of Thumb

Hu et al.'s rank ablation experiments (LoRA paper, Section 4) found that r=4 matches r=64 on most downstream tasks. The cases where higher rank helped were tasks with highly diverse training data — many different domains, styles, or task types within the fine-tuning set. For narrow, homogeneous datasets, high rank adds memorization capacity without adding generalization.

The rough guide for practice:

| Dataset size and diversity | Recommended rank |
|---|---|
| Small and narrow (< 200 examples, single task) | r=4 to r=8 |
| Moderate and focused (200–2,000 examples) | r=8 to r=16 |
| Large and diverse (2,000+ examples, multiple domains) | r=16 to r=64 |

51 Tenacious preference pairs on a single narrow task is small and homogeneous. r=4 or r=8 would have been the safer choice. The critic would have been less expressive — but more honest on uncertain inputs.

---

## What to Do With This

If you trained a LoRA adapter at the notebook default and your Delta A confidence interval touches zero, rank is one of the first things to check. A re-run at r=4 or r=8 takes the same wall time on Colab T4 (60 steps on 51 pairs is fast) and will show you whether the CI tightens — whether the critic is now making decisions based on the rubric rather than the surface shape of training examples.

Before you trust a trained judge's scores as your headline result, run `print_trainable_parameters()`, look at your dataset size, and ask whether the rank you used gave the adapter room to generalize or just room to memorize.

The default is not a choice. It is a placeholder. Make the choice yourself.

---

## Sources

- Hu et al. (2021) — *LoRA: Low-Rank Adaptation of Large Language Models*. https://arxiv.org/abs/2106.09685
- Aghajanyan et al. (2020) — *Intrinsic Dimensionality Explains the Effectiveness of Language Model Fine-Tuning*. https://arxiv.org/abs/2012.13255
- PEFT library: https://github.com/huggingface/peft
