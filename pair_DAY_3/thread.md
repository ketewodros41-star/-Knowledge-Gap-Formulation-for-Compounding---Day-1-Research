# Tweet Thread — LoRA Rank and Model Confidence on Uncertain Inputs

---

**Tweet 1**
When you fine-tune a model with LoRA, you're not updating the original weights at all. You freeze them and train two small bypass matrices A and B. The rank r controls how many "directions" that bypass can push the model. Here's why that number matters more than most people realize 🧵

---

**Tweet 2**
LoRA's update is: W_effective = W + B×A

For a 2048-dim layer at r=16, you're training ~65k parameters instead of ~4M. That's 1.6% of the layer. The pretrained weights stay frozen — you're adding a small correction on top.

The rank r = how expressive that correction can be.

---

**Tweet 3**
Here's the part that bit me: I trained an ORPO critic at r=16 on only 51 preference pairs. Delta A lift was +6.5% — but the confidence interval was [0.0, 16.1%]. Lower bound: zero.

Why? High rank + small dataset = the critic memorized what "chosen" emails look like on the surface, not what the rubric actually requires.

```python
for rank in [4, 16, 64]:
    config = LoraConfig(r=rank, target_modules=["q_proj","v_proj"])
    lora_model = get_peft_model(model, config)
    lora_model.print_trainable_parameters()
# r=4 → ~0.4% trainable (pretrained caution preserved)
# r=16 → ~1.6% trainable (can memorize 51 examples)
# r=64 → ~6.1% trainable (definitely memorized)
```

---

**Tweet 4**
The key insight from Hu et al. (2021): rank should scale with dataset size and diversity. Small, homogeneous datasets → lower rank. Their experiments show r=4 matches r=64 on most tasks.

For uncertain inputs specifically — low rank lets the pretrained model's hedging behavior dominate. High rank on small data replaces that calibration with pattern-matching confidence.

---

**Tweet 5**
The reason low rank works at all: Aghajanyan et al. (2020) showed that fine-tuning updates naturally live in a low-dimensional subspace. The model already knows language. It just needs a small push for the new task.

Setting r too high doesn't give more room to improve — it gives more room to overfit.

---

**Tweet 6**
Takeaway for FDEs training small judges or domain adapters:

- 51 pairs, narrow task → r=4 or r=8
- High r on small data = confident judge that can't handle ambiguous inputs
- Run `model.print_trainable_parameters()` before committing to a rank

Full explainer: [https://open.substack.com/pub/kidusgashaw/p/lora-rank-and-the-confidence-problem?r=8bo4le&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true]

---
*Published: https://x.com/Kidus5T99409/status/2052453988382003529?s=20*
