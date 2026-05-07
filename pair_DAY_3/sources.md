# Sources — Day 3

## Canonical Papers

**1. LoRA: Low-Rank Adaptation of Large Language Models**
Hu et al., 2021. https://arxiv.org/abs/2106.09685

The foundational paper. Section 4 contains the rank ablation experiments showing r=4 matches r=64 on most downstream tasks. Section 2 derives the BA decomposition and explains why the update is scaled by alpha/r. Read this before setting any LoRA hyperparameter.

**2. Intrinsic Dimensionality Explains the Effectiveness of Language Model Fine-Tuning**
Aghajanyan et al., 2020. https://arxiv.org/abs/2012.13255

Explains *why* low-rank updates work: fine-tuning navigates a surprisingly low-dimensional subspace of the full parameter space. This is the theoretical grounding for LoRA's design — without it, low-rank adaptation looks like a lucky hack rather than a principled approximation.

## Tool Used

**PEFT library — `model.print_trainable_parameters()`**
https://github.com/huggingface/peft

Running `get_peft_model(model, LoraConfig(r=N))` followed by `print_trainable_parameters()` produces the exact trainable parameter count and percentage for any rank setting. Used to generate the r=4/16/64 comparison in the explainer and to verify that r=16 on Qwen 3.5 2B targets ~1.6% of each adapted layer's parameters.
