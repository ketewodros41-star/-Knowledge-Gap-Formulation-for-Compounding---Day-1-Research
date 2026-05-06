# Sources — Day 2

## Canonical Papers

**1. Liu, N. F., et al. (2023). "Lost in the Middle: How Language Models Use Long Contexts."**
arXiv:2307.03172
https://arxiv.org/abs/2307.03172

Role in explainer: Load-bearing for the attention-position section. Establishes empirically that LLMs attend differently to different positions in their context window — information in the middle of long prompts is systematically less likely to influence output than information at the beginning or end. Directly explains why `Tone mode: exploratory` buried mid-prompt carries less weight than the same constraint placed immediately before the generation instruction.

---

**2. Yao, S., et al. (2022). "ReAct: Synergizing Reasoning and Acting in Language Models."**
arXiv:2210.03629
https://arxiv.org/abs/2210.03629

Role in explainer: Load-bearing for the scratchpad mechanism and the forced constraint check pattern. Original paper establishing that interleaving `Thought:` and `Action:` tokens makes retrieved context and tool outputs load-bearing in the generation path. The `Constraint check:` prefix pattern in the explainer is a direct application of the ReAct insight: reasoning tokens generated before the response body encode constraints into the probability distribution of everything that follows.

---

## Tool / Pattern Used

**OpenRouter API with Qwen3-235B-A22B (dev tier)**

Used to run the before/after prompt patterns — advisory text vs forced scratchpad — and compare actual model outputs. The two conditions were run against identical inputs with only the prompt structure changed. Results confirmed the mechanism: the advisory-text condition produced a generic outreach opening consistent with the pretraining prior; the forced scratchpad condition produced a constraint-aware response that referenced the policy field before drafting.
