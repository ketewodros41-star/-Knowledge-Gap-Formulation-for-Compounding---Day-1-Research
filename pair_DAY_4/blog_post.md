# "Not Significant" Is Not the Same as "Didn't Work"

*How statistical power determines what your agent benchmark can actually see — and what it silently misses*

---

You trained a LoRA adapter. You evaluated it on your held-out benchmark. The result came back: Delta A = +0.000, p = 0.585. You wrote "not significant" in the model card and moved on.

But look closer. Your per-family breakdown shows F2 improved +12.5% and F4 improved +14.3%. That looks like signal. You almost reported it as evidence the adapter learned something.

Here is what most people skip: before you interpret any of those numbers, you need to ask a prior question — **what is the smallest improvement this benchmark could actually detect?**

If the answer is "18 percentage points," then a genuine +10% adapter improvement is invisible in your evaluation. "Not significant" and "adapter didn't work" are not the same conclusion. One is a statement about what the test can see. The other is a statement about the world.

---

## What Statistical Power Means in Practice

Statistical power is the probability that your test detects a real effect when one actually exists. Low power means real improvements go undetected — the test returns "not significant" not because nothing changed, but because your sample was too small to tell.

For a pass-rate benchmark — where each task either passes or fails — the right framing is: given my baseline pass rate and my held-out size, what is the minimum improvement I can reliably detect?

This quantity is called the **minimum detectable effect (MDE)**. It is set by three things: your baseline pass rate, your sample size, and your chosen power and significance thresholds (conventionally 80% power, α=0.05).

---

## Computing the MDE for a Real Benchmark

Say your baseline pass rate is 14.6% (6 out of 41 tasks passing) and your held-out has 41 tasks. Here is the calculation in Python:

```python
from statsmodels.stats.proportion import proportion_effectsize
from statsmodels.stats.power import NormalIndPower
import numpy as np

p1 = 0.146       # baseline pass rate
alpha = 0.05
power = 0.80
n = 41

analysis = NormalIndPower()

for delta in np.arange(0.05, 0.60, 0.01):
    p2 = p1 + delta
    h = proportion_effectsize(p1, p2)   # Cohen's h
    achieved = analysis.power(effect_size=h, nobs1=n, alpha=alpha, ratio=1.0)
    if achieved >= 0.80:
        print(f"MDE at n=41, 80% power: +{delta*100:.1f}pp")
        break
# Output: MDE at n=41, 80% power: +18.3pp
```

**The MDE is approximately 18 percentage points.** That means your benchmark can only reliably catch improvements larger than 18pp. A genuine +10pp improvement — going from 14.6% to 24.6% — would go undetected in more than half of repeated experiments at this sample size.

The ±12pp confidence interval is the direct consequence of this. It is telling you the same thing in a different language: the data is consistent with any true effect between −12pp and +12pp. You cannot rule out a real improvement of 12pp, and you cannot rule out a regression of 12pp either.

---

## The Right Statistical Test for Paired Evaluations

One more thing most people get wrong: the test itself.

If you evaluated the same tasks under two conditions — the same 41 tasks scored by the baseline agent, then scored by the trained agent — your outcomes are **paired**. Each task either changed or it did not. The correct test for paired binary outcomes is **McNemar's test**, not a two-proportion z-test or chi-squared.

A two-proportion z-test treats the two groups as independent, which throws away the pairing information and makes your already-small benchmark even smaller in effective power. Using the right test is the first step before interpreting any p-value.

---

## What Happens at the Per-Family Level

This is where the problem becomes concrete.

F2 improved +12.5% on n=8 tasks. That is exactly one task changing from fail to pass. Fisher's exact test on a 2×8 vs 1×8 comparison returns p ≈ 0.50. The minimum detectable effect at n=8 with a 12.5% baseline is approximately **42 percentage points** — you would need the trained agent to go from 12.5% to 54.5% on that family before you could call it a finding.

The same applies to F4: +14.3% on n=7 tasks is one task changing from 0/7 to 1/7. Fisher's exact p ≈ 0.53.

These are not findings. They are hypotheses. The correct way to report them: "F2 and F4 show directional improvement in the trained condition (+12.5% and +14.3% respectively). At n=8 and n=7, these deltas cannot be distinguished from noise and should be treated as exploratory observations — targets for focused data collection in v0.2, not evidence of learned capability."

---

## How Many Tasks Would You Actually Need?

To detect a genuine +10pp improvement (14.6% → 24.6%) at 80% power, α=0.05:

```python
h = proportion_effectsize(0.146, 0.246)   # Cohen's h ≈ 0.254
n_needed = analysis.solve_power(effect_size=h, alpha=alpha, power=0.80, ratio=1.0)
print(f"Tasks needed: {int(n_needed) + 1}")
# Output: Tasks needed: 123
```

**123 tasks** per condition — three times the 41-task held-out. For per-family coverage at +10pp sensitivity, you need roughly 50 tasks per failure family.

This is expensive to label. That is the honest trade-off behind every small-N benchmark: labeling cost against evaluation sensitivity. The answer is not to pretend the sensitivity is higher than it is — it is to state the MDE explicitly in the model card and let readers judge what the null result actually proves.

---

## The One-Sentence Fix

Somewhere in every model card there is a limitation that says "small held-out size limits statistical power." That sentence is almost always correct and almost always unquantified.

The revision is one sentence longer: *"At n=41 with a 14.6% baseline, this benchmark reliably detects only improvements larger than 18pp at 80% power. A genuine 10pp adapter improvement would be missed in over half of repeated experiments at this sample size."*

That is what an honest null result looks like. It does not say the adapter failed. It says the evaluation was not built to see what you were looking for — and now you know exactly how to fix that in the next version.

---

*Sources: Dror et al. (2018), "The Hitchhiker's Guide to Testing Statistical Significance in NLP," ACL 2018. Cohen, J. (1988), "Statistical Power Analysis for the Behavioral Sciences," Ch. 6. Code: `statsmodels.stats.power`, `scipy.stats.fisher_exact`.*
