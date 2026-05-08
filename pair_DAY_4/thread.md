# Tweet Thread — Day 4

**Tweet 1**
"Not significant" ≠ "didn't work."

If your agent benchmark has 41 held-out tasks and a 15% baseline pass rate, you can only reliably detect improvements larger than 18pp. A real +10% adapter improvement would be invisible more than half the time. Here's how to know what your benchmark can and cannot see. 🧵

---

**Tweet 2**
First: use the right test.

If you evaluated the same tasks under two conditions (baseline vs. trained), outcomes are paired. Use McNemar's test, not a two-proportion z-test. The paired structure gives you more power — wasting it with an unpaired test makes a small benchmark even smaller.

---

**Tweet 3**
Computing your minimum detectable effect (MDE):

```python
from statsmodels.stats.proportion import proportion_effectsize
from statsmodels.stats.power import NormalIndPower

p1 = 0.146  # your baseline pass rate
h = proportion_effectsize(p1, p1 + 0.10)  # 10pp improvement
n_needed = NormalIndPower().solve_power(
    effect_size=h, alpha=0.05, power=0.80
)
# n_needed ≈ 122 — you need 3x your current held-out
```

At n=41 with 14.6% baseline, MDE ≈ 18pp. You cannot see a +10% lift with this benchmark.

---

**Tweet 4**
Per-family deltas at n=7–8 are not findings.

+12.5% on F2 (n=8) = 1 task changing. Fisher's exact p ≈ 0.50.
+14.3% on F4 (n=7) = 1 task changing. Fisher's exact p ≈ 0.53.

MDE at n=8 is ~42pp. These are hypotheses for your next training run, not evidence of learned capability. Label them exploratory.

---

**Tweet 5**
What a CI of [−12pp, +12pp] actually says:

Your data is consistent with any true effect in that range — including +12pp improvement and −12pp regression. "p=0.585" means the test cannot distinguish signal from noise here, not that the effect is zero. The conclusion to write is "underpowered to detect effects below 18pp," not "adapter didn't work."

---

**Tweet 6**
The fix: one sentence added to Limitation #7.

"At n=41 with 14.6% baseline, this benchmark detects only improvements above 18pp at 80% power. A genuine 10pp adapter improvement would be missed in over half of repeated experiments at this sample size."

That is an honest null result. Full explainer: https://open.substack.com/pub/kidusgashaw/p/not-significant-is-not-the-same-as?r=8bo4le&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true

Sources: Dror et al. 2018 (ACL, statistical testing for NLP); Cohen 1988 (Statistical Power Analysis, Ch. 6)
