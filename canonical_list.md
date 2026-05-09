# Canonical Reading List — Week 12

**TRP1 Challenge | Week 12 | Kidus Gashaw**

Annotated papers, tools, and patterns across all four days. Each entry notes the specific question it answered and how it applies to the Tenacious Conversion Engine.

---

## Papers

**Hu, E., Shen, Y., Wallis, P., Allen-Zhu, Z., Li, Y., Wang, S., ... & Chen, W. (2022). LoRA: Low-Rank Adaptation of Large Language Models. ICLR 2022.**

The foundational paper for parameter-efficient fine-tuning. Introduces the low-rank decomposition W_effective = W + BA where B and A are trainable low-rank matrices. Relevant to Days 3 and 4: explains why LoRA makes a persistent change to the weight function rather than adding runtime context, which is the mechanistic basis for why LoRA can fix failures that SCAP cannot. Also explains why rank (r) controls the expressivity of the adapter — too high a rank relative to training pairs produces an overconfident judge (Day 4 concern: r=16 on 69 pairs).

URL: https://arxiv.org/abs/2106.09685

---

**Dror, R., Baumer, G., Shlomov, S., and Reichart, R. (2018). The Hitchhiker's Guide to Testing Statistical Significance in Natural Language Processing. ACL 2018.**

The practical reference for statistical testing in NLP and ML evaluation. Section 4 covers McNemar's test for paired binary comparisons — the correct test for comparing pass rates when the same tasks are evaluated under two conditions (baseline vs. adapted model). Section 5 covers bootstrap and permutation tests for aggregate metrics. Used in Day 3 to write Mistire's explainer: at n=41 with 14.6% baseline, McNemar's test is the right procedure, and the MDE is approximately 18pp.

URL: https://aclanthology.org/P18-1128

---

**Cohen, J. (1988). Statistical Power Analysis for the Behavioral Sciences. 2nd edition. Lawrence Erlbaum Associates.**

The canonical reference for effect size and power calculations. Chapter 6 covers Cohen's h — the effect size measure for proportions — and the arcsine transformation that makes proportion comparisons scale-independent. The n≈122 calculation in Day 3 (tasks needed to detect a 10pp improvement at 80% power) derives directly from the formulas in this chapter. Used to explain why per-family deltas at n=7–8 are exploratory, not findings: MDE at n=8 is approximately 42pp.

---

**Dubois, Y., Galambosi, B., Liang, P., and Hashimoto, T. (2024). Length-Controlled AlpacaEval: A Simple Way to Debias Automatic Evaluators. arXiv:2404.04475.**

Demonstrates that LLM judges trained or prompted without length controls systematically prefer longer outputs independent of quality. Introduces length-controlled win rate as a debiased evaluation metric. Directly relevant to Day 4: the core concern behind Kidus's verbosity bias question is the same phenomenon documented here — a judge that rewards elaborateness rather than rubric compliance will inflate scores on verbose outputs while missing genuine failures.

URL: https://arxiv.org/abs/2404.04475

---

**Zheng, L., Chiang, W., Sheng, Y., Zhuang, S., Wu, Z., Zhuang, Y., ... & Stoica, I. (2023). Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena. NeurIPS 2023.**

The paper that systematizes LLM-as-a-judge evaluation and names the three canonical biases: position bias (preferring the first or second response), verbosity bias (preferring longer responses), and self-preference bias (preferring outputs from the same model family). Used in Day 4 as the source taxonomy for the biases Kidus needed to separate from rubric learning. Section 3 describes verbosity bias directly and notes that human evaluators are also susceptible.

URL: https://arxiv.org/abs/2306.05685

---

**Li, M., et al. (2025). Preference leakage in LLM-as-a-judge training and the rotation policy fix.**

Documents preference leakage — the contamination of judge training when the model that generates training outputs is the same model used to judge them. The rotation policy (DeepSeek generates, Qwen judges) in Tenacious-Bench v0.1 directly addresses this. Used in Day 4 as background for why Kidus's chosen outputs come from DeepSeek V3.2 rather than Qwen — this was a deliberate design decision to avoid leakage, and it introduced the length confound that created the verbosity bias question.

---

**Wang, P., Li, S., Chen, L., Zhu, W., Lin, B., Cao, Y., ... & Wang, H. (2023). Large Language Models Are Not Robust Multiple Choice Selectors. ICLR 2024.**

Documents position bias in LLM evaluation — models consistently favor responses in certain positions in pairwise comparisons regardless of quality. Used in Day 4 as context for the broader LLM-as-a-judge bias landscape. The verbosity bias and position bias are both documented here as systematic failure modes that affect small judges more than large ones.

URL: https://arxiv.org/abs/2309.03882

---

## Tools and Patterns

**Python `statsmodels.stats.power` + `scipy.stats.fisher_exact`**

Standard scientific Python libraries used in Day 3.
- `NormalIndPower().solve_power()`: computes MDE and required sample size for two-proportion comparisons using Cohen's h
- `proportion_effectsize()`: computes Cohen's h from two raw proportions
- `fisher_exact()`: exact p-values for small-n 2×2 tables; used for F2 (n=8, p≈0.50) and F4 (n=7, p≈0.53) per-family analysis in Mistire's model card

All code runs without additional dependencies.

---

**Langfuse tracing — TTFT isolation pattern (Day 1)**

Pattern for isolating prefill vs. decode latency in LLM API calls:
1. Record timestamp at API call start
2. Record timestamp when first token of streaming response arrives → `ttft_ms`
3. Record timestamps at generation end → compute `tokens_per_second` over decode phase
4. Log both fields as separate Langfuse observation metadata

This pattern is applicable to any agent that constructs long prompts and monitors latency.

---

**Scaffolding pre-call pattern vs. model-invoked tool pattern (Day 2)**

Two distinct patterns for enforcing constraints in agentic pipelines:
- **Scaffolding pre-call**: external code runs before model invocation; result injected as context; model cannot bypass; deterministic
- **Model-invoked tool**: model decides whether to call; probabilistic; subject to instruction drift and prompt phrasing

Decision rule: use scaffolding pre-calls for hard constraints (must run every time, cannot be bypassed); use model-invoked tools for soft constraints (model judgment about when to use an external resource is acceptable).

---

**Verbosity bias detection — three-test protocol (Day 4)**

For any judge trained on preference pairs where chosen outputs may be systematically more elaborate than rejected outputs:

- **Test 1 — Wilcoxon signed-rank on training pair lengths**: compare word counts of chosen vs. rejected outputs across all training pairs. If ratio > 1.3x AND p < 0.05, the confound is present.
- **Test 2 — Spearman ρ on held-out scored outputs**: compute correlation between output word count and judge score across the held-out evaluation set. If ρ > 0.3, length is a real driver.
- **Test 3 — Adversarial pairs**: hand-write 4 pairs where a short, rubric-compliant output is opposed to a long, rubric-violating output. If the judge does not consistently prefer the short one, verbosity bias is confirmed.

Run Test 1 first. If it triggers, run Test 2. If ρ > 0.3, run Test 3 to confirm, then retrain on length-matched pairs.
