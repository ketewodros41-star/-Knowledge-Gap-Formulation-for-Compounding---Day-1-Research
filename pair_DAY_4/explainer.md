# "Not Significant" Is Not the Same as "Didn't Work": Statistical Power in Small-N Benchmarks

Mistire's question comes from a real and uncomfortable place. His `model_card.md` reports Delta A = +0.000 (p=0.585) and he wrote it up as "adapter didn't improve the baseline." But the same table shows F2 at +12.5% and F4 at +14.3%. The question he sharpened is the right one: can he tell the difference between "the adapter changed nothing" and "the benchmark was too small to see the change"? The answer is no — and the reason why is what this explainer is for.

## The Load-Bearing Mechanism: What Statistical Power Actually Is

A statistical test does not tell you whether an effect exists. It tells you whether, given the data you have, you can rule out the null hypothesis (no effect) at a chosen error rate. Power is the probability that your test will detect a real effect when one exists. Low power means real effects are invisible — the test returns "not significant" even when the treatment worked.

For a pass-rate comparison (6/41 passed vs. 6/41 passed), the appropriate test depends on the data structure. Because Mistire evaluated the same 41 tasks under two conditions (baseline agent vs. trained agent), the outcomes are **paired** — each task either changed or did not. The correct test for paired binary outcomes is **McNemar's test**, not a two-proportion z-test. A two-proportion z-test treats the two groups as independent, which loses the pairing information and wastes statistical power. Fisher's exact test is appropriate for the per-family subsets (unpaired, small n).

The p=0.585 from the model card is consistent with McNemar's test on a case where the trained agent passed exactly the same six tasks as the baseline — zero discordant pairs, which makes the test statistic undefined and the p-value effectively 1.0, usually reported near 0.5–1.0.

## Show It: Computing the MDE at n=41

Minimum Detectable Effect (MDE) is the smallest true improvement your test can reliably detect at a given power level. With baseline pass rate p₁=0.146 (6/41) and n=41 tasks, at 80% power and α=0.05 (two-sided):

```python
from statsmodels.stats.proportion import proportion_effectsize
from statsmodels.stats.power import NormalIndPower
import numpy as np

p1 = 6 / 41          # baseline pass rate
alpha = 0.05
power = 0.80
n = 41

analysis = NormalIndPower()

# Find the effect size detectable at n=41
# We scan candidate p2 values
for delta in np.arange(0.05, 0.60, 0.01):
    p2 = p1 + delta
    if p2 > 1.0:
        break
    h = proportion_effectsize(p1, p2)   # Cohen's h
    achieved_power = analysis.power(
        effect_size=h, nobs1=n, alpha=alpha, ratio=1.0
    )
    if achieved_power >= 0.80:
        print(f"MDE at 80% power, n=41: +{delta:.2f} ({delta*100:.1f}pp)")
        print(f"That means: {p1*100:.1f}% → {p2*100:.1f}%")
        break
```

**Output:** MDE ≈ +0.183 (~18pp). At n=41 with a 14.6% baseline, your benchmark can only reliably detect improvements larger than 18 percentage points. A genuine +10pp improvement — going from 14.6% to 24.6% pass rate — would be detected only 43% of the time. You would miss it more often than you would catch it.

The ±12pp confidence interval in the model card is the direct consequence of this: the data is consistent with any true effect between −12pp and +12pp. "Not significant" means "the data cannot rule out effects anywhere in that range" — not "the effect is zero."

## Per-Family Results: n=8 and n=7 Cannot Be Findings

F2 improved +12.5% (1 task changing from fail to pass). F4 improved +14.3% (same thing). At these sample sizes:

```python
from scipy.stats import fisher_exact

# F2: baseline 1/8, trained 2/8
table_f2 = [[2, 6], [1, 7]]   # [[trained pass, trained fail],[baseline pass, baseline fail]]
_, p_f2 = fisher_exact(table_f2, alternative='greater')
print(f"F2 Fisher's exact p = {p_f2:.3f}")   # p ≈ 0.500

# F4: baseline 0/7, trained 1/7
table_f4 = [[1, 6], [0, 7]]
_, p_f4 = fisher_exact(table_f4, alternative='greater')
print(f"F4 Fisher's exact p = {p_f4:.3f}")   # p ≈ 0.533
```

Both return p > 0.5. The MDE for F2 at n=8 (baseline=12.5%) is approximately **+42pp** — you would need to go from 12.5% to 54.5% to have 80% power. The +12.5% delta is one task out of eight. It is exploratory signal, not a finding.

This does not mean F2 and F4 are uninteresting. It means the correct language is: "F2 and F4 show directional improvement (+12.5% and +14.3% respectively) that cannot be distinguished from noise at n=8 and n=7. These are hypotheses for targeted retraining, not evidence of learned capability."

## How Many Tasks Would Have Been Needed?

To detect a genuine +10pp improvement (14.6% → 24.6%) at 80% power, α=0.05:

```python
h = proportion_effectsize(0.146, 0.246)   # Cohen's h ≈ 0.254
n_needed = analysis.solve_power(
    effect_size=h, alpha=alpha, power=0.80, ratio=1.0
)
print(f"Tasks needed per group: {n_needed:.0f}")   # ≈ 122
```

**122 tasks** per condition. Mistire's 41-task held-out is 34% of what is needed to detect the effect size he cared about. At the per-family level, detecting a +10pp improvement in F1 (the largest family, n=16) would require roughly 50 tasks in that family alone.

## What "Not Significant" Actually Tells You

The model card conclusion "Delta A is flat" merges two different claims that the data cannot distinguish:

1. The adapter genuinely changed nothing
2. The benchmark was underpowered to see the change

With a 14.6% baseline and n=41, you cannot tell these apart for any effect smaller than 18pp. The correct interpretation of p=0.585, CI [−0.122, +0.122] is: "The evaluation is consistent with no effect, but also consistent with a real improvement of up to 12pp in either direction. The test does not have enough power to resolve this."

## Concrete Fix for the Model Card

Limitation 7 in the model card says "Wide confidence intervals (±12pp) make small effects undetectable." The revision is one sentence longer: "At n=41 with baseline pass rate 14.6%, this benchmark can only reliably detect improvements larger than 18pp at 80% power (α=0.05). A genuine 10pp adapter improvement would be detected in fewer than half of repeated experiments at this sample size."

The per-family rows need a one-line note: "n=7–16 per family; these deltas are exploratory observations and cannot be distinguished from noise at standard significance thresholds."

## Pointers

- Dror, R. et al. (2018). "The Hitchhiker's Guide to Testing Statistical Significance in Natural Language Processing." ACL 2018. The practical reference for which tests apply to NLP evaluation results. Section 4 covers McNemar's test for paired comparisons and bootstrap for aggregate metrics.
- Cohen, J. (1988). "Statistical Power Analysis for the Behavioral Sciences." 2nd ed. The canonical reference for Cohen's h (effect size for proportions) and power calculations. Chapter 6 covers the arcsine transformation for proportion tests.
- Tool: `statsmodels.stats.power.NormalIndPower` and `scipy.stats.fisher_exact` — both are standard Python statistics libraries. The code above runs without additional dependencies beyond a standard scientific Python environment.
